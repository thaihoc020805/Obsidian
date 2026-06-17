### StatefulSet là gì? Tưởng tượng cho dễ hiểu

Hãy bắt đầu bằng một ví dụ đơn giản.

**1. Ứng dụng "Vô danh" (Stateless) - Dùng `Deployment`**

Tưởng tượng bạn có một trang web tin tức đơn giản. Bạn cần 3 máy chủ web để chạy nó. Cả 3 máy chủ này đều giống hệt nhau, không có gì đặc biệt. Nếu một máy chủ bị sập, hệ thống chỉ cần tạo một cái mới thay thế là xong. Người dùng không hề biết và cũng không quan tâm họ đang truy cập vào máy chủ nào. Chúng giống như những nhân viên trong một tổng đài, ai cũng có thể trả lời cuộc gọi như nhau.

Đây được gọi là ứng dụng **stateless** (không trạng thái), và chúng ta thường dùng `Deployment` hoặc `ReplicaSet` để quản lý chúng. Các Pods là những bản sao vô danh, có thể thay thế cho nhau.

**2. Ứng dụng "Có định danh" (Stateful) - Dùng `StatefulSet`**

Bây giờ, hãy tưởng tượng bạn đang chạy một hệ thống cơ sở dữ liệu (database) có 3 nút (nodes): một nút chính (master) và hai nút phụ (replicas).

- **Nút chính** có vai trò đặc biệt: nó ghi dữ liệu.
    
- **Các nút phụ** sao chép dữ liệu từ nút chính.
    
- Mỗi nút phải có **ổ cứng riêng** để lưu dữ liệu của nó. Nếu nút chính bị sập và khởi động lại, nó phải kết nối lại đúng với ổ cứng cũ của mình để không mất dữ liệu.
    
- Các nút cần có **tên miền (network name) cố định** để chúng có thể tìm thấy và giao tiếp với nhau. Nút phụ luôn phải biết địa chỉ của nút chính.
    

Những Pods này **không thể thay thế cho nhau**. Mỗi Pod có một "danh tính" riêng biệt và quan trọng. Đây chính là lúc chúng ta cần **StatefulSet**.

**Tóm lại:** `StatefulSet` được sinh ra để quản lý các ứng dụng mà mỗi Pod trong đó có một danh tính độc nhất và không đổi, cần lưu trữ dữ liệu bền vững và có thứ tự triển khai/xóa được đảm bảo.

### Các đặc điểm Vàng của StatefulSet

StatefulSet cung cấp 3 đảm bảo chính mà `Deployment` không có:

#### 1. Danh tính Mạng Ổn định và Duy nhất (Stable, Unique Network Identifiers)

Mỗi Pod do StatefulSet tạo ra sẽ có một cái tên (hostname) cố định và có thể dự đoán được.

- **Cú pháp tên Pod:** `$(tên-statefulset)-$(số-thứ-tự)`
    
- **Ví dụ:** Nếu StatefulSet của bạn tên là `web` và có `replicas: 3`, nó sẽ tạo ra các Pod tên là: `web-0`, `web-1`, `web-2`.
    

Để làm được điều này, StatefulSet yêu cầu một thứ gọi là **Headless Service**.

- **Headless Service là gì?** Là một Service bình thường nhưng có `clusterIP: None`. Thay vì cung cấp một địa chỉ IP duy nhất để cân bằng tải (load balance) đến các Pod, nó lại tạo ra một bản ghi DNS cho **từng Pod**.
    
- **Ví dụ:** Pod `web-0` sẽ có một địa chỉ DNS riêng là `web-0.nginx.default.svc.cluster.local`. Nhờ vậy, các Pod khác luôn có thể tìm thấy `web-0` bằng cái tên cố định này, kể cả khi `web-0` bị sập và được khởi động lại ở một node vật lý khác.
    

#### 2. Lưu trữ Ổn định và Bền vững (Stable, Persistent Storage)

Mỗi Pod sẽ được gắn với một ổ đĩa lưu trữ riêng biệt và cố định (`PersistentVolumeClaim` - PVC).

- Pod `web-0` sẽ dùng ổ đĩa `www-web-0`.
    
- Pod `web-1` sẽ dùng ổ đĩa `www-web-1`.
    
- Pod `web-2` sẽ dùng ổ đĩa `www-web-2`.
    

Điều này được định nghĩa trong mục `volumeClaimTemplates` của file YAML. Nó giống như một cái "khuôn" để tự động tạo ra một PVC riêng cho mỗi Pod.

**Quan trọng:** Nếu Pod `web-1` bị lỗi và được tạo lại, nó sẽ tự động kết nối lại với đúng ổ đĩa `www-web-1`. Dữ liệu của nó vẫn còn nguyên vẹn!

#### 3. Triển khai và Mở rộng có Thứ tự (Ordered, Graceful Deployment and Scaling)

Đây là một đặc điểm cực kỳ quan trọng cho các ứng dụng stateful.

- **Khi tạo hoặc Scale Up (tăng số lượng):** Các Pod được tạo ra tuần tự, từ số thứ tự nhỏ đến lớn. `web-0` sẽ được tạo trước. Chỉ khi `web-0` đã `Running` và `Ready` (sẵn sàng hoạt động), `web-1` mới được tạo. Tương tự, `web-2` chỉ được tạo sau khi `web-1` đã sẵn sàng.
    
- **Khi xóa hoặc Scale Down (giảm số lượng):** Quá trình diễn ra ngược lại, từ số thứ tự lớn đến nhỏ. `web-2` sẽ bị xóa trước. Chỉ khi `web-2` đã bị xóa hoàn toàn, `web-1` mới bắt đầu bị xóa, và cứ thế.
    

Điều này đảm bảo rằng ứng dụng có thể tắt hoặc mở rộng một cách an toàn. Ví dụ, với database, bạn sẽ muốn các nút phụ (replica) tắt trước, và nút chính (master) tắt sau cùng.

### Phân tích file YAML ví dụ

Hãy cùng "mổ xẻ" file YAML để thấy rõ các thành phần:

```YAML
# Phần 1: Headless Service - Tạo danh tính mạng
apiVersion: v1
kind: Service
metadata:
  name: nginx  # Tên của Service
spec:
  clusterIP: None # <-- ĐÂY LÀ ĐIỂM MẤU CHỐT! Biến nó thành Headless Service.
  selector:
    app: nginx # Service này sẽ tìm các Pod có label là "app: nginx"
---
# Phần 2: StatefulSet - Quản lý các Pod có định danh
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web # Tên của StatefulSet
spec:
  serviceName: "nginx" # <-- Rất quan trọng! Chỉ định Headless Service dùng để tạo DNS
  replicas: 3 # Tạo ra 3 Pod
  selector:
    matchLabels:
      app: nginx # StatefulSet này quản lý các Pod có label "app: nginx"
  template: # <-- Khuôn mẫu để tạo ra các Pod
    metadata:
      labels:
        app: nginx # Label của Pod, phải khớp với selector ở trên
    spec:
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.24
        ports:
        - containerPort: 80
          name: web
        volumeMounts: # <-- Gắn ổ đĩa vào container
        - name: www # Tên của volume mount
          mountPath: /usr/share/nginx/html # Đường dẫn bên trong container
  volumeClaimTemplates: # <-- Khuôn mẫu để tự động tạo ổ đĩa cho mỗi Pod
  - metadata:
      name: www # Tên của ổ đĩa (PersistentVolumeClaim)
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi # Mỗi Pod sẽ có một ổ đĩa 1GiB riêng
```

### Những điểm cần lưu ý (Limitations)

1. **Ổ đĩa không tự xóa:** Khi bạn xóa StatefulSet hoặc giảm `replicas`, các ổ đĩa (`PVC`) tương ứng **sẽ không bị xóa**. Đây là một biện pháp an toàn để tránh mất dữ liệu. Bạn phải xóa chúng thủ công nếu không cần nữa. (Trừ khi bạn cấu hình `persistentVolumeClaimRetentionPolicy` trên các phiên bản Kubernetes mới).
    
2. **Bắt buộc phải có Headless Service:** Bạn phải tự tay tạo ra nó. StatefulSet cần nó để hoạt động.
    
3. **Cập nhật (Update Strategies):**
    
    - `RollingUpdate` (mặc định): Tự động cập nhật các Pod theo thứ tự ngược (từ lớn đến nhỏ). Rất tiện lợi.
        
    - `OnDelete`: StatefulSet sẽ không tự làm gì cả. Bạn phải tự tay xóa một Pod cũ đi, sau đó controller mới tạo một Pod mới với cấu hình đã cập nhật.


Retention Policy (xóa/giữ PVC khi delete/scale)

yaml

CopyEdit

`spec:   persistentVolumeClaimRetentionPolicy:     whenDeleted: Retain | Delete     whenScaled:  Retain | Delete`

- **Retain (mặc định)**: **không** xóa PVC khi xóa StatefulSet/scale down.
    
- **Delete**: **xóa** PVC tương ứng **sau khi** Pod bị xóa (an toàn tháo mount trước).
    

> Lưu ý: chính sách chỉ tác động khi **xóa StatefulSet** hoặc **scale down**. Không đụng tới trường hợp Pod chết do node lỗi (PVC vẫn giữ nguyên và gắn lại vào Pod thay thế).

Pod Management Policy

- **OrderedReady** (mặc định): theo thứ tự (an toàn, phù hợp DB).
    
- **Parallel**: tạo/xóa **song song** khi scale (nhanh hơn, nhưng **không** áp dụng cho rolling update).


### 1. Với Service "Bình Thường" (Loại `ClusterIP`)

Hãy tưởng tượng một Service bình thường giống như **số tổng đài của một công ty**.

- Khi bạn tạo một Service tên là `my-app-service`, Kubernetes sẽ cho nó một địa chỉ IP ảo duy nhất (gọi là `ClusterIP`), ví dụ `10.96.100.200`.
    
- Service này sẽ theo dõi tất cả các Pod khỏe mạnh mà nó quản lý (ví dụ: `pod-a`, `pod-b`, `pod-c`).
    
- Khi bạn gửi một yêu cầu đến IP `10.96.100.200`, Service sẽ đóng vai trò **cân bằng tải (load balancer)**: nó sẽ chuyển tiếp yêu cầu của bạn đến **một trong số** các Pod (`pod-a` hoặc `pod-b` hoặc `pod-c`).
    
- **Về DNS:** Tên miền `my-app-service.default.svc.cluster.local` sẽ được phân giải ra địa chỉ IP ảo duy nhất là `10.96.100.200`.
    

**Kết luận:** Với Service bình thường, bạn chỉ có một địa chỉ chung. Bạn không thể chọn nói chuyện với một Pod cụ thể nào. Giống như gọi đến tổng đài, bạn chỉ được nối máy đến một nhân viên bất kỳ đang rảnh.

### 2. Với Headless Service (`clusterIP: None`)

Bây giờ, hãy tưởng tượng Headless Service giống như **danh bạ nội bộ của công ty**. Nó không có số tổng đài chung, thay vào đó, nó cung cấp cho bạn số máy lẻ (địa chỉ IP) của từng nhân viên (từng Pod).

Khi bạn tạo một Headless Service (bằng cách đặt `clusterIP: None`):

1. **Không có IP ảo, không cân bằng tải:** Service này sẽ không có `ClusterIP`.
    
2. **Nó biến thành một máy chủ DNS:** Thay vì trả về một IP duy nhất, khi bạn truy vấn tên của Service, nó sẽ trả về **một danh sách các địa chỉ IP của tất cả các Pod** mà nó đang quản lý.
    
    - Truy vấn DNS cho `nginx.default.svc.cluster.local` sẽ trả về:
        
        - IP của Pod `web-0` (ví dụ: `10.1.2.3`)
            
        - IP của Pod `web-1` (ví dụ: `10.1.2.4`)
            
        - IP của Pod `web-2` (ví dụ: `10.1.2.5`)
            
3. **Và đây là điều kỳ diệu nhất - "Bản ghi DNS cho từng Pod":** Kết hợp với StatefulSet, Headless Service còn làm một việc đặc biệt hơn: Nó tạo ra một **bản ghi DNS A** riêng biệt và ổn định cho mỗi Pod theo cú pháp:
    
    `$(tên-pod).$(tên-service).$(namespace).svc.cluster.local`
    
    Áp dụng vào ví dụ của chúng ta:
    
    - Tên miền `web-0.nginx.default.svc.cluster.local` sẽ được phân giải **trực tiếp** ra IP của Pod `web-0` (là `10.1.2.3`).
        
    - Tên miền `web-1.nginx.default.svc.cluster.local` sẽ được phân giải **trực tiếp** ra IP của Pod `web-1` (là `10.1.2.4`).
        
    - Tên miền `web-2.nginx.default.svc.cluster.local` sẽ được phân giải **trực tiếp** ra IP của Pod `web-2` (là `10.1.2.5`).
        

### Tại sao điều này lại Cực kỳ Quan trọng?

Bởi vì nó cho phép các Pod giao tiếp **trực tiếp và có chủ đích** với nhau.

Hãy quay lại ví dụ về cơ sở dữ liệu với một nút master (`db-0`) và các nút replica (`db-1`, `db-2`).

- Các nút replica (`db-1`, `db-2`) cần phải biết **chính xác địa chỉ** của nút master (`db-0`) để sao chép dữ liệu.
    
- Nhờ Headless Service, Pod `db-1` có thể luôn luôn kết nối đến Pod `db-0` bằng cách sử dụng tên DNS cố định là `db-0.mysql.default.svc.cluster.local`.
    
- **Quan trọng nhất:** Nếu Pod `db-0` bị sập và được Kubernetes khởi động lại ở một node khác với một địa chỉ IP mới (ví dụ `10.1.5.6`), Kubernetes sẽ **tự động cập nhật** bản ghi DNS. Tên miền `db-0.mysql.default.svc.cluster.local` bây giờ sẽ trỏ đến IP mới `10.1.5.6`. Pod `db-1` không cần thay đổi bất cứ gì cả, nó vẫn kết nối đến cái tên quen thuộc đó và mọi thứ lại hoạt động bình thường.

Chắc chắn rồi! Hiểu được luồng chi tiết này chính là chìa khóa để nắm vững StatefulSet. Chúng ta sẽ đi từng bước một, xem các thành phần của Kubernetes nói chuyện với nhau như thế nào.

**Bối cảnh:**

- Bạn có một cluster Kubernetes đang hoạt động.
    
- Bên trong cluster có một "dịch vụ DNS" đang chạy, thường là **CoreDNS**. Hãy coi CoreDNS như một cuốn **danh bạ điện thoại** cho toàn bộ cluster.
    
- Bạn chuẩn bị chạy lệnh `kubectl apply -f my-statefulset.yaml`.
    
- File YAML của bạn định nghĩa 2 thứ:
    
    1. Một **Headless Service** tên là `nginx`.
        
    2. Một **StatefulSet** tên là `web`, yêu cầu `replicas: 2` và được liên kết với service `nginx`.
        

---

### Luồng Chi Tiết Từng Bước

**Bước 1: Người dùng Gửi Yêu Cầu (`kubectl apply`)**

Bạn gõ lệnh `kubectl apply...`. File YAML được gửi đến **API Server** của Kubernetes. API Server nhận được yêu cầu tạo 2 tài nguyên: một Service và một StatefulSet.

**Bước 2: Xử Lý Headless Service "nginx"**

1. API Server lưu định nghĩa của Service `nginx` vào cơ sở dữ liệu của nó (etcd).
    
2. Một controller chuyên về Service sẽ thấy có Service mới. Nó đọc cấu hình và thấy `clusterIP: None`.
    
3. Controller này sẽ **không** tạo ra một IP ảo. Thay vào đó, nó tương tác với **CoreDNS** (cuốn danh bạ) và thông báo:
    
    > "Này CoreDNS, hãy chuẩn bị quản lý một tên miền mới là `nginx.default.svc.cluster.local`. Tên miền này sẽ không trỏ đến một IP duy nhất, mà sẽ là một danh sách các IP. Khi nào có các Pod phù hợp xuất hiện, tôi sẽ báo cho anh biết IP của chúng."
    

Tại thời điểm này, chưa có Pod nào tồn tại, nên nếu bạn truy vấn `nginx.default.svc.cluster.local`, nó sẽ không trả về kết quả gì.

**Bước 3: Xử Lý StatefulSet "web" - Bắt đầu tạo Pod `web-0`**

1. **StatefulSet Controller** (một bộ não khác trong Kubernetes) thấy có StatefulSet `web` mới được tạo.
    
2. Nó đọc cấu hình: `replicas: 2`, `serviceName: "nginx"`.
    
3. Nó tuân theo quy tắc thứ tự, bắt đầu với Pod đầu tiên: "OK, tôi cần tạo Pod `web-0`."
    
4. Nó tạo ra một bản thiết kế (manifest) cho Pod `web-0` dựa trên phần `template` trong file YAML.
    
5. Bản thiết kế này được gửi đi để lên lịch chạy.
    

**Bước 4: Pod `web-0` được Tạo và có Địa chỉ IP**

1. **Scheduler** (người điều phối) nhận bản thiết kế của `web-0` và quyết định chạy nó trên một Node phù hợp, ví dụ `node-A`.
    
2. **Kubelet** trên `node-A` nhận lệnh, kéo image container về và khởi chạy Pod `web-0`.
    
3. Hệ thống mạng của Kubernetes cấp cho Pod `web-0` một địa chỉ IP duy nhất trong cluster, ví dụ: `10.244.1.5`.
    
4. Trạng thái của `web-0` được cập nhật về API Server: "Tôi đang chạy và IP của tôi là `10.244.1.5`."
    

**Bước 5: PHÉP MÀU DNS XẢY RA cho `web-0`**

Đây là bước quan trọng nhất!

1. Hệ thống điều khiển của Kubernetes (Control Plane) nhận thấy sự kiện: "Pod `web-0` với IP `10.244.1.5` đã khởi động".
    
2. Nó kiểm tra các label của Pod này và thấy label `app: nginx`.
    
3. Nó thấy rằng label này khớp với `selector` của Headless Service `nginx`.
    
4. Vì Pod `web-0` thuộc StatefulSet `web` được liên kết với Service `nginx`, Control Plane liền ra lệnh cho **CoreDNS**:
    
    > "Này CoreDNS, cập nhật danh bạ đi! Hãy tạo một bản ghi **A** (Address record) mới như sau:"
    
    > `web-0.nginx.default.svc.cluster.local. IN A 10.244.1.5`
    
    > "Đồng thời, hãy thêm IP `10.244.1.5` này vào danh sách cho tên miền chung `nginx.default.svc.cluster.local`."
    

**Kết quả:** Ngay từ giây phút này, bất kỳ Pod nào khác trong cluster cũng có thể tìm thấy `web-0` bằng cách hỏi CoreDNS tên miền `web-0.nginx.default.svc.cluster.local` và sẽ nhận được câu trả lời là `10.244.1.5`.

**Bước 6: Lặp lại quy trình cho Pod `web-1`**

1. StatefulSet Controller thấy `web-0` đã ở trạng thái `Running` và `Ready` (sẵn sàng).
    
2. Nó nói: "Tốt, `web-0` đã xong. Giờ đến lượt `web-1`."
    
3. **Lặp lại Bước 4 và 5 cho `web-1`**:
    
    - `web-1` được Scheduler đặt lên một Node (ví dụ `node-B`).
        
    - `web-1` được cấp một IP mới, ví dụ: `10.244.2.8`.
        
    - Control Plane lại ra lệnh cho **CoreDNS**:
        
        > "CoreDNS, thêm một bản ghi nữa:" `web-1.nginx.default.svc.cluster.local. IN A 10.244.2.8` "Và cũng thêm IP `10.244.2.8` vào danh sách của `nginx.default.svc.cluster.local`."
        

### Tổng Kết Luồng

1. **Headless Service** đóng vai trò **đăng ký một "không gian tên" DNS** và **đặt ra luật chơi** (dựa trên selector).
    
2. **StatefulSet Controller** lần lượt **tạo ra các Pod** theo đúng thứ tự.
    
3. Mỗi khi một Pod được tạo ra, có IP và khớp với luật chơi của Headless Service, **Control Plane** sẽ đóng vai trò **người cập nhật danh bạ**, ra lệnh cho **CoreDNS** tạo ra một bản ghi DNS cụ thể trỏ đến IP của Pod đó.

