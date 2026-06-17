### Giới thiệu: Tại sao cần quản lý tài nguyên?

Hãy tưởng tượng bạn có một cụm Kubernetes (cluster) giống như một đội xe tải (Nodes), và các ứng dụng của bạn (Pods) là những chiếc hộp hàng cần được xếp lên xe.

Nếu bạn không khai báo kích thước của các hộp hàng, người điều phối (Kubernetes Scheduler) sẽ xếp chúng một cách ngẫu nhiên. Điều này có thể dẫn đến:

- Một chiếc xe tải bị nhồi nhét quá nhiều hàng, bị quá tải và hỏng hóc giữa đường (Node bị sập).
    
- Một chiếc hộp quá lớn được xếp lên một chiếc xe tải nhỏ không đủ chỗ (Pod không thể chạy).
    

Quản lý tài nguyên là cách bạn "dán nhãn" kích thước lên mỗi hộp hàng (Pod/Container) để người điều phối có thể xếp chúng lên xe một cách thông minh và hiệu quả, đảm bảo mọi thứ vận hành trơn tru.

---

### Request vs. Limit: Hai Khái Niệm Cốt Lõi

Đây là hai khái niệm quan trọng nhất bạn cần nắm vững. Chúng được định nghĩa cho mỗi container.

#### 📦 Request (Yêu cầu)

- **Ý nghĩa:** Đây là lượng tài nguyên **tối thiểu được đảm bảo** mà container của bạn sẽ có.
    
- **Ai sử dụng:** **Kubernetes Scheduler** (Người điều phối).
    
- **Analogy:** Khi bạn gửi hàng, bạn nói: "Cái hộp này cần một chỗ **tối thiểu rộng 50cm**." Người điều phối sẽ tìm một chiếc xe tải (Node) còn trống ít nhất 50cm để xếp hộp của bạn lên.
    
- **Tóm lại:** `Request` được dùng để **lên lịch (scheduling)**. Nó đảm bảo Pod của bạn có đủ tài nguyên để khởi động và chạy một cách ổn định.
    

#### ⛔ Limit (Giới hạn)

- **Ý nghĩa:** Đây là lượng tài nguyên **tối đa tuyệt đối** mà container của bạn được phép sử dụng.
    
- **Ai sử dụng:** **Kubelet** (Người giám sát trên mỗi Node).
    
- **Analogy:** Bạn dặn người tài xế: "Cái hộp này dù có thể phình ra, nhưng **tuyệt đối không được chiếm quá 100cm**."
    
- **Tóm lại:** `Limit` được dùng để **thi hành (enforcement)**. Nó ngăn một container "tham lam" chiếm hết tài nguyên của Node, ảnh hưởng đến các container khác.
    

**Một quy tắc quan trọng:**

- Container được phép sử dụng nhiều hơn `request` nếu Node còn tài nguyên rảnh rỗi.
    
- Container **KHÔNG BAO GIỜ** được phép sử dụng vượt quá `limit`.
    

**Lưu ý:** Nếu bạn chỉ định `limit` mà không chỉ định `request`, Kubernetes sẽ tự động đặt `request` bằng với `limit`.

---

### Cách Kubernetes Xử Lý Request và Limit

Cách Kubernetes thi hành giới hạn cho CPU và Memory hơi khác nhau một chút:

#### CPU

- **Cách thi hành:** **CPU Throttling (Bóp băng thông CPU)**.
    
- **Điều gì xảy ra:** Khi container của bạn cố gắng sử dụng nhiều CPU hơn `limit`, Kernel của Linux sẽ "bóp" nó lại, làm cho ứng dụng của bạn chạy chậm đi trong khoảng thời gian đó. Ứng dụng **không bị "chết"**. Đây là một giới hạn cứng.
    

#### Memory (RAM)

- **Cách thi hành:** **OOM Kill (Out Of Memory Kill)**.
    
- **Điều gì xảy ra:** Khi container của bạn sử dụng vượt quá `memory` `limit`, nó sẽ bị đưa vào "danh sách đen". Nếu Kernel phát hiện Node đang bị áp lực về bộ nhớ (thiếu RAM), nó sẽ ra tay và **"giết" (terminate)** một trong các process trong container của bạn. Nếu process đó là process chính (PID 1), container sẽ bị khởi động lại.
    
- **Tính chất:** Đây là hành động mang tính **phản ứng (reactive)**. Container có thể tạm thời dùng lố `limit` một chút nếu Node còn nhiều RAM, nhưng nếu nó làm vậy, nó có nguy cơ bị "giết" bất cứ lúc nào.
    

---

### Các Loại Tài Nguyên (Resource Types)

#### 1. CPU và Memory (Tài nguyên tính toán)

Đây là hai loại phổ biến nhất.

- **Đơn vị CPU:**
    
    - Được đo bằng đơn vị "Kubernetes CPUs". `1` CPU tương đương với 1 core vật lý hoặc 1 vCPU.
        
    - Bạn có thể dùng các đơn vị nhỏ hơn, gọi là "millicpu" hoặc "millicores".
        
    - `1000m` = `1` CPU.
        
    - `500m` = 0.5 CPU.
        
    - `250m` = 0.25 CPU.
        
    - **Lưu ý:** Kubernetes không cho phép độ chính xác nhỏ hơn `1m`. `0.5m` là không hợp lệ.
        
- **Đơn vị Memory:**
    
    - Được đo bằng bytes.
        
    - Bạn có thể dùng các hậu tố thông thường (E, P, T, G, M, k) hoặc hậu tố lũy thừa của 2 (Ei, Pi, Ti, Gi, Mi, Ki).
        
    - `128Mi` (Mebibytes) = 128 * 2^20 bytes.
        
    - `129M` (Megabytes) = 129 * 10^6 bytes.
        
    - **Cảnh báo phổ biến:** `400m` memory có nghĩa là 0.4 bytes, không phải 400 megabytes! Hãy luôn dùng `M` hoặc `Mi`.
        

#### 2. Local Ephemeral Storage (Bộ nhớ tạm cục bộ)

- **Nó là gì:** Đây là không gian đĩa cứng trên chính Node đó, được dùng cho các mục đích tạm thời như:
    
    - `emptyDir` volumes (không phải loại `tmpfs`).
        
    - Lớp có thể ghi (writable layer) của container.
        
    - Log của container.
        
- **Tại sao cần quản lý:** Nếu một Pod ghi quá nhiều log hoặc dữ liệu tạm, nó có thể làm đầy ổ đĩa của Node, gây sập cả Node.
    
- **Hành động khi vượt limit:** Nếu một Pod sử dụng vượt quá `limit` của `ephemeral-storage`, Kubelet sẽ đánh dấu Pod đó để **trục xuất (eviction)**, tức là "đuổi" nó đi để bảo vệ Node.
    

#### 3. Extended Resources (Tài nguyên mở rộng)

Dùng cho các tài nguyên không được tích hợp sẵn, thường là phần cứng đặc biệt như GPU, FPGA...

- **Cách hoạt động:**
    
    1. Quản trị viên cluster phải "quảng cáo" tài nguyên này trên Node (ví dụ: "Node này có 2 GPU `nvidia.com/gpu`").
        
    2. Người dùng yêu cầu tài nguyên đó trong Pod spec, giống như CPU/Memory. `requests` và `limits` cho tài nguyên mở rộng phải bằng nhau.
        

---

### Ví dụ về cấu hình tài nguyên

Pod sau có 2 container. Tổng `request` của Pod là `500m` CPU và `128Mi` Memory. Tổng `limit` của Pod là `1` CPU và `256Mi` Memory.

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: my-app:v4
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: log-aggregator
    image: my-log-aggregator:v6
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

---

### Gỡ rối (Troubleshooting)

#### 1. Pod của tôi bị Pending với lỗi `FailedScheduling`

Điều này có nghĩa là Scheduler không thể tìm thấy Node nào trong cluster có đủ tài nguyên (dựa trên `request` của Pod) để chạy nó.

**Làm sao để kiểm tra?**

Bash

```
kubectl describe pod my-pending-pod
```

Nhìn vào phần `Events`, bạn sẽ thấy thông báo như: `0/5 nodes available: insufficient cpu`.

**Cách giải quyết:**

- Thêm Node mới vào cluster.
    
- Xóa bớt các Pod không cần thiết để giải phóng tài nguyên.
    
- Kiểm tra lại xem `request` của bạn có quá lớn so-với-sức-chứa của các Node không.
    
- Kiểm tra `Node taints` (có thể Node có tài nguyên nhưng bị "đánh dấu" không cho Pod thông thường chạy trên đó).
    

#### 2. Container của tôi liên tục bị chấm dứt và khởi động lại

Đây rất có thể là do nó đã vi phạm `limit`.

**Làm sao để kiểm tra?**

Bash

```
kubectl describe pod my-restarting-pod
```

Nhìn vào phần `Containers` -> `Last State` (Trạng thái lần cuối), nếu bạn thấy:

```
Last State:     Terminated
  Reason:       OOMKilled
```

`OOMKilled` (Out Of Memory Killed) là bằng chứng rõ ràng cho thấy container đã cố gắng dùng nhiều Memory hơn `limit` của nó và đã bị Kernel "giết".

**Cách giải quyết:**

- Kiểm tra code ứng dụng xem có bị rò rỉ bộ nhớ (memory leak) không.
    
- Nếu ứng dụng hoạt động đúng, hãy xem xét tăng `memory` `limit` (và cả `request`) cho nó.
    

Bằng cách hiểu rõ và áp dụng đúng `request` và `limit`, bạn có thể đảm bảo các ứng dụng của mình chạy một cách ổn định, hiệu quả và không "gây sự" với các ứng dụng khác trong cùng một cluster.