Đúng, file mới này là **phần tiếp theo của Kubernetes API Concepts**. Nó gồm 6 nhóm chính:

1. Xóa resource và `finalizer`
    
2. Force deletion
    
3. Single-resource API
    
4. Field validation
    
5. Dry-run
    
6. Cập nhật bằng `PUT`, `PATCH`, Server-Side Apply
    

Mình giải thích lần lượt.

---

# 1. Resource deletion: xóa object không phải lúc nào cũng biến mất ngay

Tài liệu nói quá trình xóa có hai giai đoạn:

```text
1. Finalization
2. Removal
```

Tức là:

```text
Yêu cầu xóa
    ↓
đánh dấu object đang bị xóa
    ↓
các controller dọn dẹp tài nguyên liên quan
    ↓
xóa object thật khỏi etcd
```

## Ví dụ object ban đầu

```json
{
  "kind": "ConfigMap",
  "metadata": {
    "name": "demo",
    "finalizers": [
      "url.io/neat-finalization",
      "other-url.io/my-finalizer"
    ],
    "deletionTimestamp": null
  }
}
```

Ở đây:

```text
deletionTimestamp = null
```

nghĩa là object chưa bị yêu cầu xóa.

Còn:

```text
finalizers:
- url.io/neat-finalization
- other-url.io/my-finalizer
```

nghĩa là trước khi object được xóa thật, có hai công việc dọn dẹp cần hoàn tất.

---

# 2. Finalizer là gì?

Finalizer chỉ là một chuỗi nằm trong:

```yaml
metadata:
  finalizers:
    - example.com/cleanup
```

Bản thân chuỗi đó không tự chạy code.

Nó giống như một “chốt chặn” nói với API Server:

> Chưa được xóa object này khỏi etcd, vì vẫn còn controller cần xử lý công việc dọn dẹp.

Controller tương ứng sẽ watch object, thấy finalizer của mình, thực hiện cleanup rồi xóa finalizer đó khỏi danh sách.

Ví dụ một custom resource quản lý database bên ngoài:

```yaml
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: production-db
  finalizers:
    - database.example.com/delete-external-db
```

Controller có thể đã tạo một database thật bên ngoài Kubernetes:

```text
AWS RDS
Cloud SQL
PostgreSQL bên ngoài
```

Khi bạn xóa Kubernetes object, nếu xóa ngay khỏi etcd thì controller có thể không còn thông tin để xóa database ngoài.

Vì vậy finalizer giữ object lại cho đến khi cleanup hoàn tất.

---

# 3. Khi gửi DELETE, API Server làm gì?

Giả sử bạn chạy:

```bash
kubectl delete configmap demo
```

Client gửi:

```http
DELETE /api/v1/namespaces/default/configmaps/demo
```

Nếu object có finalizer, API Server không xóa ngay khỏi etcd.

Thay vào đó nó cập nhật:

```yaml
metadata:
  deletionTimestamp: "2026-07-04T15:30:00Z"
```

Object lúc này vẫn tồn tại:

```bash
kubectl get configmap demo -o yaml
```

vẫn có thể thấy nó.

Nhưng nó đã bước vào trạng thái:

```text
Terminating / đang được finalization
```

Luồng:

```text
Client gửi DELETE
        ↓
API Server đặt deletionTimestamp
        ↓
object vẫn tồn tại trong etcd
        ↓
controller nhìn thấy deletionTimestamp
        ↓
controller thực hiện cleanup
        ↓
controller xóa finalizer của mình
        ↓
finalizer cuối cùng bị xóa
        ↓
API Server xóa object khỏi etcd
```

---

# 4. Vì sao sau khi có `deletionTimestamp` không thể hủy xóa dễ dàng?

Một khi:

```yaml
metadata:
  deletionTimestamp: ...
```

đã được đặt, object được coi là đang bị xóa.

Thông thường bạn không thể chỉ sửa:

```yaml
deletionTimestamp: null
```

để cứu nó, vì trường này do API Server quản lý.

Trong giai đoạn này, các controller chỉ nên:

- cleanup
    
- xóa finalizer của chính mình
    

Chứ không nên cố đưa object quay lại trạng thái bình thường.

---

# 5. Controller xử lý finalizer như thế nào?

Pseudo-code:

```go
func reconcile(obj Object) {
    if obj.DeletionTimestamp == nil {
        if !contains(obj.Finalizers, myFinalizer) {
            addFinalizer(obj)
        }

        ensureExternalResourceExists(obj)
        return
    }

    if contains(obj.Finalizers, myFinalizer) {
        deleteExternalResource(obj)
        removeFinalizer(obj)
    }
}
```

Logic có hai nhánh.

## Object chưa bị xóa

```text
deletionTimestamp == nil
```

Controller:

- thêm finalizer nếu chưa có
    
- tạo hoặc quản lý tài nguyên bên ngoài
    

## Object đang bị xóa

```text
deletionTimestamp != nil
```

Controller:

- dọn dẹp tài nguyên ngoài
    
- xóa finalizer của mình
    

Khi không còn finalizer nào, API Server xóa object thật.

---

# 6. Tại sao finalizer không có thứ tự bắt buộc?

Ví dụ:

```yaml
finalizers:
  - controller-a.example.com/cleanup
  - controller-b.example.com/cleanup
```

Bạn có thể tưởng Kubernetes sẽ làm:

```text
A xong trước
rồi mới đến B
```

Nhưng thực tế Kubernetes không đảm bảo thứ tự đó.

Cả A và B đều có thể cleanup:

```text
bất cứ lúc nào
theo bất cứ thứ tự nào
thậm chí đồng thời
```

## Vì sao không ép thứ tự?

Vì có nguy cơ deadlock.

Giả sử nếu Kubernetes buộc A phải hoàn thành trước B:

```text
Finalizer A đang chờ B tạo một tín hiệu.
Finalizer B chưa được phép chạy vì A đứng trước.
```

Kết quả:

```text
A chờ B
B chờ A xong
→ object kẹt vĩnh viễn
```

Ngoài ra, `metadata.finalizers` là danh sách dùng chung. Bất kỳ actor nào có quyền update object đều có thể đổi thứ tự danh sách.

Do đó controller không được dựa vào:

```text
“Tôi đứng đầu danh sách nên tôi chạy trước.”
```

Mỗi finalizer phải được thiết kế độc lập và có khả năng xử lý khi các finalizer khác chạy trước hoặc sau.

---

# 7. Khi nào object bị xóa thật khỏi etcd?

Ví dụ ban đầu:

```yaml
finalizers:
  - a.example.com
  - b.example.com
```

Controller A hoàn tất:

```yaml
finalizers:
  - b.example.com
```

Object vẫn tồn tại.

Controller B hoàn tất:

```yaml
finalizers: []
```

Khi finalizer cuối cùng bị xóa:

```text
API Server xóa record của object khỏi etcd
```

Watch client sẽ nhận:

```json
{
  "type": "DELETED",
  "object": {
    "metadata": {
      "name": "demo"
    }
  }
}
```

---

# 8. Trường hợp object bị kẹt `Terminating`

Một object thường bị kẹt khi:

```text
deletionTimestamp đã có
nhưng finalizer không bị xóa
```

Ví dụ:

```yaml
metadata:
  deletionTimestamp: "2026-07-04T15:30:00Z"
  finalizers:
    - operator.example.com/cleanup
```

Nhưng operator tương ứng:

- bị crash
    
- đã bị uninstall
    
- không còn quyền RBAC
    
- cleanup bị lỗi
    
- external API không phản hồi
    

Khi đó object cứ nằm mãi ở trạng thái terminating.

Bạn có thể xem:

```bash
kubectl get <resource> <name> -o yaml
```

và kiểm tra:

```yaml
metadata:
  deletionTimestamp:
  finalizers:
```

---

# 9. Xóa finalizer thủ công có nguy hiểm không?

Bạn có thể patch:

```bash
kubectl patch <resource> <name> \
  --type=json \
  -p='[
    {
      "op": "remove",
      "path": "/metadata/finalizers"
    }
  ]'
```

Nhưng làm vậy nghĩa là:

> Bỏ qua cleanup mà controller định thực hiện.

Hậu quả có thể là:

- cloud load balancer còn sót
    
- volume còn sót
    
- database ngoài không bị xóa
    
- firewall rule còn sót
    
- dependent resource bị orphan
    
- storage chưa detach đúng cách
    

Do đó chỉ nên xóa finalizer thủ công khi đã hiểu finalizer đó bảo vệ cái gì.

---

# 10. Finalizer khác ownerReference như thế nào?

Hai cơ chế liên quan đến xóa nhưng không giống nhau.

## Finalizer

Ngăn object bị xóa cho đến khi cleanup hoàn tất:

```text
Đừng xóa tôi cho đến khi công việc X hoàn thành.
```

## `ownerReferences`

Mô tả quan hệ cha-con để Garbage Collector biết tài nguyên phụ thuộc vào ai:

```text
ReplicaSet thuộc Deployment
Pod thuộc ReplicaSet
```

Ví dụ:

```yaml
metadata:
  ownerReferences:
    - kind: ReplicaSet
      name: nginx-abc
```

Khi owner bị xóa, Garbage Collector xử lý dependent tùy `propagationPolicy`.

Hai cơ chế có thể kết hợp:

```text
OwnerReferences:
ai sở hữu ai?

Finalizers:
trước khi xóa phải cleanup gì?
```

---

# 11. Force deletion trong đoạn này là loại đặc biệt

Phần tài liệu này không nói về lệnh quen thuộc:

```bash
kubectl delete pod --force --grace-period=0
```

Nó nói về một cơ chế nguy hiểm hơn:

```text
unsafe deletion của object bị hỏng hoặc không thể đọc từ storage
```

Option:

```text
ignoreStoreReadErrorWithClusterBreakingPotential=true
```

Tên dài như vậy để nhấn mạnh:

> Nó có khả năng phá cluster.

Tính năng này ở Kubernetes v1.36 vẫn là alpha và mặc định tắt. Muốn dùng phải bật feature gate trên API Server:

```text
--feature-gates=AllowUnsafeMalformedObjectDeletion=true
```

([Kubernetes](https://kubernetes.io/docs/reference/using-api/api-concepts/?utm_source=chatgpt.com "Kubernetes API Concepts | Kubernetes"))

---

# 12. “Corrupt resource” nghĩa là gì?

Object được coi là hỏng khi API Server không thể đọc nó từ storage do:

## Transformation error

Ví dụ dữ liệu trong etcd được mã hóa, nhưng:

- encryption key bị mất
    
- encryption config thay đổi sai
    
- không giải mã được ciphertext
    

```text
etcd có record
nhưng API Server không giải mã được
```

## Decode error

API Server đọc được bytes nhưng không chuyển được thành object hợp lệ.

Ví dụ:

```text
stored bytes
    ↓
decode JSON/Protobuf
    ↓
lỗi
```

Khi đó API Server có thể không:

- GET được object
    
- update được object
    
- xóa theo flow bình thường
    

---

# 13. Unsafe force delete hoạt động thế nào?

API Server trước tiên vẫn thử xóa bình thường:

```text
DELETE bình thường
    ↓
đọc object từ storage
    ↓
nếu đọc được → xử lý bình thường
```

Chỉ nếu gặp lỗi corrupt resource và option nguy hiểm đã được bật:

```text
bỏ qua một số kiểm tra
xóa thẳng record khỏi storage
```

Cơ chế này:

- bỏ qua finalizer
    
- bỏ qua precondition
    
- có thể bỏ qua flow cleanup
    
- có thể làm workload hoặc cluster không nhất quán
    

Do đó tên option có chữ:

```text
ClusterBreakingPotential
```

không phải để trang trí.

---

# 14. Vì sao object đọc được thì API Server lại từ chối unsafe delete?

Nếu object vẫn đọc bình thường, bạn phải dùng flow deletion bình thường.

API Server không cho phép sử dụng option này như một cách tiện lợi để:

```text
bỏ finalizer
bỏ ownerReferences
bỏ precondition
```

Nó chỉ dành cho object thực sự corrupt.

---

# 15. Quyền RBAC đặc biệt

Người dùng phải có cả hai quyền:

```text
delete
unsafe-delete-ignore-read-errors
```

Ví dụ có quyền `delete` thông thường vẫn chưa đủ.

Đây là lớp bảo vệ vì thao tác có thể xóa object mà không chạy cleanup.

---

# 16. Single resource API nghĩa là gì?

Các verb sau chỉ tác động lên một resource mỗi request:

```text
get
create
update
patch
delete
proxy
```

Ví dụ:

```http
GET /api/v1/namespaces/default/pods/nginx
```

lấy một Pod.

```http
DELETE /api/v1/namespaces/default/pods/nginx
```

xóa một Pod.

Bạn không thể gửi một request kiểu:

```http
DELETE pod-a, pod-b, pod-c
```

và yêu cầu API Server xử lý chúng như một transaction nguyên tử.

---

# 17. Khi chạy `kubectl delete -f file.yaml` có nhiều object thì sao?

Giả sử file có:

```yaml
Deployment
---
Service
---
ConfigMap
```

`kubectl` sẽ gửi nhiều request riêng:

```text
DELETE Deployment
DELETE Service
DELETE ConfigMap
```

Không phải:

```text
một transaction chứa cả ba object
```

Do đó có thể xảy ra:

```text
Deployment xóa thành công
Service xóa thành công
ConfigMap xóa thất bại
```

Kubernetes API không tự rollback hai object đầu.

Đây là điểm quan trọng:

> Nhiều thao tác qua `kubectl` không đồng nghĩa với transaction database.

---

# 18. `list`, `watch` và `deletecollection`

Các verb hỗ trợ collection:

## `list`

```http
GET /api/v1/namespaces/default/pods
```

trả nhiều Pod.

## `watch`

Theo dõi nhiều object trong collection.

## `deletecollection`

Xóa một collection theo selector.

Ví dụ về logic:

```http
DELETE /api/v1/namespaces/default/pods?labelSelector=app%3Ddemo
```

Nhưng ngay cả `deletecollection` cũng không nên được hiểu như transaction ACID:

```text
hoặc tất cả xóa
hoặc không object nào xóa
```

Nó là API collection-level, không phải transaction nhiều object theo kiểu SQL.

---

# 19. Field validation: Kubernetes kiểm tra trường dữ liệu như thế nào?

Kubernetes luôn kiểm tra kiểu của field.

Ví dụ schema nói:

```yaml
spec:
  replicas:
    type: integer
```

Thì hợp lệ:

```yaml
replicas: 3
```

Không hợp lệ:

```yaml
replicas: "ba"
```

Nếu field là array string:

```yaml
command:
  - sleep
  - "3600"
```

thì không thể gửi object thay thế hoàn toàn sai kiểu.

Ngoài type còn có:

- required field
    
- enum
    
- range
    
- format
    
- structural schema
    
- custom validation với CEL ở một số API/CRD
    

---

# 20. Unknown field là gì?

Ví dụ bạn viết sai:

```yaml
spec:
  replica: 3
```

trong khi field đúng là:

```yaml
spec:
  replicas: 3
```

`replica` không tồn tại trong OpenAPI schema của Deployment.

Đó là unknown field.

Một ví dụ khác:

```yaml
spec:
  containers:
    - name: nginx
      imagee: nginx
```

Bạn gõ nhầm:

```text
imagee
```

thay vì:

```text
image
```

Nếu unknown field bị bỏ âm thầm, Pod có thể dùng image mặc định không tồn tại hoặc request thất bại ở chỗ khác, rất khó debug.

---

# 21. Duplicate field là gì?

JSON/YAML có thể chứa cùng field hai lần:

```yaml
spec:
  replicas: 3
  replicas: 5
```

Vấn đề là parser khác nhau có thể:

- lấy giá trị đầu
    
- lấy giá trị cuối
    
- báo lỗi
    

Kubernetes phát hiện duplicate field để tránh kết quả mơ hồ.

---

# 22. Ba mức `fieldValidation`

Query parameter:

```text
fieldValidation=Ignore
fieldValidation=Warn
fieldValidation=Strict
```

## Ignore

API Server:

- bỏ unknown field
    
- xử lý duplicate field theo hành vi parser
    
- không cảnh báo
    
- vẫn chấp nhận request nếu không có lỗi khác
    

Ví dụ:

```yaml
spec:
  replicas: 3
  unknownField: hello
```

`unknownField` bị loại bỏ.

Client không được thông báo.

## Warn

Đây là mặc định phía API Server.

API Server:

- vẫn chấp nhận request
    
- bỏ field không hợp lệ
    
- trả warning qua HTTP response header
    

Ví dụ:

```http
Warning: 299 - "unknown field \"spec.replica\""
```

`kubectl` có thể hiển thị:

```text
Warning: unknown field "spec.replica"
```

## Strict

API Server từ chối toàn bộ request:

```http
400 Bad Request
```

và liệt kê unknown hoặc duplicate fields.

Đây là chế độ an toàn nhất để phát hiện typo.

---

# 23. `kubectl --validate`

Theo tài liệu hiện tại, `kubectl` dùng:

```bash
--validate=ignore
--validate=warn
--validate=strict
```

Ngoài ra:

```bash
--validate=true
```

tương đương strict.

```bash
--validate=false
```

tương đương ignore.

Mặc định hiện nay:

```bash
--validate=true
```

tức strict server-side validation. ([Kubernetes](https://kubernetes.io/docs/reference/using-api/api-concepts?utm_source=chatgpt.com "Kubernetes API Concepts | Kubernetes"))

Ví dụ:

```bash
kubectl apply -f deployment.yaml --validate=strict
```

Nếu có:

```yaml
replicass: 3
```

request sẽ bị từ chối.

---

# 24. Vì sao vẫn có thể gặp lỗi khác trước unknown field?

Giả sử object có cả hai lỗi:

```yaml
spec:
  replicas: "three"
  unknowField: abc
```

`replicas` sai type và `unknownField` không tồn tại.

API Server có thể gặp lỗi type trước:

```text
cannot unmarshal string into integer
```

và trả `400 Bad Request`.

Nó chưa chắc liệt kê unknown field, vì quá trình decode đã thất bại trước khi tới bước validation đó.

---

# 25. CRD và unknown fields

Với CRD, mặc định structural schema có thể prune field không khai báo.

Ví dụ CRD chỉ khai báo:

```yaml
spec:
  properties:
    size:
      type: integer
```

User gửi:

```yaml
spec:
  size: 3
  secretExtraField: hello
```

`secretExtraField` có thể bị bỏ.

CRD có thể chọn giữ unknown fields bằng:

```yaml
x-kubernetes-preserve-unknown-fields: true
```

Nhưng dùng nó cần cẩn thận vì:

- schema validation yếu hơn
    
- field ownership khó hơn
    
- client khó hiểu dữ liệu
    
- Server-Side Apply có thể phức tạp hơn
    

---

# 26. Dry-run là gì?

Dry-run cho phép request đi qua gần như toàn bộ pipeline xử lý nhưng không ghi object xuống storage.

Ví dụ:

```bash
kubectl apply -f pod.yaml --dry-run=server -o yaml
```

Luồng:

```text
request
  ↓
authentication
  ↓
authorization
  ↓
defaulting
  ↓
mutating admission
  ↓
validation
  ↓
validating admission
  ↓
merge / patch logic
  ↓
trả object kết quả
  ✗ không lưu vào etcd
```

Mục đích:

> Kiểm tra xem request thật sẽ được xử lý thành object nào mà không thay đổi cluster.

---

# 27. `dryRun=All`

HTTP request:

```http
POST /api/v1/namespaces/test/pods?dryRun=All
```

`All` hiện là giá trị hợp lệ chính của dry-run API.

API Server thực hiện:

- default fields
    
- mutation
    
- validation
    
- admission
    
- patch merge
    
- schema checking
    
- authorization
    

Nhưng không persist object xuống etcd. ([Kubernetes](https://kubernetes.io/docs/reference/using-api/api-concepts/?utm_source=chatgpt.com "Kubernetes API Concepts | Kubernetes"))

Một lưu ý về đoạn văn trong file: phần:

```text
[no value set]
Allow side effects
```

nên hiểu là **không đặt tham số `dryRun` thì request thật được phép tạo side effect**. Request dry-run chuẩn là:

```text
?dryRun=All
```

Không nên hiểu rằng chỉ viết `?dryRun` rỗng là dry-run hợp lệ.

---

# 28. Ví dụ dry-run Pod

```bash
kubectl run nginx \
  --image=nginx \
  --dry-run=server \
  -o yaml
```

API Server có thể trả:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  containers:
    - image: nginx
      name: nginx
  restartPolicy: Always
```

Nhưng:

```bash
kubectl get pod nginx
```

sẽ không thấy Pod, vì object chưa được lưu.

---

# 29. Client dry-run và server dry-run

## Client dry-run

```bash
--dry-run=client
```

`kubectl` tự tạo object cục bộ.

Không gửi request đầy đủ tới API Server.

Do đó có thể không kiểm tra được:

- admission webhook
    
- server-side defaulting
    
- CRD schema mới nhất
    
- policy của cluster
    
- server-side validation thực tế
    

## Server dry-run

```bash
--dry-run=server
```

Request được gửi tới API Server và chạy qua pipeline thật, chỉ bỏ bước persist.

Thường chính xác hơn.

---

# 30. Admission webhook và dry-run

Một admission webhook có thể làm side effect ngoài Kubernetes:

```text
gọi API cloud
ghi database
gửi message
tạo resource ngoài
```

Nếu dry-run vẫn gọi webhook như bình thường, nó có thể vô tình tạo side effect dù object không được lưu.

Do đó webhook phải khai báo:

```yaml
sideEffects: None
```

hoặc:

```yaml
sideEffects: NoneOnDryRun
```

## `None`

Webhook cam kết không có side effect.

## `NoneOnDryRun`

Webhook có thể có side effect với request thật, nhưng biết cách không thực hiện side effect khi:

```text
AdmissionReview.request.dryRun == true
```

Nếu webhook có side effect mà không hỗ trợ dry-run an toàn, API Server có thể từ chối dry-run.

---

# 31. Giá trị sinh ra trong dry-run không đáng tin tuyệt đối

Một số field được tạo động:

```text
name từ generateName
UID
creationTimestamp
deletionTimestamp
resourceVersion
Service clusterIP
NodePort
field do mutating webhook thêm
```

Ví dụ:

```yaml
metadata:
  generateName: job-
```

Dry-run có thể trả:

```yaml
name: job-x8abc
```

Nhưng request thật có thể sinh:

```yaml
name: job-f92kd
```

Không được lấy tên dry-run rồi giả định request thật sẽ giống hệt.

Tương tự:

```text
UID dry-run ≠ UID thật
resourceVersion dry-run ≠ resourceVersion thật
```

Vì `resourceVersion` chỉ thực sự có ý nghĩa với object đã persist.

---

# 32. Dry-run vẫn cần quyền RBAC thật

Bạn không thể dùng dry-run để xem mình “có thể làm gì” nếu chưa có quyền.

Ví dụ muốn:

```bash
kubectl patch deployment nginx --dry-run=server
```

thì bạn vẫn phải có:

```yaml
verbs:
  - patch
resources:
  - deployments
```

Lý do:

> Dữ liệu trả về từ admission/defaulting có thể nhạy cảm, và dry-run mô phỏng đúng thao tác thật.

Dry-run không phải cơ chế bypass authorization.

---

# 33. Update existing resources: `PUT` là thay toàn bộ object

HTTP:

```http
PUT /apis/apps/v1/namespaces/default/deployments/nginx
```

Client thường phải:

1. GET object hiện tại
    
2. sửa object cục bộ
    
3. gửi lại toàn bộ object bằng PUT
    

Ví dụ GET được:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo
  resourceVersion: "100"
data:
  a: one
  b: two
```

Client muốn đổi `a`:

```yaml
data:
  a: changed
  b: two
```

Nó gửi lại cả object kèm:

```yaml
resourceVersion: "100"
```

---

# 34. Vì sao PUT phải gửi `resourceVersion`?

Để tránh lost update.

Giả sử hai client cùng đọc object ở version `100`.

```text
Client A GET RV=100
Client B GET RV=100
```

Client A sửa `a` và PUT:

```text
RV 100 → thành RV 101
```

Client B vẫn giữ bản cũ RV `100`, sửa `b`, rồi PUT.

Nếu API Server chấp nhận, bản của B có thể ghi đè thay đổi A.

Kubernetes kiểm tra:

```text
resourceVersion client gửi = resourceVersion hiện tại?
```

Với B:

```text
B gửi:     100
Hiện tại:  101
```

Không khớp nên trả:

```http
409 Conflict
```

Luồng:

```text
Client A            API Server            Client B
   |                    |                    |
   | GET RV=100         |      GET RV=100   |
   |<-------------------|------------------->|
   |                    |                    |
   | PUT RV=100         |                    |
   |------------------->|                    |
   |       success, RV=101                   |
   |                    |<---- PUT RV=100 ---|
   |                    |---- 409 Conflict ->|
```

Client B phải:

```text
GET lại bản RV=101
áp dụng thay đổi của mình
PUT lại
```

---

# 35. Lost update là gì?

Giả sử object ban đầu:

```yaml
data:
  countA: 0
  countB: 0
```

Client A đổi:

```yaml
countA: 1
```

Client B đổi:

```yaml
countB: 1
```

Nếu B ghi toàn bộ object cũ sau A, kết quả có thể là:

```yaml
countA: 0
countB: 1
```

Thay đổi của A biến mất.

Đó là lost update.

`resourceVersion` giúp phát hiện thay vì âm thầm ghi đè.

---

# 36. Nhược điểm của PUT

## Phải gửi toàn bộ object

Bạn chỉ đổi một field nhưng vẫn gửi toàn object.

## Dễ làm rơi field

Giả sử server có field mới nhưng client library cũ không hiểu.

Client:

```text
GET object có field mới
decode vào struct cũ
field mới bị bỏ
encode lại
PUT
```

Kết quả field mới có thể biến mất.

## Conflict nhiều

Dù hai client sửa hai field không liên quan, chỉ cần object resourceVersion đổi là PUT cũ bị conflict.

Ví dụ:

```text
Controller A sửa metadata.annotation
Controller B sửa spec.replicas
```

Cả hai vẫn tranh chấp trên version của toàn object.

---

# 37. PATCH khác PUT thế nào?

## PUT

```text
Đây là toàn bộ object mới.
Hãy thay object cũ bằng object này.
```

## PATCH

```text
Đây là thay đổi cần áp dụng lên object hiện tại.
```

Ví dụ chỉ đổi replicas:

```json
{
  "spec": {
    "replicas": 5
  }
}
```

PATCH thường gửi ít dữ liệu hơn và ít xung đột với field không liên quan hơn.

---

# 38. Bốn loại PATCH

Kubernetes phân biệt loại patch bằng header:

```http
Content-Type: ...
```

Có bốn loại trong tài liệu.

---

# 39. Server-Side Apply

Content-Type:

```http
application/apply-patch+yaml
```

Ví dụ:

```bash
kubectl apply --server-side -f deployment.yaml
```

Bạn gửi trạng thái mong muốn của những field mình quản lý.

API Server theo dõi ownership trong:

```yaml
metadata:
  managedFields:
```

Ví dụ:

```text
kubectl quản lý spec.replicas
operator quản lý metadata.annotations["operator"]
HPA quản lý spec.replicas
```

Nếu hai manager muốn quản lý cùng một field với giá trị khác nhau, Server-Side Apply có thể báo conflict.

## Create-or-update

Nếu object chưa tồn tại:

```text
logical operation = create
```

Nếu đã tồn tại:

```text
logical operation = patch
```

Nó gần giống UPSERT.

---

# 40. Field manager trong Server-Side Apply

Ví dụ:

```bash
kubectl apply --server-side \
  --field-manager=my-deployer \
  -f deployment.yaml
```

API Server ghi rằng:

```text
my-deployer sở hữu các field được apply
```

Nếu manager khác apply cùng field:

```text
field ownership conflict
```

Muốn chiếm quyền có thể dùng:

```bash
--force-conflicts
```

Nhưng điều này nghĩa là manager mới giành ownership của field, nên phải dùng cẩn thận.

---

# 41. JSON Patch

Content-Type:

```http
application/json-patch+json
```

Theo RFC 6902.

Body là danh sách operation:

```json
[
  {
    "op": "replace",
    "path": "/spec/replicas",
    "value": 5
  }
]
```

Các operation phổ biến:

```text
add
remove
replace
move
copy
test
```

Ví dụ:

```bash
kubectl patch deployment nginx \
  --type=json \
  -p='[
    {
      "op": "replace",
      "path": "/spec/replicas",
      "value": 5
    }
  ]'
```

---

# 42. JSON Patch có điều kiện bằng `test`

Ví dụ chỉ thay replicas nếu hiện tại đang là 3:

```json
[
  {
    "op": "test",
    "path": "/spec/replicas",
    "value": 3
  },
  {
    "op": "replace",
    "path": "/spec/replicas",
    "value": 5
  }
]
```

Nếu replicas không còn là `3`, patch thất bại.

Điều này giúp tránh lost update mà không nhất thiết khóa cả object bằng resourceVersion.

Bạn có thể hiểu:

```text
Nếu giá trị hiện tại đúng như tôi mong đợi
thì mới thực hiện thay đổi
```

---

# 43. JSON Merge Patch

Content-Type:

```http
application/merge-patch+json
```

Bạn gửi object một phần:

```json
{
  "spec": {
    "replicas": 5
  }
}
```

API Server merge phần đó vào object hiện tại.

Thông thường:

```json
{
  "field": null
}
```

có nghĩa là xóa field.

JSON Merge Patch khá dễ hiểu với object thường, nhưng xử lý array không thông minh.

Ví dụ muốn sửa một container trong danh sách `containers`, merge patch có thể thay toàn bộ array thay vì merge theo `name`.

---

# 44. Strategic Merge Patch

Content-Type:

```http
application/strategic-merge-patch+json
```

Đây là extension riêng của Kubernetes.

Nó hiểu một số list theo “merge key”.

Ví dụ danh sách container:

```yaml
containers:
  - name: app
    image: app:v1
  - name: sidecar
    image: sidecar:v1
```

Strategic Merge Patch có thể biết rằng container được định danh theo:

```text
name
```

Nên patch:

```yaml
spec:
  template:
    spec:
      containers:
        - name: app
          image: app:v2
```

chỉ sửa container `app`, không thay toàn bộ danh sách.

## Hạn chế

Strategic Merge Patch:

- hỗ trợ built-in API
    
- không dùng được cho CRD thông thường
    
- đang dần được Server-Side Apply thay thế
    

---

# 45. So sánh nhanh bốn loại patch

|Cơ chế|Ý tưởng|
|---|---|
|Server-Side Apply|Gửi desired fields, server quản lý field ownership|
|JSON Patch|Gửi chuỗi thao tác chính xác `add/remove/replace/test`|
|JSON Merge Patch|Gửi object một phần để merge|
|Strategic Merge Patch|Merge kiểu Kubernetes, hiểu list key của built-in resources|

---

# 46. Khi nào dùng PUT?

PUT phù hợp khi:

- bạn quản lý toàn bộ object
    
- muốn replace rõ ràng
    
- đã GET object hiện tại
    
- muốn dùng `resourceVersion` để chống lost update
    
- có retry logic cho `409 Conflict`
    

Ví dụ controller sở hữu toàn bộ một custom resource status hoặc một object đơn giản.

Nhưng trong Kubernetes controller, thường phải cẩn thận vì nhiều actor cùng sửa một object.

---

# 47. Khi nào dùng JSON Patch?

JSON Patch phù hợp khi:

- cần sửa field chính xác
    
- cần operation `test`
    
- cần add/remove item theo path
    
- muốn thay đổi nhỏ
    
- muốn tránh gửi toàn object
    

Ví dụ xóa finalizer:

```json
[
  {
    "op": "remove",
    "path": "/metadata/finalizers/0"
  }
]
```

Hoặc chỉ sửa replicas khi giá trị hiện tại đúng kỳ vọng.

---

# 48. Khi nào dùng Server-Side Apply?

Phù hợp khi:

- declarative configuration
    
- một manager quản lý một tập field
    
- muốn API Server theo dõi ownership
    
- muốn create-or-update một lần
    
- muốn phát hiện field conflict rõ ràng
    
- đang xây công cụ tương tự `kubectl apply`
    

Không phù hợp khi thay đổi phụ thuộc vào giá trị hiện tại.

Ví dụ:

```text
count = count + 1
```

Server-Side Apply chỉ nói desired value, không có ý nghĩa “đọc hiện tại rồi cộng một”.

Trường hợp đó cần:

- GET + update có conflict retry
    
- JSON Patch với `test`
    
- endpoint/subresource chuyên biệt
    

---

# 49. Tại sao patch thường ít conflict hơn PUT?

Giả sử object:

```yaml
spec:
  replicas: 3
metadata:
  annotations:
    foo: bar
```

Client A liên tục sửa annotation.

Client B chỉ muốn sửa replicas.

## Với PUT

B GET object.

Trong lúc đó A sửa annotation làm resourceVersion đổi.

B PUT bản cũ:

```text
409 Conflict
```

Dù B không quan tâm annotation.

## Với patch đơn giản

B gửi:

```json
{
  "spec": {
    "replicas": 5
  }
}
```

API Server có thể áp dụng ngay lên object hiện tại mà không cần B gửi lại annotation.

Do đó ít retry hơn.

Nhưng nếu thay đổi phải dựa trên trạng thái cũ, B vẫn nên dùng điều kiện như:

- `resourceVersion`
    
- JSON Patch `test`
    
- Server-Side Apply conflict semantics
    

---

# 50. Toàn bộ luồng API request liên quan các phần này

Một request thay đổi object thường đi qua:

```text
Client
  ↓
TLS
  ↓
Authentication
  ↓
Authorization
  ↓
Decode request
  ↓
Field validation
  ↓
Defaulting
  ↓
Mutating admission
  ↓
Validation
  ↓
Validating admission
  ↓
Conflict / resourceVersion checks
  ↓
Persist vào etcd
  ↓
Watch event phát cho clients
```

Nếu là dry-run:

```text
mọi bước gần như bình thường
nhưng bỏ persist
```

Nếu là DELETE có finalizer:

```text
persist deletionTimestamp
chưa xóa record
controller cleanup
xóa finalizer cuối
xóa record khỏi etcd
```

Nếu là PUT với version cũ:

```text
dừng ở conflict check
trả 409
```

---

# 51. Ví dụ tổng hợp: xóa một custom database

Ban đầu:

```yaml
apiVersion: db.example.com/v1
kind: Database
metadata:
  name: orders-db
  resourceVersion: "500"
  finalizers:
    - db.example.com/cloud-cleanup
spec:
  provider: aws
```

Bạn chạy:

```bash
kubectl delete database orders-db
```

API Server update object:

```yaml
metadata:
  resourceVersion: "501"
  deletionTimestamp: "2026-07-04T15:30:00Z"
  finalizers:
    - db.example.com/cloud-cleanup
```

Watch stream gửi:

```json
{
  "type": "MODIFIED",
  "object": {
    "metadata": {
      "resourceVersion": "501",
      "deletionTimestamp": "..."
    }
  }
}
```

Database controller nhận event:

```text
thấy deletionTimestamp
    ↓
gọi AWS xóa RDS
    ↓
đợi RDS bị xóa
    ↓
PATCH bỏ finalizer
```

Sau patch:

```yaml
metadata:
  resourceVersion: "502"
  finalizers: []
```

API Server xóa object khỏi etcd.

Watch stream gửi:

```json
{
  "type": "DELETED",
  "object": {
    "metadata": {
      "name": "orders-db"
    }
  }
}
```

Đây cũng cho thấy phần file mới liên kết trực tiếp với phần watch trước:

```text
DELETE request
không nhất thiết tạo DELETED event ngay

Có thể trước tiên tạo MODIFIED:
deletionTimestamp được đặt

Sau khi finalizer hết:
mới có DELETED
```

Đây là điểm quan trọng nhất để hiểu deletion trong Kubernetes.