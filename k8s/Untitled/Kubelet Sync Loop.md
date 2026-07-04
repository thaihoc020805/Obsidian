Hiểu đơn giản: **kubelet là một vòng lặp điều khiển chạy trên từng node**, liên tục so sánh:

```text
Pod đáng lẽ phải chạy như thế nào
            vs
Pod/container thực tế đang chạy như thế nào
```

Nếu hai bên không khớp, kubelet sẽ cố sửa cho khớp.

---

## 1. Desired state đến từ đâu?

Giả sử API Server đang lưu Pod sau:

```yaml
spec:
  nodeName: fdp-k8s-wk1
  containers:
    - name: nginx
      image: nginx:1.27
```

Điều này có nghĩa:

```text
Trạng thái mong muốn:
Trên node fdp-k8s-wk1 phải có Pod này,
bên trong có container nginx:1.27 đang chạy.
```

Kubelet trên `fdp-k8s-wk1` theo dõi các Pod có:

```yaml
spec.nodeName: fdp-k8s-wk1
```

Sau khi nhận Pod spec, kubelet không chỉ chạy lệnh một lần rồi bỏ mặc. Nó liên tục kiểm tra:

- Pod sandbox đã có chưa?
    
- Network namespace đã có chưa?
    
- Container đã được tạo chưa?
    
- Container có đang chạy không?
    
- Volume đã mount chưa?
    
- Secret, ConfigMap đã được chuẩn bị chưa?
    
- Liveness probe có thất bại không?
    
- Pod có đang bị terminate không?
    

---

# 2. Sync Loop là gì?

Sync Loop là vòng lặp chính của kubelet.

Có thể hình dung như:

```go
for {
    event := waitForPodEvent()

    pod := event.Pod

    reconcile(pod)
}
```

Nhưng thực tế phức tạp hơn. Nó nhận công việc từ nhiều nguồn:

```text
API Server báo Pod mới
API Server báo Pod bị sửa
API Server báo Pod bị xóa
PLEG báo container chết
Probe báo container unhealthy
Timer định kỳ chạy
Volume manager báo volume thay đổi
Runtime báo lỗi
```

Các sự kiện này được đưa vào vòng xử lý của kubelet.

Ví dụ:

```text
API Server
    |
    | Pod nginx được assign vào wk1
    v
kubelet Sync Loop
    |
    | đưa Pod vào pod worker
    v
SyncPod()
    |
    | kiểm tra trạng thái hiện tại
    v
containerd
```

Điểm quan trọng là Sync Loop **không chỉ chạy theo chu kỳ cố định**.

Nó có thể được đánh thức bởi event, chẳng hạn:

```text
container vừa chết
Pod spec vừa thay đổi
probe vừa fail
Pod vừa bị xóa
```

Ngoài ra vẫn có các lần đồng bộ lại định kỳ để tránh bỏ sót trạng thái.

---

# 3. Pod worker là gì?

Kubelet có thể quản lý rất nhiều Pod trên một node.

Nó không xử lý toàn bộ Pod tuần tự trong một vòng lặp lớn như:

```text
Pod A xong
rồi Pod B
rồi Pod C
```

Thay vào đó, mỗi Pod có luồng xử lý logic riêng gọi là **pod worker**.

Ví dụ node có ba Pod:

```text
Pod A ── pod worker A ── SyncPod(A)
Pod B ── pod worker B ── SyncPod(B)
Pod C ── pod worker C ── SyncPod(C)
```

Các Pod khác nhau có thể được xử lý đồng thời.

Nhưng đối với **cùng một Pod**, kubelet cố gắng tuần tự hóa việc xử lý để tránh:

```text
vừa tạo container
vừa xóa container
vừa restart container
```

cùng một lúc.

Ví dụ Pod A nhận liên tiếp ba sự kiện:

```text
1. Pod được tạo
2. Image bị thay đổi
3. Pod bị xóa
```

Pod worker của A sẽ xử lý có thứ tự, tránh nhiều SyncPod cho cùng Pod chạy chồng chéo hỗn loạn.

---

# 4. SyncPod là gì?

`SyncPod()` là phần xử lý chính để đưa một Pod về trạng thái mong muốn.

Giả sử Pod cần chạy nhưng hiện tại chưa tồn tại.

`SyncPod()` có thể thực hiện logic kiểu:

```text
1. Kiểm tra Pod có được phép chạy không
2. Tạo thư mục dữ liệu cho Pod
3. Chuẩn bị Secret và ConfigMap
4. Mount volume
5. Tạo Pod sandbox
6. Thiết lập network cho sandbox
7. Pull image nếu cần
8. Tạo container
9. Start container
10. Chạy probes
11. Cập nhật Pod status về API Server
```

Không phải toàn bộ công việc đều nằm trực tiếp trong một hàm duy nhất, nhưng về mặt ý tưởng, `SyncPod()` là đầu mối điều phối quá trình reconcile Pod.

Ví dụ:

```text
Desired:
nginx phải Running

Actual:
chưa có sandbox
chưa có container
```

Kubelet sẽ đi theo hướng:

```text
create sandbox
    ↓
setup networking
    ↓
create nginx container
    ↓
start nginx container
```

Một lúc sau:

```text
Desired:
nginx phải Running

Actual:
nginx đã exit với code 1
```

Nếu restart policy cho phép, kubelet sẽ cố restart container.

---

# 5. `SyncTerminatingPod` và `SyncTerminatedPod`

Không phải Pod lúc nào cũng ở trạng thái “cần chạy”.

Kubelet có các luồng xử lý khác nhau tùy lifecycle.

## `SyncPod`

Dùng khi Pod còn cần hoạt động:

```text
Pending
ContainerCreating
Running
Restarting
```

## `SyncTerminatingPod`

Dùng khi Pod đang bị xóa.

Ví dụ bạn chạy:

```bash
kubectl delete pod nginx
```

API Server đặt:

```yaml
metadata:
  deletionTimestamp: ...
```

Kubelet thấy Pod đang terminating và thực hiện:

```text
1. Chạy preStop hook nếu có
2. Gửi SIGTERM cho container
3. Chờ terminationGracePeriodSeconds
4. Nếu chưa dừng thì gửi SIGKILL
5. Dừng sandbox
6. Unmount volume
```

## `SyncTerminatedPod`

Dùng khi container đã dừng xong và kubelet cần dọn phần còn lại:

```text
cleanup sandbox
cleanup volume
cleanup cgroup
cleanup thư mục Pod
cập nhật trạng thái cuối
```

Có thể hình dung:

```text
SyncPod
   |
   | Pod bị delete
   v
SyncTerminatingPod
   |
   | container đã dừng
   v
SyncTerminatedPod
   |
   v
cleanup hoàn tất
```

---

# 6. Kubelet không tự chạy container trực tiếp

Kubelet không trực tiếp gọi Linux để tạo container theo kiểu:

```text
kubelet tự tạo namespace
kubelet tự unpack image
kubelet tự tạo overlay filesystem
```

Thay vào đó, kubelet nói chuyện với container runtime qua **CRI**.

Trong cluster của bạn:

```text
kubelet → CRI gRPC → containerd
```

Thông thường socket là:

```text
unix:///run/containerd/containerd.sock
```

Kubelet là client, containerd là server.

Ví dụ kubelet gửi các request CRI như:

```text
RunPodSandbox
PullImage
CreateContainer
StartContainer
StopContainer
RemoveContainer
PodSandboxStatus
ContainerStatus
ListContainers
```

Luồng tạo Pod nhìn gần đúng như sau:

```text
kubelet
   |
   | RunPodSandbox
   v
containerd
   |
   | tạo sandbox container
   | tạo network namespace
   | gọi CNI để cấu hình network
   v
Pod sandbox sẵn sàng

kubelet
   |
   | CreateContainer
   v
containerd

kubelet
   |
   | StartContainer
   v
containerd
```

---

# 7. Pod sandbox là gì?

Một Pod không chỉ là tập hợp container độc lập.

Các container trong cùng Pod chia sẻ một số tài nguyên, đặc biệt là network namespace.

Container runtime thường tạo một sandbox trước:

```text
Pod nginx
├── sandbox / pause container
├── nginx container
└── sidecar container
```

Sandbox giữ môi trường dùng chung của Pod, ví dụ:

```text
network namespace
IP address
Linux namespaces liên quan
cấu hình cấp Pod
```

Sau đó các container của Pod được gắn vào network namespace của sandbox.

Vì vậy:

```text
nginx container: port 80
sidecar container: gọi localhost:80
```

Hai container có thể nói chuyện qua `localhost` vì chúng chia sẻ network namespace.

---

# 8. PLEG là gì?

PLEG là viết tắt của:

```text
Pod Lifecycle Event Generator
```

Nó giúp kubelet phát hiện trạng thái container trong runtime đã thay đổi.

Ví dụ container đang chạy:

```text
nginx: Running
```

Sau đó process nginx crash:

```text
nginx: Exited
```

Kubelet cần biết chuyện này để:

- cập nhật status;
    
- restart container nếu cần;
    
- chạy logic probe/lifecycle;
    
- gọi SyncPod lại.
    

PLEG định kỳ hỏi container runtime:

```text
Cho tôi trạng thái hiện tại của các Pod và container.
```

Ví dụ:

```text
PLEG
  |
  | ListPodSandbox / ListContainers / Status
  v
containerd
```

Sau đó PLEG so sánh:

```text
Trạng thái lần trước:
Container nginx = Running

Trạng thái lần này:
Container nginx = Exited
```

Nó tạo lifecycle event:

```text
ContainerDied
```

Event này đánh thức phần xử lý Pod:

```text
PLEG phát hiện container chết
            ↓
Sync Loop nhận event
            ↓
pod worker của Pod chạy
            ↓
SyncPod kiểm tra restartPolicy
            ↓
kubelet yêu cầu containerd restart
```

---

# 9. Tại sao cần PLEG khi kubelet là người tạo container?

Vì sau khi container được tạo, rất nhiều thứ có thể xảy ra ngoài lời gọi ban đầu của kubelet.

Ví dụ:

- application tự crash;
    
- kernel OOM-kill container;
    
- container runtime restart;
    
- process trong container tự exit;
    
- node bị thiếu tài nguyên;
    
- một thao tác mức runtime làm container biến mất.
    

Kubelet từng yêu cầu:

```text
StartContainer
```

không có nghĩa container sẽ sống mãi.

Nó cần liên tục quan sát runtime để biết thực tế hiện giờ là gì.

---

# 10. Ví dụ đầy đủ: tạo một Pod

Bạn chạy:

```bash
kubectl run nginx --image=nginx
```

Luồng gần đúng:

```text
kubectl
   |
   | POST Pod
   v
API Server
   |
   | lưu Pod vào etcd
   v
Scheduler
   |
   | chọn fdp-k8s-wk1
   | cập nhật spec.nodeName
   v
API Server
   |
   | watch event
   v
kubelet trên wk1
   |
   | Sync Loop nhận Pod mới
   v
pod worker
   |
   | SyncPod()
   v
containerd qua CRI
   |
   | RunPodSandbox
   | PullImage
   | CreateContainer
   | StartContainer
   v
container chạy
```

Sau đó kubelet cập nhật status:

```text
kubelet
   |
   | PATCH/UPDATE Pod status
   v
API Server
```

Bạn chạy:

```bash
kubectl get pod
```

thì `kubectl` đọc status từ API Server, không trực tiếp hỏi containerd.

---

# 11. Ví dụ container bị crash

Giả sử process nginx bị lỗi:

```text
nginx process exit
        ↓
containerd thấy container stopped
        ↓
PLEG poll containerd
        ↓
PLEG phát hiện Running → Exited
        ↓
tạo ContainerDied event
        ↓
Sync Loop kích hoạt pod worker
        ↓
SyncPod kiểm tra restartPolicy
        ↓
kubelet yêu cầu containerd start lại
```

Nếu lỗi liên tục:

```text
restart
crash
restart
crash
```

kubelet áp dụng backoff và Pod có thể hiện:

```text
CrashLoopBackOff
```

`CrashLoopBackOff` không phải container state thật trong runtime. Nó là trạng thái Kubernetes mô tả rằng kubelet đang trì hoãn lần restart tiếp theo vì container crash liên tục.

---

# 12. Vì sao `kubectl get pod` có thể chậm hơn thực tế?

Giả sử container chết tại thời điểm:

```text
14:00:00.000
```

Nhưng trình tự có thể là:

```text
14:00:00.000 container thực tế chết
14:00:00.200 runtime ghi nhận trạng thái
14:00:00.800 PLEG poll và phát hiện
14:00:01.000 kubelet xử lý event
14:00:01.300 kubelet update Pod status lên API Server
14:00:01.500 kubectl đọc status mới
```

Trong khoảng rất ngắn đó:

```text
container thực tế: đã chết
kubectl get pod: vẫn có thể hiện Running
```

Bởi vì `kubectl` nhìn vào:

```text
PodStatus trong API Server
```

chứ không nhìn trực tiếp vào:

```text
trạng thái process ngay trong kernel/containerd
```

Luồng status là:

```text
Linux process
    ↓
container runtime
    ↓
PLEG / kubelet
    ↓
API Server
    ↓
kubectl
```

Mỗi bước đều có thể tạo ra một chút độ trễ.
