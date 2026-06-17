### DaemonSet là gì? Tưởng tượng cho dễ hiểu

Để hiểu DaemonSet, hãy tưởng tượng bạn là người quản lý một khu phức hợp lớn gồm nhiều tòa nhà (đây là **Cluster Kubernetes** của bạn). Mỗi tòa nhà là một **Node**.

Bây giờ, bạn có một số nhiệm vụ cần phải được thực hiện ở **mọi tòa nhà**:

1. **Lắp camera an ninh**: Để giám sát mọi hoạt động (giống như **node monitoring**).
    
2. **Đặt một nhân viên thu gom rác**: Để thu gom rác thải hàng ngày (giống như **logs collection**).
    
3. **Đặt một nhân viên bảo vệ**: Để đảm bảo an ninh cho tòa nhà (giống như một **agent an ninh mạng**).
    

Bạn sẽ làm gì? Bạn sẽ không thuê một đội 10 nhân viên rồi bảo họ "cứ đi đâu làm thì làm". Thay vào đó, bạn sẽ ra một quy tắc: **"Mỗi tòa nhà PHẢI CÓ và CHỈ CÓ một nhân viên cho mỗi nhiệm vụ trên."**

**DaemonSet chính là cái QUY TẮC đó trong Kubernetes.**

> **DaemonSet** đảm bảo rằng một bản sao của Pod sẽ được chạy trên **mỗi Node** (hoặc một số Node được chỉ định) trong cluster.
> 
> - Khi bạn thêm một Node mới vào cluster, DaemonSet sẽ tự động triển khai Pod đó lên Node mới.
>     
> - Khi bạn xóa một Node khỏi cluster, Pod trên Node đó sẽ tự động bị dọn dẹp.
>     

Nó không quan tâm đến `replicas` (số lượng bản sao) như `Deployment`. Nó chỉ quan tâm đến **độ bao phủ** trên các Node.

### So sánh nhanh: DaemonSet vs. Deployment

|Tiêu chí|Deployment|DaemonSet|
|---|---|---|
|**Mục tiêu**|Duy trì một **số lượng Pod** nhất định trong toàn cluster.|Đảm bảo **mỗi Node** có một bản sao của Pod.|
|**Số lượng Pod**|Do bạn quyết định (ví dụ: `replicas: 5`).|Bằng số lượng Node (mà nó quản lý).|
|**Ví dụ**|Chạy 5 Pod cho web server frontend.|Chạy agent thu thập log trên mọi Node.|
|**Tương tự**|"Cần 5 nhân viên thu ngân cho cả siêu thị."|"Cần 1 nhân viên bảo vệ ở mỗi cửa ra vào."|

Xuất sang Trang tính

### Các trường hợp sử dụng điển hình

Như trong ví dụ, DaemonSet hoàn hảo cho các tác vụ "cơ sở hạ tầng" cấp Node:

- **Thu thập log**: Chạy một agent như `Fluentd` hoặc `Logstash` trên mỗi Node để thu thập log từ tất cả các Pod khác trên Node đó và gửi về một nơi tập trung.
    
- **Giám sát (Monitoring)**: Chạy một agent như `Prometheus Node Exporter` hoặc `Datadog agent` trên mỗi Node để thu thập các chỉ số về CPU, Memory, Disk... của chính Node đó.
    
- **Mạng (Networking)**: Các plugin mạng như Calico, Flannel thường sử dụng DaemonSet để chạy một agent trên mỗi Node, đảm bảo việc kết nối mạng trong cluster hoạt động.
    
- **Lưu trữ (Storage)**: Chạy các daemon lưu trữ như `GlusterFS` hoặc `Ceph` trên mỗi Node để cung cấp dịch vụ lưu trữ phân tán.
    

### Phân tích file YAML ví dụ

Hãy cùng "mổ xẻ" file YAML `fluentd-elasticsearch` để xem nó hoạt động như thế nào.

YAML

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template: # <--- Khuôn mẫu Pod, giống hệt Deployment/StatefulSet
    metadata:
      labels:
        name: fluentd-elasticsearch # Phải khớp với selector ở trên
    spec:
      # --- Phần 1: "Siêu năng lực" đặc biệt ---
      tolerations:
      # Toleration này cho phép Pod chạy trên cả các Node control-plane
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      
      # --- Phần 2: Cấu hình container ---
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v5.0.1
        # ...cấu hình resources...
        volumeMounts:
        # Gắn thư mục /var/log của Node vào trong container
        - name: varlog
          mountPath: /var/log
      
      # --- Phần 3: Định nghĩa Volume đặc biệt ---
      volumes:
      - name: varlog
        hostPath: # <--- Đây là điểm mấu chốt!
          path: /var/log
```

#### Điểm đáng chú ý trong YAML:

1. **`kind: DaemonSet`**: Khai báo đây là một DaemonSet.
    
2. **`tolerations`**: Đây là một "siêu năng lực" của DaemonSet.
    
    - **Taint & Toleration là gì?** Tưởng tượng `Taint` là một cái biển "CẤM VÀO" treo trên một Node (ví dụ, các Node `control-plane` thường có biển này để các Pod ứng dụng thông thường không chạy trên đó). `Toleration` là một cái "GIẤY PHÉP ĐẶC BIỆT" cho phép Pod lờ đi cái biển cấm đó và vẫn chạy được.
        
    - **Tại sao DaemonSet cần nó?** DaemonSet thường chạy các dịch vụ nền tảng. Ví dụ, một plugin mạng phải chạy trên Node để Node đó có thể kết nối mạng. Nhưng Node đó lại chưa được coi là `Ready` vì chưa có mạng. Đây là một vòng luẩn quẩn (deadlock). Kubernetes tự động thêm các "giấy phép đặc biệt" này cho Pod của DaemonSet để chúng có thể chạy trên các Node ngay cả khi Node đó chưa sẵn sàng, phá vỡ vòng luẩn quẩn.
        
3. **`volumes` và `hostPath`**:
    
    - Đây là một mẫu rất phổ biến trong DaemonSet. `hostPath` cho phép Pod truy cập **trực tiếp** vào một thư mục trên chính cái Node mà nó đang chạy.
        
    - Trong ví dụ này, container `fluentd` (thu thập log) được cấp quyền truy cập vào thư mục `/var/log` của Node. Nhờ vậy, nó có thể đọc tất cả các file log hệ thống và log của các container khác nằm trong thư mục đó.
        

### Làm thế nào để giao tiếp với các Pod của DaemonSet?

- **Push**: Đây là cách phổ biến nhất. Pod của DaemonSet (ví dụ: `fluentd`) sẽ chủ động gửi dữ liệu (log, metrics) đến một dịch vụ trung tâm khác (ví dụ: Elasticsearch, Prometheus).
    
- **NodeIP và `hostPort`**: Bạn có thể cho Pod của DaemonSet mở một port trực tiếp trên IP của Node. Khi đó, bạn có thể kết nối đến Pod đó thông qua `(địa-chỉ-IP-của-Node):(port)`.
    
- **Service**: Bạn có thể tạo một Service trỏ đến các Pod của DaemonSet. Tuy nhiên, nếu là Service bình thường, nó sẽ kết nối bạn đến một Pod ngẫu nhiên. Nếu bạn cần kết nối đến Pod trên một Node cụ thể, bạn cần dùng các phương pháp khác.
    

### Kết luận

Hãy nhớ về DaemonSet với hình ảnh **người nhân viên bảo vệ ở mỗi tòa nhà**:

- **Mục đích:** Cung cấp các dịch vụ nền tảng, gắn liền với Node.
    
- **Hành vi:** Một Pod trên mỗi Node. Tự động co giãn theo số lượng Node.
    
- **Đặc điểm:** Thường dùng `hostPath` để tương tác với Node và có "siêu năng lực" `Toleration` để chạy ở những nơi đặc biệt.
    

Nó là một công cụ cực kỳ mạnh mẽ và thiết yếu để xây dựng một cluster Kubernetes hoàn chỉnh và đáng tin cậy.