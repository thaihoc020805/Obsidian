Expose an application running in your cluster behind a single outward-facing endpoint, even when the workload is split across multiple backends.

In Kubernetes, a Service is a method for exposing a network application that is running as one or more [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) in your cluster.

A key aim of Services in Kubernetes is that you don't need to modify your existing application to use an unfamiliar service discovery mechanism. You can run code in Pods, whether this is a code designed for a cloud-native world, or an older app you've containerized. You use a Service to make that set of Pods available on the network so that clients can interact with it.

If you use a [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) to run your app, that Deployment can create and destroy Pods dynamically. From one moment to the next, you don't know how many of those Pods are working and healthy; you might not even know what those healthy Pods are named. Kubernetes [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) are created and destroyed to match the desired state of your cluster. Pods are ephemeral resources (you should not expect that an individual Pod is reliable and durable).

Each Pod gets its own IP address (Kubernetes expects network plugins to ensure this). For a given Deployment in your cluster, the set of Pods running in one moment in time could be different from the set of Pods running that application a moment later.

This leads to a problem: if some set of Pods (call them "backends") provides functionality to other Pods (call them "frontends") inside your cluster, how do the frontends find out and keep track of which IP address to connect to, so that the frontend can use the backend part of the workload?

Enter _Services_.

## Services in Kubernetes[](https://kubernetes.io/docs/concepts/services-networking/service/#services-in-kubernetes)

The Service API, part of Kubernetes, is an abstraction to help you expose groups of Pods over a network. Each Service object defines a logical set of endpoints (usually these endpoints are Pods) along with a policy about how to make those pods accessible.

For example, consider a stateless image-processing backend which is running with 3 replicas. Those replicas are fungible—frontends do not care which backend they use. While the actual Pods that compose the backend set may change, the frontend clients should not need to be aware of that, nor should they need to keep track of the set of backends themselves.

The Service abstraction enables this decoupling.

The set of Pods targeted by a Service is usually determined by a [selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) that you define. To learn about other ways to define Service endpoints, see [Services _without_ selectors](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors).

If your workload speaks HTTP, you might choose to use an [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) to control how web traffic reaches that workload. Ingress is not a Service type, but it acts as the entry point for your cluster. An Ingress lets you consolidate your routing rules into a single resource, so that you can expose multiple components of your workload, running separately in your cluster, behind a single listener.

The [Gateway](https://gateway-api.sigs.k8s.io/#what-is-the-gateway-api) API for Kubernetes provides extra capabilities beyond Ingress and Service. You can add Gateway to your cluster - it is a family of extension APIs, implemented using [CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) - and then use these to configure access to network services that are running in your cluster.

### Cloud-native service discovery[](https://kubernetes.io/docs/concepts/services-networking/service/#cloud-native-service-discovery)

If you're able to use Kubernetes APIs for service discovery in your application, you can query the [API server](https://kubernetes.io/docs/concepts/architecture/#kube-apiserver) for matching EndpointSlices. Kubernetes updates the EndpointSlices for a Service whenever the set of Pods in a Service changes.

For non-native applications, Kubernetes offers ways to place a network port or load balancer in between your application and the backend Pods.

Either way, your workload can use these [service discovery](https://kubernetes.io/docs/concepts/services-networking/service/#discovering-services) mechanisms to find the target it wants to connect to.

## Defining a Service[](https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service)

A Service is an [object](https://kubernetes.io/docs/concepts/overview/working-with-objects/#kubernetes-objects) (the same way that a Pod or a ConfigMap is an object). You can create, view or modify Service definitions using the Kubernetes API. Usually you use a tool such as `kubectl` to make those API calls for you.

For example, suppose you have a set of Pods that each listen on TCP port 9376 and are labelled as `app.kubernetes.io/name=MyApp`. You can define a Service to publish that TCP listener:

[`service/simple-service.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/service/simple-service.yaml)![](https://kubernetes.io/images/copycode.svg "Copy service/simple-service.yaml to clipboard")

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

Applying this manifest creates a new Service named "my-service" with the default ClusterIP [service type](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types). The Service targets TCP port 9376 on any Pod with the `app.kubernetes.io/name: MyApp` label.

Kubernetes assigns this Service an IP address (the _cluster IP_), that is used by the virtual IP address mechanism. For more details on that mechanism, read [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/).

The controller for that Service continuously scans for Pods that match its selector, and then makes any necessary updates to the set of EndpointSlices for the Service.

The name of a Service object must be a valid [RFC 1035 label name](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#rfc-1035-label-names).

#### Note:

A Service can map _any_ incoming `port` to a `targetPort`. By default and for convenience, the `targetPort` is set to the same value as the `port` field.

### Port definitions[](https://kubernetes.io/docs/concepts/services-networking/service/#field-spec-ports)

Port definitions in Pods have names, and you can reference these names in the `targetPort` attribute of a Service. For example, we can bind the `targetPort` of the Service to the Pod port in the following way:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:stable
    ports:
      - containerPort: 80
        name: http-web-svc

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 80
    targetPort: http-web-svc
```

This works even if there is a mixture of Pods in the Service using a single configured name, with the same network protocol available via different port numbers. This offers a lot of flexibility for deploying and evolving your Services. For example, you can change the port numbers that Pods expose in the next version of your backend software, without breaking clients.

The default protocol for Services is [TCP](https://kubernetes.io/docs/reference/networking/service-protocols/#protocol-tcp); you can also use any other [supported protocol](https://kubernetes.io/docs/reference/networking/service-protocols/).

Because many Services need to expose more than one port, Kubernetes supports [multiple port definitions](https://kubernetes.io/docs/concepts/services-networking/service/#multi-port-services) for a single Service. Each port definition can have the same `protocol`, or a different one.

### Services without selectors[](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors)

Services most commonly abstract access to Kubernetes Pods thanks to the selector, but when used with a corresponding set of [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) objects and without a selector, the Service can abstract other kinds of backends, including ones that run outside the cluster.

For example:

- You want to have an external database cluster in production, but in your test environment you use your own databases.
- You want to point your Service to a Service in a different [Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces) or on another cluster.
- You are migrating a workload to Kubernetes. While evaluating the approach, you run only a portion of your backends in Kubernetes.

In any of these scenarios you can define a Service _without_ specifying a selector to match Pods. For example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
```

Because this Service has no selector, the corresponding EndpointSlice objects are not created automatically. You can map the Service to the network address and port where it's running, by adding an EndpointSlice object manually. For example:

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-1 # by convention, use the name of the Service
                     # as a prefix for the name of the EndpointSlice
  labels:
    # You should set the "kubernetes.io/service-name" label.
    # Set its value to match the name of the Service
    kubernetes.io/service-name: my-service
addressType: IPv4
ports:
  - name: http # should match with the name of the service port defined above
    appProtocol: http
    protocol: TCP
    port: 9376
endpoints:
  - addresses:
      - "10.4.5.6"
  - addresses:
      - "10.1.2.3"
```

### . `name: http` trong `ports`: Sao phải trùng?

> `name: http` là gì ? sao phải trùng ?

- **Nó là gì?** `name: http` là một **tên định danh** (symbolic name) mà bạn đặt cho một cặp cổng (`port` và `targetPort`) trong Service. Nó giúp việc tham chiếu dễ dàng và dễ đọc hơn thay vì phải nhớ con số cụ thể.
    
- **Tại sao phải trùng?** Đây là **cơ chế liên kết giữa các cổng (ports)**. Một Service có thể mở nhiều cổng cùng lúc. Ví dụ:
    
    - Cổng `80` cho traffic `http`.
        
    - Cổng `443` cho traffic `https`
        
    - Cổng `50051` cho traffic `grpc`.
        
    
    YAML
    
    ```
    # Trong Service (Thực đơn)
    spec:
      ports:
        - name: web         # Món "web"
          port: 80
          targetPort: 8080
        - name: metrics     # Món "metrics"
          port: 9090
          targetPort: 9191
    ```
    
    Khi đó, trong EndpointSlice (phiếu gọi món), bạn cũng phải chỉ rõ địa chỉ IP này là dành cho "món web" hay "món metrics". Việc đặt tên trùng nhau giúp Kubernetes biết rằng:
    
    - Định nghĩa cổng tên `web` trong EndpointSlice sẽ cung cấp các IP backend cho cổng tên `web` trong Service.
        
    - Định nghĩa cổng tên `metrics` trong EndpointSlice sẽ cung cấp các IP backend cho cổng tên `metrics` trong Service.
        
    
    Nếu không có tên trùng khớp, Kubernetes sẽ không biết danh sách địa chỉ IP `$endpoints` (`"10.4.5.6"`, `"10.1.2.3"`) này nên được dùng cho cổng nào của Service. Việc đặt tên trùng nhau đảm bảo việc ánh xạ được chính xác.
    

---

### 2. `kubernetes.io/service-name: my-service`: Sao phải trùng với tên service?

> `kubernetes.io/service-name: my-service` sao phải trùng với tên service ?

- **Nó là gì?** Đây là một **nhãn (label) đặc biệt và bắt buộc**. Tên của label này (`kubernetes.io/service-name`) là một quy ước cố định của Kubernetes. Giá trị của nó (`my-service`) phải là tên của Service mà bạn muốn liên kết tới.
    
- **Tại sao phải trùng?** Đây là **sợi dây liên kết chính** giữa đối tượng Service và đối tượng EndpointSlice.
    
    Khi một yêu cầu được gửi đến Service có tên `my-service`, bộ điều khiển của Kubernetes (Kubernetes control plane) sẽ thực hiện một việc rất đơn giản: **Nó sẽ tìm kiếm tất cả các đối tượng EndpointSlice trong cùng Namespace có cái nhãn `kubernetes.io/service-name` với giá trị là `my-service`**.
    
    Tất cả các địa chỉ IP được tìm thấy trong các EndpointSlice này sẽ được tổng hợp lại thành danh sách các điểm cuối (endpoints) hợp lệ cho Service đó.
    
    Nếu bạn đặt tên khác hoặc không có nhãn này, Service `my-service` sẽ không bao giờ tìm thấy EndpointSlice của nó. Kết quả là Service đó sẽ không có endpoint nào và không thể định tuyến traffic đi đâu cả. Nó sẽ bị "mồ côi".
    
#### Custom EndpointSlices[](https://kubernetes.io/docs/concepts/services-networking/service/#custom-endpointslices)

When you create an [EndpointSlice](https://kubernetes.io/docs/concepts/services-networking/service/#endpointslices) object for a Service, you can use any name for the EndpointSlice. Each EndpointSlice in a namespace must have a unique name. You link an EndpointSlice to a Service by setting the `kubernetes.io/service-name` [label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels) on that EndpointSlice.

#### Note:

The endpoint IPs _must not_ be: loopback (127.0.0.0/8 for IPv4, ::1/128 for IPv6), or link-local (169.254.0.0/16 and 224.0.0.0/24 for IPv4, fe80::/64 for IPv6).

The endpoint IP addresses cannot be the cluster IPs of other Kubernetes Services, because [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) doesn't support virtual IPs as a destination.

For an EndpointSlice that you create yourself, or in your own code, you should also pick a value to use for the label [`endpointslice.kubernetes.io/managed-by`](https://kubernetes.io/docs/reference/labels-annotations-taints/#endpointslicekubernetesiomanaged-by). If you create your own controller code to manage EndpointSlices, consider using a value similar to `"my-domain.example/name-of-controller"`. If you are using a third party tool, use the name of the tool in all-lowercase and change spaces and other punctuation to dashes (`-`). If people are directly using a tool such as `kubectl` to manage EndpointSlices, use a name that describes this manual management, such as `"staff"` or `"cluster-admins"`. You should avoid using the reserved value `"controller"`, which identifies EndpointSlices managed by Kubernetes' own control plane.

#### Accessing a Service without a selector[](https://kubernetes.io/docs/concepts/services-networking/service/#service-no-selector-access)

Accessing a Service without a selector works the same as if it had a selector. In the [example](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors) for a Service without a selector, traffic is routed to one of the two endpoints defined in the EndpointSlice manifest: a TCP connection to 10.1.2.3 or 10.4.5.6, on port 9376.

#### Note:

The Kubernetes API server does not allow proxying to endpoints that are not mapped to pods. Actions such as `kubectl port-forward service/<service-name> forwardedPort:servicePort` where the service has no selector will fail due to this constraint. This prevents the Kubernetes API server from being used as a proxy to endpoints the caller may not be authorized to access.

An `ExternalName` Service is a special case of Service that does not have selectors and uses DNS names instead. For more information, see the [ExternalName](https://kubernetes.io/docs/concepts/services-networking/service/#externalname) section.

### EndpointSlices[](https://kubernetes.io/docs/concepts/services-networking/service/#endpointslices)

FEATURE STATE: `Kubernetes v1.21 [stable]`

[EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) are objects that represent a subset (a _slice_) of the backing network endpoints for a Service.

Your Kubernetes cluster tracks how many endpoints each EndpointSlice represents. If there are so many endpoints for a Service that a threshold is reached, then Kubernetes adds another empty EndpointSlice and stores new endpoint information there. By default, Kubernetes makes a new EndpointSlice once the existing EndpointSlices all contain at least 100 endpoints. Kubernetes does not make the new EndpointSlice until an extra endpoint needs to be added.

See [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) for more information about this API.

### Endpoints (deprecated)[](https://kubernetes.io/docs/concepts/services-networking/service/#endpoints)

FEATURE STATE: `Kubernetes v1.33 [deprecated]`

The EndpointSlice API is the evolution of the older [Endpoints](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/endpoints-v1/) API. The deprecated Endpoints API has several problems relative to EndpointSlice:

- It does not support dual-stack clusters.
- It does not contain information needed to support newer features, such as [trafficDistribution](https://kubernetes.io/docs/concepts/services-networking/service/#traffic-distribution).
- It will truncate the list of endpoints if it is too long to fit in a single object.

Because of this, it is recommended that all clients use the EndpointSlice API rather than Endpoints.

#### Over-capacity endpoints[](https://kubernetes.io/docs/concepts/services-networking/service/#over-capacity-endpoints)

Kubernetes limits the number of endpoints that can fit in a single Endpoints object. When there are over 1000 backing endpoints for a Service, Kubernetes truncates the data in the Endpoints object. Because a Service can be linked with more than one EndpointSlice, the 1000 backing endpoint limit only affects the legacy Endpoints API.

In that case, Kubernetes selects at most 1000 possible backend endpoints to store into the Endpoints object, and sets an [annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations) on the Endpoints: [`endpoints.kubernetes.io/over-capacity: truncated`](https://kubernetes.io/docs/reference/labels-annotations-taints/#endpoints-kubernetes-io-over-capacity). The control plane also removes that annotation if the number of backend Pods drops below 1000.

Traffic is still sent to backends, but any load balancing mechanism that relies on the legacy Endpoints API only sends traffic to at most 1000 of the available backing endpoints.

The same API limit means that you cannot manually update an Endpoints to have more than 1000 endpoints.

### Application protocol[](https://kubernetes.io/docs/concepts/services-networking/service/#application-protocol)

FEATURE STATE: `Kubernetes v1.20 [stable]`

The `appProtocol` field provides a way to specify an application protocol for each Service port. This is used as a hint for implementations to offer richer behavior for protocols that they understand. The value of this field is mirrored by the corresponding Endpoints and EndpointSlice objects.

This field follows standard Kubernetes label syntax. Valid values are one of:

- [IANA standard service names](https://www.iana.org/assignments/service-names).
    
- Implementation-defined prefixed names such as `mycompany.com/my-custom-protocol`.
    
- Kubernetes-defined prefixed names:
    

|Protocol|Description|
|---|---|
|`kubernetes.io/h2c`|HTTP/2 over cleartext as described in [RFC 7540](https://www.rfc-editor.org/rfc/rfc7540)|
|`kubernetes.io/ws`|WebSocket over cleartext as described in [RFC 6455](https://www.rfc-editor.org/rfc/rfc6455)|
|`kubernetes.io/wss`|WebSocket over TLS as described in [RFC 6455](https://www.rfc-editor.org/rfc/rfc6455)|

### Multi-port Services[](https://kubernetes.io/docs/concepts/services-networking/service/#multi-port-services)

For some Services, you need to expose more than one port. Kubernetes lets you configure multiple port definitions on a Service object. When using multiple ports for a Service, you must give all of your ports names so that these are unambiguous. For example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```

#### Note:

As with Kubernetes [names](https://kubernetes.io/docs/concepts/overview/working-with-objects/names) in general, names for ports must only contain lowercase alphanumeric characters and `-`. Port names must also start and end with an alphanumeric character.

For example, the names `123-abc` and `web` are valid, but `123_abc` and `-web` are not.

## Service type[](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)

For some parts of your application (for example, frontends) you may want to expose a Service onto an external IP address, one that's accessible from outside of your cluster.

Kubernetes Service types allow you to specify what kind of Service you want.

The available `type` values and their behaviors are:

[`ClusterIP`](https://kubernetes.io/docs/concepts/services-networking/service/#type-clusterip)

Exposes the Service on a cluster-internal IP. Choosing this value makes the Service only reachable from within the cluster. This is the default that is used if you don't explicitly specify a `type` for a Service. You can expose the Service to the public internet using an [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) or a [Gateway](https://gateway-api.sigs.k8s.io/).

[`NodePort`](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)

Exposes the Service on each Node's IP at a static port (the `NodePort`). To make the node port available, Kubernetes sets up a cluster IP address, the same as if you had requested a Service of `type: ClusterIP`.

[`LoadBalancer`](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)

Exposes the Service externally using an external load balancer. Kubernetes does not directly offer a load balancing component; you must provide one, or you can integrate your Kubernetes cluster with a cloud provider.

[`ExternalName`](https://kubernetes.io/docs/concepts/services-networking/service/#externalname)

Maps the Service to the contents of the `externalName` field (for example, to the hostname `api.foo.bar.example`). The mapping configures your cluster's DNS server to return a `CNAME` record with that external hostname value. No proxying of any kind is set up.

The `type` field in the Service API is designed as nested functionality - each level adds to the previous. However there is an exception to this nested design. You can define a `LoadBalancer` Service by [disabling the load balancer `NodePort` allocation](https://kubernetes.io/docs/concepts/services-networking/service/#load-balancer-nodeport-allocation).

### `type: ClusterIP`[](https://kubernetes.io/docs/concepts/services-networking/service/#type-clusterip)

This default Service type assigns an IP address from a pool of IP addresses that your cluster has reserved for that purpose.

Several of the other types for Service build on the `ClusterIP` type as a foundation.

If you define a Service that has the `.spec.clusterIP` set to `"None"` then Kubernetes does not assign an IP address. See [headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) for more information.

#### Choosing your own IP address[](https://kubernetes.io/docs/concepts/services-networking/service/#choosing-your-own-ip-address)

You can specify your own cluster IP address as part of a `Service` creation request. To do this, set the `.spec.clusterIP` field. For example, if you already have an existing DNS entry that you wish to reuse, or legacy systems that are configured for a specific IP address and difficult to re-configure.

The IP address that you choose must be a valid IPv4 or IPv6 address from within the `service-cluster-ip-range` CIDR range that is configured for the API server. If you try to create a Service with an invalid `clusterIP` address value, the API server will return a 422 HTTP status code to indicate that there's a problem.

Read [avoiding collisions](https://kubernetes.io/docs/reference/networking/virtual-ips/#avoiding-collisions) to learn how Kubernetes helps reduce the risk and impact of two different Services both trying to use the same IP address.

### `type: NodePort`[](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)

If you set the `type` field to `NodePort`, the Kubernetes control plane allocates a port from a range specified by `--service-node-port-range` flag (default: 30000-32767). Each node proxies that port (the same port number on every Node) into your Service. Your Service reports the allocated port in its `.spec.ports[*].nodePort` field.

Using a NodePort gives you the freedom to set up your own load balancing solution, to configure environments that are not fully supported by Kubernetes, or even to expose one or more nodes' IP addresses directly.

For a node port Service, Kubernetes additionally allocates a port (TCP, UDP or SCTP to match the protocol of the Service). Every node in the cluster configures itself to listen on that assigned port and to forward traffic to one of the ready endpoints associated with that Service. You'll be able to contact the `type: NodePort` Service, from outside the cluster, by connecting to any node using the appropriate protocol (for example: TCP), and the appropriate port (as assigned to that Service).

#### Choosing your own port[](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport-custom-port)

If you want a specific port number, you can specify a value in the `nodePort` field. The control plane will either allocate you that port or report that the API transaction failed. This means that you need to take care of possible port collisions yourself. You also have to use a valid port number, one that's inside the range configured for NodePort use.

Here is an example manifest for a Service of `type: NodePort` that specifies a NodePort value (30007, in this example):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - port: 80
      # By default and for convenience, the `targetPort` is set to
      # the same value as the `port` field.
      targetPort: 80
      # Optional field
      # By default and for convenience, the Kubernetes control plane
      # will allocate a port from a range (default: 30000-32767)
      nodePort: 30007
```

#### Reserve Nodeport ranges to avoid collisions[](https://kubernetes.io/docs/concepts/services-networking/service/#avoid-nodeport-collisions)

The policy for assigning ports to NodePort services applies to both the auto-assignment and the manual assignment scenarios. When a user wants to create a NodePort service that uses a specific port, the target port may conflict with another port that has already been assigned.

To avoid this problem, the port range for NodePort services is divided into two bands. Dynamic port assignment uses the upper band by default, and it may use the lower band once the upper band has been exhausted. Users can then allocate from the lower band with a lower risk of port collision.

#### Custom IP address configuration for `type: NodePort` Services[](https://kubernetes.io/docs/concepts/services-networking/service/#service-nodeport-custom-listen-address)

You can set up nodes in your cluster to use a particular IP address for serving node port services. You might want to do this if each node is connected to multiple networks (for example: one network for application traffic, and another network for traffic between nodes and the control plane).

If you want to specify particular IP address(es) to proxy the port, you can set the `--nodeport-addresses` flag for kube-proxy or the equivalent `nodePortAddresses` field of the [kube-proxy configuration file](https://kubernetes.io/docs/reference/config-api/kube-proxy-config.v1alpha1/) to particular IP block(s).

This flag takes a comma-delimited list of IP blocks (e.g. `10.0.0.0/8`, `192.0.2.0/25`) to specify IP address ranges that kube-proxy should consider as local to this node.

For example, if you start kube-proxy with the `--nodeport-addresses=127.0.0.0/8` flag, kube-proxy only selects the loopback interface for NodePort Services. The default for `--nodeport-addresses` is an empty list. This means that kube-proxy should consider all available network interfaces for NodePort. (That's also compatible with earlier Kubernetes releases.)

#### Note:

This Service is visible as `<NodeIP>:spec.ports[*].nodePort` and `.spec.clusterIP:spec.ports[*].port`. If the `--nodeport-addresses` flag for kube-proxy or the equivalent field in the kube-proxy configuration file is set, `<NodeIP>` would be a filtered node IP address (or possibly IP addresses).

### `type: LoadBalancer`[](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)

On cloud providers which support external load balancers, setting the `type` field to `LoadBalancer` provisions a load balancer for your Service. The actual creation of the load balancer happens asynchronously, and information about the provisioned balancer is published in the Service's `.status.loadBalancer` field. For example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```

Traffic from the external load balancer is directed at the backend Pods. The cloud provider decides how it is load balanced.

To implement a Service of `type: LoadBalancer`, Kubernetes typically starts off by making the changes that are equivalent to you requesting a Service of `type: NodePort`. The cloud-controller-manager component then configures the external load balancer to forward traffic to that assigned node port.

You can configure a load balanced Service to [omit](https://kubernetes.io/docs/concepts/services-networking/service/#load-balancer-nodeport-allocation) assigning a node port, provided that the cloud provider implementation supports this.

Some cloud providers allow you to specify the `loadBalancerIP`. In those cases, the load-balancer is created with the user-specified `loadBalancerIP`. If the `loadBalancerIP` field is not specified, the load balancer is set up with an ephemeral IP address. If you specify a `loadBalancerIP` but your cloud provider does not support the feature, the `loadbalancerIP` field that you set is ignored.

Đây là quy trình từng bước khi bạn `kubectl apply` một file YAML có `type: LoadBalancer`:

**Analogy:** Hãy tưởng tượng bạn là **chủ nhà** (người dùng Kubernetes), và nhà cung cấp đám mây là một **công ty giao hàng chuyên nghiệp** (ví dụ: Grab).

**Bước 1: Bạn yêu cầu một Service `LoadBalancer`** Bạn tạo một Service trong Kubernetes. Đây giống như bạn đặt một yêu cầu "Tôi muốn có một người giao hàng chuyên dụng để nhận đồ cho nhà tôi".

**Bước 2: Kubernetes chuẩn bị các "Điểm giao hàng tạm thời" (`NodePort`)**

- **Đây là phần quan trọng nhất:** Trước khi gọi cho công ty giao hàng, Kubernetes sẽ tự động làm tất cả các bước của một `Service` loại `NodePort`.
    
- Nó sẽ cấp phát một cổng `NodePort` (ví dụ: `31500`) và mở cổng này trên **tất cả các Node** trong cluster.
    
- _Analogy:_ Trợ lý của bạn (Kubernetes) thiết lập sẵn các tủ đồ thông minh (NodePort) ở tất cả các con đường xung quanh khu phố của bạn, tất cả đều có mã `31500`.
    

**Bước 3: `cloud-controller-manager` vào cuộc**

- Trong cluster Kubernetes của bạn có một thành phần gọi là `cloud-controller-manager`. Đây là "người phiên dịch" giữa Kubernetes và API của nhà cung cấp đám mây.
    
- Nó thấy yêu cầu tạo `Service LoadBalancer` của bạn.
    
- _Analogy:_ Trợ lý của bạn nhấc máy lên và gọi cho tổng đài Grab.
    

**Bước 4: Tạo Load Balancer trên Cloud**

- `cloud-controller-manager` sẽ gửi một yêu cầu API đến nhà cung cấp đám mây: "Này AWS/Google Cloud, hãy tạo cho tôi một Network Load Balancer mới."
    
- Nhà cung cấp đám mây sẽ khởi tạo một Load Balancer thực sự trong tài khoản của bạn. Quá trình này mất vài phút và Load Balancer này sẽ có một địa chỉ IP public, ổn định của riêng nó (ví dụ: `54.169.100.200`).
    
- _Analogy:_ Grab chỉ định một shipper tên Nguyễn Văn A, số điện thoại `54.169.100.200`, chuyên nhận hàng cho bạn.
    

**Bước 5: Cập nhật lại Service trong Kubernetes**

- Sau khi Load Balancer được tạo xong, `cloud-controller-manager` sẽ lấy địa chỉ IP public đó (`54.169.100.200`) và cập nhật ngược lại vào trường `status` của đối tượng `Service` của bạn trong Kubernetes.
    
- Đây là lý do tại sao sau khi tạo Service một lúc, bạn chạy lệnh `kubectl get service` sẽ thấy cột `EXTERNAL-IP` hiện ra một địa chỉ IP.
    

### "Và nó trỏ vào đâu?"

Đây là luồng traffic khi một người dùng bên ngoài truy cập ứng dụng của bạn:

**Người dùng bên ngoài** `|` `v` **1. IP Public của Load Balancer** (ví dụ: `54.169.100.200:80`)

- Đây là địa chỉ mà bạn đưa cho người dùng.
    

`|` `v` **2. Cloud Load Balancer** (Tài nguyên của AWS/Google Cloud)

- Load Balancer này đã được `cloud-controller-manager` cấu hình để chuyển tiếp traffic đến **tất cả các Node của bạn trên cổng `NodePort` đã được cấp phát**.
    
- Nó sẽ trỏ đến:
    
    - `IP_Node_A:31500`
        
    - `IP_Node_B:31500`
        
    - `IP_Node_C:31500`
        
    - ...
        

`|` `v` **3. Một Node bất kỳ trong Cluster** (ví dụ: traffic đến `IP_Node_B:31500`)

- Traffic bây giờ đã vào bên trong mạng của cluster.
    

`|` `v` **4. `kube-proxy` trên Node B**

- `kube-proxy` trên Node B nhận lấy traffic này và chuyển nó đến một Pod backend đang khỏe mạnh của Service, thông qua `ClusterIP`.
    

`|` `v` **5. Pod Backend** (có thể đang chạy trên Node A, B, hoặc C)

- Traffic cuối cùng đã đến được ứng dụng của bạn.
#### Note:

The`.spec.loadBalancerIP` field for a Service was deprecated in Kubernetes v1.24.

This field was under-specified and its meaning varies across implementations. It also cannot support dual-stack networking. This field may be removed in a future API version.

If you're integrating with a provider that supports specifying the load balancer IP address(es) for a Service via a (provider specific) annotation, you should switch to doing that.

If you are writing code for a load balancer integration with Kubernetes, avoid using this field. You can integrate with [Gateway](https://gateway-api.sigs.k8s.io/) rather than Service, or you can define your own (provider specific) annotations on the Service that specify the equivalent detail.

#### Node liveness impact on load balancer traffic[](https://kubernetes.io/docs/concepts/services-networking/service/#node-liveness-impact-on-load-balancer-traffic)

Load balancer health checks are critical to modern applications. They are used to determine which server (virtual machine, or IP address) the load balancer should dispatch traffic to. The Kubernetes APIs do not define how health checks have to be implemented for Kubernetes managed load balancers, instead it's the cloud providers (and the people implementing integration code) who decide on the behavior. Load balancer health checks are extensively used within the context of supporting the `externalTrafficPolicy` field for Services.

#### Load balancers with mixed protocol types[](https://kubernetes.io/docs/concepts/services-networking/service/#load-balancers-with-mixed-protocol-types)

FEATURE STATE: `Kubernetes v1.26 [stable]` (enabled by default: true)

By default, for LoadBalancer type of Services, when there is more than one port defined, all ports must have the same protocol, and the protocol must be one which is supported by the cloud provider.

The feature gate `MixedProtocolLBService` (enabled by default for the kube-apiserver as of v1.24) allows the use of different protocols for LoadBalancer type of Services, when there is more than one port defined.

#### Note:

The set of protocols that can be used for load balanced Services is defined by your cloud provider; they may impose restrictions beyond what the Kubernetes API enforces.

#### Disabling load balancer NodePort allocation[](https://kubernetes.io/docs/concepts/services-networking/service/#load-balancer-nodeport-allocation)

FEATURE STATE: `Kubernetes v1.24 [stable]`

You can optionally disable node port allocation for a Service of `type: LoadBalancer`, by setting the field `spec.allocateLoadBalancerNodePorts` to `false`. This should only be used for load balancer implementations that route traffic directly to pods as opposed to using node ports. By default, `spec.allocateLoadBalancerNodePorts` is `true` and type LoadBalancer Services will continue to allocate node ports. If `spec.allocateLoadBalancerNodePorts` is set to `false` on an existing Service with allocated node ports, those node ports will **not** be de-allocated automatically. You must explicitly remove the `nodePorts` entry in every Service port to de-allocate those node ports.

#### Specifying class of load balancer implementation[](https://kubernetes.io/docs/concepts/services-networking/service/#load-balancer-class)

FEATURE STATE: `Kubernetes v1.24 [stable]`

For a Service with `type` set to `LoadBalancer`, the `.spec.loadBalancerClass` field enables you to use a load balancer implementation other than the cloud provider default.

By default, `.spec.loadBalancerClass` is not set and a `LoadBalancer` type of Service uses the cloud provider's default load balancer implementation if the cluster is configured with a cloud provider using the `--cloud-provider` component flag.

If you specify `.spec.loadBalancerClass`, it is assumed that a load balancer implementation that matches the specified class is watching for Services. Any default load balancer implementation (for example, the one provided by the cloud provider) will ignore Services that have this field set. `spec.loadBalancerClass` can be set on a Service of type `LoadBalancer` only. Once set, it cannot be changed. The value of `spec.loadBalancerClass` must be a label-style identifier, with an optional prefix such as "`internal-vip`" or "`example.com/internal-vip`". Unprefixed names are reserved for end-users.

#### Load balancer IP address mode[](https://kubernetes.io/docs/concepts/services-networking/service/#load-balancer-ip-mode)

FEATURE STATE: `Kubernetes v1.32 [stable]` (enabled by default: true)

For a Service of `type: LoadBalancer`, a controller can set `.status.loadBalancer.ingress.ipMode`. The `.status.loadBalancer.ingress.ipMode` specifies how the load-balancer IP behaves. It may be specified only when the `.status.loadBalancer.ingress.ip` field is also specified.

There are two possible values for `.status.loadBalancer.ingress.ipMode`: "VIP" and "Proxy". The default value is "VIP" meaning that traffic is delivered to the node with the destination set to the load-balancer's IP and port. There are two cases when setting this to "Proxy", depending on how the load-balancer from the cloud provider delivers the traffics:

- If the traffic is delivered to the node then DNATed to the pod, the destination would be set to the node's IP and node port;
- If the traffic is delivered directly to the pod, the destination would be set to the pod's IP and port.

Service implementations may use this information to adjust traffic routing.

#### Internal load balancer[](https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer)

In a mixed environment it is sometimes necessary to route traffic from Services inside the same (virtual) network address block.

In a split-horizon DNS environment you would need two Services to be able to route both external and internal traffic to your endpoints.

To set an internal load balancer, add one of the following annotations to your Service depending on the cloud service provider you're using:

- [Default](https://kubernetes.io/docs/concepts/services-networking/service/#service-tabs-0)
- [GCP](https://kubernetes.io/docs/concepts/services-networking/service/#service-tabs-1)
- [AWS](https://kubernetes.io/docs/concepts/services-networking/service/#service-tabs-2)
- [Azure](https://kubernetes.io/docs/concepts/services-networking/service/#service-tabs-3)
- [IBM Cloud](https://kubernetes.io/docs/concepts/services-networking/service/#service-tabs-4)
- [OpenStack](https://kubernetes.io/docs/concepts/services-networking/service/#service-tabs-5)
- [Baidu Cloud](https://kubernetes.io/docs/concepts/services-networking/service/#service-tabs-6)
- [Tencent Cloud](https://kubernetes.io/docs/concepts/services-networking/service/#service-tabs-7)
- [Alibaba Cloud](https://kubernetes.io/docs/concepts/services-networking/service/#service-tabs-8)
- [OCI](https://kubernetes.io/docs/concepts/services-networking/service/#service-tabs-9)

Select one of the tabs.

### `type: ExternalName`[](https://kubernetes.io/docs/concepts/services-networking/service/#externalname)

Services of type ExternalName map a Service to a DNS name, not to a typical selector such as `my-service` or `cassandra`. You specify these Services with the `spec.externalName` parameter.

This Service definition, for example, maps the `my-service` Service in the `prod` namespace to `my.database.example.com`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

#### Note:

A Service of `type: ExternalName` accepts an IPv4 address string, but treats that string as a DNS name comprised of digits, not as an IP address (the internet does not however allow such names in DNS). Services with external names that resemble IPv4 addresses are not resolved by DNS servers.

Đây là một cạm bẫy phổ biến. Bạn có thể nghĩ rằng mình có thể dùng `ExternalName` để trỏ đến một địa chỉ IP bên ngoài, ví dụ:

YAML

```
# CÁCH LÀM SAI
apiVersion: v1
kind: Service
metadata:
  name: my-external-ip-service
spec:
  type: ExternalName
  externalName: "8.8.8.8" # <-- Vấn đề ở đây
```

Kubernetes sẽ chấp nhận file YAML này, nhưng nó sẽ **KHÔNG** hoạt động như bạn mong đợi. Nó sẽ coi `"8.8.8.8"` là một **tên miền (hostname)**, chứ không phải là một địa chỉ IP. Hệ thống DNS sẽ cố gắng tìm một bản ghi cho tên miền `8.8.8.8`, và dĩ nhiên là sẽ thất bại.

**Cách làm đúng để trỏ Service đến một IP cụ thể:** Như tài liệu gợi ý, bạn nên dùng một **Headless Service** (Service không có `ClusterIP`) và tạo một `EndpointSlice` thủ công để trỏ đến địa chỉ IP đó.

If you want to map a Service directly to a specific IP address, consider using [headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services).

When looking up the host `my-service.prod.svc.cluster.local`, the cluster DNS Service returns a `CNAME` record with the value `my.database.example.com`. Accessing `my-service` works in the same way as other Services but with the crucial difference that redirection happens at the DNS level rather than via proxying or forwarding. Should you later decide to move your database into your cluster, you can start its Pods, add appropriate selectors or endpoints, and change the Service's `type`.

`Service` loại `ExternalName` là loại Service đơn giản nhất. Nó không làm gì liên quan đến proxy hay chuyển tiếp traffic. Nó chỉ đơn giản là tạo ra một **bí danh (alias)** ở tầng DNS cho các ứng dụng bên trong cluster của bạn.

**Analogy:** Hãy tưởng tượng trong công ty nội bộ (cluster), mọi người quen gọi một đối tác quan trọng là **"Đối tác Vàng"** (`my-service`). Nhưng địa chỉ thật của đối tác đó ở bên ngoài là **`my.database.example.com`**.

Khi bạn tạo một `Service ExternalName` tên là "Đối tác Vàng" và trỏ nó đến `my.database.example.com`, bạn đang nói với hệ thống DNS nội bộ của công ty (Cluster DNS) rằng:

> "Này DNS, nếu có ai hỏi địa chỉ của 'Đối tác Vàng', hãy bảo họ rằng đó thực chất là `my.database.example.com`."

Về mặt kỹ thuật, DNS sẽ trả về một bản ghi `CNAME`. Sau đó, máy của người hỏi sẽ tự thực hiện một truy vấn DNS mới để tìm địa chỉ IP của `my.database.example.com` và kết nối trực tiếp đến đó.

**Sự linh hoạt:** Phần cuối nói rằng sau này, nếu bạn quyết định chuyển "Đối tác Vàng" vào hoạt động bên trong công ty, bạn có thể chỉ cần thay đổi `type` của Service này (ví dụ: thành `ClusterIP`) và thêm `selector` để trỏ đến các Pod mới. Các ứng dụng nội bộ vẫn gọi "Đối tác Vàng" mà không cần thay đổi bất kỳ dòng code nào.
#### Caution:

You may have trouble using ExternalName for some common protocols, including HTTP and HTTPS. If you use ExternalName then the hostname used by clients inside your cluster is different from the name that the ExternalName references.

For protocols that use hostnames this difference may lead to errors or unexpected responses. HTTP requests will have a `Host:` header that the origin server does not recognize; TLS servers will not be able to provide a certificate matching the hostname that the client connected to.

Đây là **vấn đề lớn nhất và quan trọng nhất** của `ExternalName`. Nó xảy ra do sự không khớp về tên máy chủ (hostname mismatch).

Hãy quay lại ví dụ "Đối tác Vàng" (`my-service`) và `my.database.example.com`.

Khi ứng dụng của bạn kết nối đến "Đối tác Vàng", nó nghĩ rằng nó đang nói chuyện với `my-service`. Nhưng thực tế, nó đang nói chuyện với server `my.database.example.com`.

**1. Vấn đề với HTTP:**

- Ứng dụng của bạn gửi một request HTTP. Trong header của request đó, nó sẽ khai báo: `Host: my-service`.
    
- Server `my.database.example.com` nhận được request này. Nó nhìn vào header `Host` và có thể sẽ không hiểu. Nó được cấu hình để chỉ trả lời các request có `Host: my.database.example.com`.
    
- **Kết quả:** Server có thể trả về lỗi `404 Not Found` hoặc một trang web mặc định, không phải nội dung bạn mong muốn.
    

**2. Vấn đề với HTTPS (nghiêm trọng hơn):**

- Ứng dụng của bạn cố gắng thiết lập một kết nối TLS/SSL (HTTPS) an toàn tới `my-service`.
    
- Server `my.database.example.com` trả về chứng chỉ số (certificate) của nó. Trên chứng chỉ đó ghi rõ: **"Chứng chỉ này được cấp cho `my.database.example.com`"**.
    
- Ứng dụng của bạn so sánh hai thứ: tên nó yêu cầu (`my-service`) và tên trên chứng chỉ (`my.database.example.com`).
    
- **Hai tên này không khớp!**
    
- **Kết quả:** Để đảm bảo an toàn, ứng dụng của bạn sẽ ngay lập tức hủy kết nối và báo lỗi "Certificate validation error" hoặc "Hostname mismatch". Đây là một cơ chế bảo mật cơ bản của TLS để chống lại các cuộc tấn âcông xen giữa (man-in-the-middle).
    

**Khi nào thì có thể dùng `ExternalName`?** Nó hoạt động tốt nhất với các giao thức không nhạy cảm với hostname, hoặc khi bạn có thể cấu hình client bỏ qua việc xác thực hostname (ví dụ: một số kết nối database TCP thuần túy). Đối với HTTP/HTTPS, hãy hết sức cẩn thận.
## Headless Services[](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)

Sometimes you don't need load-balancing and a single Service IP. In this case, you can create what are termed _headless Services_, by explicitly specifying `"None"` for the cluster IP address (`.spec.clusterIP`).

You can use a headless Service to interface with other service discovery mechanisms, without being tied to Kubernetes' implementation.

For headless Services, a cluster IP is not allocated, kube-proxy does not handle these Services, and there is no load balancing or proxying done by the platform for them.

A headless Service allows a client to connect to whichever Pod it prefers, directly. Services that are headless don't configure routes and packet forwarding using [virtual IP addresses and proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/); instead, headless Services report the endpoint IP addresses of the individual pods via internal DNS records, served through the cluster's [DNS service](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/). To define a headless Service, you make a Service with `.spec.type` set to ClusterIP (which is also the default for `type`), and you additionally set `.spec.clusterIP` to None.

The string value None is a special case and is not the same as leaving the `.spec.clusterIP` field unset.

How DNS is automatically configured depends on whether the Service has selectors defined:

### With selectors[](https://kubernetes.io/docs/concepts/services-networking/service/#with-selectors)

For headless Services that define selectors, the endpoints controller creates EndpointSlices in the Kubernetes API, and modifies the DNS configuration to return A or AAAA records (IPv4 or IPv6 addresses) that point directly to the Pods backing the Service.

### Without selectors[](https://kubernetes.io/docs/concepts/services-networking/service/#without-selectors)

For headless Services that do not define selectors, the control plane does not create EndpointSlice objects. However, the DNS system looks for and configures either:

- DNS CNAME records for [`type: ExternalName`](https://kubernetes.io/docs/concepts/services-networking/service/#externalname) Services.
- DNS A / AAAA records for all IP addresses of the Service's ready endpoints, for all Service types other than `ExternalName`.
    - For IPv4 endpoints, the DNS system creates A records.
    - For IPv6 endpoints, the DNS system creates AAAA records.

When you define a headless Service without a selector, the `port` must match the `targetPort`.

## Discovering services[](https://kubernetes.io/docs/concepts/services-networking/service/#discovering-services)

For clients running inside your cluster, Kubernetes supports two primary modes of finding a Service: environment variables and DNS.

### Environment variables[](https://kubernetes.io/docs/concepts/services-networking/service/#environment-variables)

When a Pod is run on a Node, the kubelet adds a set of environment variables for each active Service. It adds `{SVCNAME}_SERVICE_HOST` and `{SVCNAME}_SERVICE_PORT` variables, where the Service name is upper-cased and dashes are converted to underscores.

For example, the Service `redis-primary` which exposes TCP port 6379 and has been allocated cluster IP address 10.0.0.11, produces the following environment variables:

```shell
REDIS_PRIMARY_SERVICE_HOST=10.0.0.11
REDIS_PRIMARY_SERVICE_PORT=6379
REDIS_PRIMARY_PORT=tcp://10.0.0.11:6379
REDIS_PRIMARY_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_PRIMARY_PORT_6379_TCP_PROTO=tcp
REDIS_PRIMARY_PORT_6379_TCP_PORT=6379
REDIS_PRIMARY_PORT_6379_TCP_ADDR=10.0.0.11
```

#### Note:

When you have a Pod that needs to access a Service, and you are using the environment variable method to publish the port and cluster IP to the client Pods, you must create the Service _before_ the client Pods come into existence. Otherwise, those client Pods won't have their environment variables populated.

If you only use DNS to discover the cluster IP for a Service, you don't need to worry about this ordering issue.

Kubernetes also supports and provides variables that are compatible with Docker Engine's "_[legacy container links](https://docs.docker.com/network/links/)_" feature. You can read [`makeLinkVariables`](https://github.com/kubernetes/kubernetes/blob/dd2d12f6dc0e654c15d5db57a5f9f6ba61192726/pkg/kubelet/envvars/envvars.go#L72) to see how this is implemented in Kubernetes.

### DNS[](https://kubernetes.io/docs/concepts/services-networking/service/#dns)

You can (and almost always should) set up a DNS service for your Kubernetes cluster using an [add-on](https://kubernetes.io/docs/concepts/cluster-administration/addons/).

A cluster-aware DNS server, such as CoreDNS, watches the Kubernetes API for new Services and creates a set of DNS records for each one. If DNS has been enabled throughout your cluster then all Pods should automatically be able to resolve Services by their DNS name.

For example, if you have a Service called `my-service` in a Kubernetes namespace `my-ns`, the control plane and the DNS Service acting together create a DNS record for `my-service.my-ns`. Pods in the `my-ns` namespace should be able to find the service by doing a name lookup for `my-service` (`my-service.my-ns` would also work).

Pods in other namespaces must qualify the name as `my-service.my-ns`. These names will resolve to the cluster IP assigned for the Service.

Kubernetes also supports DNS SRV (Service) records for named ports. If the `my-service.my-ns` Service has a port named `http` with the protocol set to `TCP`, you can do a DNS SRV query for `_http._tcp.my-service.my-ns` to discover the port number for `http`, as well as the IP address.

The Kubernetes DNS server is the only way to access `ExternalName` Services. You can find more information about `ExternalName` resolution in [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).

Giả sử bạn có một Pod `frontend` cần nói chuyện với một Service `backend`. Vấn đề là `backend` có một địa chỉ IP nội bộ (`ClusterIP`) được gán tự động và có thể thay đổi. Vậy làm thế nào để `frontend` biết được địa chỉ IP và cổng của `backend` để kết nối?

Kubernetes cung cấp hai cơ chế chính để giải quyết vấn đề này.

---

### Cách 1: Biến môi trường (Environment Variables)

#### Nó là gì?

Đây là cơ chế mặc định và đơn giản nhất. Khi một Pod được khởi tạo trên một Node, `kubelet` (agent của Kubernetes trên Node đó) sẽ tự động "tiêm" một loạt các biến môi trường vào bên trong Pod. Các biến này chứa thông tin về **tất cả các Service đang hoạt động (active) tại thời điểm đó**.

#### Nó hoạt động như thế nào?

- Tên Service sẽ được chuyển thành chữ IN HOA, và các dấu gạch ngang (`-`) được đổi thành dấu gạch dưới (`_`).
    
- Các biến quan trọng nhất được tạo ra là:
    
    - `{TÊN_SERVICE}_SERVICE_HOST`: Chứa địa chỉ `ClusterIP` của Service.
        
    - `{TÊN_SERVICE}_SERVICE_PORT`: Chứa số cổng của Service.
        

**Ví dụ trong tài liệu:**

- Một Service tên là `redis-primary` có `ClusterIP` là `10.0.0.11` và cổng `6379`.
    
- Khi một Pod mới được tạo, nó sẽ có các biến môi trường sau:
    
    Shell
    
    ```
    REDIS_PRIMARY_SERVICE_HOST=10.0.0.11
    REDIS_PRIMARY_SERVICE_PORT=6379
    # và một số biến phụ trợ khác...
    ```
    
- Ứng dụng của bạn trong Pod chỉ cần đọc các biến môi trường này để biết địa chỉ cần kết nối.
    

#### **Lưu ý CỰC KỲ QUAN TRỌNG (Điểm yếu chí mạng):**

> You must create the Service **before** the client Pods come into existence.

- Bạn **BẮT BUỘC** phải tạo `Service` (`redis-primary`) **TRƯỚC KHI** bạn tạo Pod `client`.
    
- Lý do là vì các biến môi trường chỉ được tiêm vào Pod **một lần duy nhất lúc Pod khởi tạo**. Nếu Service chưa tồn tại vào thời điểm đó, Pod sẽ không có các biến môi trường tương ứng. Nếu bạn tạo Service sau, các Pod đã chạy sẽ không tự động cập nhật. Bạn sẽ phải khởi động lại chúng.
    
- Điều này tạo ra một sự phụ thuộc chặt chẽ về thứ tự, rất phiền phức và dễ gây lỗi.
    

---

### Cách 2: DNS (Phương pháp được khuyến nghị)

#### Nó là gì?

Đây là phương pháp hiện đại, linh hoạt và được khuyến khích sử dụng trong hầu hết mọi trường hợp. Hầu hết các cluster Kubernetes đều được cài đặt sẵn một dịch vụ DNS nội bộ (thường là CoreDNS).

#### Nó hoạt động như thế nào?

1. **Tự động tạo bản ghi DNS:** Server DNS này sẽ theo dõi API của Kubernetes. Mỗi khi một `Service` mới được tạo, nó sẽ tự động tạo ra một bản ghi DNS cho Service đó.
    
2. **Tự động phân giải tên:** Tất cả các Pod trong cluster đều được tự động cấu hình để sử dụng server DNS nội bộ này. Do đó, chúng có thể phân giải (resolve) tên của các Service khác một cách dễ dàng.
    

**Ví dụ trong tài liệu:**

- Bạn có một `Service` tên là `my-service` trong `namespace` là `my-ns`.
    
- Một bản ghi DNS `my-service.my-ns` sẽ được tạo ra và trỏ đến địa chỉ `ClusterIP` của Service đó.
    
- **Nếu Pod client ở trong cùng namespace `my-ns`:** Nó có thể kết nối bằng tên ngắn gọn: `my-service`.
    
- **Nếu Pod client ở một namespace khác:** Nó phải dùng tên đầy đủ (FQDN - Fully Qualified Domain Name): `my-service.my-ns`.
    
- **Với các cổng được đặt tên:** DNS còn hỗ trợ các bản ghi `SRV` nâng cao. Nếu cổng của bạn được đặt tên là `http`, client có thể thực hiện một truy vấn `SRV` tới `_http._tcp.my-service.my-ns` để tìm ra cả địa chỉ IP và số cổng chính xác.
    
- **Với `ExternalName`:** DNS là cách **duy nhất** để truy cập các Service loại `ExternalName`.

## Virtual IP addressing mechanism[](https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ip-addressing-mechanism)

Read [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/) explains the mechanism Kubernetes provides to expose a Service with a virtual IP address.

### Traffic policies[](https://kubernetes.io/docs/concepts/services-networking/service/#traffic-policies)

You can set the `.spec.internalTrafficPolicy` and `.spec.externalTrafficPolicy` fields to control how Kubernetes routes traffic to healthy (“ready”) backends.

See [Traffic Policies](https://kubernetes.io/docs/reference/networking/virtual-ips/#traffic-policies) for more details.

### Traffic distribution[](https://kubernetes.io/docs/concepts/services-networking/service/#traffic-distribution)

FEATURE STATE: `Kubernetes v1.33 [stable]` (enabled by default: true)

The `.spec.trafficDistribution` field provides another way to influence traffic routing within a Kubernetes Service. While traffic policies focus on strict semantic guarantees, traffic distribution allows you to express _preferences_ (such as routing to topologically closer endpoints). This can help optimize for performance, cost, or reliability. In Kubernetes 1.33, the following field value is supported:

`PreferClose`

Indicates a preference for routing traffic to endpoints that are in the same zone as the client.

FEATURE STATE: `Kubernetes v1.33 [alpha]` (enabled by default: false)

Two additional values are available when the `PreferSameTrafficDistribution` [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) is enabled:

`PreferSameZone`

This is an alias for `PreferClose` that is clearer about the intended semantics.

`PreferSameNode`

Indicates a preference for routing traffic to endpoints that are on the same node as the client.

If the field is not set, the implementation will apply its default routing strategy.

See [Traffic Distribution](https://kubernetes.io/docs/reference/networking/virtual-ips/#traffic-distribution) for more details

### Session stickiness[](https://kubernetes.io/docs/concepts/services-networking/service/#session-stickiness)

If you want to make sure that connections from a particular client are passed to the same Pod each time, you can configure session affinity based on the client's IP address. Read [session affinity](https://kubernetes.io/docs/reference/networking/virtual-ips/#session-affinity) to learn more.

## External IPs[](https://kubernetes.io/docs/concepts/services-networking/service/#external-ips)

If there are external IPs that route to one or more cluster nodes, Kubernetes Services can be exposed on those `externalIPs`. When network traffic arrives into the cluster, with the external IP (as destination IP) and the port matching that Service, rules and routes that Kubernetes has configured ensure that the traffic is routed to one of the endpoints for that Service.

When you define a Service, you can specify `externalIPs` for any [service type](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types). In the example below, the Service named `"my-service"` can be accessed by clients using TCP, on `"198.51.100.32:80"` (calculated from `.spec.externalIPs[]` and `.spec.ports[].port`).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 49152
  externalIPs:
    - 198.51.100.32
```
Tính năng `externalIPs` cho phép bạn "gán" một hoặc nhiều địa chỉ IP bên ngoài (mà bạn đã có và kiểm soát) vào một `Service` trong Kubernetes.

Khi đó, bất kỳ traffic nào gửi đến địa chỉ IP bên ngoài đó trên đúng cổng của `Service`, Kubernetes sẽ tự động bắt lấy và chuyển tiếp nó đến các Pod backend của `Service` đó.

Nó giống như việc bạn tạo ra một **lối vào riêng** cho `Service` của mình bằng một địa chỉ IP mà bạn đã sở hữu từ trước.

---

### Analogy dễ hiểu: Số Điện Thoại Chuyển Hướng

Hãy tưởng tượng:

- **Cluster Kubernetes** là một **tòa nhà văn phòng lớn**.
    
- **Các Service** bên trong là các **phòng ban/nhân viên** với các số máy lẻ nội bộ (`ClusterIP`).
    
- **Bạn (ứng dụng của bạn)** muốn có một số điện thoại đẹp, dễ nhớ (`198.51.100.32`) để khách hàng gọi trực tiếp thay vì phải qua tổng đài.
    

Tính năng `externalIPs` hoạt động như sau:

1. **Phần việc của bạn (Quản trị viên cluster):** Bạn phải tự mình làm việc với công ty viễn thông (tức là cấu hình router, mạng bên ngoài) để **chuyển hướng (route)** tất cả các cuộc gọi đến số `198.51.100.32` về tổng đài của tòa nhà văn phòng của bạn (tức là một trong các Node).
    
2. **Phần việc của Kubernetes:** Bạn khai báo trong hệ thống nội bộ của Kubernetes (thông qua `Service` YAML) rằng: "Này hệ thống, bất kỳ cuộc gọi nào được chuyển tiếp từ số `198.51.100.32`, hãy nối máy thẳng đến phòng ban của tôi (Pod)."
    

---

### Giải thích chi tiết theo tài liệu

#### 1. Luồng traffic hoạt động như thế nào?

> When network traffic arrives into the cluster, with the external IP (as destination IP) and the port matching that Service, rules and routes that Kubernetes has configured ensure that the traffic is routed to one of the endpoints for that Service.

Luồng đi của một gói tin sẽ như sau:

1. **Bên ngoài:** Một client gửi request đến `198.51.100.32:80`.
    
2. **Hạ tầng mạng của bạn:** Router/firewall của bạn nhận được gói tin này và (do bạn đã cấu hình từ trước) nó biết rằng phải chuyển gói tin này đến một trong các Node của cluster Kubernetes (ví dụ: Node có IP `10.0.0.5`).
    
3. **Bên trong Node:** Gói tin đến Node `10.0.0.5`. `kube-proxy` (agent mạng của Kubernetes) trên Node này đã được cài đặt sẵn các quy tắc mạng. Nó nhìn vào địa chỉ IP đích của gói tin (`198.51.100.32`) và nhận ra: "Aha! IP này đã được gán cho `my-service`."
    
4. **Đến Pod:** `kube-proxy` ngay lập tức chuyển hướng gói tin này đến một trong các Pod backend của `my-service` (ví dụ Pod có IP `172.17.0.10:49152`).
    

Quá trình này diễn ra một cách liền mạch.

#### 2. Ví dụ YAML

YAML

```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80 # Cổng mà client bên ngoài sẽ truy cập
      targetPort: 49152 # Cổng mà container đang lắng nghe bên trong Pod
  externalIPs:
    - 198.51.100.32 # Địa chỉ IP bên ngoài mà bạn gán cho Service này
```

Với cấu hình này, `my-service` sẽ có thể được truy cập tại `198.51.100.32:80`.

#### 3. "Note" - Lưu ý quan trọng về trách nhiệm

> Kubernetes does not manage allocation of `externalIPs`; these are the responsibility of the cluster administrator.

Đây là điểm khác biệt mấu chốt so với `Service` loại `LoadBalancer`:

- **Với `type: LoadBalancer`:** Bạn chỉ cần yêu cầu, Kubernetes sẽ tự động gọi API của cloud để **tạo ra** một Load Balancer và một IP public mới cho bạn.
    
- **Với `externalIPs`:** Bạn phải **tự chuẩn bị** địa chỉ IP đó từ trước. Kubernetes **KHÔNG** làm bất cứ điều gì bên ngoài cluster để IP đó có thể truy cập được. Nó chỉ đơn giản là lắng nghe traffic đến từ IP đó.
    

Trách nhiệm của quản trị viên cluster là phải đảm bảo rằng hạ tầng mạng bên ngoài có thể định tuyến traffic từ `198.51.100.32` đến ít nhất một trong các Node của cluster.

### So sánh nhanh

- **`NodePort`:** Truy cập qua `IP_của_Node:Cổng_cao`. Bất tiện vì cổng cao và phải biết IP của Node.
    
- **`LoadBalancer`:** Tự động tạo IP public và cân bằng tải. Tiện lợi nhất nhưng chỉ có trên cloud và có thể tốn tiền.
    
- **`externalIPs`:** Cho phép dùng IP bạn đã có và truy cập qua cổng chuẩn (80, 443). Rất linh hoạt, đặc biệt cho môi trường on-premise, nhưng đòi hỏi bạn phải tự cấu hình mạng bên ngoài.
#### Note:

Kubernetes does not manage allocation of `externalIPs`; these are the responsibility of the cluster administrator.

## API Object[](https://kubernetes.io/docs/concepts/services-networking/service/#api-object)

Service is a top-level resource in the Kubernetes REST API. You can find more details about the [Service API object](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.33/#service-v1-core).