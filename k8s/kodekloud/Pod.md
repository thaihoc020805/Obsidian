![[Pasted image 20250801111128.png]]

At this point, we ==assume== that the ==application is already developed and built into==
==Docker Images and it is available on a Docker repository like Docker hub==, so
==kubernetes can pull it down==. We ==also assume== that the ==Kubernetes cluster has already==
==been setup and is working.==


![[Pasted image 20250801111223.png]]

As we discussed before, with kubernetes our ultimate aim is to deploy our
application in the form of containers on a set of machines that are configured as
worker nodes in a cluster.However, ==kubernetes does not deploy containers==
==directly on the worker nodes==. The ==containers are encapsulated into== a Kubernetes
object known as ==PODs==. ==A POD is a single instance of an application==. ==A POD is the==
==smallest object==, ==that you can create in kubernetes==. So ==what happens when you want==
==to scale up?==


![[Pasted image 20250801111535.png]]
Do you add more containers to the same pod? No! <mark style="background: #FF5582A6;">You create more pods</mark>. <mark style="background: #BBFABBA6;">Typically an
application instance running as a container has a 1-to-1 relationship with a Pod</mark>.<mark style="background: #BBFABBA6;"> To
create more instances of application you create more Pods</mark>. However the <mark style="background: #FFF3A3A6;">1-to-1
relationship is not a strict rule.</mark>


![[Pasted image 20250801112728.png]]

It is a <mark style="background: #FFF3A3A6;">common practice</mark> to <mark style="background: #FFF3A3A6;">have a helper container or a sidecar container along with
the main application</mark>. Such <mark style="background: #BBFABBA6;">as an agent that collects logs or monitors the application
and reports to a third party</mark>. And that's absolutely fine.

![[Pasted image 20250801112857.png]]

Let us now look at <mark style="background: #FFF3A3A6;">how to create PODs</mark>. For this we <mark style="background: #FFF3A3A6;">run</mark> the <mark style="background: #BBFABBA6;">kubectl run
command</mark>. We <mark style="background: #FFF3A3A6;">specify a name for the pod and the image to be used to create</mark> the pod. 

<mark style="background: #FFF3A3A6;">What</mark> this <mark style="background: #FFF3A3A6;">command</mark> really <mark style="background: #FFF3A3A6;">does</mark> is it<mark style="background: #FFF3A3A6;"> deploys a container by creating a POD</mark>. So it
<mark style="background: #BBFABBA6;">first creates a POD automatically and deploys an instance of the nginx docker image</mark>.

But where does it get the application image from?  For that you need to
<mark style="background: #FFF3A3A6;">specify the image name using the –-image parameter.</mark> The <mark style="background: #FFF3A3A6;">application image</mark>, in this
case the nginx image, <mark style="background: #FFF3A3A6;">is downloaded from the docker hub repository</mark>. Docker hub as
we discussed is a public repository were latest docker images of various applications
are stored. <mark style="background: #FFF3A3A6;">You could configure kubernetes to pull the image from the public docker
hub or a private repository within the organization.</mark>

Now that we have a POD created, <mark style="background: #FFF3A3A6;">how do we see the list of PODs available? </mark>
The <mark style="background: #BBFABBA6;">kubectl get PODs command</mark> helps us <mark style="background: #FFF3A3A6;">see the list of pods in our cluster.</mark> In this
case we see the pod is in a<mark style="background: #FFF3A3A6;"> ContainerCreating state</mark> and<mark style="background: #FFF3A3A6;"> soon changes to a Running
state</mark> when it is actually running.
Also remember that we haven’t really talked about the concepts on how a user can
access the nginx web server. And so in the current state we haven’t made the web
server accessible to external users. <mark style="background: #FFF3A3A6;">You can access it internally from the Node though.</mark>

For <mark style="background: #FFF3A3A6;">now we will just see how to deploy a POD </mark>and in a <mark style="background: #FFF3A3A6;">later lecture once we
learn about networking and services</mark> we will get to know <mark style="background: #FFF3A3A6;">how to make this service
accessible to end users.</mark>
Now this is called as the <mark style="background: #FFF3A3A6;">imperative way to create a POD</mark>. Let us now see the
declarative way to create a POD.


![[Pasted image 20250801114941.png]]


So that was the <mark style="background: #FFF3A3A6;">imperative way </mark>of creating an object in Kubernetes. <mark style="background: #FFF3A3A6;">You run a
command to create one object at a time</mark>. <mark style="background: #BBFABBA6;">When there are many objects and services
in your application this is not a viable option.</mark>
The more preferred approach is the <mark style="background: #BBFABBA6;">declarative way</mark>, <mark style="background: #FFF3A3A6;">where you create a YAML file
with the specifications of the object – the pod in this case</mark>. And have <mark style="background: #FFF3A3A6;">kubernetes
apply that configuration</mark>. This way <mark style="background: #FFF3A3A6;">you can define the state of your application and its
services as code and store it in source code repositories and version them</mark>. This
approach <mark style="background: #FFF3A3A6;">enables version control, CI/CD and sharing these with others and
collaborating together.</mark>

![[Pasted image 20250801115922.png]]

Now we will learn how to develop YAML files specifically for Kubernetes.<mark style="background: #FFF3A3A6;"> Kubernetes
uses YAML files as input for the creation of objects such as PODs, Replicas,
Deployments, Services etc</mark>. <mark style="background: #BBFABBA6;">All of these follow similar structure</mark>. <mark style="background: #BBFABBA6;">A kubernetes
definition file always contains 4 top level fields. The apiVersion, kind,
metadata and spec.</mark> <mark style="background: #FFF3A3A6;">T</mark><mark style="background: #FFF3A3A6;">hese are top level or root level properties. Think of them as
siblings, children of the same parent</mark>. These are all <mark style="background: #FF5582A6;">REQUIRED fields, so you MUST
have them in your configuration file.</mark>

Let us look at each one of them. The first one is the<mark style="background: #FFF3A3A6;"> apiVersion</mark>. <mark style="background: #FFF3A3A6;">This is the version of
the kubernetes API we’re using to create the object</mark>. <mark style="background: #FFF3A3A6;">Depending on what we are
trying to create we must use the RIGHT apiVersion</mark>. For now <mark style="background: #FFF3A3A6;">since we are working on
PODs, we will set the apiVersion as v1</mark>. If you are creating a service,
replicaset or deployments you will use the versions listed here. We will see what
these are later in this course.

Có hai nhóm API chính mà bạn cần biết:

1. **Nhóm API "Core"**:
    
    - Nhóm này chứa các đối tượng cơ bản và cốt lõi nhất của Kubernetes.
        
    - Các đối tượng này đã rất ổn định, vì vậy `apiVersion` của chúng luôn là **`v1`**.
        
    - **Ví dụ:** `Pod`, `Service`, `ConfigMap`, `Namespace`, `Volume`.
        
2. **Nhóm API khác**:
    
    - Nhóm này chứa các đối tượng phức tạp hơn. `apiVersion` của chúng thường có cấu trúc `ten-nhom/phien-ban` (ví dụ: `apps/v1`).
        
    - **Ví dụ:**
        
        - **`apps/v1`**: Chứa các đối tượng dùng để quản lý ứng dụng như `Deployment`, `ReplicaSet`.
            
        - **`batch/v1`**: Chứa các đối tượng dùng để chạy các tác vụ định kỳ như `Job`, `CronJob`.

Next is the <mark style="background: #FFF3A3A6;">kind</mark>. The <mark style="background: #FFF3A3A6;">kind refers to the type of object we are trying to create,</mark> which i<mark style="background: #FFF3A3A6;">n
this case happens to be a POD</mark>. So we will set it as  Pod. Some other possible
values here could be ReplicaSet or Deployment or Service, which is what you see in
the kind field in the table on the right.

![[Pasted image 20250801121129.png]]

The next is <mark style="background: #FFF3A3A6;">metadata</mark>. The metadata is <mark style="background: #FFF3A3A6;">data about the object like its name,
labels etc</mark>. As you can see unlike the <mark style="background: #FFF3A3A6;">first two where you specified a string
value</mark>, this, is in the<mark style="background: #FFF3A3A6;"> form of a dictionary</mark>. So <mark style="background: #FFF3A3A6;">everything under metadata is
intended to the right a little bit and so names and labels are children of metadata</mark>. The number of <mark style="background: #FFF3A3A6;">spaces before the two properties name and labels doesn’t
matter</mark>, but <mark style="background: #FFF3A3A6;">they should be the same as they are siblings</mark>. In this case <mark style="background: #FFF3A3A6;">labels
has more spaces on the left than name and so it is now a child of the name property
instead of a sibling</mark>. Also the <mark style="background: #FFF3A3A6;">two properties must have MORE spaces than its
parent, which is metadata,</mark> so that its intended to the right a little bit. Under metadata, the name is a string value – so you can name your POD myapp-pod - and the labels is a dictionary. So labels is a dictionary within the metadata dictionary. And it can have any key and value pairs as you wish. For now I have added a label app with the value myapp. Similarly you could add other labels as you see fit which will help you identify these objects at a later point in time.

Say for example there are 1<mark style="background: #FFF3A3A6;">00s of PODs running a front-end application, and 100’s of them
running a backend application or a database</mark>, it will be <mark style="background: #FFF3A3A6;">DIFFICULT for you to group
these PODs once they are deployed</mark>. <mark style="background: #BBFABBA6;">If you label them now as front-end, back-end or
database, you will be able to filter the PODs based on this label at a later point in
time.</mark>


<mark style="background: #FF5582A6;">It’s IMPORTANT to note that under metadata, you can only specify name or labels or
anything else that kubernetes expects to be under metadata.</mark> You <mark style="background: #FF5582A6;">CANNOT add any
other property as you wish under this</mark>. However, under labels you CAN have any kind
of key or value pairs as you see fit. So its<mark style="background: #FF5582A6;"> IMPORTANT to understand what each of
these parameters expect.</mark>


![[Pasted image 20250801122212.png]]

So far we have only mentioned the type and name of the object we need to create
which happens to be a POD with the name myapp-pod, <mark style="background: #FFF3A3A6;">but we haven’t really
specified the container or image we need in the pod</mark>. The <mark style="background: #FFF3A3A6;">last section in the
configuration file is the specification which is written as spec</mark>. <mark style="background: #BBFABBA6;">Depending on the
object we are going to create, this is where we provide additional information to
kubernetes pertaining to that object</mark>. <mark style="background: #BBFABBA6;">This is going to be different for different objects,
so its important to understand or refer to the documentation section to get the right
format for each</mark>. Since we are only creating a pod with a single container in it, it is
easy. Spec is a dictionary so add a property under it called containers,which is
a list or an array. The reason this property <mark style="background: #FFF3A3A6;">is a list is because</mark> the<mark style="background: #FFF3A3A6;"> PODs can have
multiple containers within them</mark> as we learned in the lecture earlier.<mark style="background: #FFF3A3A6;"> In this case
though, we will only add a single item in the list, since we plan to have only a
single container in the POD.</mark> The <mark style="background: #FFF3A3A6;">item in the list is a dictionary, so add a name and
image property. The value for image is nginx.</mark>

Once the file is created, run the <mark style="background: #BBFABBA6;">command kubectl create -f followed by the file name
which is pod-definition.yml and kubernetes creates the pod</mark>.
So to summarize remember the 4 top level properties. apiVersion, kind, metadata
and spec. Then start by adding values to those depending on the object you are
creating.

Khi bạn chạy lệnh `kubectl create -f filename.yml`, Kubernetes sẽ kiểm tra xem đối tượng đó đã tồn tại hay chưa dựa vào sự kết hợp của 3 thuộc tính chính:

1. **`kind`**: Loại đối tượng (ví dụ: `Pod`, `Deployment`, `Service`).
    
2. **`metadata.name`**: Tên của đối tượng (ví dụ: `my-web-app-pod`).
    
3. **`metadata.namespace`**: Tên của không gian tên (namespace) mà đối tượng đó thuộc về.
    

Kubernetes coi một đối tượng là "duy nhất" nếu sự kết hợp của 3 thuộc tính trên là duy nhất.

- **`kubectl create -f`**: Chỉ dùng để tạo mới đối tượng. Nếu đối tượng đã tồn tại, lệnh sẽ báo lỗi.
    
- **`kubectl apply -f`**: Đây là cách làm chuẩn của phương pháp khai báo. Lệnh này sẽ:
    
    - **Tạo** đối tượng nếu nó chưa tồn tại.
        
    - **Cập nhật** đối tượng nếu nó đã tồn tại và có thay đổi.


![[Pasted image 20250801122906.png]]

Once we create the pod, how do you see it? <mark style="background: #FFF3A3A6;">Use the kubectl get pods
command to see a list of pods available</mark>. In this case its just one. <mark style="background: #BBFABBA6;">To see detailed
information about the pod run the kubectl describe pod command</mark>. This <mark style="background: #FFF3A3A6;">will
tell you information about the POD, when it was created, what labels are assigned to
it, what docker containers are part of it and the events associated with that POD.</mark>

<mark style="background: #BBFABBA6;">to delete a pod, use kubectl delete pod name-pod</mark>


https://kodekloud.com/topic/labs-pods-with-yaml/


### Phân biệt các loại Labels phổ biến

- **`app`**:
    
    - **Ý nghĩa:** Đây là nhãn dùng để xác định **tên của toàn bộ ứng dụng** của bạn.
        
    - **Mục đích:** Để bạn có thể dễ dàng tìm kiếm và quản lý tất cả các tài nguyên (Pods, Services, Deployments, v.v.) thuộc cùng một ứng dụng.
        
    - **Ví dụ:** Bạn có một ứng dụng thương mại điện tử, bạn có thể gán `app: e-commerce`. Tất cả các Pods của frontend, backend, database đều có nhãn này.
        
- **`tier`**:
    
    - **Ý nghĩa:** Nhãn này dùng để xác định **tầng (layer)** logic của ứng dụng.
        
    - **Mục đích:** Rất hữu ích cho các ứng dụng có kiến trúc nhiều tầng (multi-tier).
        
    - **Ví dụ:** Với ứng dụng thương mại điện tử trên, bạn có thể phân tầng các Pod như sau:
        
        - Frontend: `tier: frontend`
            
        - Backend API: `tier: backend`
            
        - Database: `tier: database`
            
- **`type`** (hoặc `component`):
    
    - **Ý nghĩa:** Dùng để mô tả **thành phần cụ thể** hoặc **loại công việc** bên trong một tầng. Nhãn này chi tiết hơn `tier`.
        
    - **Mục đích:** Giúp bạn phân biệt các dịch vụ bên trong cùng một tầng.
        
    - **Ví dụ:**
        
        - Tầng `backend` có thể có các Pod làm nhiệm vụ khác nhau, bạn có thể phân loại chúng bằng `type`: `type: api-gateway`, `type: order-service`, `type: payment-service`.
            


```YAML
metadata:
  labels:
    app: e-commerce          # Thuộc ứng dụng "e-commerce"
    tier: frontend           # Là tầng giao diện người dùng
    component: web-ui        # Là thành phần giao diện web
    env: production          # Môi trường chạy là "production"
```

lọc
kubectl get pods -l app=e-commerce,tier=frontend



#### 1. `volumeMounts`

YAML

```
volumeMounts:
  - mountPath: /var/lib/postgresql/data
    name: db-data
```

- **`volumeMounts`**: Đây là một danh sách các volume mà bạn muốn gắn vào bên trong container. Nó được định nghĩa trong cấu hình của container.
    
- **`mountPath`**: Đường dẫn bên trong container mà volume sẽ được gắn vào. Trong trường hợp này, dữ liệu sẽ được lưu trữ tại `/var/lib/postgresql/data` bên trong container.
    
- **`name`**: Tên của volume được gắn vào. Tên này phải khớp với tên của một volume được khai báo trong phần `volumes` ở cấp độ Pod. Ở đây, nó khớp với `db-data`.
    

**Ý nghĩa:** Container sẽ thấy một thư mục tại `/var/lib/postgresql/data`. Mọi dữ liệu được ghi vào thư mục này sẽ thực sự được lưu trữ trên volume có tên là `db-data`.

#### 2. `volumes`

YAML

```
volumes:
  - name: db-data
    emptyDir: {}
```

- **`volumes`**: Đây là danh sách các volume được khai báo ở cấp độ Pod (nghĩa là nó được định nghĩa cùng cấp với `spec` của Pod, không phải trong cấu hình của container).
    
- **`name`**: Tên của volume. Tên này sẽ được sử dụng trong phần `volumeMounts` để tham chiếu và gắn vào container.
    
- **`emptyDir: {}`**: Đây là loại volume được sử dụng. `emptyDir` là một loại volume đơn giản nhất trong Kubernetes.
    

**Đặc điểm của `emptyDir`:**

- **Được tạo ra khi Pod được tạo**: Volume `emptyDir` được tạo ra khi Pod được lên lịch chạy trên một node.
    
- **Tồn tại trong suốt vòng đời của Pod**: Volume này tồn tại chừng nào Pod còn tồn tại.
    
- **Bị xóa khi Pod bị xóa**: Khi Pod bị xóa khỏi node (vì bị terminate, bị lỗi, hoặc bị xóa thủ công), dữ liệu trong `emptyDir` cũng sẽ bị xóa vĩnh viễn. Kubernetes sẽ xóa hoàn toàn thư mục này cùng với tất cả dữ liệu bên trong.
    
- **Không được chia sẻ giữa các Pod**: Mỗi Pod có `emptyDir` riêng của nó.
    
- **Lưu trữ ở đâu?**: Theo mặc định, `emptyDir` được lưu trữ trên bộ nhớ của node (thường là ổ đĩa SSD hoặc HDD của node đó). Tuy nhiên, bạn có thể cấu hình để nó được lưu trên RAM bằng cách thêm `medium: Memory` vào cấu hình.
    

### Tổng hợp và ví dụ minh họa

Trong ví dụ của bạn, có vẻ như đây là một phần của manifest cho một cơ sở dữ liệu (ví dụ: PostgreSQL, vì có đường dẫn `/var/lib/postgresql/data`).

- **Mục đích:** Để đảm bảo rằng dữ liệu của cơ sở dữ liệu được lưu trữ ở một nơi ổn định trong suốt vòng đời của Pod.
    
- **Quy trình hoạt động:**
    
    1. Kubernetes tạo một Pod.
        
    2. Khi Pod được tạo, nó cũng tạo một volume `emptyDir` có tên `db-data`.
        
    3. Container được khởi động, và volume `db-data` được gắn vào đường dẫn `/var/lib/postgresql/data`.
        
    4. Cơ sở dữ liệu (ví dụ PostgreSQL) sẽ khởi tạo và lưu trữ tất cả các file dữ liệu của nó (tablespace, log files, v.v.) vào thư mục `/var/lib/postgresql/data`.
        
    5. Nếu container bị crash và khởi động lại, volume `emptyDir` vẫn còn đó, và dữ liệu vẫn được bảo toàn.
        
    6. **Nhược điểm quan trọng:** Nếu Pod bị xóa hoàn toàn (chứ không phải chỉ container bị crash), toàn bộ dữ liệu trong volume `emptyDir` sẽ **mất vĩnh viễn**. Do đó, `emptyDir` không được khuyến khích sử dụng cho dữ liệu quan trọng và cần được duy trì lâu dài (persistent data). Thay vào đó, người ta sẽ sử dụng các loại volume khác như `PersistentVolume` và `PersistentVolumeClaim`.