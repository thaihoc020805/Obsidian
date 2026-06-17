 ![[Pasted image 20250804114044.png]]
In a Kubernetes cluster, every node must have a container runtime installed to execute containers inside application pods. This could be Docker or some other technology. 
![[Pasted image 20250804114109.png]]
The process that actually schedules these pods and the containers underneath is kubelet, a Kubernetes process. Unlike the container runtime, kubelet has an interface with both the container runtime and the machine itself, as it is responsible for taking the pod's configuration, starting the pod with a container inside, and assigning resources like CPU, RAM, and storage from the node to the container. 


![[Pasted image 20250804114133.png]]

A Kubernetes cluster is typically made up of multiple nodes, which all must have container runtime and kubelet services installed. These nodes can run hundreds of other pods and their replicas. Communication between these pods is handled using Services, which act as a load balancer to match requests directed to a specific application and forward them to the respective pods. The third process responsible for forwarding requests from services to pods is Cube proxy, which also must be installed on every node. Cube proxy has intelligent forwarding logic to ensure performant communication with low overhead. For example, if a replica of an application pod makes a request to a database, Cube proxy will forward the request to the database replica running on the same node to avoid network overhead. To summarize, two Kubernetes processes, kubelet and Cube proxy, along with an independent container runtime, must be installed on every Kubernetes worker node for the cluster to function.


Chào bạn, đây là giải thích chi tiết về tác dụng của 3 tiến trình quan trọng trên mỗi Worker Node trong Kubernetes: **Container Runtime**, **Kubelet**, và **Kube-proxy**.

Để dễ hình dung, hãy tưởng tượng Worker Node như một công trường xây dựng:

- **Control Plane (máy chủ Master):** Là văn phòng chỉ huy, nơi có kiến trúc sư và quản lý dự án, giữ các bản thiết kế (khai báo YAML).
    
- **Kubelet:** Là người quản đốc tại công trường. Anh ta nhận bản thiết kế từ văn phòng chỉ huy và chỉ đạo công nhân.
    
- **Container Runtime:** Là đội ngũ công nhân (thợ xây, thợ điện...). Họ trực tiếp thực hiện công việc (chạy container) theo lệnh của quản đốc.
    
- **Kube-proxy:** Là người điều phối giao thông và logistics. Anh ta đảm bảo đường đi lối lại (mạng) thông suốt để các khu vực khác nhau có thể giao tiếp với nhau.
    

Bây giờ, hãy đi vào chi tiết từng thành phần:

---

### 1. Container Runtime (Ví dụ: containerd, CRI-O)

Đây là thành phần nền tảng nhất, chịu trách nhiệm thực thi và quản lý vòng đời của các container. Kubernetes không tự mình chạy container mà nó ra lệnh cho Container Runtime làm việc đó.

#### Tác dụng chính: **Công nhân thực thi**

- **Kéo (Pull) Container Image:** Tải các image (khuôn mẫu của ứng dụng) từ một registry (như Docker Hub, Google Container Registry) về Worker Node.
    
- **Tạo và Chạy Container:** Dựa trên image đã được kéo về và các cấu hình do Kubelet cung cấp (ví dụ: tài nguyên CPU/RAM, volume lưu trữ, biến môi trường), nó sẽ tạo ra và khởi chạy các container.
    
- **Dừng và Xóa Container:** Dừng hoạt động và xóa container khi được Kubelet yêu cầu (ví dụ: khi một Pod bị xóa).
    
- **Quản lý tài nguyên cấp thấp:** Nó trực tiếp tương tác với kernel của hệ điều hành để cấp phát và giới hạn tài nguyên (CPU, memory) cho từng container, cũng như thiết lập môi trường biệt lập (namespaces, cgroups).
    

**Tóm lại:** Nếu không có Container Runtime, Worker Node chỉ là một máy chủ bình thường, không thể chạy được bất kỳ container nào. Nó là "đôi tay" thực hiện công việc vật lý.

---

### 2. Kubelet

Kubelet là "đại lý" (agent) chính của Kubernetes chạy trên mỗi Worker Node. Nó đóng vai trò là cầu nối liên lạc giữa Worker Node và Control Plane (cụ thể là API Server).

#### Tác dụng chính: **Quản đốc tại chỗ**

- **Đăng ký Node với Cluster:** Khi khởi động, Kubelet tự đăng ký Worker Node của mình với API Server trong Control Plane, làm cho Node đó trở thành một phần của cluster Kubernetes.
    
- **Nhận và diễn giải PodSpec:** Nó liên tục "lắng nghe" các chỉ thị từ API Server. Khi một Pod được lên lịch (schedule) để chạy trên Node của nó, Kubelet sẽ nhận được bản đặc tả của Pod đó (gọi là `PodSpec`).
    
- **Ra lệnh cho Container Runtime:** Kubelet sẽ đọc `PodSpec` và dịch nó thành các lệnh cụ thể cho **Container Runtime**. Ví dụ: "Này Container Runtime, hãy kéo image `nginx:1.21` và chạy một container từ image đó, gắn cho nó volume tên là `web-data`."
    
- **Quản lý Vòng đời Pod:** Kubelet đảm bảo rằng các container trong Pod mà nó quản lý luôn ở trạng thái mong muốn (desired state). Nếu một container bị chết, Kubelet sẽ cố gắng khởi động lại nó dựa trên chính sách khởi động lại (`restartPolicy`) của Pod.
    
- **Báo cáo Trạng thái:** Kubelet thường xuyên gửi "báo cáo sức khỏe" về Control Plane, bao gồm:
    
    - **Trạng thái của Node:** Node có sẵn sàng không, có bị cạn kiệt tài nguyên (bộ nhớ, đĩa) không.
        
    - **Trạng thái của Pod:** Các Pod trên Node đang ở trạng thái `Running`, `Pending`, `Succeeded` hay `Failed`.
        
- **Thực thi các Probe:** Nó chịu trách nhiệm thực thi các kiểm tra sức khỏe (`livenessProbe`, `readinessProbe`, `startupProbe`) được định nghĩa trong Pod để xác định xem container có đang hoạt động đúng cách hay không.
    

**Tóm lại:** Kubelet là "bộ não" của Worker Node. Nó không tự chạy container nhưng nó biết cần phải chạy container nào, chạy ra sao, và đảm bảo chúng luôn hoạt động theo đúng "bản thiết kế" từ Control Plane.

---

### 3. Kube-proxy

Kube-proxy là một tiến trình chạy trên mỗi Worker Node, chịu trách nhiệm về các khía cạnh mạng trong Kubernetes, cụ thể là triển khai khái niệm **Service**.

#### Tác dụng chính: **Điều phối mạng (Network Plumber)**

- **Triển khai Service Abstraction:** Trong Kubernetes, Pod có thể bị hủy và tạo lại bất cứ lúc nào, dẫn đến địa chỉ IP của chúng thay đổi liên tục. **Service** cung cấp một địa chỉ IP ảo, ổn định (gọi là `ClusterIP`) và một DNS name cố định để truy cập vào một nhóm các Pod. Kube-proxy chính là thành phần biến khái niệm trừu tượng này thành hiện thực.
    
- **Theo dõi sự thay đổi của Service và Endpoint:** Kube-proxy "theo dõi" API Server để phát hiện bất kỳ thay đổi nào đối với các đối tượng Service và Endpoint. (Endpoint là danh sách các cặp `IP:Port` thực tế của các Pod đang khỏe mạnh thuộc về một Service).
    
- **Quản lý Quy tắc Mạng:** Khi có sự thay đổi, Kube-proxy sẽ cập nhật các quy tắc mạng trên chính hệ điều hành của Worker Node để điều hướng lưu lượng truy cập. Cụ thể:
    
    - Nó sẽ tạo ra các quy tắc để bất kỳ gói tin nào được gửi đến địa chỉ `ClusterIP:Port` của một Service sẽ được chặn lại.
        
    - Sau đó, nó sẽ chuyển tiếp (forward) gói tin đó đến một trong các địa chỉ `PodIP:Port` thực tế của một Pod khỏe mạnh thuộc Service đó. Quá trình này cũng bao gồm việc cân bằng tải (load balancing) đơn giản.
        
- **Các chế độ hoạt động:** Kube-proxy có thể hoạt động ở nhiều chế độ khác nhau để quản lý các quy tắc mạng này, phổ biến nhất là:
    
    - **iptables:** Chế độ mặc định và tương thích rộng rãi. Nó tạo ra các quy tắc `iptables` trong kernel Linux để thực hiện việc chuyển hướng và NAT.
        
    - **IPVS (IP Virtual Server):** Một chế độ hiện đại hơn, được thiết kế để có hiệu năng cao hơn `iptables`, đặc biệt là trong các cluster có số lượng Service rất lớn.
        

**Tóm lại:** Kube-proxy không quản lý Pod hay container. Vai trò của nó là đảm bảo rằng kết nối mạng _đến_ các Pod (thông qua Service) luôn hoạt động chính xác và ổn định, bất kể các Pod đó có bị di chuyển hay khởi động lại hay không.