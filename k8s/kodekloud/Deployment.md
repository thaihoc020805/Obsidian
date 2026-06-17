![[Pasted image 20250802130306.png]]

So we saw how to deploy an application to Kubernetes by creating a pod and deploying multiple instances using replicasets. But deploying and managing the number of replicas won't cut it (không đủ) when it comes to deploying applications for production use cases. When <mark style="background: #FFF3A3A6;">newer versions of application is released</mark>, you would like to <mark style="background: #FFF3A3A6;">UPGRADE your application instances seamlessly</mark>. <mark style="background: #BBFABBA6;">When you upgrade your instances, you may want to upgrade them one after the other. And that kind of upgrade is known as Rolling Updates.</mark>

Suppose <mark style="background: #FFF3A3A6;">one of the upgrades</mark> you performed resulted in an<mark style="background: #FFF3A3A6;"> unexpected error</mark> and you are asked to <mark style="background: #FFF3A3A6;">undo the recent update</mark>. You would like to be able to <mark style="background: #FFF3A3A6;">rollBACK the changes that were recently carried out</mark>. Finally, say for example <mark style="background: #FFF3A3A6;">you would like to make multiple changes to your environment</mark> such as upgrading the underlying WebServer versions, as well as scaling your environment and also modifying the resource allocations etc. You do not want each change to be applied immediately after the command is run, instead you would like to apply <mark style="background: #FFF3A3A6;">a pause to your environment</mark>, <mark style="background: #FFF3A3A6;">make the changes and then resume</mark> so that <mark style="background: #BBFABBA6;">all changes are rolled-out together</mark>. All of these capabilities are available with the kubernetes Deployments. So far in this course we discussed about PODs, which deploy single instances of our application such as the web application in this case. Each container is encapsulated in PODs. Multiple such PODs are deployed using Replica Sets. And then comes Deployment which is a kubernetes object that comes higher in the hierarchy. <mark style="background: #BBFABBA6;">The deployment provides us with capabilities to upgrade the underlying instances seamlessly using rolling updates, undo changes, and pause and resume changes to applications running on the cluster.</mark>

![[Pasted image 20250802160817.png]]
So how do we create a deployment? As with the previous components, we<mark style="background: #FFF3A3A6;"> first create a deployment definition file</mark>. The contents of the deployment definition file are exactly similar to the replicaset definition file, except for the kind, which is now going to be Deployment. If we walk through the contents of the file it has an <mark style="background: #BBFABBA6;">apiVersion which is apps/v1,</mark> <mark style="background: #BBFABBA6;">metadata which has name and labels</mark> and a <mark style="background: #BBFABBA6;">spec that has template, replicas and selector.</mark> The <mark style="background: #BBFABBA6;">template has a POD definition inside it</mark>. Once the file is ready run the kubectl create command and specify deployment definition file. <mark style="background: #FFF3A3A6;">Then run the kubectl get deployments command to see the newly created deployment</mark>. The <mark style="background: #FFF3A3A6;">deployment automatically creates a replica set</mark>. So if you run the kubectl get replicaset command you will be able to see a new replicaset in the name of the deployment. The replicasets ultimately create pods, so if you run the kubectl get pods command you will be able to see the pods with the name of the deployment and the replicaset.

![[Pasted image 20250802161440.png]]

<mark style="background: #FF5582A6;">To see all the created objects at once run the kubectl get all command.</mark>

![[Pasted image 20250802162012.png]]

Now <mark style="background: #FFF3A3A6;">once a deployment is created and you have a newer version of the app available,</mark> how do you upgrade your application? As before one way is to updated the deployment definition file to <mark style="background: #FFF3A3A6;">update the new image name with the newer version of the app</mark>. <mark style="background: #BBFABBA6;">The imperative approach would be to use the kubectl set image command and specify the deployment name and the image name </mark>like this.

# Cập nhật và Rollback trong Kubernetes Deployment

## Phần 1: Cập nhật ứng dụng (Update)

Khi bạn thay đổi `spec.template` của một Deployment (ví dụ: đổi image container, cập nhật biến môi trường, sửa command...), Deployment sẽ kích hoạt một quá trình cập nhật. Chiến lược cập nhật mặc định và phổ biến nhất là **Rolling Update (Cập nhật cuốn chiếu)**.

### A. Khái niệm Rolling Update

Mục tiêu của Rolling Update là cập nhật các Pod sang phiên bản mới một cách từ từ, tuần tự, đảm bảo rằng ứng dụng luôn có một số lượng Pod nhất định đang chạy để phục vụ người dùng. Quá trình này diễn ra như sau:

1. **Tạo ReplicaSet mới**: Deployment tạo một ReplicaSet mới với cấu hình Pod mới (v2), nhưng ban đầu scale nó ở mức 0 replica.
    
2. **Scale Up ReplicaSet mới**: Nó bắt đầu tăng số replica của ReplicaSet mới (v2) lên 1.
    
3. **Scale Down ReplicaSet cũ**: Ngay sau khi Pod mới (v2) sẵn sàng (sẵn sàng nhận traffic), nó sẽ giảm số replica của ReplicaSet cũ (v1) xuống 1 (bằng cách chấm dứt một Pod cũ).
    
4. **Lặp lại**: Quá trình này lặp đi lặp lại cho đến khi tất cả các Pod cũ (v1) đã được thay thế bằng Pod mới (v2), và ReplicaSet cũ có số replica bằng 0.
    

### B. Kiểm soát quá trình cập nhật

Tốc độ và sự an toàn của quá trình Rolling Update được kiểm soát bởi hai tham số quan trọng trong `spec.strategy.rollingUpdate`:

- **`maxUnavailable`**: Số lượng Pod tối đa được phép _không khả dụng_ trong quá trình cập nhật. Nó có thể là một số tuyệt đối (`1`) hoặc một tỷ lệ phần trăm (`25%`).
    
    > Ví dụ: Nếu bạn có `replicas: 4` và `maxUnavailable: 25%`, Kubernetes sẽ luôn đảm bảo có ít nhất `4 - (4*25%) = 3` Pod đang chạy và sẵn sàng trong suốt quá trình cập nhật.
    
- **`maxSurge`**: Số lượng Pod tối đa được phép tạo _vượt quá_ số `replicas` mong muốn. Nó cũng có thể là số tuyệt đối hoặc tỷ lệ phần trăm.
    
    > Ví dụ: Nếu bạn có `replicas: 4` và `maxSurge: 25%`, trong quá trình cập nhật, tổng số Pod (cũ và mới) có thể lên tới `4 + (4*25%) = 5` Pod.
    

### C. Ví dụ thực hành cập nhật

#### Bước 1: Triển khai phiên bản đầu tiên (v1)

Tạo file `deployment-v1.yaml`:

YAML

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx-app
spec:
  replicas: 4
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
        image: nginx:1.21.0 # <-- Phiên bản 1
        ports:
        - containerPort: 80
```

Triển khai:

Bash

```
kubectl apply -f deployment-v1.yaml
```

#### Bước 2: Kích hoạt và theo dõi cập nhật (lên v2)

Cập nhật image bằng lệnh `kubectl set image`:

Bash

```
# Cập nhật image của container tên 'nginx' trong deployment 'my-nginx-app' sang phiên bản 1.22.0
kubectl set image deployment/my-nginx-app nginx=nginx:1.22.0
```

Theo dõi quá trình:

Bash

```
# Theo dõi trạng thái rollout theo thời gian thực
kubectl rollout status deployment/my-nginx-app

# Hoặc xem các Pod được tạo/xóa
kubectl get pods -l app=nginx -w
```

#### Bước 3: Kiểm tra ReplicaSet

Sau khi cập nhật, kiểm tra các ReplicaSet:

Bash

```
$ kubectl get rs
NAME                      DESIRED   CURRENT   READY   AGE
my-nginx-app-698f68c56c   4         4         4       2m
my-nginx-app-544dc6b6d5   0         0         0       10m
```

Bạn sẽ thấy ReplicaSet mới (`...-698f...`) đang chạy 4 Pod, và ReplicaSet cũ (`...-544d...`) vẫn còn đó nhưng có 0 Pod. Đây chính là "chìa khóa" cho việc rollback.

---

## Phần 2: Quay lui phiên bản (Rollback)

Giả sử phiên bản `nginx:1.22.0` bị lỗi. Bạn cần nhanh chóng quay trở lại phiên bản `1.21.0`.

### A. Khái niệm Rollback

Một cuộc rollback thực chất chỉ là một cuộc **Rolling Update khác**, trong đó "phiên bản mới" chính là cấu hình của một phiên bản cũ đã được lưu lại trong lịch sử (revision) của Deployment.

### B. Ví dụ thực hành Rollback

#### Bước 1: Xem lịch sử các phiên bản

Bash

```
$ kubectl rollout history deployment/my-nginx-app

deployment.apps/my-nginx-app
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

- **REVISION**: Số thứ tự của phiên bản. `2` là phiên bản hiện tại, `1` là phiên bản trước đó.
    
- **CHANGE-CAUSE**: Lý do thay đổi (nếu có).
    

#### Bước 2: Thực hiện Rollback

- **Quay về phiên bản ngay trước đó (phổ biến nhất):**
    
    Bash
    
    ```
    kubectl rollout undo deployment/my-nginx-app
    ```
    
- **Quay về một phiên bản cụ thể:**
    
    Bash
    
    ```
    # Quay về revision số 1
    kubectl rollout undo deployment/my-nginx-app --to-revision=1
    ```
    

#### Bước 3: Theo dõi và xác minh

Sử dụng `kubectl rollout status` để theo dõi. Sau khi hoàn tất, ứng dụng của bạn đã an toàn quay trở lại phiên bản `nginx:1.21.0`.

---

## Tóm tắt các lệnh chính

| Hành động                       | Lệnh `kubectl`                                               |
| ------------------------------- | ------------------------------------------------------------ |
| **Cập nhật Image**              | `kubectl set image deployment/[tên] [container]=[image:tag]` |
| **Theo dõi tiến trình**         | `kubectl rollout status deployment/[tên]`                    |
| **Xem lịch sử**                 | `kubectl rollout history deployment/[tên]`                   |
| **Rollback về phiên bản trước** | `kubectl rollout undo deployment/[tên]`                      |
| **Rollback về phiên bản N**     | `kubectl rollout undo deployment/[tên] --to-revision=[N]`    |
| **Tạm dừng/Tiếp tục**           | `kubectl rollout pause/resume deployment/[tên]`              |

Xuất sang Trang tính

---

## Tìm hiểu sâu hơn: `kubectl rollout pause` và `resume`

Đây là những công cụ mạnh mẽ để kiểm soát quá trình cập nhật một cách thủ công.

### 1. `kubectl rollout pause` - Tạm dừng

#### Cơ chế hoạt động

Lệnh này "đóng băng" trạng thái của Deployment, ngăn nó tiếp tục tạo Pod mới và xóa Pod cũ.

#### Các trường hợp sử dụng chính

- **A. Kiểm thử Canary (Canary Release/Testing)**: Đưa một thay đổi mới ra cho một nhóm nhỏ người dùng/request để kiểm tra trước khi triển khai toàn bộ. Bạn sẽ `pause` ngay sau khi một vài Pod mới được tạo để kiểm thử chúng.
    
- **B. Áp dụng nhiều thay đổi cùng lúc**: `Pause` Deployment, thực hiện nhiều thay đổi (set image, patch, set env...), sau đó `resume` để kích hoạt một đợt rollout duy nhất cho tất cả thay đổi.
    

### 2. `kubectl rollout resume` - Tiếp tục

#### Cơ chế hoạt động

Lệnh này "rã đông" Deployment, cho phép nó tiếp tục quá trình Rolling Update từ đúng chỗ đã dừng lại để đưa hệ thống về trạng thái mong muốn.

#### Ví dụ thực hành chi tiết: Kiểm thử Canary

Giả sử có Deployment `my-app` với `replicas: 10`, đang chạy `image:v1.0`.

**Bước 1: Pause Deployment**

Bash

```
# Tạm dừng Deployment trước khi thay đổi bất cứ điều gì.
kubectl rollout pause deployment/my-app
```

**Bước 2: Cập nhật Image**

Bash

```
# Sẽ KHÔNG có rollout nào xảy ra vì đang bị pause.
kubectl set image deployment/my-app my-container=my-app-image:v2.0-beta
```

**Bước 3: Resume để bắt đầu cập nhật có kiểm soát**

Bash

```
kubectl rollout resume deployment/my-app
```

**Bước 4: Nhanh chóng Pause lại** Ngay sau khi resume, quá trình rollout sẽ tạo 1-2 Pod mới. Hãy nhanh chóng `pause` một lần nữa để tạo ra môi trường Canary.

Bash

```
kubectl rollout pause deployment/my-app
```

Lúc này bạn sẽ có một môi trường hỗn hợp (ví dụ: 8 Pod v1.0, 2 Pod v2.0-beta) để kiểm thử.

**Bước 5: Ra quyết định**

- **Nếu phiên bản beta hoạt động tốt:**
    
    Bash
    
    ```
    kubectl rollout resume deployment/my-app
    ```
    
- **Nếu phiên bản beta có lỗi:**
    
    Bash
    
    ```
    # 1. Undo rollout (cấu hình sẽ đổi về v1.0 nhưng chưa chạy)
    kubectl rollout undo deployment/my-app
    
    # 2. Resume để thực hiện quá trình rollback
    kubectl rollout resume deployment/my-app
    ```