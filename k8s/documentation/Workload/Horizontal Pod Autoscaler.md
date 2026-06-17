
## I. Điều Kiện Tiên Quyết (Bắt buộc)
Trước khi bạn có thể dùng HPA, cluster của bạn bắt buộc phải có 2 thứ sau:

### 1. Metrics Server được cài đặt
> **Nó là gì?**
> Metrics Server là một thành phần trong cluster có nhiệm vụ thu thập các chỉ số tài nguyên cơ bản (như mức sử dụng CPU, Memory) từ các Node và Pod.
> 
> **Tại sao cần?**
> HPA cần một nguồn dữ liệu để biết được các Pod đang dùng bao nhiêu CPU/Memory. Metrics Server chính là nguồn dữ liệu đó. Nếu không có nó, HPA sẽ "bị mù" và không biết khi nào cần co giãn.

Hầu hết các nhà cung cấp cloud Kubernetes (GKE, EKS, AKS) đều cài sẵn nó. Nếu bạn tự dựng cluster, bạn có thể phải cài nó thủ công.

### 2. Deployment/Workload phải có `resources.requests`
HPA tính toán việc co giãn dựa trên tỷ lệ phần trăm của tài nguyên đã được **yêu cầu (requested)**.

> **Tại sao cần?**
> Bạn đặt mục tiêu HPA là 50% CPU. Nếu Pod của bạn không khai báo `requests.cpu`, HPA sẽ không biết 50% của "không có gì" là bao nhiêu, và nó sẽ không hoạt động.

Đây là một ví dụ khai báo đúng trong file `deployment.yaml` của bạn:
```yaml
spec:
  template:
    spec:
      containers:
      - name: my-app
        image: my-image
        resources:
          requests: # <-- HPA CẦN CÁI NÀY!
            cpu: "200m" # 200 millicores
            memory: "256Mi"
````

---

## II. Cách tạo HPA

### Cách 1: Dùng Lệnh `kubectl autoscale` (Nhanh và Dễ)

Đây là cách nhanh nhất để tạo một HPA cho một workload đã tồn tại. Rất phù hợp khi bạn đang thử nghiệm hoặc cho các kịch bản đơn giản.

**Cú pháp:**

Bash

```
kubectl autoscale deployment [TÊN_DEPLOYMENT] --cpu-percent=[MỤC_TIÊU] --min=[SỐ_POD_TỐI_THIỂU] --max=[SỐ_POD_TỐI_ĐA]
```

**Ví dụ thực tế:** Giả sử bạn có một `Deployment` tên là `my-app`. Bạn muốn tự động co giãn nó như sau:

- Giữ mức sử dụng CPU trung bình ở mức `50%`.
    
- Số Pod tối thiểu là `2`.
    
- Số Pod tối đa là `10`.
    

Bạn sẽ chạy lệnh:

Bash

```
kubectl autoscale deployment my-app --cpu-percent=50 --min=2 --max=10
```

> Lệnh này sẽ tự động tạo ra một đối tượng `HorizontalPodAutoscaler` trong cluster của bạn với các cấu hình trên.

### Cách 2: Khai báo bằng file YAML (Chuẩn và Chuyên nghiệp)

Đây là cách được khuyên dùng cho môi trường production. Việc khai báo bằng YAML giúp bạn quản lý cấu hình bằng mã nguồn (Infrastructure as Code), lưu trữ trong Git, và tích hợp vào các quy trình CI/CD.

**Ví dụ file `hpa.yaml`:**

YAML

```
apiVersion: autoscaling/v2 # Luôn ưu tiên dùng v2, nó mạnh mẽ hơn v1
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa # Đặt tên cho đối tượng HPA
spec:
  # 1. Chỉ định mục tiêu cần co giãn
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app # Tên của Deployment mục tiêu

  # 2. Định nghĩa giới hạn co giãn
  minReplicas: 2
  maxReplicas: 10

  # 3. Định nghĩa các chỉ số (metrics) để ra quyết định
  metrics:
  - type: Resource # Co giãn dựa trên tài nguyên của Pod (CPU/Memory)
    resource:
      name: cpu
      target:
        type: Utilization # Tính theo tỷ lệ %
        averageUtilization: 50 # Mục tiêu là 50%

  # Bạn cũng có thể thêm metric cho Memory
  # - type: Resource
  #   resource:
  #     name: memory
  #     target:
  #       type: Utilization
  #       averageUtilization: 70
```

Sau khi tạo file này, bạn chỉ cần áp dụng nó:

Bash

```
kubectl apply -f hpa.yaml
```

---

## III. Làm thế nào để kiểm tra HPA?

Sau khi tạo HPA bằng một trong hai cách trên, bạn có thể kiểm tra trạng thái của nó bằng lệnh:

Bash

```
kubectl get hpa
```

Kết quả sẽ trông giống như thế này:

```
NAME         REFERENCE           TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
my-app-hpa   Deployment/my-app   15%/50%   2         10        2          5m
```

**Giải thích các cột:**

- **REFERENCE:** Đối tượng mà HPA đang theo dõi (`Deployment/my-app`).
    
- **TARGETS:** Cột quan trọng nhất! Nó cho bạn biết mức sử dụng hiện tại so với mục tiêu. `15%/50%` có nghĩa là CPU trung bình hiện tại là 15%, trong khi mục tiêu là 50%. Vì hiện tại thấp hơn mục tiêu, HPA đang giữ số `REPLICAS` ở mức tối thiểu (`MINPODS`). Nếu con số này vượt quá 50%, bạn sẽ thấy số `REPLICAS` tăng lên.
    
- **REPLICAS:** Số lượng Pod hiện tại mà `Deployment` đang chạy.
    

Để xem chi tiết các sự kiện co giãn (ví dụ: "Scaled up replicas to 5"), bạn có thể dùng lệnh:

Bash

```
kubectl describe hpa my-app-hpa
```


---
### Phần 2: So sánh Deployment và HPA


# So sánh Deployment và HPA

**Deployment và HPA (Horizontal Pod Autoscaler) không phải là hai thứ để lựa chọn một trong hai, mà chúng bổ trợ và làm việc cùng nhau.**

Hãy dùng một ví dụ đơn giản để dễ hình dung:

- **Deployment** giống như người quản lý của một nhà hàng. Anh ta quyết định rằng: "Nhà hàng của chúng ta _luôn cần có 3 đầu bếp_ làm việc". Dù vắng khách hay đông khách, anh ta vẫn đảm bảo luôn có đúng 3 người.
- **HPA** giống như một người quản lý thông minh và linh hoạt hơn. Anh ta sẽ quan sát lượng khách hàng đang chờ.
    - Nếu khách quá đông (tải tăng cao), anh ta sẽ gọi thêm đầu bếp vào làm.
    - Nếu nhà hàng vắng hoe (tải giảm xuống), anh ta sẽ cho một vài đầu bếp về nghỉ để tiết kiệm chi phí.

Trong thế giới Kubernetes, "đầu bếp" chính là các **Pod** (container chứa ứng dụng của bạn).

Bây giờ, hãy đi vào chi tiết kỹ thuật:

### Vai trò của Deployment

**`Deployment`** là một đối tượng cốt lõi trong Kubernetes, có nhiệm vụ chính là:

1.  **Khai báo trạng thái mong muốn:** Bạn nói với Kubernetes rằng: "Tôi muốn ứng dụng X của tôi chạy với 3 bản sao (replicas)".
2.  **Đảm bảo số lượng Pod:** `Deployment` sẽ làm mọi cách để đảm bảo _luôn có đúng 3 Pod_ đang chạy. Nếu một Pod bị lỗi và chết đi, `Deployment` sẽ tự động tạo một Pod mới để thay thế.
3.  **Quản lý việc cập nhật (Rolling Update):** Khi bạn muốn nâng cấp ứng dụng lên phiên bản mới, `Deployment` sẽ thực hiện việc này một cách từ từ, không gây gián đoạn dịch vụ. Nó sẽ tạo Pod mới với phiên bản mới, chờ nó sẵn sàng rồi mới xóa Pod cũ đi.

**Vấn đề khi chỉ dùng Deployment:** Số lượng Pod là **cố định**.

- **Khi traffic tăng đột biến:** 3 Pod có thể bị quá tải, dẫn đến ứng dụng chạy chậm, treo hoặc sập. Người dùng sẽ có trải nghiệm tồi tệ.
- **Khi không có traffic (ví dụ: ban đêm):** 3 Pod vẫn chạy ở đó, gây lãng phí tài nguyên (CPU, RAM) và tốn tiền một cách không cần thiết.

---

### Vai trò của HPA (Horizontal Pod Autoscaler)

Đây chính là lúc **HPA** phát huy tác dụng. HPA giải quyết chính xác vấn đề trên.

1.  **Tự động co giãn (Autoscaling):** HPA theo dõi các chỉ số (metrics) của các Pod, phổ biến nhất là CPU và Memory.
2.  **Ra quyết định:** Bạn đặt ra các quy tắc, ví dụ: "Nếu mức sử dụng CPU trung bình của các Pod vượt quá 70%, hãy tăng số lượng Pod lên".
3.  **Hành động:** Khi điều kiện được đáp ứng, HPA sẽ **tự động cập nhật lại trường `replicas` trong chính `Deployment` của bạn**.

Nói cách khác, HPA là "bộ não" ra lệnh cho `Deployment` phải thay đổi số lượng Pod cho phù hợp với tình hình thực tế.

### Bảng so sánh trực quan

| Tiêu chí | Chỉ dùng Deployment | Dùng Deployment + HPA |
| :--- | :--- | :--- |
| **Mục đích chính** | Đảm bảo một số lượng Pod **cố định** luôn chạy. Quản lý cập nhật. | Tự động điều chỉnh số lượng Pod dựa trên tải thực tế. |
| **Khả năng co giãn** | Thủ công (Bạn phải tự sửa file YAML và `apply` lại). | **Tự động** và nhanh chóng. |
| **Tối ưu tài nguyên** | Kém. Gây lãng phí khi tải thấp và có nguy cơ sập khi tải cao. | **Rất tốt**. Tiết kiệm chi phí khi tải thấp, đảm bảo ổn định khi tải cao. |
| **Tính ổn định** | Phụ thuộc vào việc bạn dự đoán tải có chính xác hay không. | **Cao**. Tự động thích ứng với các biến động không lường trước. |

### Khi nào thì chỉ cần dùng Deployment?

Bạn chỉ nên dùng `Deployment` mà không cần HPA khi ứng dụng của bạn có lượng tải **rất ổn định và có thể dự đoán được**. Ví dụ:

- Một công cụ nội bộ chỉ có vài người dùng cố định.
- Một tiến trình chạy nền (background job) xử lý một lượng công việc không đổi.

### Kết luận

**HPA không thay thế `Deployment`, mà nó làm việc _cùng_ với `Deployment` để tạo ra một hệ thống có khả năng co giãn tự động.**

- **`Deployment`** tạo ra nền tảng vững chắc, đảm bảo ứng dụng của bạn luôn chạy.
- **HPA** đặt lên trên đó một lớp thông minh, giúp hệ thống của bạn trở nên linh hoạt, hiệu quả về chi phí và có khả năng chống chịu lỗi tốt hơn nhiều.

Vì vậy, trong hầu hết các ứng dụng thực tế, đặc biệt là các ứng dụng phục vụ người dùng cuối (web, API, e-commerce), việc **sử dụng kết hợp `Deployment` và HPA là tiêu chuẩn vàng.**
