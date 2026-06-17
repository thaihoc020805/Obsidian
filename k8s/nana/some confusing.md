**Controller Manager trong Kubernetes là gì?**

Controller Manager là một thành phần quan trọng của **Control Plane** (mặt phẳng điều khiển) trong Kubernetes, chạy trên master node. Bản chất, nó là một tiến trình duy nhất tổng hợp và vận hành nhiều controller khác nhau nhằm đảm bảo trạng thái hệ thống luôn đạt trạng thái mong muốn do người dùng khai báo.

## Chi tiết về Controller Manager:

- **Vai trò chính:**  
    Controller Manager liên tục **theo dõi trạng thái hiện tại của các đối tượng trong cluster** (Pod, Node, Service, ReplicaSet, Endpoints, v.v.) thông qua API Server và so sánh với trạng thái mong muốn được định nghĩa trong các resource (YAML, JSON).
    
- **Nguyên lý hoạt động:**
    
    - Thu thập dữ liệu trạng thái từ API Server.
        
    - So sánh trạng thái thực tế với trạng thái mong muốn.
        
    - Nếu phát hiện sự khác biệt, Controller Manager sẽ thực thi các hành động cần thiết để điều chỉnh, ví dụ: tạo hay xóa Pod, cập nhật Endpoint, phát hiện Node lỗi, tạo token mới cho Service Account,...
        
    - Quá trình này diễn ra liên tục, tự động duy trì trạng thái ổn định của cluster và ứng dụng.
        
- **Các loại controller chính chạy bên trong Controller Manager:**
    
    - **Node Controller:** Quản lý trạng thái các node (thêm, xóa, cảnh báo node down).
        
    - **Replication Controller (hoặc ReplicaSet Controller):** Đảm bảo số lượng Pod mong muốn được duy trì.
        
    - **Endpoints Controller:** Tạo và cập nhật danh sách endpoints cho Service, giúp các Pod kết nối lẫn nhau.
        
    - **Service Account Controller và Token Controller:** Tạo tài khoản và quản lý token API cho Pod.
        
    - Và nhiều controller khác phục vụ các mục đích khác nhau trong quản lý cluster.
        
- **Cloud Controller Manager (một thành phần tách biệt):**  
    Chạy các controller dành riêng cho tích hợp với nhà cung cấp cloud, giúp Kubernetes quản lý tài nguyên đám mây như node, cân bằng tải (load balancer) hoặc storage.
    

## Tóm tắt:

|Thành phần|Controller Manager|
|---|---|
|Bản chất|Một tiến trình trên master node chạy nhiều controller (bộ "bộ não nhỏ")|
|Chức năng chính|Giữ trạng thái cluster luôn trùng với "desired state" theo resource đã định nghĩa|
|Phương thức hoạt động|Quan sát state qua API Server, so sánh và hành động điều chỉnh nếu lệch|
|Controller chính|Node Controller, Replication Controller, Endpoints Controller, Service Account Controller, v.v.|
|Automation|Tự động tạo, xóa, sửa tài nguyên để duy trì trạng thái ổn định|
|Tách riêng cloud|Cloud Controller Manager chuyên xử lý tương tác với nhà cung cấp cloud|

Kết luận:  
Controller Manager là **trung tâm điều phối tự động và quản lý trạng thái các tài nguyên nền tảng của Kubernetes**, đảm bảo hệ thống vận hành liên tục, nhất quán và chính xác theo định nghĩa của người dùng[](https://locker.io/vi/blog/kubernetes-architecture)[](https://viblo.asia/p/k8s-basic-tong-quan-cac-thanh-phan-cua-kubernetes-aAY4ql7qVPw)[](https://bizflycloud.vn/tin-tuc/kubernetes-architecture-la-gi-nhung-dung-cua-kubernetes-architecture-trong-thuc-te-20240223104503118.htm)[](https://sunteco.vn/kien-truc-kubernetes/).

Nếu bạn cần tôi có thể cung cấp thêm tài liệu mô tả sâu về từng controller nhỏ bên trong Controller Manager.



**etcd** trong Kubernetes được gọi bất cứ khi nào có sự thay đổi hoặc truy vấn liên quan đến trạng thái tài nguyên trong cluster.

## Khi nào etcd được gọi?

- **Khi TẠO, CHỈNH SỬA, XOÁ bất kỳ resource nào:**  
    Mỗi khi bạn tạo, cập nhật, hoặc xóa các đối tượng như Pod, Deployment, Service... qua kubectl hoặc API, API Server sẽ ghi (write/update/delete) dữ liệu đó vào etcd.
    
- **Khi ĐỌC trạng thái hệ thống:**  
    Khi cần lấy trạng thái thực tế (ví dụ: kubectl get pod, get deployment), API Server đọc dữ liệu từ etcd ra để trả về client hoặc cho nội bộ các controller/các thành phần khác dùng.
    
- **Khi các thành phần Control Plane CẦN GIÁM SÁT/so sánh trạng thái:**  
    Controller Manager, Scheduler, các controller khác, đều cần biết trạng thái tài nguyên nên những thay đổi/tình trạng mới nhất đều được đọc từ etcd (thông qua API Server).
    
- **Khi SYNCHRONIZE trạng thái cluster:**  
    Nếu có sự kiện như Node bị mất, Pod bị crash, các event này được cập nhật vào etcd.
    

## etcd được gọi như thế nào?

- **Chỉ có API Server giao tiếp trực tiếp với etcd:**  
    Các thành phần khác (controller, scheduler, kubelet, kube-proxy...) đều chỉ đọc/ghi dữ liệu cluster thông qua API Server.
    
- **Còn các controller, scheduler, worker node KHÔNG truy cập trực tiếp etcd mà luôn thông qua API Server**.
    
- **Được truy cập liên tục:**  
    Các thao tác như triển khai mới, thay đổi cấu hình, cập nhật trạng thái, heartbeat, đều khiến API Server phải đọc/ghi vào etcd. Vì vậy, etcd luôn được truy cập một cách liên tục, không có “chu kỳ cố định”, mà mỗi thay đổi trong cluster đều là một lần etcd được sử dụng[](https://matthewpalmer.net/kubernetes-app-developer/articles/how-does-kubernetes-use-etcd.html)[](https://www.datadoghq.com/blog/managing-etcd-storage/)[](https://www.plural.sh/blog/etcd-kubernetes-guide/)[](https://viblo.asia/p/etcd-bo-nao-cua-kubernetes-va-cach-cai-dat-cum-etcd-cluster-high-availability-RnB5pOBJlPG).
    

## Tóm tắt

- etcd **là nơi lưu trữ dữ liệu nguồn duy nhất** (source of truth) cho tất cả tài nguyên và trạng thái cluster.
    
- **Bất kỳ thay đổi hoặc thao tác lấy trạng thái nào về resource** (Pod, Deployment, ConfigMap, Node, Secret, Service, Endpoint, v.v.) đều sẽ đi qua etcd.
    
- **API Server là cổng duy nhất gọi đến etcd**, các thành phần khác giao tiếp qua API Server[](https://matthewpalmer.net/kubernetes-app-developer/articles/how-does-kubernetes-use-etcd.html)[](https://www.ibm.com/think/topics/etcd)[](https://www.plural.sh/blog/etcd-kubernetes-guide/).
    

Nếu etcd gặp sự cố, cluster sẽ không tạo/sửa/xóa resource được, và trạng thái cluster sẽ không còn được ghi nhận chính xác.



**Quản lý rolling update trong Kubernetes do ai đảm nhiệm?**

- **Rolling update là chiến lược cập nhật phiên bản mới cho ứng dụng mà không gây downtime, bằng cách cập nhật từng Pod một cách tuần tự, giữ cho phiên bản cũ vẫn chạy cho đến khi phiên bản mới sẵn sàng.**
    
- Trong Kubernetes, **việc quản lý rolling update được thực hiện bởi đối tượng Deployment**, cụ thể là **Deployment Controller**, một controller chạy trong **Controller Manager**.
    

## Cách hoạt động rolling update:

1. **Bạn cập nhật một Deployment**, ví dụ thay đổi version image mới trong Pod template.
    
2. **Deployment Controller phát hiện thay đổi này** qua event từ API Server.
    
3. Một **ReplicaSet mới sẽ được tạo ra** để quản lý các Pod phiên bản mới (có image mới).
    
4. Deployment Controller sẽ **từ từ tăng số bản sao (replicas) cho ReplicaSet mới**, đồng thời **giảm số bản sao Pod trong ReplicaSet cũ**, theo các cấu hình rolling update như maxSurge (số Pod mới tạo vượt quá mong muốn) và maxUnavailable (số Pod cũ bị xóa trong quá trình cập nhật).
    
5. Việc này được lặp lại từng bước một, đảm bảo luôn có đủ Pod phục vụ, giảm downtime.
    
6. Khi ReplicaSet mới đủ Pod, ReplicaSet cũ sẽ được scale xuống 0 và cuối cùng bị xóa.
    

## Tóm tắt:

|Thành phần chịu trách nhiệm|Công việc chính với rolling update|
|---|---|
|Deployment (resource)|Khai báo trạng thái, cấu hình rolling update|
|Controller Manager|Chạy Deployment Controller thực hiện thuật toán rolling update|
|Deployment Controller|Tạo ReplicaSet mới, tăng giảm số lượng Pod theo rollout từng bước|
|ReplicaSet Controller|Quản lý số lượng Pod trên từng ReplicaSet|

## Các chiến lược triển khai phổ biến trong Deployment:

- **RollingUpdate:** Cập nhật dần dần từng Pod theo tuần tự, không downtime.
    
- **Recreate:** Xoá toàn bộ Pod cũ trước khi tạo Pod mới, gây downtime.
    
- Có thể kết hợp thêm Blue/Green hoặc Canary để tùy biến nâng cao.
    

**Nguồn tham khảo:**

- Kubernetes official docs, các bài hướng dẫn về Deployment và rolling update.
    
- Giới thiệu về Deployment Controller trong Controller Manager quản lý các bản cập nhật phiên bản[](https://bizflycloud.vn/tin-tuc/kubernetes-deployment-la-gi-cac-chien-luoc-trien-khai-kubernetes-deployment-20231013115021186.htm)[](https://newsletter.grokking.org/p/134-hi-u-ro-v-rolling-update-m-t-deployment-trong-kubernetes-266125)[](https://blog.cloud-ace.vn/cac-chien-luoc-trien-khai-kubernetes/)[](https://viblo.asia/p/dam-bao-kha-nang-san-sang-cao-cho-ung-dung-tren-kubernetes-high-availability-for-application-on-kubernetes-EbNVQAg1VvR)[](https://blog.vietnamlab.vn/kubernetes-best-pratice-zero-downtime-rolling-update/).




**Tác dụng của kube-proxy trong Kubernetes:**

- Kube-proxy là một thành phần chạy trên **mỗi node** trong cluster Kubernetes, chịu trách nhiệm **quản lý mạng và chuyển tiếp lưu lượng tới các Pod vui thuộc Service**.
    
- Kube-proxy giúp triển khai **abstraction Service** bằng cách duy trì các quy tắc mạng (NAT rules) để đảm bảo rằng khi một Client truy cập vào địa chỉ IP của Service (ClusterIP), request đó sẽ được kube-proxy chuyển tiếp tới một Pod backend phù hợp.
    
- Nó quản lý **load balancing** ở mức node, phân phối lưu lượng đến các Pod một cách hiệu quả, sử dụng các thuật toán như round-robin hay least connections.
    
- Kube-proxy **giám sát liên tục sự thay đổi của Service và Endpoints** qua API Server, tự động cập nhật các quy tắc chuyển tiếp khi Pod được thêm, xóa hoặc thay đổi.
    

## Cách kube-proxy hoạt động:

1. **Kết nối với API Server:** Kube-proxy lắng nghe các thay đổi về Service và Endpoints trên cluster.
    
2. **Xây dựng bảng ánh xạ:** Khi Service được tạo, kube-proxy biết được các Pod backend thông qua Endpoints, thiết lập bảng ánh xạ IP Service tới IP Pod.
    
3. **Thiết lập quy tắc mạng:** Tạo các quy tắc NAT (thường là iptables hoặc IPVS) trên node để chuyển tiếp request.
    
4. **Chuyển tiếp lưu lượng:** Khi có request đến IP Service, kube-proxy chuyển tiếp đến Pod phù hợp dựa trên các quy tắc load balancing.
    
5. **Cập nhật liên tục:** Khi Pod hoặc Service thay đổi, kube-proxy cập nhật các quy tắc chuyển tiếp tương ứng.
    

## Các chế độ vận hành kube-proxy phổ biến:

- **Userspace mode:** Xử lý chuyển tiếp trong không gian người dùng (hiệu năng thấp hơn, ít dùng).
    
- **iptables mode:** Sử dụng iptables để định tuyến trực tiếp, nhanh và hiệu quả.
    
- **IPVS mode:** Dùng Linux IP Virtual Server, hiệu năng cao và khả năng mở rộng tốt hơn iptables.
    

## Tóm lại tác dụng chính của kube-proxy:

|Tác dụng|Giải thích|
|---|---|
|Định tuyến traffic Service|Chuyển hướng traffic từ IP Service tới Pod backend phù hợp|
|Load balancing|Cân bằng tải traffic qua nhiều Pod để tránh Pod bị quá tải|
|Service Discovery|Giữ cho các Service luôn “địa chỉ cố định”, bất kể Pod thay đổi IP|
|Dynamic updates|Cập nhật quy tắc mạng khi Service hoặc Pod thay đổi|

**Kube-proxy là thành phần quan trọng đảm bảo mạng nội bộ trong Kubernetes hoạt động trơn tru, giúp các Service dễ dàng liên lạc với các Pod trong môi trường thay đổi liên tục của cluster**[](https://zesty.co/finops-glossary/kubernetes-kube-proxy/)[](https://kodekloud.com/blog/kube-proxy/)[](https://www.geeksforgeeks.org/devops/understanding-kubernetes-kube-proxy-and-its-role-in-service-networking/)[](https://seifrajhi.github.io/blog/kubernetes-networking/)[](https://dev.to/rajeshgheware/mastering-service-to-pod-communication-in-kubernetes-unveiling-the-role-of-iptables-and-kube-proxy-4jmh).




**Cách hoạt động của kubelet trong Kubernetes**

## 1. Định nghĩa

- **Kubelet** là một agent chạy trên mỗi node trong cluster Kubernetes.
    
- Nhiệm vụ chính là **đảm bảo các container thuộc Pod được tạo ra và chạy đúng theo mô tả (PodSpec)** do control plane chỉ định, đồng thời giám sát và báo cáo trạng thái Pod/container về API Server.
    

## 2. Quy trình hoạt động

## a. Nhận định nghĩa Pod

- Kubelet liên tục **theo dõi API Server để nhận các Pod được gán cho node đó** (qua trường `spec.nodeName`).
    
- Khi phát hiện Pod mới được gán cho node, kubelet lấy thông tin PodSpec (cấu hình container, volume, tài nguyên...) từ API Server.
    

## b. Tương tác với Container Runtime

- Kubelet giao tiếp với Container Runtime (như Docker, containerd) thông qua **Container Runtime Interface (CRI)**, một giao diện chuẩn để tương tác với nhiều loại runtime khác nhau.
    
- Kubelet yêu cầu runtime **kéo (pull) image container** nếu chưa có, và **khởi chạy container** theo đúng định nghĩa Pod.
    

## c. Theo dõi trạng thái Pod và container

- Kubelet giám sát quá trình chạy container: trạng thái đang chạy, lỗi, tài nguyên sử dụng.
    
- Nếu container gặp lỗi hoặc chết bất ngờ, kubelet có thể thực hiện khởi động lại theo chính sách định sẵn (restart policy).
    

## d. Báo cáo và đồng bộ trạng thái

- Định kỳ, kubelet gửi **thông tin trạng thái Pod, container** và node lên API Server để Control Plane cập nhật trạng thái cụm.
    
- Giúp Scheduler, Controller Manager biết được tình trạng thực tế của các Pod trên node.
    

## e. Quản lý các tài nguyên node

- Kubelet giám sát tài nguyên như CPU, bộ nhớ, disk, network để đảm bảo Pod sử dụng đúng quota.
    
- Kubelet cũng thực hiện các tác vụ liên quan như mount volume, cấu hình mạng cho Pod.
    

## 3. Lợi ích của kubelet

- Giúp phân tán công việc quản lý container từ Control Plane xuống từng node cụ thể.
    
- Đảm bảo rằng Pod được triển khai chính xác, hoạt động ổn định theo mô tả.
    
- Cải thiện khả năng mở rộng của cluster bằng cách cho phép nhiều node hoạt động song song và tự quản lý.
    

## 4. Tóm tắt chu trình làm việc hộp đen của kubelet

|Bước|Mô tả|
|---|---|
|Theo dõi Pod mới|Lấy thông tin Pod gán cho node từ API Server|
|Giao tiếp với Runtime|Yêu cầu container runtime pull image, tạo container|
|Giám sát trạng thái|Theo dõi container, báo lỗi hoặc restart container nếu cần|
|Cập nhật trạng thái|Gửi trạng thái Pod/kết quả chạy container về API Server|
|Quản lý tài nguyên|Giám sát sử dụng CPU, memory, volume và các ràng buộc tài nguyên khác trên node|

Kubelet là thành phần thiết yếu trên mỗi node, chịu trách nhiệm vận hành Pod/container thực tế trên node, gắn kết chặt chẽ với Control Plane để đồng bộ và duy trì trạng thái hệ thống[](https://viblo.asia/p/tong-quan-ve-kien-truc-cua-kubernetes-E375zVpq5GW)[](https://viblo.asia/p/k8s-basic-tong-quan-cac-thanh-phan-cua-kubernetes-aAY4ql7qVPw)[](https://sunteco.vn/huong-dan-trien-khai-kubernetes/)[](https://viblo.asia/p/tong-quan-ve-kien-truc-cua-kubernetes-E375zVpq5GW).