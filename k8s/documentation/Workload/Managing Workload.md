
# Quản lý Workloads trong Kubernetes

Sau khi bạn đã triển khai ứng dụng và đưa nó ra ngoài thông qua một `Service`, bước tiếp theo là gì? Kubernetes cung cấp nhiều công cụ để giúp bạn quản lý việc triển khai ứng dụng, bao gồm co giãn (scaling) và cập nhật (updating).

---

## 1. Tổ chức cấu hình tài nguyên (Organizing resource configurations)

Nhiều ứng dụng đòi hỏi phải tạo ra nhiều tài nguyên cùng lúc, ví dụ như một `Deployment` đi kèm với một `Service`. Để quản lý dễ dàng hơn, bạn có thể nhóm chúng vào chung một file YAML, phân tách bởi dấu `---`.

**Ví dụ:** file `application/nginx-app.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-svc
  labels:
    app: nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
````

> **Ghi chú:** Việc đặt `Service` lên trước `Deployment` là một thực hành tốt. Điều này đảm bảo rằng Service tồn tại trước, giúp bộ lập lịch (scheduler) có thể phân bổ các Pod một cách hiệu quả hơn khi `Deployment` tạo ra chúng.

Bạn có thể tạo nhiều tài nguyên này bằng một lệnh duy nhất:

Bash

```
kubectl apply -f [https://k8s.io/examples/application/nginx-app.yaml](https://k8s.io/examples/application/nginx-app.yaml)
```

Hoặc bạn có thể chỉ định nhiều file `-f` cùng lúc:

Bash

```
kubectl apply -f /path/to/nginx-svc.yaml -f /path/to/nginx-deployment.yaml
```

---

## 2. Các công cụ bên ngoài (External tools)

### Helm

Là một công cụ quản lý các "gói" tài nguyên Kubernetes đã được cấu hình sẵn. Các gói này được gọi là **Helm charts**.

### Kustomize

Giúp bạn tùy chỉnh các file manifest của Kubernetes mà không cần sửa trực tiếp vào file gốc. Nó có thể thêm, xóa, hoặc cập nhật các tùy chọn cấu hình. Kustomize hiện đã được tích hợp sẵn vào `kubectl`.

---

## 3. Các thao tác hàng loạt trong kubectl (Bulk operations in kubectl)

`kubectl` không chỉ tạo tài nguyên hàng loạt mà còn có thể thực hiện các thao tác khác.

- **Xóa tài nguyên từ file:**
    
    Bash
    
    ```
    kubectl delete -f [https://k8s.io/examples/application/nginx-app.yaml](https://k8s.io/examples/application/nginx-app.yaml)
    ```
    
- **Xóa nhiều tài nguyên cụ thể:**
    
    Bash
    
    ```
    kubectl delete deployments/my-nginx services/my-nginx-svc
    ```
    
- **Xóa tài nguyên bằng label selector (`-l`):** Đây là cách rất mạnh mẽ để thao tác trên một nhóm tài nguyên có chung một nhãn (label).
    
    Bash
    
    ```
    # Xóa tất cả deployment và service có nhãn app=nginx
    kubectl delete deployment,services -l app=nginx
    ```
    
- **Thao tác đệ quy (Recursive operations):** Nếu bạn tổ chức các file manifest trong nhiều thư mục con, bạn có thể dùng cờ `--recursive` (hoặc `-R`) để `kubectl` xử lý tất cả các file trong cây thư mục.
    
    Bash
    
    ```
    # Giả sử cấu trúc thư mục là project/k8s/development/...
    kubectl apply -f project/k8s/development --recursive
    ```
    

---

## 4. Cập nhật ứng dụng không gây gián đoạn (Zero-Downtime Updates)

Đây là một trong những tính năng mạnh mẽ nhất của Kubernetes. Bạn có thể cập nhật ứng dụng lên phiên bản mới mà không làm dịch vụ bị ngưng. Quá trình này được gọi là **Rolling Update**.

### Ví dụ các bước cập nhật:

1. **Triển khai phiên bản đầu tiên (`v1.14.2`):**
    
    Bash
    
    ```
    kubectl create deployment my-nginx --image=nginx:1.14.2
    ```
    
2. **Cập nhật lên phiên bản mới (`v1.16.1`):** Cách đơn giản nhất là dùng lệnh `kubectl edit`.
    
    Bash
    
    ```
    kubectl edit deployment/my-nginx
    ```
    
    Lệnh này sẽ mở file YAML của `Deployment` trong trình soạn thảo. Bạn chỉ cần tìm và sửa `image: nginx:1.14.2` thành `image: nginx:1.16.1` rồi lưu lại.
    

> **Quá trình Rolling Update:**
> 
> - Kubernetes sẽ tạo một Pod mới với phiên bản `1.16.1`.
>     
> - Chờ đến khi Pod mới sẵn sàng hoạt động (healthy).
>     
> - Sau đó, nó sẽ xóa một Pod cũ chạy phiên bản `1.14.2`.
>     
> - Quá trình này lặp lại cho đến khi tất cả các Pod cũ được thay thế bằng Pod mới.
>     

### Quản lý quá trình Rollout

Bạn có thể theo dõi, tạm dừng, hoặc hủy bỏ một quá trình cập nhật bằng lệnh `kubectl rollout`.

- **Xem trạng thái:**
    
    Bash
    
    ```
    kubectl rollout status deployment/my-deployment
    ```
    
- **Tạm dừng:**
    
    Bash
    
    ```
    kubectl rollout pause deployment/my-deployment
    ```
    
- **Tiếp tục:**
    
    Bash
    
    ```
    kubectl rollout resume deployment/my-deployment
    ```
    
- **Quay về phiên bản trước:**
    
    Bash
    
    ```
    kubectl rollout undo deployment/my-deployment
    ```
    

---

## 5. Triển khai theo kiểu Canary (Canary Deployments)

Đây là một kỹ thuật triển khai nâng cao, cho phép bạn đưa một phiên bản mới ra cho một phần nhỏ người dùng trước khi triển khai toàn bộ.

Cách thực hiện là tạo hai `Deployment` song song:

1. **Deployment "stable"**: Chạy phiên bản cũ, phục vụ phần lớn traffic. Có label `track: stable`.
    
2. **Deployment "canary"**: Chạy phiên bản mới, phục vụ một phần nhỏ traffic. Có label `track: canary`.
    

`Service` sẽ được cấu hình để gửi traffic đến **cả hai** `Deployment` này bằng cách chọn các label chung (ví dụ: `app: guestbook`, `tier: frontend` mà bỏ qua label `track`). Bằng cách điều chỉnh số lượng `replicas` của hai `Deployment`, bạn có thể kiểm soát tỷ lệ traffic đi vào phiên bản mới.

---

## 6. Co giãn ứng dụng (Scaling Your Application)

### Co giãn thủ công (Manual Scaling)

Bash

```
# Scale số replicas của my-nginx xuống còn 1
kubectl scale deployment/my-nginx --replicas=1
```

### Co giãn tự động (Automatic Scaling)

Đây chính là lúc bạn dùng `HPA (Horizontal Pod Autoscaler)`.

Bash

```
# Cho phép hệ thống tự động scale my-nginx từ 1 đến 3 replicas
# khi CPU trung bình vượt 80%
kubectl autoscale deployment/my-nginx --min=1 --max=3 --cpu-percent=80
```

Lệnh này tạo ra một tài nguyên `HPA`. Nó sẽ theo dõi CPU của các Pod và tự động điều chỉnh số lượng `replicas` cho phù hợp.

---

## 7. Các cách cập nhật tài nguyên khác

### `kubectl apply` (Khuyến khích)

Đây là cách tốt nhất để quản lý cấu hình. Bạn lưu các file YAML trong hệ thống quản lý phiên bản (như Git). Khi có thay đổi, bạn chỉ cần chạy lại `kubectl apply -f <file.yaml>`. `kubectl` sẽ tự so sánh và chỉ áp dụng những thay đổi đó.

### `kubectl edit`

Dùng để chỉnh sửa nhanh một tài nguyên trực tiếp trên cluster. Phù hợp cho các thay đổi nhỏ, mang tính tương tác.

### `kubectl patch`

Dùng để cập nhật một phần nhỏ của một tài nguyên một cách có chủ đích, thường được dùng trong các kịch bản tự động hóa (script).

### `kubectl replace --force` (Cập nhật gây gián đoạn)

Một số trường của tài nguyên không thể thay đổi sau khi đã được tạo. Trong trường hợp này, nếu bạn bắt buộc phải thay đổi chúng, bạn phải dùng `replace --force`.

> **Cảnh báo:** Lệnh này sẽ **xóa tài nguyên cũ và tạo lại một tài nguyên mới** từ file YAML của bạn. Hãy cẩn thận khi sử dụng lệnh này vì nó sẽ gây gián đoạn dịch vụ.

Nguồn