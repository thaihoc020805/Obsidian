

```text
Control plane:
fdp-k8s-cp1

Worker:
fdp-k8s-wk1
fdp-k8s-wk2
fdp-k8s-wk3
```

---

# 1. Node là gì?

Hiểu đơn giản:

```text
Kubernetes cluster
├── Control plane: bộ não, quản lý cluster
└── Nodes: các máy thực sự chạy ứng dụng
```

Một **Node** có thể là:

- máy vật lý;
    
- máy ảo;
    
- cloud VM;
    
- thậm chí máy laptop trong môi trường học tập.
    

Ví dụ:

```text
fdp-k8s-wk1
CPU: 4 core
RAM: 12 GB
IP: 172.21.92.175
```

Đây là một máy Linux thật hoặc VM.

Nhưng Kubernetes không chỉ nhìn nó như một máy Linux. Trong Kubernetes còn tồn tại một object:

```yaml
kind: Node
metadata:
  name: fdp-k8s-wk1
```

Phải phân biệt:

```text
Máy Linux thật                  Node object trong Kubernetes
─────────────────────────────────────────────────────────────
Có CPU, RAM, disk               Bản ghi đại diện cho máy đó
Chạy kubelet                    Được lưu trong API server/etcd
Có IP 172.21.92.175             Có status, labels, taints...
Có containerd                   Scheduler đọc object này
```

Khi bạn chạy:

```bash
kubectl get nodes
```

Bạn đang xem **Node objects**, không phải trực tiếp SSH vào các máy.

---

# 2. Trên một Node có những thành phần nào?

Tài liệu nhắc ba thành phần chính:

```text
Node
├── kubelet
├── container runtime
└── kube-proxy
```

## 2.1. Kubelet

Kubelet là **agent của Kubernetes chạy trên từng Node**.

Ví dụ trên `fdp-k8s-wk1`:

```bash
systemctl status kubelet
```

Nhiệm vụ của kubelet:

- kết nối tới API server;
    
- đăng ký Node;
    
- theo dõi Pod nào được gán cho Node;
    
- yêu cầu container runtime tạo container;
    
- kiểm tra container còn sống không;
    
- gửi trạng thái Node và Pod về API server;
    
- mount volume;
    
- chạy readiness probe, liveness probe, startup probe.
    

Có thể hiểu:

```text
API server:
"wk1, hãy chạy Pod nginx này."

kubelet trên wk1:
"Được, tôi sẽ nhờ containerd kéo image và tạo container."
```

Kubelet không tự quyết định Pod nào chạy ở Node mình. **Scheduler quyết định**, kubelet chỉ thi hành.

---

## 2.2. Container runtime

Đây là phần thực sự chạy container.

Cluster của bạn dùng:

```text
containerd
```

Luồng cơ bản:

```text
kubelet
   │ CRI
   ▼
containerd
   │
   ▼
runc / Linux namespaces / cgroups
   │
   ▼
Container
```

Kubelet không trực tiếp gọi:

```bash
docker run ...
```

Nó giao tiếp với containerd thông qua chuẩn **CRI — Container Runtime Interface**.

---

## 2.3. Kube-proxy

Kube-proxy phụ trách phần mạng liên quan đến **Service**.

Ví dụ:

```yaml
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
```

Kube-proxy theo dõi Service và EndpointSlice rồi tạo rule mạng bằng:

- iptables;
    
- hoặc IPVS;
    
- tùy cấu hình cluster.
    

Luồng có thể là:

```text
Client Pod
   │
   ▼
Service IP: 10.96.20.10
   │ kube-proxy rules
   ▼
Pod backend: 192.168.10.23
```

CNI như Calico và kube-proxy không hoàn toàn giống nhau:

```text
Calico:
- cấp mạng cho Pod
- route Pod-to-Pod
- NetworkPolicy
- VXLAN trong cluster của bạn

kube-proxy:
- xử lý Service IP
- chuyển Service traffic đến backend Pod
```

---

# 3. Node tham gia cluster như thế nào?

Tài liệu nói có hai cách:

1. kubelet tự đăng ký;
    
2. người quản trị tự tạo Node object.
    

Trong thực tế, cách 1 gần như luôn được dùng.

---

# 4. Self-registration: kubelet tự đăng ký Node

Trong cluster kubeadm của bạn, khi chạy trên worker:

```bash
kubeadm join ...
```

Kubeadm sẽ cấu hình kubelet để kubelet có thể kết nối API server.

Sau đó kubelet gửi request đại loại:

```http
POST /api/v1/nodes
```

Nội dung logic:

```yaml
kind: Node
metadata:
  name: fdp-k8s-wk1
status:
  addresses:
    - type: InternalIP
      address: 172.21.92.175
  capacity:
    cpu: "4"
    memory: ...
```

Luồng đầy đủ:

```text
kubelet khởi động trên wk1
        │
        │ dùng kubeconfig + certificate
        ▼
kết nối kube-apiserver
        │
        │ đăng ký Node fdp-k8s-wk1
        ▼
API server xác thực và phân quyền
        │
        ▼
ghi Node object vào etcd
        │
        ▼
kubectl get nodes nhìn thấy wk1
```

---

# 5. Tại sao chỉ tạo Node object thôi chưa đủ?

Bạn có thể thủ công tạo:

```yaml
apiVersion: v1
kind: Node
metadata:
  name: fake-node
```

Kubernetes có thể lưu object này.

Nhưng không có máy thật nào chạy kubelet với tên `fake-node`.

Kết quả:

```text
Node object tồn tại
Nhưng:
- không có heartbeat;
- không có capacity thật;
- không có kubelet nhận Pod;
- trạng thái không Ready.
```

Nó giống như bạn tạo một dòng nhân viên trong cơ sở dữ liệu:

```text
Tên: Nguyễn Văn A
Trạng thái: chưa từng đăng nhập
```

Bản ghi có tồn tại, nhưng người đó chưa thực sự xuất hiện làm việc.

Kubernetes giữ Node object đó và tiếp tục chờ xem kubelet có xuất hiện không. Nó không tự xóa object chỉ vì Node không hợp lệ.

---

# 6. Node name quan trọng thế nào?

Ví dụ:

```text
fdp-k8s-wk1
```

Tên này là định danh của Node object.

Trong cùng một thời điểm không thể có hai Node cùng tên.

Quan trọng hơn, Kubernetes giả định:

> Cùng Node name thì đó vẫn là cùng một máy với cùng trạng thái cơ bản.

Ví dụ nguy hiểm:

```text
Máy cũ:
hostname: fdp-k8s-wk1
IP: 172.21.92.175
disk chứa dữ liệu A

Máy mới:
hostname: fdp-k8s-wk1
IP: 172.21.92.200
disk hoàn toàn khác
```

Nếu bạn phá máy cũ rồi tạo máy mới nhưng giữ nguyên tên mà không xóa Node object cũ, Kubernetes có thể giữ lại:

- labels cũ;
    
- taints cũ;
    
- thông tin topology cũ;
    
- Pod references cũ;
    
- trạng thái cũ trong một khoảng thời gian.
    

Vì vậy khi thay thế Node lớn, nên:

```bash
kubectl drain fdp-k8s-wk1
kubectl delete node fdp-k8s-wk1
```

Sau đó cho máy mới join lại.

Tên Node phải đúng định dạng DNS subdomain, nghĩa là thường:

```text
fdp-k8s-wk1
worker-01
k8s-node.example.local
```

Không nên dùng:

```text
Worker_01
My Node
NODE@123
```

---

# 7. Các tùy chọn quan trọng khi kubelet đăng ký

## 7.1. `--register-node`

Mặc định:

```text
--register-node=true
```

Nghĩa là kubelet tự tạo Node object nếu chưa có.

Nếu đặt:

```text
--register-node=false
```

kubelet không tự đăng ký; quản trị viên phải tự tạo Node object.

Thông thường không cần đổi cờ này.

---

## 7.2. `--kubeconfig`

Ví dụ:

```text
/etc/kubernetes/kubelet.conf
```

File này cho kubelet biết:

```yaml
server: https://k8s-api:6443
certificate-authority-data: ...
client-certificate-data: ...
client-key-data: ...
```

Nó trả lời ba câu hỏi:

```text
API server ở đâu?
Tin certificate server nào?
Kubelet dùng danh tính nào để đăng nhập?
```

Kubelet thường đăng nhập dưới danh tính:

```text
system:node:fdp-k8s-wk1
```

Và thuộc group:

```text
system:nodes
```

---

## 7.3. `--cloud-provider`

Cờ này dùng để tích hợp cloud provider.

Ví dụ cloud provider có thể cung cấp:

- instance ID;
    
- zone;
    
- region;
    
- địa chỉ public/private;
    
- load balancer;
    
- disk cloud.
    

Với bare-metal cluster của bạn thường không dùng cloud provider tích hợp sẵn.

---

## 7.4. `--register-with-taints`

Cho phép Node vừa đăng ký đã có taint.

Ví dụ:

```text
dedicated=gpu:NoSchedule
```

Node đăng ký xong sẽ có:

```yaml
spec:
  taints:
    - key: dedicated
      value: gpu
      effect: NoSchedule
```

Pod thông thường không được scheduler vào đó, trừ khi có toleration phù hợp.

Ví dụ control-plane Node thường có taint:

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

Nó mang ý nghĩa:

```text
Không đặt workload thường vào control plane.
```

---

## 7.5. `--node-ip`

Cờ này chỉ định IP kubelet khai báo là địa chỉ của Node.

Ví dụ trên `wk1`:

```text
--node-ip=172.21.92.175
```

Nếu không đặt, kubelet tự chọn IP mặc định của máy.

Vấn đề xảy ra khi máy có nhiều card mạng:

```text
ens160: 172.21.92.175
docker0: 172.17.0.1
tun0: 10.8.0.2
wlan0: 192.168.1.20
```

Nếu kubelet chọn sai IP, các thành phần khác có thể không kết nối đúng tới Node.

Bạn kiểm tra bằng:

```bash
kubectl get nodes -o wide
```

Cột:

```text
INTERNAL-IP
```

---

## 7.6. `--node-labels`

Dùng để gắn labels khi Node đăng ký.

Ví dụ:

```text
--node-labels=node-type=worker,disk=ssd,environment=production
```

Sau khi đăng ký:

```bash
kubectl get node fdp-k8s-wk1 --show-labels
```

Có thể thấy:

```text
disk=ssd
environment=production
```

Pod có thể yêu cầu chạy trên Node phù hợp:

```yaml
spec:
  nodeSelector:
    disk: ssd
```

Scheduler chỉ chọn Node có:

```text
disk=ssd
```

---

## 7.7. `--node-status-update-frequency`

Đây là tần suất kubelet cập nhật **Node status đầy đủ** lên API server.

Node status gồm:

- Ready;
    
- MemoryPressure;
    
- DiskPressure;
    
- PIDPressure;
    
- addresses;
    
- capacity;
    
- allocatable;
    
- image list;
    
- kubelet version;
    
- container runtime version.
    

Cần phân biệt:

```text
Node status update:
- nặng hơn
- nhiều thông tin
- không nhất thiết gửi liên tục từng giây

Lease renewal:
- rất nhẹ
- chủ yếu để báo "tôi vẫn sống"
- cập nhật thường xuyên hơn
```

---

# 8. NodeAuthorization và NodeRestriction là gì?

Kubelet cần quyền sửa Node object của chính nó.

Nhưng ta không muốn kubelet của `wk1` có thể sửa `wk2`.

Vì vậy Kubernetes có các lớp bảo vệ.

## Node Authorization

Cho phép kubelet:

```text
system:node:fdp-k8s-wk1
```

thực hiện các hành động cần thiết cho chính nó:

- đọc Pod được gán cho mình;
    
- cập nhật status của Pod trên Node mình;
    
- cập nhật Node status của mình;
    
- cập nhật Lease của mình;
    
- đọc Secret/ConfigMap liên quan đến Pod trên mình.
    

Nhưng không được tự do thao túng toàn cluster.

## NodeRestriction admission plugin

Hạn chế thêm việc kubelet được sửa gì.

Ví dụ:

```text
wk1 chỉ được sửa Node fdp-k8s-wk1
không được sửa fdp-k8s-wk2
```

Nó cũng hạn chế kubelet tự gắn một số label nhạy cảm để gian lận scheduling.

Ví dụ nếu có label bảo mật:

```text
company.example/security-level=trusted
```

Ta không muốn một Node bị chiếm quyền tự gắn label `trusted` cho chính nó rồi nhận workload nhạy cảm.

---

# 9. Tại sao thay `--node-labels` rồi restart kubelet có thể không hiệu lực?

Một số label từ `--node-labels` được thiết lập lúc **Node registration**.

Nếu Node object đã tồn tại, kubelet restart với cùng tên Node có thể không thực hiện lại toàn bộ đăng ký như Node mới.

Ví dụ ban đầu:

```text
disk=hdd
```

Sau đó bạn sửa kubelet:

```text
disk=ssd
```

rồi chỉ:

```bash
systemctl restart kubelet
```

Không phải lúc nào nó cũng được áp dụng như một Node mới đăng ký.

Cách đơn giản để đổi label thông thường là:

```bash
kubectl label node fdp-k8s-wk1 disk=ssd --overwrite
```

Còn nếu thay đổi cấu hình Node lớn, có thể drain, xóa Node object rồi join lại.

---

# 10. Manual Node administration

Bạn có thể thay đổi Node bằng `kubectl`.

## Thêm label

```bash
kubectl label node fdp-k8s-wk1 disk=ssd
```

## Xóa label

```bash
kubectl label node fdp-k8s-wk1 disk-
```

## Xem labels

```bash
kubectl get nodes --show-labels
```

## Thêm taint

```bash
kubectl taint node fdp-k8s-wk1 dedicated=database:NoSchedule
```

## Xóa taint

```bash
kubectl taint node fdp-k8s-wk1 dedicated=database:NoSchedule-
```

---

# 11. Node role thực chất chỉ là label

Khi bạn thấy:

```bash
kubectl get nodes
```

```text
NAME          STATUS   ROLES           AGE
fdp-k8s-cp1   Ready    control-plane   ...
fdp-k8s-wk1   Ready    <none>          ...
```

`ROLES` không phải một trường quyền lực đặc biệt theo kiểu:

```yaml
role: worker
```

Nó chủ yếu được suy ra từ label:

```text
node-role.kubernetes.io/control-plane
```

Bạn cũng có thể thêm label:

```bash
kubectl label node fdp-k8s-wk1 node-role.kubernetes.io/worker=worker
```

Lúc đó `kubectl get nodes` có thể hiển thị role `worker`.

Nhưng chỉ gắn label `worker` không tự biến đổi chức năng nội bộ của máy.

Tương tự, xóa label `control-plane` cũng không tự gỡ:

- kube-apiserver;
    
- etcd;
    
- scheduler;
    
- controller-manager.
    

Role label chủ yếu dùng cho:

- hiển thị;
    
- scheduling;
    
- quy ước quản trị.
    

---

# 12. Labels và nodeSelector

Giả sử:

```bash
kubectl label node fdp-k8s-wk1 storage=ssd
kubectl label node fdp-k8s-wk2 storage=hdd
```

Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  nodeSelector:
    storage: ssd
  containers:
    - name: postgres
      image: postgres:17
```

Scheduler lọc:

```text
wk1: storage=ssd → phù hợp
wk2: storage=hdd → loại
wk3: không có label → loại
```

Pod chỉ có thể được đưa lên `wk1`.

Node selector không phải là “ra lệnh kubelet”. Scheduler dùng nó trước khi Pod được gán Node.

---

# 13. Cordon là gì?

Lệnh:

```bash
kubectl cordon fdp-k8s-wk1
```

đặt:

```yaml
spec:
  unschedulable: true  
```

Ý nghĩa:

```text
Không đặt Pod mới lên wk1 nữa.
```

Nhưng Pod đang chạy vẫn tiếp tục chạy.

Ví dụ trước cordon:

```text
wk1:
- nginx-1
- postgres-0
- kafka-0
```

Sau cordon:

```text
Các Pod trên vẫn chạy.
Pod mới sẽ không được scheduler lên wk1.
```

Kiểm tra:

```bash
kubectl get nodes
```

Có thể thấy:

```text
Ready,SchedulingDisabled
```

Mở lại:

```bash
kubectl uncordon fdp-k8s-wk1
```

---

# 14. Drain khác cordon thế nào?

```bash
kubectl drain fdp-k8s-wk1
```

Drain thường làm hai việc:

1. cordon Node;
    
2. yêu cầu di dời/evict các Pod phù hợp khỏi Node.
    

So sánh:

```text
cordon:
- cấm Pod mới
- Pod cũ vẫn chạy

drain:
- cấm Pod mới
- cố đẩy Pod cũ sang Node khác
```

Ví dụ:

```bash
kubectl drain fdp-k8s-wk1 \
  --ignore-daemonsets \
  --delete-emptydir-data
```

Sau bảo trì:

```bash
kubectl uncordon fdp-k8s-wk1
```

---

# 15. Vì sao DaemonSet vẫn liên quan đến Node đã cordon/drain?

DaemonSet thường chạy một Pod trên từng Node:

```text
calico-node
kube-proxy
node-exporter
log agent
```

Chúng là dịch vụ gắn với chính Node.

Ví dụ Calico cần chạy trên mỗi Node để thiết lập mạng Node đó.

Do đó DaemonSet thường có toleration thích hợp và được xử lý đặc biệt khi drain.

Khi drain, bạn thường phải thêm:

```bash
--ignore-daemonsets
```

Nghĩa là:

```text
Đừng cố xóa các DaemonSet Pod như workload thông thường.
```

Nếu không thêm:

```
--ignore-daemonsets
```

thì `kubectl drain` thường sẽ báo lỗi và không drain tiếp, vì nó thấy có DaemonSet Pod mà nó không nên xóa.
---

# 16. Node status chứa những gì?

Node status có bốn nhóm chính:

```text
Addresses
Conditions
Capacity / Allocatable
Info
```

Bạn xem bằng:

```bash
kubectl describe node fdp-k8s-wk1
```

---

# 17. Addresses

Ví dụ:

```text
Addresses:
  InternalIP:  172.21.92.175
  Hostname:    fdp-k8s-wk1
```

Các loại phổ biến:

## InternalIP

IP dùng trong nội bộ cluster hoặc mạng private.

```text
172.21.92.175
```

## ExternalIP

IP có thể truy cập từ bên ngoài, thường gặp trên cloud.

## Hostname

```text
fdp-k8s-wk1
```

Một số thành phần dùng địa chỉ này để kết nối tới kubelet hoặc Node.

Ví dụ API server có thể kết nối kubelet tại:

```text
https://172.21.92.175:10250
```

---

# 18. Conditions

Ví dụ:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

## Ready

```text
Ready=True
```

Node khỏe và có thể nhận Pod.

```text
Ready=False
```

Node đã báo rõ rằng nó không khỏe.

```text
Ready=Unknown
```

Control plane không nhận được heartbeat và không biết Node đang thế nào.

Điểm rất quan trọng:

```text
False:
Node vẫn có thể nói chuyện với control plane và báo "tôi có vấn đề".

Unknown:
Control plane không nghe được gì từ Node.
```

---

## MemoryPressure

```text
MemoryPressure=True
```

Node đang thiếu RAM.

Kubelet có thể bắt đầu eviction Pod để bảo vệ Node.

## DiskPressure

```text
DiskPressure=True
```

Dung lượng disk hoặc inode quá thấp.

Có thể do:

- image quá nhiều;
    
- container logs quá lớn;
    
- emptyDir đầy;
    
- container writable layer đầy.
    

## PIDPressure

Máy sắp cạn số process ID.

Ví dụ ứng dụng fork quá nhiều process.

## NetworkUnavailable

Mạng Node chưa sẵn sàng, thường liên quan CNI.

---

# 19. Capacity và Allocatable

Ví dụ Node có:

```text
Capacity:
  cpu: 4
  memory: 12 GiB
  pods: 110
```

Nhưng:

```text
Allocatable:
  cpu: 3800m
  memory: 10.8 GiB
  pods: 110
```

## Capacity

Tổng tài nguyên máy có.

## Allocatable

Phần Kubernetes cho phép Pod sử dụng.

Công thức gần đúng:

```text
Allocatable
= Capacity
- tài nguyên dành cho hệ điều hành
- tài nguyên dành cho Kubernetes
- ngưỡng eviction
```

Ví dụ:

```text
Máy có 12 GB RAM
- OS và system daemons: 1 GB
- kubelet/containerd: 0.5 GB
- eviction reserve: 0.5 GB
= khoảng 10 GB allocatable
```

Scheduler nên dựa vào **Allocatable**, không đơn giản dùng toàn bộ RAM vật lý.

---

# 20. Info

Thông tin Node có thể gồm:

```text
Kernel Version
OS Image
Operating System
Architecture
Container Runtime Version
Kubelet Version
Kube-Proxy Version
Boot ID
Machine ID
System UUID
```

Ví dụ:

```text
OS Image: Ubuntu 24.04 LTS
Kernel Version: 6.x
Container Runtime Version: containerd://2.2.5
Kubelet Version: v1.36.2
Architecture: amd64
```

---

# 21. Heartbeat của Node là gì?

Control plane cần biết:

```text
Node còn sống không?
```

Node gửi hai loại tín hiệu:

1. cập nhật `.status` của Node;
    
2. cập nhật Lease object.
    

---

# 22. Node status update

Kubelet định kỳ cập nhật:

```yaml
status:
  conditions:
    - type: Ready
      status: "True"
  capacity:
    cpu: "4"
  allocatable:
    cpu: "4"
```

Đây là bản cập nhật tương đối lớn.

Nếu hàng nghìn Node liên tục sửa toàn bộ Node status, API server và etcd sẽ chịu tải cao.

Vì vậy Kubernetes dùng thêm Lease.

---

# 23. Lease object là gì?

Mỗi Node có một Lease trong namespace:

```text
kube-node-lease
```

Xem:

```bash
kubectl get lease -n kube-node-lease
```

Ví dụ:

```text
fdp-k8s-cp1
fdp-k8s-wk1
fdp-k8s-wk2
fdp-k8s-wk3
```

Lease có nội dung gần giống:

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: fdp-k8s-wk1
  namespace: kube-node-lease
spec:
  holderIdentity: fdp-k8s-wk1
  renewTime: "2026-07-04T03:20:10Z"
```

Nó có nghĩa:

```text
wk1 vừa báo "tôi vẫn sống" vào thời điểm renewTime.
```

Lease nhẹ hơn cập nhật toàn bộ Node status.

So sánh:

```text
Node status:
"Tôi còn sống, CPU là 4, RAM là ..., DiskPressure=False,
kubelet version ..., addresses ..."

Lease:
"Tôi còn sống, timestamp mới đây."
```

---

# 24. Lease có “hết hạn rồi biến mất” không?

Không theo kiểu TTL tự xóa.

Khi kubelet dừng:

```text
Lease object thường vẫn còn.
renewTime chỉ không được cập nhật nữa.
```

Ví dụ:

```text
Lease fdp-k8s-wk1:
renewTime = 10:00:00

Bây giờ = 10:02:00
Không có cập nhật mới
```

Node controller nhận ra heartbeat đã quá cũ.

Lease không phải Pod và cũng không phải “Lease của Pod”. Nó là Lease gắn với **Node**.

Nếu Pod chết, Lease của Node vẫn còn miễn Node vẫn là thành viên cluster.

Khi Node bị xóa khỏi cluster, Lease liên quan thường được dọn theo.

---

# 25. Node controller là gì?

Node controller là logic nằm trong:

```text
kube-controller-manager
```

Nó phụ trách quản lý vòng đời và sức khỏe Node.

Luồng:

```text
kubelet cập nhật Lease/Node status
          │
          ▼
API server lưu dữ liệu
          │
          ▼
Node controller theo dõi
          │
          ├── Node khỏe → không làm gì
          └── mất heartbeat → Ready=Unknown, thêm taint, xử lý Pod
```

---

# 26. Node controller gán PodCIDR

Nếu cluster dùng cơ chế cấp Pod CIDR theo Node, Node controller có thể cấp một block mạng cho mỗi Node.

Ví dụ:

```text
wk1: 192.168.10.0/26
wk2: 192.168.20.0/26
wk3: 192.168.30.0/26
```

Pod trên `wk1` nhận IP từ block của `wk1`.

Trong cluster Calico, việc quản lý IP có thể liên quan IPAM của Calico; chi tiết thực tế phụ thuộc cấu hình CNI và controller.

Node status có thể chứa:

```yaml
spec:
  podCIDR: 192.168.10.0/26
```

---

# 27. Node controller kiểm tra cloud VM

Trong cloud:

```text
Node object: worker-5
Cloud VM: instance i-12345
```

Nếu Node mất liên lạc, Node controller/cloud controller có thể hỏi cloud provider:

```text
VM i-12345 còn tồn tại không?
```

Nếu VM đã bị xóa, controller có thể dọn Node object.

Bare-metal của bạn không có cloud API để tự hỏi:

```text
Máy vật lý này còn tồn tại không?
```

Do đó nhiều việc phải do quản trị viên xử lý.

---

# 28. Khi Node mất liên lạc thì chuyện gì xảy ra?

Giả sử `wk1` mất điện.

## Giai đoạn 1: kubelet ngừng heartbeat

```text
Lease không còn được renew
Node status không còn được cập nhật
```

## Giai đoạn 2: Node controller phát hiện

Node controller kiểm tra định kỳ.

Sau khi quá ngưỡng, nó đổi:

```text
Ready=True
```

thành:

```text
Ready=Unknown
```

## Giai đoạn 3: thêm taint

Ví dụ:

```text
node.kubernetes.io/unreachable:NoExecute
```

Hoặc:

```text
node.kubernetes.io/not-ready:NoExecute
```

## Giai đoạn 4: Pod bị eviction sau thời gian chịu đựng

Pod bình thường thường có toleration mặc định khoảng 300 giây cho hai taint trên.

Nghĩa là:

```text
Node mất liên lạc
   │
   ├── trong vài phút: chờ Node có thể quay lại
   │
   └── quá thời gian: bắt đầu evict/reschedule Pod
```

Tài liệu nêu mặc định thường chờ khoảng 5 phút trước khi bắt đầu eviction Pod trên Node unreachable.

---

# 29. Tại sao không reschedule Pod ngay lập tức?

Vì mất heartbeat chưa chắc Node thật sự chết.

Có thể:

- mất mạng 10 giây;
    
- API server tạm nghẽn;
    
- switch restart;
    
- kubelet restart;
    
- control plane tạm mất kết nối;
    
- network partition.
    

Nếu cứ mất vài giây là reschedule, sẽ gây:

- tạo container trùng;
    
- tải image hàng loạt;
    
- ứng dụng chập chờn;
    
- volume attach/detach liên tục;
    
- database có nguy cơ hai bản cùng nghĩ mình active.
    

Do đó Kubernetes đợi một khoảng thời gian.

---

# 30. API-initiated eviction là gì?

Khi Node chết, kubelet trên Node đó không còn chạy để tự xóa Pod.

Control plane sẽ thay đổi trạng thái Pod qua API, từ đó các controller tạo Pod thay thế ở Node khác.

Ví dụ Deployment:

```yaml
replicas: 3
```

Ban đầu:

```text
pod-a → wk1
pod-b → wk2
pod-c → wk3
```

`wk1` chết:

```text
pod-a bị coi là mất
ReplicaSet nhận thấy chỉ còn 2 replica hoạt động
ReplicaSet tạo pod-d
Scheduler gán pod-d → wk2 hoặc wk3
```

Điểm quan trọng:

> Không phải Pod cũ được “dịch chuyển” nguyên vẹn sang Node khác.

Kubernetes tạo **Pod mới**, với:

- tên hoặc UID mới;
    
- IP mới;
    
- container mới;
    
- filesystem cục bộ mới.
    

---

# 31. Eviction rate limit là gì?

Nếu hàng trăm Node cùng chết, Kubernetes không muốn evict tất cả Pod cùng lúc.

Mặc định tài liệu đề cập:

```text
--node-eviction-rate=0.1 node/giây
```

Tức trung bình:

```text
1 Node mỗi 10 giây
```

Đây là rate theo **Node**, không phải đơn giản 0.1 Pod/giây.

Mục đích:

- tránh API server bị bão request;
    
- tránh scheduler bị dồn hàng nghìn Pod;
    
- tránh registry bị kéo image đồng loạt;
    
- tránh cluster còn sống bị quá tải ngay lập tức.
    

---

# 32. Availability Zone là gì?

Trong cloud có thể có:

```text
Region: asia-southeast1
├── Zone A
├── Zone B
└── Zone C
```

Mỗi zone thường là một cụm hạ tầng tách biệt.

Nếu một số Node trong Zone A chết, có thể là Node riêng lẻ hỏng.

Nhưng nếu 100% Node Zone A cùng mất kết nối, có thể cả zone hoặc đường mạng tới zone bị lỗi.

Node controller tính tỷ lệ Node unhealthy theo zone để quyết định tốc độ eviction.

---

# 33. `--unhealthy-zone-threshold=0.55`

Nếu ít nhất 55% Node trong một zone không khỏe:

```text
unhealthy ratio >= 0.55
```

Kubernetes coi đây có thể là sự cố diện rộng, không phải lỗi cá biệt.

Ví dụ zone có 10 Node:

```text
4 Node hỏng → 40% → dưới ngưỡng
6 Node hỏng → 60% → vượt ngưỡng
```

Khi vượt ngưỡng, eviction rate bị giảm hoặc dừng tùy quy mô cluster.

---

# 34. Vì sao cluster nhỏ có thể dừng eviction?

Theo tài liệu, nếu cluster nhỏ hơn hoặc bằng ngưỡng mặc định 50 Node và một zone có tỷ lệ unhealthy cao, Kubernetes có thể dừng eviction.

Lý do:

Giả sử cluster chỉ có 4 Node:

```text
3 Node mất kết nối control plane
1 Node còn kết nối
```

Nếu control plane lập tức reschedule toàn bộ workload từ 3 Node kia sang 1 Node còn lại:

- Node còn lại chắc chắn quá tải;
    
- có nguy cơ tạo duplicate workload;
    
- có thể vấn đề thực tế chỉ là network partition;
    
- các Pod bên phía bị ngắt mạng có thể vẫn đang chạy.
    

Kubernetes chọn thận trọng.

---

# 35. Network partition nguy hiểm thế nào?

Ví dụ:

```text
Control plane ──X── wk1
```

Control plane không nói chuyện được với `wk1`.

Nhưng trên `wk1`:

```text
PostgreSQL container vẫn chạy
Ứng dụng vẫn ghi dữ liệu cục bộ
```

Nếu control plane tạo một PostgreSQL mới ở `wk2`, có thể xảy ra:

```text
PostgreSQL cũ trên wk1 vẫn chạy
PostgreSQL mới trên wk2 cũng chạy
```

Đây là một dạng split-brain nếu hệ thống không có cơ chế leader/quorum/fencing phù hợp.

Vì vậy Kubernetes không luôn vội vàng kết luận:

```text
Không liên lạc được = máy chắc chắn đã chết.
```

---

# 36. Taints do Node controller thêm

Khi Node có vấn đề, Node controller có thể thêm:

```text
node.kubernetes.io/not-ready
node.kubernetes.io/unreachable
node.kubernetes.io/memory-pressure
node.kubernetes.io/disk-pressure
node.kubernetes.io/pid-pressure
node.kubernetes.io/network-unavailable
```

Scheduler nhìn taint để tránh đưa Pod mới vào Node không khỏe.

Ví dụ:

```text
node.kubernetes.io/not-ready:NoSchedule
```

Nghĩa là:

```text
Đừng schedule Pod mới lên đây.
```

Nếu:

```text
node.kubernetes.io/unreachable:NoExecute
```

Nó còn có thể đẩy Pod đang chạy ra, trừ Pod có toleration.

---

# 37. NoSchedule và NoExecute khác nhau thế nào?

## NoSchedule

```text
Pod mới không được vào.
Pod đang có vẫn ở lại.
```

## PreferNoSchedule

```text
Cố tránh Node này, nhưng không tuyệt đối.
```

## NoExecute

```text
Pod mới không được vào.
Pod đang có cũng bị đuổi nếu không tolerate.
```

Ví dụ:

```text
dedicated=database:NoSchedule
```

Chỉ Pod database có toleration mới vào.

```text
node.kubernetes.io/unreachable:NoExecute
```

Pod đang nằm trên Node unreachable có thể bị eviction.

---

# 38. Toleration seconds

Pod có thể có:

```yaml
tolerations:
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 300
```

Nghĩa là:

```text
Khi Node bị taint unreachable,
Pod được phép ở lại logic trong 300 giây.
Sau đó có thể bị eviction.
```

Nếu không có `tolerationSeconds`:

```yaml
tolerations:
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
```

Pod có thể tolerate vô thời hạn.

DaemonSet và một số Pod hệ thống có thể có toleration mạnh hơn workload thường.

---

# 39. Scheduler kiểm tra tài nguyên như thế nào?

Tài liệu nói scheduler đảm bảo tổng `requests` không vượt quá tài nguyên Node.

Ví dụ Node:

```text
Allocatable CPU: 4 core
Allocatable RAM: 10 GiB
```

Đã có Pod:

```text
Pod A requests:
cpu: 1
memory: 2 GiB

Pod B requests:
cpu: 2
memory: 4 GiB
```

Tổng:

```text
CPU request = 3 core
RAM request = 6 GiB
```

Pod mới yêu cầu:

```text
cpu: 2
memory: 3 GiB
```

Nếu lên Node đó:

```text
CPU = 3 + 2 = 5 > 4
```

Scheduler không chọn Node này.

Dù CPU thực tế lúc đó chỉ dùng 5%, scheduler vẫn dựa chủ yếu vào **requests**, không phải mức sử dụng tức thời.

---

# 40. Requests không phải usage thật

Ví dụ Pod:

```yaml
resources:
  requests:
    cpu: 1
    memory: 2Gi
  limits:
    cpu: 2
    memory: 4Gi
```

Ý nghĩa:

```text
requests:
- lượng tài nguyên dùng để scheduler đặt chỗ
- Node phải có chỗ cho 1 CPU và 2 GiB

limits:
- mức tối đa container được phép sử dụng
```

Pod có thể thực tế chỉ dùng:

```text
0.1 CPU
500 MiB RAM
```

Nhưng scheduler vẫn tính nó chiếm:

```text
1 CPU request
2 GiB request
```

---

# 41. Scheduler có tính process ngoài Kubernetes không?

Tài liệu nhấn mạnh rằng scheduler tính các container do kubelet quản lý, nhưng không trực tiếp biết đầy đủ các process ngoài sự quản lý của kubelet.

Ví dụ trên Node:

```text
Pod requests: 8 GiB
Một Java process chạy bằng systemd: 3 GiB
OS + cache: 1 GiB
Máy có 12 GiB
```

Scheduler có thể chỉ thấy Pod requests và không hiểu hết Java process bên ngoài.

Do đó cần reserve tài nguyên cho hệ thống:

```text
system-reserved
kube-reserved
eviction-hard
```

Nếu không, scheduler có thể tưởng Node còn nhiều RAM trong khi process ngoài Kubernetes đang chiếm đáng kể.

---

# 42. Container chạy trực tiếp bằng containerd thì sao?

Giả sử ai đó trên Node tự chạy container ngoài kubelet.

Kubernetes không có Pod object cho container đó.

Scheduler không coi đó là workload Kubernetes đã đặt chỗ.

Nhưng container đó vẫn tiêu thụ:

- CPU;
    
- RAM;
    
- disk;
    
- PID.
    

Đây là lý do không nên chạy workload tùy tiện trực tiếp trên Kubernetes Node.

Node nên được coi là hạ tầng do Kubernetes quản lý.

---

# 43. Reserve resources cho system daemons

Có thể cấu hình kubelet:

```text
--system-reserved=cpu=500m,memory=1Gi
--kube-reserved=cpu=500m,memory=1Gi
```

Ví dụ Node có:

```text
CPU capacity: 4
RAM capacity: 12 GiB
```

Reserve:

```text
system-reserved:
  CPU 0.5
  RAM 1 GiB

kube-reserved:
  CPU 0.5
  RAM 1 GiB
```

Allocatable gần đúng:

```text
CPU: 3
RAM: 10 GiB
```

`system-reserved` dành cho:

- systemd;
    
- sshd;
    
- journald;
    
- OS daemon.
    

`kube-reserved` dành cho:

- kubelet;
    
- containerd;
    
- kube-proxy;
    
- một số thành phần Kubernetes trên Node.
    

---

# 44. Node topology là gì?

Topology nghĩa là cấu trúc phần cứng bên trong hoặc vị trí logic của Node.

Có hai cách hay gặp.

## Topology ở cấp cluster

```text
region
zone
rack
hostname
```

Ví dụ labels:

```text
topology.kubernetes.io/region=asia
topology.kubernetes.io/zone=zone-a
kubernetes.io/hostname=fdp-k8s-wk1
```

Dùng để phân tán Pod:

```text
Không đặt tất cả replica trên cùng một Node hoặc cùng zone.
```

## Topology phần cứng trong Node

Một server lớn có thể có:

```text
NUMA node 0
├── CPU 0-7
└── RAM bank A

NUMA node 1
├── CPU 8-15
└── RAM bank B
```

Nếu container dùng CPU ở NUMA 0 nhưng RAM ở NUMA 1, hiệu năng có thể kém hơn.

Topology Manager của kubelet phối hợp các “hints” từ:

- CPU Manager;
    
- Device Manager;
    
- Memory Manager;
    

để cố cấp tài nguyên cùng topology phù hợp.

Điều này quan trọng với:

- NFV;
    
- telecom;
    
- AI/GPU;
    
- high-performance computing;
    
- workload latency thấp.
    

Với ứng dụng web hoặc bài tập thông thường, bạn thường chưa cần chỉnh sâu.

---

# 45. Toàn bộ vòng đời của một Node

Gộp tất cả lại:

```text
1. Máy Linux khởi động
   ├── containerd chạy
   └── kubelet chạy

2. Kubelet đọc kubeconfig
   └── biết API server tại k8s-api:6443

3. Kubelet xác thực
   └── danh tính system:node:fdp-k8s-wk1

4. Kubelet đăng ký Node
   └── API server tạo Node object trong etcd

5. Kubelet gửi status
   ├── CPU/RAM
   ├── IP
   ├── conditions
   └── version

6. Kubelet renew Lease
   └── báo "tôi còn sống"

7. Scheduler thấy Node Ready
   └── có thể gán Pod vào Node

8. Kubelet thấy Pod được gán
   └── gọi containerd tạo container

9. Node gặp lỗi
   └── Lease ngừng cập nhật

10. Node controller phát hiện
    ├── Ready=Unknown
    ├── thêm unreachable taint
    └── chờ toleration/eviction timeout

11. Controller tạo Pod thay thế
    └── Scheduler đưa Pod mới lên Node khỏe khác
```

---

# 46. Các control loop đang hoạt động cùng nhau

Không có một thành phần duy nhất làm tất cả.

```text
kubelet:
- quản lý Pod trên Node
- báo health

API server:
- cổng giao tiếp
- lưu thay đổi qua etcd

scheduler:
- chọn Node cho Pod chưa được gán

node controller:
- theo dõi Node
- xử lý Node unhealthy

ReplicaSet controller:
- duy trì số replica

taint-eviction-controller:
- xử lý Pod trên Node bị taint

containerd:
- thực sự chạy container

Calico:
- mạng Pod

kube-proxy:
- Service routing
```

Ví dụ `wk1` chết:

```text
Node controller:
"wk1 unreachable."

Taint eviction controller:
"Pod trên wk1 đã hết thời gian tolerate."
"Gọi API server để xóa/evict Pod"

ReplicaSet controller:
"Thiếu một replica."

Scheduler:
"Pod mới chạy trên wk2."

Kubelet wk2:
"Tôi sẽ tạo container."

containerd wk2:
"Container đã chạy."
```

---

# 47. Những nhầm lẫn dễ gặp

## “Node object chính là máy vật lý”

Không hoàn toàn.

```text
Node object = đại diện trong Kubernetes
Máy/VM = hạ tầng thật
```

## “Tạo Node YAML là tạo ra một máy”

Không. Kubernetes không tự sinh máy bare-metal chỉ vì có Node object.

## “Cordon làm Pod hiện tại bị chuyển đi”

Không. Cordon chỉ chặn scheduling mới.

## “Drain di chuyển nguyên Pod”

Không. Pod cũ bị xóa/evict và controller tạo Pod mới.

## “Lease hết hạn sẽ tự mất object”

Không. `renewTime` ngừng cập nhật; object thường vẫn còn.

## “Node NotReady là kubelet chắc chắn chết”

Không nhất thiết. Có thể do network partition, API server không tới được Node hoặc lỗi khác.

## “Scheduler xem CPU usage hiện tại để chọn Node”

Chủ yếu scheduler dựa vào `requests`, constraints, labels, taints và các rule scheduling.

## “Capacity là toàn bộ tài nguyên Pod dùng được”

Không. Phần dùng cho Pod là `Allocatable`.

## “Node role là một loại Node đặc biệt trong API”

Role hiển thị chủ yếu dựa trên labels; việc một máy thực sự chạy control-plane components là vấn đề khác.

---

# 48. Các lệnh nên thử trên cluster của bạn

## Xem Node

```bash
kubectl get nodes -o wide
```

## Xem chi tiết Node

```bash
kubectl describe node fdp-k8s-wk1
```

## Xem Node object YAML

```bash
kubectl get node fdp-k8s-wk1 -o yaml
```

## Xem conditions

```bash
kubectl get node fdp-k8s-wk1 \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'
```

## Xem capacity

```bash
kubectl get node fdp-k8s-wk1 \
  -o jsonpath='{.status.capacity}'
```

## Xem allocatable

```bash
kubectl get node fdp-k8s-wk1 \
  -o jsonpath='{.status.allocatable}'
```

## Xem Lease

```bash
kubectl get leases -n kube-node-lease
```

## Xem chi tiết Lease

```bash
kubectl get lease fdp-k8s-wk1 \
  -n kube-node-lease \
  -o yaml
```

## Xem taints

```bash
kubectl describe node fdp-k8s-wk1 | grep -A2 Taints
```

## Xem Pod nào đang chạy trên Node

```bash
kubectl get pods -A \
  --field-selector spec.nodeName=fdp-k8s-wk1 \
  -o wide
```

## Cordon

```bash
kubectl cordon fdp-k8s-wk1
```

## Uncordon

```bash
kubectl uncordon fdp-k8s-wk1
```

---

# 49. Sơ đồ cuối cùng dễ nhớ nhất

```text
                         CONTROL PLANE
┌────────────────────────────────────────────────────────┐
│                                                        │
│  kube-apiserver ◄────────────── kubectl                │
│        │                                               │
│        ├────────► etcd                                 │
│        │          lưu Node, Pod, Lease                 │
│        │                                               │
│        ├────────► scheduler                            │
│        │          chọn Node cho Pod                    │
│        │                                               │
│        └────────► node controller                      │
│                   kiểm tra Node health                 │
│                                                        │
└───────────────────────▲────────────────────────────────┘
                        │ HTTPS
                        │ Node status + Lease
                        │ Pod specs
                        ▼
                         WORKER NODE
┌────────────────────────────────────────────────────────┐
│ kubelet                                                │
│   ├── đăng ký Node                                     │
│   ├── renew Lease                                      │
│   ├── cập nhật status                                  │
│   └── yêu cầu container runtime chạy Pod               │
│                                                        │
│ containerd                                             │
│   └── chạy containers                                  │
│                                                        │
│ kube-proxy                                             │
│   └── Service routing                                  │
│                                                        │
│ Calico                                                 │
│   └── Pod networking / VXLAN / NetworkPolicy           │
└────────────────────────────────────────────────────────┘
```

Câu tóm tắt quan trọng nhất:

> **Máy Node chạy kubelet; kubelet đăng ký một Node object với API server và liên tục renew Lease. Scheduler đọc Node object để đặt Pod. Khi Lease ngừng cập nhật, Node controller đánh dấu Node không khỏe, thêm taint và sau một khoảng chờ sẽ để các controller tạo Pod thay thế trên Node khác.**