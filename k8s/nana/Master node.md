![[Pasted image 20250804115424.png]]
According to the text you provided, master nodes in a Kubernetes cluster run four completely different processes to control the cluster's state. 

![[Pasted image 20250804115541.png]]
The first is the API server, which acts as the cluster's gateway and a gatekeeper for authentication. Users interact with it via clients like a UI or command-line tools to schedule new pods, deploy applications, create services, or query the cluster's status. After validating a request, the API server forwards it to the other master processes. 
![[Pasted image 20250804115748.png]]
The second process is the scheduler. When the API server receives a request to schedule a new pod, it hands it over to the scheduler. The scheduler intelligently decides which worker node should host the pod by analyzing the application's resource requirements (CPU, RAM) and the available resources on each worker node, scheduling the pod on the least busy node. It's important to note that the scheduler only decides on the node; the actual task of starting the pod is performed by kubelet on that specific node. 
![[Pasted image 20250804115932.png]]
The third component is the controller manager, a crucial process that detects state changes, such as when a pod dies. 
![[Pasted image 20250804120015.png]]
It then initiates recovery by requesting the scheduler to reschedule the dead pods, which in turn instructs the corresponding kubelets to restart them. 
![[Pasted image 20250804120328.png]]
Finally, the last master process is etcd, a key-value store that functions as the cluster's "brain." It stores every state change, such as when a new pod is scheduled or a pod dies. This data is essential for the other master processes to function, as the scheduler, controller manager, and API server all rely on information from etcd to know about resource availability, state changes, or cluster health. The text clarifies that etcd does not store application data, such as a database's content, but only cluster state information. Given their critical role, master processes and the etcd store must be reliable. Therefore, in practice, a Kubernetes cluster typically consists of multiple master nodes, with a load-balanced API server and a distributed etcd store across all of them. 
![[Pasted image 20250804120607.png]]The text concludes by noting that master processes, though more important, require fewer hardware resources than worker nodes, which handle the actual work of running containers and therefore need more CPU, RAM, and storage. A Kubernetes cluster can be easily scaled by adding new master or worker nodes as application demands increase.

Nếu Worker Node là công trường, thì Control Plane chính là **văn phòng điều hành trung tâm**. Mọi quyết định toàn cục, mọi bản thiết kế đều được lưu trữ và xử lý tại đây.

---

### 1. etcd

Đây là thành phần nền tảng và quan trọng nhất của Control Plane. Hãy coi nó như là "tủ hồ sơ" hoặc cơ sở dữ liệu duy nhất chứa đựng toàn bộ sự thật về cluster.

#### Tác dụng chính: **Kho lưu trữ trạng thái của Cluster**

- **Lưu trữ toàn bộ cấu hình:** `etcd` là một hệ thống lưu trữ dạng key-value phân tán, có độ tin cậy cao. Nó lưu trữ tất cả thông tin về trạng thái của cluster, bao gồm:
    
    - **Trạng thái mong muốn (Desired State):** Bạn muốn chạy bao nhiêu bản sao của một ứng dụng? Cấu hình của Deployment, Service, Secret, ConfigMap... là gì?
        
    - **Trạng thái hiện tại (Current State):** Pod nào đang chạy ở Node nào? Sức khỏe của các Node ra sao?
        
- **Nguồn sự thật duy nhất (Single Source of Truth):** Tất cả các thành phần khác trong Control Plane đều đọc và ghi thông tin vào `etcd` (thông qua API Server). Điều này đảm bảo tính nhất quán và đồng bộ trên toàn bộ cluster.
    
- **Đáng tin cậy và nhất quán:** Được thiết kế để có tính sẵn sàng cao, `etcd` thường được triển khai dưới dạng một cụm (cluster) riêng để đảm bảo rằng nếu một máy `etcd` bị lỗi, dữ liệu của cluster vẫn an toàn.
    

**Tóm lại:** Nếu `etcd` bị mất dữ liệu, toàn bộ cluster của bạn sẽ mất trạng thái và không thể phục hồi. Nó là trái tim lưu trữ thông tin của Kubernetes.

---

### 2. API Server (`kube-apiserver`)

API Server là cửa ngõ duy nhất để tương tác với cluster Kubernetes. Mọi yêu cầu, dù từ người dùng hay từ các thành phần khác của hệ thống, đều phải đi qua nó.

#### Tác dụng chính: **Cổng giao tiếp và xác thực trung tâm**

- **Phơi bày Kubernetes API:** Nó cung cấp một giao diện API RESTful (qua HTTP) cho phép người dùng và các thành phần khác thực hiện các thao tác CRUD (Create, Read, Update, Delete) trên các đối tượng của Kubernetes (như Pods, Services, Deployments).
    
    - Lệnh `kubectl` bạn gõ trên máy tính thực chất là đang gửi yêu cầu API đến `kube-apiserver`.
        
- **Cổng vào duy nhất đến `etcd`:** Đây là thành phần **duy nhất** được phép nói chuyện trực tiếp với `etcd`. Nó hoạt động như một người gác cổng, đảm bảo mọi dữ liệu đọc/ghi vào `etcd` đều hợp lệ và được xác thực.
    
- **Xác thực và Ủy quyền (Authentication & Authorization):**
    
    - **Xác thực:** Kiểm tra xem người/tiến trình gửi yêu cầu là ai (ví dụ: dùng token, certificate).
        
    - **Ủy quyền:** Kiểm tra xem người/tiến trình đó có quyền thực hiện hành động được yêu cầu hay không (ví dụ: user "dev" có quyền tạo Pod trong namespace "test" không?).
        
- **Admission Control:** Sau khi xác thực và ủy quyền, API Server có thể áp dụng các bộ điều khiển "nhập học" (Admission Controllers) để sửa đổi hoặc từ chối yêu cầu trước khi lưu vào `etcd`. Ví dụ: tự động thêm một label cho mọi Pod được tạo.
    

**Tóm lại:** API Server là "lễ tân" và "bảo vệ" của văn phòng chỉ huy. Mọi người phải đi qua nó, trình giấy tờ, và nó là người duy nhất có chìa khóa của tủ hồ sơ `etcd`.

---

### 3. Scheduler (`kube-scheduler`)

Scheduler có một nhiệm vụ duy nhất và rất chuyên biệt: quyết định xem một Pod mới tạo nên được chạy trên Worker Node nào.

#### Tác dụng chính: **Người phân bổ công việc (Matchmaker)**

- **Theo dõi Pod chưa được phân công:** Scheduler liên tục theo dõi API Server để tìm các Pod vừa được tạo nhưng chưa được gán cho một Node nào (trường `nodeName` trong PodSpec còn trống).
    
- **Quá trình Lọc (Filtering):** Đối với mỗi Pod cần lên lịch, Scheduler sẽ lọc ra danh sách các Node _không_ phù hợp. Một Node có thể bị loại vì:
    
    - Không đủ tài nguyên (CPU, RAM).
        
    - Không thỏa mãn các yêu cầu về `nodeSelector`, `affinity`.
        
    - Node có một `taint` (vết bẩn) mà Pod không `tolerate` (chịu đựng) được.
        
- **Quá trình Chấm điểm (Scoring):** Sau khi có danh sách các Node phù hợp, Scheduler sẽ chấm điểm cho từng Node để tìm ra Node _tốt nhất_. Các tiêu chí chấm điểm có thể là:
    
    - Ưu tiên Node đang có ít Pod chạy nhất.
        
    - Ưu tiên Node đã có sẵn container image cần thiết.
        
    - Các quy tắc do người dùng định nghĩa.
        
- **Gán Node (Binding):** Sau khi chọn được Node tốt nhất, Scheduler sẽ không tự mình chạy Pod. Thay vào đó, nó gửi một yêu cầu ngược lại cho **API Server** để cập nhật đối tượng Pod, điền tên của Node đã chọn vào trường `nodeName`.
    

**Tóm lại:** Scheduler là một "nhà hoạch định tài nguyên" thông minh. Nó không thực thi công việc mà chỉ ra quyết định "công việc này nên được làm ở đâu là tối ưu nhất".

---

### 4. Controller Manager (`kube-controller-manager`)

Đây là một tiến trình "tổng hợp" chạy nhiều vòng lặp điều khiển (controller loops) khác nhau. Mỗi vòng lặp là một "bộ máy tự động hóa" chịu trách nhiệm đưa trạng thái _hiện tại_ của cluster về trạng thái _mong muốn_.

#### Tác dụng chính: **Người giám sát và tự động sửa lỗi**

Controller Manager hoạt động giống như một chiếc máy điều nhiệt. Nó liên tục so sánh trạng thái mong muốn (được lưu trong `etcd`) với trạng thái thực tế. Nếu có sự khác biệt, nó sẽ hành động để khắc phục.

Một số controller quan trọng chạy bên trong `kube-controller-manager`:

- **Replication Controller / ReplicaSet Controller:** Đảm bảo số lượng Pod của một ReplicaSet luôn đúng như khai báo. Nếu một Pod bị chết, controller này sẽ phát hiện ra và yêu cầu API Server tạo một Pod mới để thay thế.
    
- **Node Controller:** Theo dõi sức khỏe của các Worker Node. Nếu một Node không gửi tín hiệu về trong một khoảng thời gian, controller này sẽ đánh dấu Node đó là `NotReady` và có thể sẽ di tản các Pod đang chạy trên đó sang Node khác.
    
- **Service Controller:** Tương tác với cơ sở hạ tầng của nhà cung cấp đám mây (nếu có) để tạo các tài nguyên như Load Balancer khi bạn tạo một Service loại `LoadBalancer`.
    
- **Endpoint Controller:** Cập nhật đối tượng Endpoint với danh sách IP của các Pod khỏe mạnh tương ứng với một Service.
    

**Tóm lại:** Controller Manager là một "đội ngũ quản lý dự án" cần mẫn. Họ liên tục đi kiểm tra, so sánh bản thiết kế (`desired state`) với thực tế công trường (`current state`) và tự động nộp yêu cầu (qua API Server) để sửa chữa mọi sai lệch.


Luồng hoạt động tổng thể khi tạo một Deployment mà bạn mô tả là **chính xác** về mặt nguyên lý, đúng với kiến trúc và hành vi của Kubernetes hiện nay. Cụ thể các bước sau đây đều khớp với tài liệu chính thức và các hướng dẫn hiện đại về cách tạo Deployment:

- **Chạy lệnh `kubectl apply -f my-deployment.yaml`:** Lệnh này gửi file YAML khai báo Deployment lên API Server của Kubernetes[](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)[](https://octopus.com/devops/kubernetes-deployments/)[](https://kubernetes.io/docs/tutorials/kubernetes-basics/deploy-app/deploy-intro/).
    
- **API Server xác thực, ghi vào etcd:** API Server sẽ tiến hành xác thực, ủy quyền và lưu đối tượng Deployment mới vào hệ thống lưu trữ vĩnh viễn etcd. Đây là bước chuẩn hóa và bảo đảm nhất quán trạng thái cụm[](https://octopus.com/devops/kubernetes-deployments/).
    
- **Controller Manager/Deployment Controller phát hiện Deployment mới:** Controller này sẽ đọc trạng thái mới và tạo ReplicaSet dựa vào thông tin được khai báo trong Deployment. Nếu bạn khai báo 3 replicas thì ReplicaSet cũng sẽ yêu cầu 3 Pod[](https://codefresh.io/learn/kubernetes-deployment/)[](https://www.qovery.com/blog/what-is-kubernetes-deployment-guide-how-to-use/)[](https://octopus.com/devops/kubernetes-deployments/)[](https://viblo.asia/p/kubernetes-series-bai-5-deployment-cap-nhat-ung-dung-bang-deployment-RQqKL6q0l7z).
    
- **ReplicaSet Controller đảm bảo số lượng Pod phù hợp:** Nếu chưa có Pod nào, nó sẽ yêu cầu API Server tạo đủ 3 đối tượng Pod[](https://octopus.com/devops/kubernetes-deployments/)[](https://viblo.asia/p/kubernetes-series-bai-5-deployment-cap-nhat-ung-dung-bang-deployment-RQqKL6q0l7z).
    
- **API Server lưu các Pod vào etcd:** Các đối tượng Pod này khi mới tạo sẽ chưa được gán Node cụ thể để chạy thật[](https://octopus.com/devops/kubernetes-deployments/).
    
- **Scheduler phát hiện các Pod “mồ côi”:** Scheduler chọn Node phù hợp cho từng Pod (dựa theo tài nguyên, nodeSelector, affinity, taint/tolerations,…) và cập nhật thông tin Node của các Pod đó trong etcd[](https://octopus.com/devops/kubernetes-deployments/)[](https://kubernetes.io/docs/concepts/architecture/).
    
- **Kubelet tại các Node chạy Pod được giao:** Kubelet trên từng node sẽ nhận thấy các Pod mới gán cho mình, tiến hành kéo image và thực thi container[](https://octopus.com/devops/kubernetes-deployments/).
    
- **Kube-proxy cấu hình mạng cho Service (nếu có):** Nếu có Service liên kết với các Pod này, kube-proxy trên các node sẽ cập nhật lại các quy tắc mạng để route traffic tới các Pod mới tạo[](https://octopus.com/devops/kubernetes-deployments/).




**Cloud Controller Manager (CCM)** là một thành phần trong control plane của Kubernetes, đóng vai trò làm cầu nối giữa Kubernetes cluster và các dịch vụ của cloud provider như AWS, Google Cloud, Azure... Nó giúp Kubernetes có thể tự động quản lý, tích hợp các tài nguyên cloud bên ngoài một cách linh hoạt.

## Cách hoạt động của Cloud Controller Manager

## 1. **Cầu nối với Cloud Provider**

- CCM chứa các "controller" chuyên biệt tương tác với API của cloud provider.
    
- Khi bạn tạo, sửa, xóa các resource trong cluster (ví dụ: Service kiểu LoadBalancer, PersistentVolume...), CCM sẽ gọi API của cloud để tự động tạo/điều chỉnh tài nguyên ngoài đời thực (load balancer, IP, storage, routes, v.v.) phù hợp với state mong muốn của cluster[](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)[](https://blog.techiescamp.com/docs/cloud-controller-manager/)[](https://www.geeksforgeeks.org/devops/kubernetes-cloud-controller-manager/).
    

## 2. **Các controller chính bên trong CCM**

- **Node Controller:** Theo dõi, đồng bộ các node trong cloud với các Node object trong cluster, kiểm tra tình trạng node (còn sống, bị xóa, hay bị lỗi) bằng cách hỏi API của cloud.
    
- **Route Controller:** Cấu hình các route ở tầng mạng trên cloud, giúp các Pod thuộc các node khác nhau giao tiếp với nhau dễ dàng (cần cho overlay network cloud).
    
- **Service Controller:** Tự động tạo/cập nhật các dịch vụ như load balancer, public IP, và các điều khiển mạng khi bạn tạo Service kiểu LoadBalancer trong Kubernetes.
    

## 3. **Hoạt động thực tế (ví dụ tạo Service kiểu LoadBalancer)**

- Bạn tạo một Service mới và khai báo `type: LoadBalancer`.
    
- CCM nhận biết sự kiện này, sử dụng API cloud provider để tự động tạo một cloud load balancer/public IP ngoài cloud thực tế.
    
- Khi cloud load balancer được tạo xong, CCM cập nhật lại object Service trong cluster với thông tin `EXTERNAL-IP` để bạn sử dụng trực tiếp[](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)[](https://blog.techiescamp.com/docs/cloud-controller-manager/).
    
- Nếu bạn xóa Service, CCM cũng đảm nhận việc gỡ bỏ cả tài nguyên ngoài cloud tương ứng.
    

## 4. **Tách biệt logic cloud khỏi core Kubernetes**

- Tất cả logic về cloud (AWS, GCP, Azure...) đều được nhét vào CCM riêng biệt, giúp Kubernetes giữ tính "cloud-agnostic" (không ràng buộc vào nền tảng nào).
    
- CCM triển khai dưới dạng các plugin riêng cho từng nhà cung cấp dịch vụ cloud, dể dàng nâng cấp/tùy biến mà không ảnh hưởng đến logic điều phối cluster lõi.
    

## **Tóm lại**

- **CCM là “cầu nối thông minh” giữa Kubernetes và cloud provider.**
    
- Khi cluster cần tài nguyên cloud, CCM sẽ tự động gọi API cloud provider để khởi tạo, quản lý các tài nguyên tương ứng.
    
- CCM giúp Kubernetes vận hành đồng bộ, tự động hóa, không lo phụ thuộc vào bất kỳ cloud nào và dễ mở rộng cho cloud mới[](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)[](https://blog.techiescamp.com/docs/cloud-controller-manager/)[](https://www.geeksforgeeks.org/devops/kubernetes-cloud-controller-manager/).
    

**Không có Cloud Controller Manager, cluster của bạn trên cloud sẽ không tận dụng được các tính năng tạo load balancer, storage động, hoặc tự động cập nhật trạng thái node từ cloud!**


## Nếu cài **trên dịch vụ cloud**:

- Bạn sẽ tận dụng được các tiện ích như:
    
    - Service kiểu LoadBalancer sẽ tự động tạo IP/public cloud load balancer.
        
    - Volume, storage (EBS, GCE Persistent Disk...) được cấp phát động khi khai báo PVC (Persistent Volume Claim).
        
    - Node status, network route tự động sync.
        
- Những tính năng đó dùng **Cloud Controller Manager** để giao tiếp với cloud provider.
    

## Nếu bạn cài **local/bare-metal (không cloud)**:

- Cluster vẫn chạy bình thường, đầy đủ core tính năng Kubernetes: pod, deployment, service, ingress, statefulset, job, node...
    
- Nhưng các thứ như LoadBalancer service, dynamic storage provisioning, tự cấp IP public... sẽ **không dùng được "tự động"** vì không có cloud controller.
    
    - Service kiểu LoadBalancer sẽ luôn ở trạng thái `<pending>`, trừ khi bạn cài thêm phần mềm bổ trợ như MetalLB để mô phỏng load balancer ở môi trường không cloud.
        
    - Storage phải tự provision, mount thủ công hoặc dùng các add-on như Longhorn, Rook...