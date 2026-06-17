## 1) Dùng Deployment để làm gì?

- **Tạo mới** ReplicaSet/Pods và **theo dõi rollout** (trạng thái phát hành).
    
- **Cập nhật phiên bản** (đổi image, đổi label trong `.spec.template`) bằng **rolling update** an toàn.
    
- **Rollback** về **revision** trước nếu bản đang chạy bị lỗi.
    
- **Scale** lên/xuống số Pod theo tải.
    
- **Tạm dừng** rollout để gom nhiều thay đổi rồi **resume** phát hành một lần.
    
- **Phát hiện rollout kẹt** (stuck) từ status để xử lý.
    
- **Dọn rác** ReplicaSet cũ theo chính sách.
    

> **Lưu ý vàng:** **Đừng** chỉnh tay **ReplicaSet** do Deployment sở hữu. Nếu có use case đặc biệt, hãy mở issue với K8s thay vì “đi đường vòng”.

---

## 2) Ví dụ YAML & giải phẫu từng phần

yaml

CopyEdit

`apiVersion: apps/v1 kind: Deployment metadata:   name: nginx-deployment           # Tên Deployment (nền tảng cho tên RS/Pod)   labels:     app: nginx spec:   replicas: 3                      # Muốn chạy 3 Pod   selector:     matchLabels:       app: nginx                   # Phải trùng với template.metadata.labels   template:     metadata:       labels:         app: nginx                 # Nhãn dán cho Pod (phải khớp selector)     spec:       containers:       - name: nginx         image: nginx:1.14.2        # Image sẽ chạy         ports:         - containerPort: 80`

**Điểm mấu chốt:**

- `.spec.selector` **phải** khớp `.spec.template.metadata.labels`. Trong `apps/v1`, selector **bất biến** (immutable) sau khi tạo.
    
- **RestartPolicy** của Pod trong Deployment luôn **`Always`**.
    
- **replicas** mặc định là **1** nếu không ghi.
    

**Lệnh cơ bản:**

bash

CopyEdit

`kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml kubectl get deployments kubectl rollout status deployment/nginx-deployment kubectl get rs kubectl get pods --show-labels`

**Các cột khi `kubectl get deployments`:**

- **READY** = số Pod sẵn sàng / số mong muốn.
    
- **UP-TO-DATE** = số Pod đã dùng template mới nhất.
    
- **AVAILABLE** = số Pod sẵn sàng phục vụ.
    
- **AGE** = thời gian tồn tại.
    

---

## 3) `pod-template-hash` là gì? (đừng sửa!)

- Controller thêm label **`pod-template-hash`** vào mỗi ReplicaSet và Pod.
    
- Nó là **hash** của PodTemplate, giúp **phân tách** các ReplicaSet con để không **đè** lên nhau.
    
- **Tuyệt đối không đổi** label này bằng tay.
    

---

## 4) Cập nhật Deployment (trigger rollout **chỉ** khi đổi `.spec.template`)

**Rollout xảy ra khi** bạn đổi **template** (ví dụ đổi image, đổi label trong template). **Scale** không tạo rollout/revision mới.

Ví dụ đổi image:

bash

CopyEdit

`kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1 # hoặc kubectl edit deployment/nginx-deployment  # sửa image trong YAML kubectl rollout status deployment/nginx-deployment`

**Kết quả đằng sau:**

- Deployment sẽ tạo **ReplicaSet mới**, scale dần **lên**, đồng thời scale **xuống** ReplicaSet cũ theo chiến lược rolling update.
    

---

## 5) RollingUpdate — an toàn theo 2 tham số

Mặc định **`strategy.type: RollingUpdate`** với:

- **`maxUnavailable`** = 25%: tối đa **bấy nhiêu Pod** có thể **không sẵn sàng** khi update.
    
- **`maxSurge`** = 25%: tối đa **bấy nhiêu Pod** có thể **tạo vượt** số mong muốn khi update.
    

> Với 4 replicas: số Pod tại mọi thời điểm nằm trong khoảng **[3, 5]** (ít nhất 3 available, nhiều nhất 5 running tạm thời).

Ví dụ cấu hình:

yaml

CopyEdit

`strategy:   type: RollingUpdate   rollingUpdate:     maxUnavailable: 1     # Có thể là số tuyệt đối hoặc %     # maxSurge: mặc định 25% nếu không ghi`

**Quy tắc làm tròn:**

- Tính từ **%** → số nguyên: **`maxUnavailable`** làm tròn **xuống**, **`maxSurge`** làm tròn **lên**.
    
- Không được để **cả hai** bằng **0**.
    

---

## 6) `kubectl describe deployment` – đọc như chuyên gia

Bạn sẽ thấy:

- **StrategyType**: RollingUpdate hay Recreate.
    
- **MinReadySeconds**: số giây Pod phải “ready liên tục” trước khi tính **available**.
    
- **RollingUpdateStrategy**: giá trị `maxUnavailable`, `maxSurge`.
    
- **Conditions**:
    
    - **Available=True**: đạt **tối thiểu** số Pod sẵn sàng theo chiến lược.
        
    - **Progressing=True** (ví dụ **NewReplicaSetAvailable**): rollout **đang chạy** hoặc **xong**.
        

**Events** cho biết lịch sử scale up/down từng ReplicaSet.

---

## 7) Nhiều update chồng nhau (Rollover)

Nếu bạn **đổi image liên tiếp** khi rollout trước **chưa xong**:

- Deployment tạo **RS mới** cho update mới, **dừng** scale-up RS đang dở và đưa RS đó vào danh sách **old** để scale xuống.
    
- Nó luôn hướng tới trạng thái mới nhất bạn yêu cầu.
    

---

## 8) Cập nhật **label selector** – cực kỳ thận trọng

- Trong `apps/v1`, **selector immutable** sau khi tạo. Nói cách khác: **không đổi**.
    
- Nếu buộc phải “thêm” selector (trong bối cảnh cho phép), bạn **cũng phải** cập nhật **label trong template** tương ứng, nếu không sẽ **lỗi**.
    
- Thay selector có thể khiến RS/Pod cũ bị **orphan** (không còn được Deployment quản), và Deployment sẽ tạo RS **mới**.
    

> **Khuyến cáo:** **Lên kế hoạch selector ngay từ đầu**. Tránh chỉnh giữa chừng.

---

## 9) Rollback & Lịch sử (Revision)

- **Revision** mới tạo ra **chỉ khi** đổi **`.spec.template`**.
    
- Xem lịch sử:
    
    bash
    
    CopyEdit
    
    `kubectl rollout history deployment/nginx-deployment kubectl rollout history deployment/nginx-deployment --revision=2`
    
- Gắn **change-cause** để nhớ lý do:
    
    bash
    
    CopyEdit
    
    `kubectl annotate deployment/nginx-deployment \   kubernetes.io/change-cause="image updated to 1.16.1"`
    
- **Rollback**:
    
    bash
    
    CopyEdit
    
    `kubectl rollout undo deployment/nginx-deployment                # quay về revision trước kubectl rollout undo deployment/nginx-deployment --to-revision=2`
    

> **Không rollback được** nếu Deployment đang **paused** → phải **resume** trước.

---

## 10) Scale & Autoscale

- Scale tay:
    
    bash
    
    CopyEdit
    
    `kubectl scale deployment/nginx-deployment --replicas=10`
    
    > Nếu sau đó bạn `kubectl apply -f deployment.yaml` có field `replicas`, giá trị trong YAML sẽ **ghi đè** số bạn vừa scale tay.
    
- **HPA** (autoscale theo CPU, v.v.):
    
    bash
    
    CopyEdit
    
    `kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80`
    
    > Khi HPA quản lý, **đừng** tự set `.spec.replicas`.
    

### Proportional scaling (khi đang rollout)

Nếu đang rolling update dở, rồi được scale lên nữa, controller sẽ **chia đều theo tỉ lệ** các replica mới **vào cả RS cũ và RS mới**, thay vì dồn hết vào RS mới — giúp giảm rủi ro.

---

## 11) Pause / Resume rollout

- **Pause** để **gom nhiều thay đổi** mà **không** khởi chạy rollout ngay:
    
    bash
    
    CopyEdit
    
    `kubectl rollout pause deployment/nginx-deployment # thực hiện nhiều kubectl set image / set resources... kubectl rollout resume deployment/nginx-deployment`
    

> Trong lúc pause: thay đổi vào **`.spec.template`** sẽ **chưa** tạo revision/rollout.  
> **Không thể rollback** khi đang pause.

---

## 12) Trạng thái Deployment

### a) **Progressing**

Xảy ra khi:

- Tạo RS mới, hoặc
    
- Đang scale lên RS mới/scale xuống RS cũ, hoặc
    
- Có Pod mới trở **ready** (đủ **MinReadySeconds** nếu đặt).
    

Trong `.status.conditions` sẽ có:

vbnet

CopyEdit

`type: Progressing status: "True" reason: NewReplicaSetCreated | FoundNewReplicaSet | ReplicaSetUpdated`

### b) **Complete**

Khi:

- Tất cả replica đã cập nhật lên version mới,
    
- Tất cả available,
    
- Không còn RS cũ chạy.
    

Condition:

vbnet

CopyEdit

`type: Progressing status: "True" reason: NewReplicaSetAvailable`

### c) **Failed to progress (stuck)**

Nguyên nhân hay gặp:

- Hết quota
    
- **Readiness probe** fail
    
- Lỗi kéo image
    
- Thiếu quyền, limit range, cấu hình app sai…
    

Đặt **deadline** để “báo kẹt”:

bash

CopyEdit

`kubectl patch deployment/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'`

Khi quá hạn:

vbnet

CopyEdit

`type: Progressing status: "False" reason: ProgressDeadlineExceeded`

> Kubernetes **không tự rollback** khi kẹt; nó **chỉ báo**. Bạn/Orchestrator bên ngoài quyết định hành động.

**Exit code `kubectl rollout status`:**

- **0**: thành công
    
- **1**: quá deadline → lỗi
    

---

## 13) Các tham số hữu ích khác

### `spec.progressDeadlineSeconds`

- Thời gian chờ (giây) để coi như rollout **không còn tiến triển**.
    
- Phải **lớn hơn** `spec.minReadySeconds`. Mặc định **600**.
    

### `spec.minReadySeconds`

- Pod **phải ready liên tục** ít nhất chừng này giây mới tính là **available**.
    
- Mặc định **0**.
    

### **Terminating Pods** (tính năng **alpha** — `v1.33`)

- Khi scale down/xóa, Pod có thể **tắt chậm** → tổng số Pod **tạm thời vượt** `.spec.replicas`.
    
- Theo dõi qua **`.status.terminatingReplicas`**.
    
- Cần bật **feature gate** `DeploymentReplicaSetTerminatingReplicas` ở API server và controller-manager (theo tài liệu bạn đưa).
    

### `spec.revisionHistoryLimit`

- Số **ReplicaSet cũ** muốn **giữ lại** để có thể rollback. Mặc định **10**.
    
- Đặt **0** → **xóa hết** lịch sử RS có `replicas=0` → **không rollback** được về quá khứ.
    
- Cleanup chỉ chạy khi Deployment đã **complete**. Nếu rollout cứ fail/loop, có thể tạm thời có **nhiều RS** hơn con số limit.
    

### `spec.paused`

- `true/false` để **tạm dừng**/tiếp tục rollout. Khi paused, đổi `.spec.template` **không kích hoạt** rollout.
    

---

## 14) Strategy: `Recreate` vs `RollingUpdate`

- **Recreate**: **Kill hết Pod cũ** rồi mới **tạo Pod mới**. Downtime chắc chắn ngắn nhưng có thật.
    
- **RollingUpdate** (**mặc định**): Vừa tạo mới vừa hạ cũ theo `maxSurge`/`maxUnavailable`, **hạn chế downtime**.
    

> Nếu bạn cần “**tối đa là 1**” instance ở bất kỳ thời điểm (đảm bảo **at-most-one**) hoặc **danh tính/thứ tự ổn định**, hãy xem **StatefulSet** thay vì Deployment.

---

## 15) Những lỗi/gotcha hay gặp & cách tránh

- **Selector trùng với controller khác** (Deployment khác, StatefulSet…): hai controller **giành giật** cùng một nhóm Pod → hành vi **khó lường**.  
    → **Đặt label/selector riêng biệt**, không overlap.
    
- **Quên khớp selector ↔ template.labels**: API **từ chối**.
    
- **Sửa tay ReplicaSet** thuộc Deployment: **không nên**.
    
- **Scale tay rồi `apply` YAML có `replicas`**: giá trị trong YAML **ghi đè** số bạn vừa scale.
    
- **Readiness probe sai** → rollout kẹt (Available thấp) → kiểm tra probe trước.
    

---

## 16) “Sổ tay lệnh” nhanh

bash

CopyEdit

`# Tạo/cập nhật Deployment kubectl apply -f deployment.yaml  # Xem tổng quan kubectl get deploy kubectl describe deploy <name>  # Trạng thái rollout kubectl rollout status deploy/<name> kubectl rollout history deploy/<name> kubectl rollout history deploy/<name> --revision=2  # Cập nhật image kubectl set image deploy/<name> <container>=<image:tag>  # Rollback kubectl rollout undo deploy/<name> kubectl rollout undo deploy/<name> --to-revision=2  # Tạm dừng / Tiếp tục kubectl rollout pause deploy/<name> kubectl rollout resume deploy/<name>  # Scale tay kubectl scale deploy/<name> --replicas=10  # Autoscale (HPA) kubectl autoscale deploy/<name> --min=3 --max=10 --cpu-percent=80  # Xem RS & Pod kubectl get rs kubectl get pods --show-labels`

---

## 17) Khi nào nên dùng **Deployment**, khi nào **không**?

- **Nên**: Ứng dụng **stateless**, scale ngang thoải mái, cập nhật rolling, không yêu cầu danh tính Pod ổn định.
    
- **Không nên**: Cần **ổn định danh tính** từng Pod, **volume gắn cố định**, **thứ tự khởi động/tắt**, hoặc “**at most one**” nghiêm ngặt → xem **StatefulSet** (hoặc DaemonSet cho 1 Pod mỗi Node).
    

---

Nếu bạn muốn, mình có thể tạo một **mẫu Deployment “chuẩn sản xuất”** (readiness/liveness probes, resource requests/limits, rollout params hợp lý, ghi chú HPA) dựa trên app thực tế của bạn để bạn áp dụng ngay.