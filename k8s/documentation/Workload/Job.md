
# Kubernetes Job: "Người Thợ" Dùng Một Lần

Kubernetes Job là một khái niệm rất quan trọng và khác biệt so với các loại workload như `Deployment` hay `DaemonSet`.

## Job là gì? Tưởng tượng cho dễ hiểu

> Hãy quay lại với khu phức hợp của chúng ta (Cluster).
> - Bạn đã có **Deployment**: Giống như đội ngũ nhân viên lễ tân làm việc liên tục, không nghỉ.
> - Bạn đã có **DaemonSet**: Giống như đội bảo vệ, mỗi người canh gác ở một tòa nhà cố định.
>
> Bây giờ, bạn có một công việc chỉ cần làm một lần rồi thôi: **"Sơn lại hàng rào của khu phức hợp."**
>
> Bạn sẽ không thuê một nhân viên lễ tân hay một bảo vệ để làm việc này. Thay vào đó, bạn sẽ thuê một người thợ hoặc một đội thợ (Contractor).

**Job trong Kubernetes chính là người thợ đó.** Nó được tạo ra để thực hiện một tác vụ có điểm bắt đầu và điểm kết thúc rõ ràng.

- Job sẽ tạo ra một hoặc nhiều `Pod` để thực hiện công việc.
- Nó sẽ đảm bảo công việc được hoàn thành bằng cách thử lại nếu `Pod` bị lỗi.
- Khi công việc hoàn thành (`Pod` chạy xong và thoát ra với mã thành công), Job sẽ dừng lại. Nó không cố gắng giữ cho `Pod` chạy mãi mãi.

> **Tóm lại:** Nếu `Deployment` là để chạy các dịch vụ "vĩnh cửu", thì `Job` là để chạy các tác vụ "dùng một lần".

---

### Các đặc điểm Vàng của Job

-   **Chạy-để-Hoàn-thành (Run-to-Completion):** Đây là bản chất của Job. Mục tiêu của nó là kết thúc thành công.
-   **Đảm bảo Hoàn thành (Reliable Execution):** Đây là siêu năng lực của Job. Nếu `Pod` đang chạy tác vụ bị lỗi (ví dụ: Node bị sập, `Pod` bị xóa, chương trình bên trong bị crash), Job Controller sẽ tự động tạo một `Pod` mới để tiếp tục công việc cho đến khi nó thành công.
-   **Theo dõi sự Hoàn thành (Completion Tracking):** Job biết khi nào nó đã xong. Nó theo dõi số lượng `Pod` đã hoàn thành thành công.

---

### Phân tích file YAML ví dụ
Hãy xem ví dụ tính số Pi đến 2000 chữ số. Đây là một tác vụ tính toán điển hình cho Job.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  # --- Phần 1: Các tùy chọn điều khiển Job ---
  backoffLimit: 4 # <-- Số lần thử lại tối đa nếu Pod bị lỗi

  # --- Phần 2: Khuôn mẫu cho các Pod "thợ" ---
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      
      # --- Phần 3: Chính sách khởi động lại RẤT QUAN TRỌNG ---
      restartPolicy: Never # <-- Chỉ khởi động lại khi thất bại hoặc không bao giờ
````

**Các điểm đáng chú ý trong YAML:**

- `kind: Job` và `apiVersion: batch/v1`: Khai báo đây là một `Job`. "batch" có nghĩa là "xử lý theo lô".
    
- `restartPolicy: Never`: Đây là một điểm cực kỳ quan trọng và dễ nhầm lẫn.
    
    - `restartPolicy` này áp dụng cho container **bên trong một Pod**.
        
    - Nó chỉ có thể là `Never` hoặc `OnFailure`. **Không bao giờ được là `Always`**. Tại sao? Vì nếu là `Always`, `Pod` sẽ không bao giờ kết thúc, đi ngược lại bản chất của Job.
        
    - `Never`: Nếu container `perl` bị lỗi, đừng khởi động lại container đó. Thay vào đó, hãy coi toàn bộ `Pod` đã thất bại.
        
    - `OnFailure`: Nếu container `perl` bị lỗi, hãy khởi động lại nó bên trong cùng một `Pod`.
        
- `backoffLimit: 4`: Đây là chính sách thử lại ở **cấp độ Job**.
    
    - Nó có nghĩa là: "Nếu một `Pod` bị thất bại (theo `restartPolicy: Never`), hãy tạo một `Pod` mới để thử lại. Nhưng nếu đã thử lại 4 lần mà vẫn thất bại, thì hãy bỏ cuộc và đánh dấu toàn bộ Job là `Failed`."
        
    - Kubernetes sẽ không thử lại ngay lập tức mà sẽ có một khoảng trễ tăng dần (exponential back-off) như 10s, 20s, 40s...
        

> **Tóm tắt sự khác biệt:**
> 
> - **`restartPolicy`**: Quản lý việc khởi động lại container **trong 1 `Pod`**.
>     
> - **`backoffLimit`**: Quản lý việc tạo ra một **`Pod` mới** để thay thế `Pod` đã thất bại.
>     

---

### Các loại Job khác nhau (Các kiểu làm việc của "đội thợ")

1. **Job Đơn (Non-parallel Job)**
    
    > **Tình huống:** "Sơn một bức tường." Chỉ cần một thợ làm là đủ.
    
    - **Cấu hình:** `completions: 1`, `parallelism: 1` (đây là giá trị mặc định).
        
    - **Hoạt động:** Kubernetes sẽ tạo 1 `Pod`. Nếu nó thành công, Job hoàn thành. Nếu nó thất bại, Job sẽ tạo một `Pod` khác để thử lại.
        
2. **Job Song Song với Số Lần Hoàn Thành Cố Định (Fixed Completion Count)**
    
    > **Tình huống:** "Giao 50 cái bánh pizza." Bạn muốn giao xong 50 cái, và bạn có thể huy động 5 tài xế giao cùng một lúc.
    
    - **Cấu hình:** `.spec.completions: 50`, `.spec.parallelism: 5`.
        
    - **Hoạt động:** Job sẽ tạo ra 5 `Pod` chạy song song. Khi một `Pod` hoàn thành, bộ đếm tăng lên. Job sẽ tạo `Pod` mới để thay thế (miễn là tổng `Pod` chạy không quá 5 và tổng hoàn thành chưa đủ 50). Khi bộ đếm đạt 50, Job hoàn thành.
        
3. **Job Song Song với Hàng Đợi Việc (Work Queue Job)**
    
    > **Tình huống:** "Xử lý một hàng đợi email cần gửi đi." Bạn không biết trước có bao nhiêu email, nhưng bạn muốn có một đội 5 nhân viên liên tục bốc email từ hàng đợi ra để xử lý.
    
    - **Cấu hình:** `.spec.completions` không được chỉ định, `.spec.parallelism: 5`.
        
    - **Hoạt động:** Kubernetes sẽ tạo 5 `Pod`. Các `Pod` này phải được lập trình để tự kết nối đến một hàng đợi (ví dụ: RabbitMQ, SQS). Job được coi là thành công khi ít nhất một `Pod` thoát ra với mã thành công và tất cả các `Pod` khác đã dừng lại.
        

---

### Khi nào cần CronJob?

Nếu bạn muốn chạy Job "Sao lưu cơ sở dữ liệu" định kỳ vào lúc 2 giờ sáng mỗi ngày, thì đó là lúc bạn cần đến `CronJob`. `CronJob` là một bộ điều khiển "ngồi trên" `Job`, có nhiệm vụ tạo ra một `Job` mới theo một lịch trình (cron schedule) bạn định sẵn.

---
### Phần 2: Use Case thực tế của Job

# Use Case: Xử lý Video Sau Khi Upload

Đây là một ví dụ thực tế, kinh điển về việc sử dụng Job: Xử lý (transcoding) video sau khi người dùng upload.

### Bối cảnh bài toán
Tưởng tượng bạn đang xây dựng một nền tảng chia sẻ video. Người dùng upload video 4K. Bạn cần tạo ra các phiên bản có độ phân giải thấp hơn (1080p, 720p, 480p). Quá trình này rất tốn CPU và không nên được thực hiện bởi web server chính (`Deployment`).

### Luồng hoạt động

**Các thành phần tham gia:**
-   **Web Server (`Deployment`):** Dịch vụ chạy liên tục để xử lý yêu cầu upload từ người dùng.
-   **Shared Storage (`PersistentVolume`):** Một ổ đĩa mạng mà cả Web Server và các `Pod` của Job đều có thể truy cập.
-   **Video Processing Job:** "Người thợ" của chúng ta, sử dụng công cụ `ffmpeg` để làm việc.

**Các bước chi tiết:**
1.  **Người dùng upload video:** Web server nhận file và lưu vào ổ đĩa chung.
2.  **Web Server tạo ra một Job:** Thay vì tự xử lý, web server yêu cầu Kubernetes API tạo ra một `Job` mới từ một file mẫu.
3.  **Cấu hình của Job được tạo ra:**

    ```yaml
    apiVersion: batch/v1
    kind: Job
    metadata:
      # Tên Job có thể được tạo tự động để không bị trùng
      name: process-video-my-awesome-video-4k
    spec:
      # ---- Cấu hình cho "đội thợ" ----
      completions: 3        # Cần hoàn thành 3 tác vụ: 1080p, 720p, 480p
      parallelism: 3        # Có thể chạy cả 3 tác vụ song song
      completionMode: Indexed # Gán cho mỗi Pod một chỉ số (index) từ 0 đến 2

      # ---- Khuôn mẫu cho các Pod "thợ" ----
      template:
        spec:
          volumes:
          - name: video-storage
            persistentVolumeClaim:
              claimName: shared-video-pvc # Trỏ đến ổ đĩa chung
          containers:
          - name: video-processor
            image: jrottenberg/ffmpeg # Một image có sẵn công cụ ffmpeg
            env:
            - name: INPUT_FILE
              value: "/videos/uploads/my_awesome_video_4k.mp4"
            # ---- Đây là phần "ma thuật" ----
            command: ["/bin/sh", "-c"]
            args:
              - |
                echo "Bắt đầu xử lý video..."
                # Dựa vào chỉ số (index) được Kubernetes gán, Pod sẽ biết mình cần làm gì
                if [ "$JOB_COMPLETION_INDEX" = "0" ]; then
                  echo "Tạo phiên bản 1080p"
                  ffmpeg -i $INPUT_FILE -s 1920x1080 /videos/processed/my_awesome_video_1080p.mp4
                elif [ "$JOB_COMPLETION_INDEX" = "1" ]; then
                  echo "Tạo phiên bản 720p"
                  ffmpeg -i $INPUT_FILE -s 1280x720 /videos/processed/my_awesome_video_720p.mp4
                elif [ "$JOB_COMPLETION_INDEX" = "2" ]; then
                  echo "Tạo phiên bản 480p"
                  ffmpeg -i $INPUT_FILE -s 854x480 /videos/processed/my_awesome_video_480p.mp4
                fi
                echo "Xử lý xong!"
            volumeMounts:
            - name: video-storage
              mountPath: /videos # Gắn ổ đĩa chung vào thư mục /videos
          restartPolicy: Never # Nếu ffmpeg lỗi, coi như Pod thất bại
    ```
4.  **Job bắt đầu thực thi:** Job Controller tạo ra 3 `Pod` cùng lúc. `Pod` 1 có `JOB_COMPLETION_INDEX=0`, `Pod` 2 có `index=1`, `Pod` 3 có `index=2`.
5.  **Các Pod làm việc song song:** Mỗi `Pod` dựa vào index của mình để thực hiện đúng tác vụ transcoding.
6.  **Job hoàn thành:** Khi cả 3 `Pod` đều chạy xong với mã thành công, Job được đánh dấu là `Succeeded`.

### Tại sao Job là lựa chọn hoàn hảo?
-   **Tách biệt tác vụ nặng:** Web server chính vẫn nhẹ nhàng và phản hồi nhanh.
-   **Độ tin cậy:** Kubernetes sẽ tự động thử lại nếu một tác vụ bị lỗi.
-   **Hiệu quả về tài nguyên:** Chạy song song giúp xử lý nhanh hơn.
-   **Tự động hóa:** Toàn bộ quy trình được tự động hóa hoàn toàn.

---

### Phần 3: Dọn dẹp Job đã hoàn thành

# Dọn dẹp Job đã hoàn thành với TTL Controller

### Vấn đề: Điều gì xảy ra khi một Job chạy xong?
> Hãy quay lại với ví dụ người thợ sơn hàng rào (`Job`). Sau khi người thợ sơn xong (dù sơn đẹp - `Complete`, hay sơn hỏng - `Failed`), họ sẽ dừng làm việc. Nhưng theo mặc định của Kubernetes:
> **Người thợ và toàn bộ đồ đạc của họ (tức là đối tượng `Job` và các `Pod` của nó) sẽ ở lại công trường mãi mãi!**

**Tại sao đây lại là một vấn đề?**
-   **Gây "rác" cho hệ thống:** Cluster của bạn sẽ nhanh chóng bị lấp đầy bởi các `Job` cũ.
-   **Gây áp lực cho API Server:** Quá nhiều đối tượng sẽ làm tăng tải cho `etcd`.
-   **Tốn công quản lý:** Bạn sẽ phải xóa các `Job` cũ bằng tay.

### Giải pháp: TTL-after-finished Controller
> Hãy tưởng tượng bạn nói với người thợ:
> "Sau khi anh làm xong, tôi cho anh đúng 10 phút (`ttlSecondsAfterFinished: 600`) để thu dọn đồ đạc. Hết 10 phút, dịch vụ dọn dẹp tự động của tôi sẽ đến và dọn sạch mọi thứ anh để lại."

**TTL-after-finished controller** chính là "dịch vụ dọn dẹp tự động" đó.

#### Cách hoạt động
1.  **Job hoàn thành:** Trạng thái của `Job` chuyển thành `Complete` hoặc `Failed`.
2.  **Đồng hồ đếm ngược bắt đầu:** TTL controller thấy trạng thái thay đổi và bắt đầu đếm ngược thời gian bạn đã chỉ định trong trường `.spec.ttlSecondsAfterFinished`.
3.  **Hết giờ:** Khi đồng hồ về 0, TTL controller sẽ xóa `Job` đó.
4.  **Xóa theo tầng (Cascading Deletion):** Khi xóa `Job`, nó sẽ xóa luôn tất cả các đối tượng phụ thuộc, bao gồm toàn bộ các `Pod` mà `Job` đó đã tạo ra.

#### Ví dụ thực tế
Đây là cách bạn thêm trường này vào file YAML của `Job`:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-ttl
spec:
  # ---- Dịch vụ dọn dẹp tự động ----
  ttlSecondsAfterFinished: 100 # <-- ĐÂY LÀ ĐIỂM MẤU CHỐT!

  # ---- Cấu hình của Job như bình thường ----
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
````

Với cấu hình trên, sau khi `Job` hoàn thành 100 giây, nó và các `Pod` của nó sẽ bị xóa hoàn toàn khỏi cluster.

#### Lợi ích

- **Tự động hóa hoàn toàn:** Không cần lo lắng về việc dọn dẹp.
    
- **Cluster sạch sẽ:** Giữ cho danh sách tài nguyên gọn gàng.
    
- **Giảm tải cho hệ thống:** Giảm số lượng đối tượng mà API Server phải theo dõi.
    
- **Linh hoạt:** Vẫn có một khoảng thời gian để kiểm tra log trước khi `Job` bị xóa.
    