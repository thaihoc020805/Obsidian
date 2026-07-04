# Bức tranh tổng thể

Tài liệu này mô tả **các kết nối mạng giữa worker node và control plane**, chứ không mô tả toàn bộ quá trình Kubernetes quản lý Pod.

Điểm quan trọng nhất:

> Trong Kubernetes, gần như mọi thành phần đều trao đổi trạng thái thông qua **kube-apiserver**.  
> Scheduler và controller-manager không kết nối trực tiếp tới kubelet để ra lệnh.

Mô hình cơ bản:

```text
                    CONTROL PLANE

              ┌──────────────────────┐
              │    kube-apiserver    │
              │      TCP 6443        │
              └──────────┬───────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
     scheduler     controller-manager   etcd
     API client       API client       phía sau API server


                      WORKER NODE

              ┌──────────────────────┐
              │       kubelet        │
              │ API client → 6443    │
              │ HTTPS server :10250  │
              └──────────────────────┘
```

Có hai chiều giao tiếp chính:

```text
Node / Pod ───────────────→ API server
          Node to Control Plane

API server ───────────────→ Kubelet / Node / Pod / Service
          Control Plane to Node
```

Hai chiều này **không đối xứng**:

- Khi kubelet gọi API server, kubelet đóng vai trò **HTTPS client**.
    
- Khi API server gọi kubelet, kubelet đóng vai trò **HTTPS server**.
    
- Vì vậy kubelet cần cả:
    
    - client certificate để kết nối API server;
        
    - serving certificate để nhận kết nối từ API server.
        

Kubernetes liệt kê riêng server certificate và client certificate của kubelet trong mô hình PKI. ([Kubernetes](https://kubernetes.io/docs/setup/best-practices/certificates/ "PKI certificates and requirements | Kubernetes"))

---

# 1. “Hub-and-spoke API pattern” nghĩa là gì?

`Hub-and-spoke` có thể hình dung như bánh xe:

```text
                   kubelet wk1
                       │
                       │
controller-manager ─ API server ─ scheduler
                       │
                       │
                   kubelet wk2
                       │
                    Pod client
```

Trong đó:

- **Hub**: kube-apiserver.
    
- **Spokes**: kubelet, scheduler, controller-manager, kube-proxy, ứng dụng trong Pod, `kubectl`…
    

Các thành phần không cần gọi chéo trực tiếp cho nhau.

Ví dụ scheduler muốn gán Pod cho `wk1`:

```text
1. Scheduler watch các Pod chưa có node
2. Scheduler chọn wk1
3. Scheduler gửi request cập nhật Pod lên API server
4. API server lưu thay đổi
5. Kubelet wk1 đang watch API server thấy Pod đã được gán cho mình
6. Kubelet tạo container
```

Không có luồng:

```text
Scheduler ── trực tiếp ra lệnh ──→ kubelet
```

Mà là:

```text
Scheduler ── update object ──→ API server
                                  ↑
                                  │ watch
                              kubelet
```

Tương tự, controller muốn xóa Pod cũng thường gửi request đến API server. Sau đó kubelet nhận trạng thái mới thông qua API server và thực hiện công việc trên node.

Tài liệu gọi đây là mô hình hub-and-spoke vì mọi API usage từ node hoặc Pod đều kết thúc tại API server; các thành phần control plane khác không được thiết kế để cung cấp remote API cho node truy cập. ([Kubernetes](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/ "Communication between Nodes and the Control Plane | Kubernetes"))

---

# 2. Node → Control Plane

## 2.1 Ai là bên chủ động kết nối?

Trong chiều này:

```text
kubelet / kube-proxy / Pod
            │
            │ HTTPS request hoặc watch stream
            ▼
      kube-apiserver
```

Node chủ động mở kết nối ra API server.

Trong cluster kubeadm của bạn, cổng thường là:

```text
kubelet:<ephemeral-port>  ──TCP/TLS──>  k8s-api:6443
```

Tài liệu viết “typically 443” vì các cluster managed hoặc load balancer có thể công bố API trên cổng 443. Với kubeadm, cổng mặc định trực tiếp của kube-apiserver thường là `6443`. Danh sách cổng Kubernetes xác định TCP `6443` là cổng inbound của API server và TCP `10250` là Kubelet API. ([Kubernetes](https://kubernetes.io/docs/reference/networking/ports-and-protocols/?utm_source=chatgpt.com "Ports and Protocols"))

Ví dụ thực tế:

```text
fdp-k8s-wk1:45128  ─────────────→  172.21.92.160:6443
source port tạm thời                    API VIP
```

Kubelet không nhất thiết mỗi request mở một kết nối hoàn toàn mới. Nó có thể duy trì các HTTP/TLS connection và watch stream trong thời gian dài.

---

## 2.2 Kubelet gửi gì tới API server?

Kubelet sử dụng API server cho nhiều việc:

```text
Kubelet → API server
```

Bao gồm:

- đăng ký Node;
    
- cập nhật trạng thái Node;
    
- cập nhật trạng thái Pod;
    
- cập nhật Lease heartbeat;
    
- tạo CertificateSigningRequest trong quá trình TLS bootstrap;
    
- watch các Pod đã được gán cho node;
    
- đọc ConfigMap, Secret và các object cần thiết;
    
- gửi Event;
    
- gọi TokenReview hoặc SubjectAccessReview khi kubelet được cấu hình webhook authentication/authorization.
    

Ví dụ kubelet trên `wk1` theo dõi các Pod thuộc node đó:

```text
GET /api/v1/pods?fieldSelector=spec.nodeName=fdp-k8s-wk1
Authorization: client certificate của kubelet
```

Đây thường là một **watch request** kéo dài:

```text
kubelet ── GET ...?watch=true ──→ API server

API server ── ADDED Pod A ──────→ kubelet
API server ── MODIFIED Pod A ───→ kubelet
API server ── DELETED Pod A ────→ kubelet
```

Điểm dễ nhầm là kết nối TCP ban đầu vẫn do kubelet mở tới API server, nhưng sau đó API server có thể đẩy các watch event ngược lại trên chính connection đó.

Vì thế nhìn về dữ liệu, bạn thấy API server gửi dữ liệu cho kubelet; nhưng nhìn về người **khởi tạo connection**, kubelet vẫn là phía client.

---

## 2.3 Kubelet xác minh API server như thế nào?

Kubelet cần hai nhóm thông tin:

```text
1. Tin API server nào?
   → cluster CA certificate

2. Tôi là ai?
   → client certificate + private key
```

Trong kubeconfig của kubelet thường có dạng:

```yaml
clusters:
- cluster:
    certificate-authority-data: ...
    server: https://k8s-api:6443

users:
- user:
    client-certificate: ...
    client-key: ...
```

Quá trình TLS được hiểu đơn giản như sau:

```text
Kubelet                              API server
   │                                      │
   │──── TLS ClientHello ────────────────→│
   │←── server certificate ───────────────│
   │                                      │
   │ kiểm tra certificate của API server  │
   │ bằng cluster CA                      │
   │                                      │
   │──── client certificate ─────────────→│
   │                                      │
   │                        kiểm tra kubelet
   │                        client certificate
   │←══════ encrypted HTTPS ═════════════→│
```

Hai bước bảo mật khác nhau:

- **Authentication**: kubelet chứng minh danh tính.
    
- **Authorization**: API server kiểm tra kubelet được phép làm gì.
    

Ví dụ client certificate có thể khiến kubelet được nhận diện là:

```text
User:   system:node:fdp-k8s-wk1
Group:  system:nodes
```

Sau đó Node Authorizer và admission mechanisms hạn chế kubelet chỉ truy cập tài nguyên phù hợp với node của nó.

---

## 2.4 TLS bootstrapping là gì?

Khi node mới join cluster, nó chưa có kubelet client certificate lâu dài.

Quy trình khái quát:

```text
kubeadm join
    │
    ├── cấp bootstrap token + cluster CA
    │
    ▼
kubelet dùng bootstrap token gọi API server
    │
    ├── tạo CertificateSigningRequest
    ▼
CSR được approve và ký
    │
    ▼
kubelet nhận client certificate riêng
    │
    ▼
từ đó dùng certificate để kết nối API server
```

Certificate lâu dài thường được kubelet tự xoay vòng trước khi hết hạn nếu certificate rotation được bật. Kubernetes cung cấp CertificateSigningRequest API để hỗ trợ chính quá trình khởi tạo và cấp chứng chỉ này. ([Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/?utm_source=chatgpt.com "TLS bootstrapping"))

---

# 3. Pod → API server

Một ứng dụng chạy trong Pod đôi khi cần gọi Kubernetes API.

Ví dụ:

- operator;
    
- controller tự viết;
    
- dashboard;
    
- sidecar đọc metadata;
    
- External Secrets controller;
    
- ingress controller.
    

Luồng:

```text
Pod
 │
 │ HTTPS
 ▼
Service kubernetes.default.svc
 │
 │ Service routing
 ▼
kube-apiserver:6443
```

Kubernetes tạo sẵn Service:

```bash
kubectl get svc kubernetes -n default
```

Có thể thấy dạng:

```text
NAME         TYPE        CLUSTER-IP
kubernetes   ClusterIP   10.96.0.1
```

Pod gọi:

```text
https://kubernetes.default.svc
```

IP `10.96.0.1` không phải một process thực sự đang listen trên địa chỉ đó. Nó là Service virtual IP. Cơ chế Service của node, thường do kube-proxy lập trình bằng iptables, IPVS hoặc nftables tùy cấu hình, chuyển traffic tới API server endpoint.

## Pod dùng thông tin gì để xác thực?

Thông thường Kubernetes mount một projected volume vào Pod:

```text
/var/run/secrets/kubernetes.io/serviceaccount/
├── token
├── ca.crt
└── namespace
```

Trong đó:

- `ca.crt`: xác minh certificate của API server;
    
- `token`: bearer token đại diện cho ServiceAccount;
    
- `namespace`: namespace hiện tại.
    

Request có dạng:

```http
GET /api/v1/namespaces/default/pods
Authorization: Bearer eyJhbGciOi...
```

Sau khi xác thực, RBAC quyết định ServiceAccount đó có quyền đọc Pod hay không. Kubernetes tự gán ServiceAccount `default` nếu Pod không chỉ định ServiceAccount khác; mức quyền của nó vẫn phụ thuộc vào authorization policy/RBAC. ([Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/ "Configure Service Accounts for Pods | Kubernetes"))

---

# 4. Control-plane components → API server

Các thành phần control plane cũng là API client:

```text
kube-scheduler ─────────────→ kube-apiserver
kube-controller-manager ────→ kube-apiserver
cloud-controller-manager ───→ kube-apiserver
```

Ví dụ scheduler:

```text
watch Pods chưa được schedule
watch Nodes
update Pod.spec.nodeName
```

Controller-manager:

```text
watch Deployments
watch ReplicaSets
watch Pods
create/update/delete API objects
```

Chúng không ghi trực tiếp vào etcd.

Luồng đúng:

```text
controller-manager
        │ REST API
        ▼
kube-apiserver
        │ etcd client API
        ▼
       etcd
```

Luồng sai:

```text
controller-manager ─────X────→ etcd
scheduler ──────────────X────→ kubelet
```

---

# 5. Control Plane → Node

Tài liệu chia chiều này thành hai loại:

```text
1. API server → kubelet
2. API server → node / Pod / Service thông qua proxy
```

Điều rất quan trọng:

> Trong hầu hết các trường hợp, “control plane gọi node” ở đây chính là **kube-apiserver gọi**.  
> Không phải scheduler hoặc controller-manager tự mở connection tới kubelet.

---

# 6. API server → kubelet

## 6.1 Kubelet cũng là một HTTPS server

Kubelet không chỉ là client của API server.

Nó đồng thời mở một HTTPS endpoint trên node:

```text
0.0.0.0:10250
```

Ví dụ:

```text
kube-apiserver:<ephemeral-port>
             │
             │ TCP + TLS
             ▼
fdp-k8s-wk1:10250
```

Vì vậy khi chạy:

```bash
sudo ss -tlpn | grep 10250
```

bạn có thể thấy:

```text
LISTEN 0 4096 *:10250
```

Và đồng thời có các connection:

```text
ESTAB <control-plane-ip>:xxxxx <worker-ip>:10250
```

Không có gì mâu thuẫn:

- `LISTEN`: kubelet vẫn đợi thêm connection mới.
    
- `ESTAB`: đã có một connection cụ thể được thiết lập.
    

Kubelet HTTPS API có các endpoint nhạy cảm, có thể đọc thông tin node hoặc thực hiện thao tác bên trong container, nên phải được bảo vệ bằng authentication và authorization. ([Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/ "Kubelet authentication/authorization | Kubernetes"))

---

## 6.2 Khi nào API server gọi kubelet?

Tài liệu liệt kê ba trường hợp chính:

- lấy log container;
    
- attach hoặc exec vào container;
    
- port-forward tới Pod.
    

Ngoài ra kubelet endpoint còn phục vụ một số dữ liệu như stats, metrics, Pod information và node-related APIs.

## Ví dụ `kubectl logs`

Bạn chạy:

```bash
kubectl logs nginx-abc
```

Luồng thực tế:

```text
kubectl
   │ HTTPS :6443
   ▼
kube-apiserver
   │
   │ tìm Pod đang nằm trên node nào
   │ giả sử fdp-k8s-wk1
   │
   │ HTTPS :10250
   ▼
kubelet trên wk1
   │
   │ đọc log từ container runtime / log file
   ▼
kube-apiserver
   │
   ▼
kubectl
```

Chi tiết hơn:

```text
kubectl ── request logs ──→ API server
API server ── request ────→ kubelet:10250
kubelet ── log stream ────→ API server
API server ── log stream ─→ kubectl
```

`kubectl` thường không kết nối trực tiếp tới IP Pod hoặc kubelet.

---

## 6.3 Ví dụ `kubectl exec`

```bash
kubectl exec -it nginx-abc -- sh
```

Luồng:

```text
Terminal
   │
kubectl
   │ streaming connection
   ▼
API server
   │ streaming connection
   ▼
kubelet:10250
   │ CRI request
   ▼
container runtime
   │
   ▼
process "sh" trong container
```

Dữ liệu bàn phím và màn hình đi qua chuỗi:

```text
keyboard
  → kubectl
  → API server
  → kubelet
  → container runtime
  → shell
```

Output quay lại theo hướng ngược lại.

Tùy phiên bản và cơ chế streaming, kết nối có thể dùng các giao thức nâng cấp/stream multiplexing trên HTTP, nhưng về mặt kiến trúc kube-apiserver vẫn đóng vai trò trung gian tới kubelet.

---

## 6.4 Ví dụ `kubectl port-forward`

```bash
kubectl port-forward pod/nginx-abc 8080:80
```

Luồng:

```text
Browser
   │ localhost:8080
   ▼
kubectl
   │
   │ connection đến API server :6443
   ▼
kube-apiserver
   │
   │ connection đến kubelet :10250
   ▼
kubelet
   │
   ▼
network namespace của Pod
   │
   ▼
container port 80
```

Đây không phải Service routing bình thường và cũng không phải NodePort.

Nó là một đường forwarding tạm thời được thiết lập qua:

```text
kubectl → API server → kubelet → Pod
```

Tài liệu chính thức xác nhận kết nối API server tới kubelet được sử dụng cho logs, attach và port-forwarding. ([Kubernetes](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/ "Communication between Nodes and the Control Plane | Kubernetes"))

---

# 7. TLS trong chiều API server → kubelet

Đây là phần dễ nhầm nhất.

Trong kết nối này:

```text
API server = TLS client
Kubelet    = TLS server
```

API server cần thực hiện hai việc độc lập:

```text
A. Chứng minh với kubelet rằng mình là client hợp lệ
B. Kiểm tra rằng server bên kia thật sự là kubelet hợp lệ
```

## 7.1 API server xác thực với kubelet

API server sử dụng:

```text
--kubelet-client-certificate
--kubelet-client-key
```

Trong cluster kubeadm thường là:

```text
/etc/kubernetes/pki/apiserver-kubelet-client.crt
/etc/kubernetes/pki/apiserver-kubelet-client.key
```

Kubelet nhận certificate đó và xác minh client dựa trên CA được cấu hình.

Luồng:

```text
API server                              kubelet
    │                                      │
    │──── client certificate ─────────────→│
    │                              xác minh certificate
    │                              xác định danh tính client
    │                              authorize request
```

Kubeadm tạo `apiserver-kubelet-client.crt/key` để API server kết nối an toàn tới kubelet. ([Kubernetes](https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/?utm_source=chatgpt.com "Implementation details"))

---

## 7.2 API server xác minh kubelet serving certificate

Kubelet cũng trình server certificate:

```text
API server                              kubelet
    │                                      │
    │←──── kubelet serving certificate ────│
    │                                      │
    │ kiểm tra:
    │ - ai ký certificate?
    │ - còn hạn không?
    │ - hostname/IP có khớp không?
```

Flag liên quan:

```text
--kubelet-certificate-authority=/path/to/ca.crt
```

Nó nói với API server:

> Chỉ tin serving certificate của kubelet nếu certificate đó được CA này ký.

Tài liệu bạn đưa ra cảnh báo rằng theo cách cấu hình mặc định được mô tả, API server có thể không xác minh kubelet serving certificate. Khi đó traffic vẫn mã hóa nhưng API server không chắc endpoint kia có thật sự là kubelet đúng hay không, tạo nguy cơ man-in-the-middle trên mạng không đáng tin cậy. Kubernetes khuyến nghị cấu hình `--kubelet-certificate-authority` để xác minh certificate của kubelet. ([Kubernetes](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/ "Communication between Nodes and the Control Plane | Kubernetes"))

Đây là sự khác biệt giữa:

```text
TLS mã hóa
```

và:

```text
TLS mã hóa + xác minh đúng danh tính server
```

Chỉ mã hóa mà không xác minh certificate thì kẻ đứng giữa vẫn có thể tạo một TLS endpoint giả.

---

# 8. Kubelet authentication và authorization

Kubelet API ở `10250` có thể rất quyền lực. Không nên nghĩ rằng chỉ cần firewall nội bộ là đủ.

Kubelet phải kiểm tra hai tầng:

```text
Request tới kubelet
       │
       ▼
Authentication: người gọi là ai?
       │
       ▼
Authorization: người đó được phép làm gì?
```

## Authentication

Các cách có thể gồm:

- X.509 client certificate;
    
- bearer token qua TokenReview webhook;
    
- anonymous, nếu không tắt.
    

Cấu hình quan trọng:

```yaml
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
```

## Authorization

Thường dùng:

```yaml
authorization:
  mode: Webhook
```

Khi đó kubelet hỏi API server:

```text
“Danh tính X có được gọi endpoint Y trên node này không?”
```

Thông qua `SubjectAccessReview`.

Kubernetes hiện cảnh báo rằng quyền `nodes/proxy` rất mạnh: nó có thể cho phép truy cập các kubelet APIs dùng để chạy lệnh trong container, nên không thể coi đó là quyền “chỉ đọc”. ([Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/ "Kubelet authentication/authorization | Kubernetes"))

---

# 9. API server → Node, Pod hoặc Service qua proxy

Đây là một chức năng khác với API server → kubelet.

Kubernetes API có các proxy-style endpoint, ví dụ về mặt ý tưởng:

```text
/api/v1/nodes/<node-name>/proxy/...
/api/v1/namespaces/<namespace>/pods/<pod-name>/proxy/...
/api/v1/namespaces/<namespace>/services/<service-name>/proxy/...
```

Luồng:

```text
Client
   │
   ▼
API server
   │
   ├──→ Node endpoint
   ├──→ Pod IP
   └──→ Service backend
```

Ví dụ:

```text
kubectl proxy
```

sau đó client gọi một URL API proxy. API server có thể chuyển request tiếp tới Pod hoặc Service.

## Vì sao tài liệu cảnh báo plain HTTP?

Trong kiểu proxy này, target cuối có thể chỉ cung cấp HTTP:

```text
API server ── plain HTTP ──→ Pod:8080
```

Đoạn từ client tới API server có thể là HTTPS, nhưng đoạn:

```text
API server → target
```

có thể không mã hóa và không xác thực.

Nếu URL target được chỉ rõ bằng `https`, kết nối có thể được mã hóa, nhưng theo tài liệu này API server không nhất thiết xác minh certificate của target và cũng không nhất thiết cung cấp client credential cho target. Do đó nó không đảm bảo đầy đủ danh tính và tính toàn vẹn trên một mạng không đáng tin cậy. ([Kubernetes](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/ "Communication between Nodes and the Control Plane | Kubernetes"))

Điều này **không có nghĩa mọi traffic Pod trong Kubernetes mặc định đều plain HTTP**. Nó chỉ đang nói về nhóm kết nối cụ thể:

```text
API server proxy → Node/Pod/Service target
```

Ứng dụng của bạn vẫn có thể tự sử dụng HTTPS hoặc mTLS.

---

# 10. Tại sao Node → Control Plane an toàn hơn Control Plane → Node?

## Node → API server

Thường có đầy đủ:

```text
kubelet
  ├── biết CA của API server
  ├── xác minh server certificate
  ├── có client credential
  └── dùng HTTPS
```

Do đó:

```text
Node ── authenticated encrypted HTTPS ──→ API server
```

## API server → kubelet

Cần cấu hình đúng cả hai phía:

```text
API server có client cert
kubelet xác minh client cert
API server có CA để xác minh serving cert của kubelet
```

Nếu thiếu `--kubelet-certificate-authority`, có thể xảy ra:

```text
API server ── encrypted nhưng không xác minh đúng server ──→ ?
```

## API server proxy → Pod/Service

Target thường là ứng dụng do người dùng triển khai, nên Kubernetes không thể mặc định đảm bảo:

- target có TLS hay không;
    
- certificate do CA nào ký;
    
- cần client certificate nào;
    
- ứng dụng dùng authentication gì.
    

Vì vậy tài liệu đánh giá loại đường đi này không phù hợp cho mạng public/untrusted nếu không có lớp bảo vệ bổ sung.

---

# 11. SSH tunnels trong tài liệu là gì?

Cơ chế cũ cho phép API server mở SSH connection tới từng node:

```text
kube-apiserver
      │
      │ SSH TCP 22
      ▼
sshd trên worker node
      │
      ├──→ kubelet:10250
      ├──→ Pod IP
      └──→ Service endpoint
```

Traffic control-plane-to-node được bọc bên trong SSH tunnel:

```text
API server → SSH tunnel → worker network target
```

Ưu điểm:

- không cần expose trực tiếp nhiều endpoint;
    
- traffic đi qua tunnel mã hóa;
    
- hữu ích khi control plane không route trực tiếp được tới mạng Pod.
    

Nhưng Kubernetes đã đánh dấu cách này là **deprecated** và khuyến nghị Konnectivity thay thế. ([Kubernetes](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/ "Communication between Nodes and the Control Plane | Kubernetes"))

Đây không phải SSH tunnel mà bạn tự tạo bằng:

```bash
ssh -L ...
ssh -R ...
```

cho việc quản trị thông thường. Nó là cơ chế egress của chính API server trong một kiểu triển khai Kubernetes cũ.

---

# 12. Konnectivity Service

Konnectivity giải quyết bài toán:

> Control plane cần kết nối tới node/Pod, nhưng control plane không thể hoặc không nên chủ động mở kết nối trực tiếp vào mạng worker.

Nó có hai phần:

```text
Control-plane network                 Node network

┌────────────────────┐              ┌────────────────────┐
│ Konnectivity Server│◄─────────────│Konnectivity Agent  │
└─────────▲──────────┘ persistent   └────────────────────┘
          │             connection       chạy phía node
          │
┌─────────┴──────────┐
│   kube-apiserver   │
└────────────────────┘
```

Điểm quan trọng:

```text
Agent phía node chủ động kết nối ra Server
```

chứ không phải server luôn phải kết nối vào node.

Điều này phù hợp với firewall/NAT:

```text
Node network:
- cho outbound tới Konnectivity server
- không cần mở inbound từ control plane tới mọi worker
```

## Luồng `kubectl logs` khi có Konnectivity

```text
kubectl
   │
   ▼
API server
   │
   ▼
Konnectivity server
   │ sử dụng connection mà agent đã duy trì
   ▼
Konnectivity agent phía node
   │
   ▼
kubelet:10250
```

Tức là:

```text
API server không dial trực tiếp wk1:10250
```

mà dùng:

```text
API server → Konnectivity server → agent connection → wk1:10250
```

Konnectivity hoạt động như TCP-level proxy cho traffic control-plane-to-cluster. API server được cấu hình bằng egress selector; giữa API server và Konnectivity server có thể dùng gRPC hoặc HTTP CONNECT, và có thể dùng Unix domain socket nếu hai thành phần nằm cùng host. ([Kubernetes](https://kubernetes.io/docs/tasks/extend-kubernetes/setup-konnectivity/ "Set up Konnectivity service | Kubernetes"))

## Konnectivity không phải CNI

Nó không thay Calico, Cilium hoặc Flannel.

```text
Calico:
  cung cấp Pod network và NetworkPolicy

Konnectivity:
  tạo đường proxy cho control plane kết nối vào cluster network
```

Nó cũng không thay kube-proxy cho Service routing.

---
