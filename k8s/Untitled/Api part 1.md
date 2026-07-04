Đoạn này mô tả **cách Kubernetes client theo dõi thay đổi của object một cách hiệu quả**, thay vì cứ vài giây lại tải toàn bộ dữ liệu từ API Server.

Ví dụ các thành phần như:

- kubelet
    
- scheduler
    
- controller-manager
    
- các operator/controller tự viết
    
- `kubectl get --watch`
    

đều có thể dùng cơ chế này.

---

# 1. Vấn đề Kubernetes cần giải quyết

Giả sử một controller cần quản lý tất cả Pod trong namespace `test`.

Cách ngây thơ là cứ liên tục gọi:

```http
GET /api/v1/namespaces/test/pods
```

Ví dụ mỗi giây gọi một lần:

```text
Controller                    API Server
    |                             |
    |------ GET all Pods -------->|
    |<----- 10.000 Pods ----------|
    |                             |
    |------ GET all Pods -------->|
    |<----- 10.000 Pods ----------|
    |                             |
    |------ GET all Pods -------->|
```

Cách này rất tốn:

- CPU của API Server
    
- RAM
    
- băng thông mạng
    
- CPU của client để parse JSON
    
- tải lên etcd hoặc cache của API Server
    

Trong khi có thể trong 10.000 Pod chỉ có **một Pod thay đổi**.

Kubernetes giải quyết bằng mô hình:

```text
LIST trạng thái hiện tại
          +
WATCH các thay đổi tiếp theo
```

Tức là:

```text
1. Lấy ảnh chụp hiện tại của hệ thống.
2. Ghi nhớ mốc phiên bản.
3. Mở kết nối watch từ mốc đó.
4. Chỉ nhận những thay đổi mới.
```

---

# 2. `resourceVersion` là gì?

Mỗi Kubernetes object có trường:

```yaml
metadata:
  resourceVersion: "10596"
```

`resourceVersion` là một **mốc phiên bản của dữ liệu trong tầng lưu trữ bên dưới**.

Trong Kubernetes thông thường, tầng lưu trữ đó là:

```text
API Server
    |
    v
etcd
```

Nói đơn giản:

> `resourceVersion` cho biết object hoặc tập kết quả này tương ứng với trạng thái dữ liệu tại mốc nào.

Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  resourceVersion: "10596"
```

Sau khi Pod được cập nhật:

```yaml
metadata:
  name: nginx
  resourceVersion: "11020"
```

## Không nên hiểu nó là version nghiệp vụ

`resourceVersion` không phải:

```text
Pod version 1
Pod version 2
Pod version 3
```

Và không nên tự thực hiện phép toán như:

```text
11020 - 10596 = Pod đã sửa 424 lần
```

Điều đó sai.

Nó là token/mốc dùng cho:

- concurrency control
    
- watch
    
- cache consistency
    
- phát hiện object đã thay đổi
    

Trong hệ thống dùng etcd, giá trị này thường liên quan đến revision của etcd, nhưng client nên coi nó là **chuỗi opaque**, tức là chỉ lưu lại và gửi lại, không tự diễn giải ý nghĩa số học.

---

# 3. `resourceVersion` của một object và của một danh sách

Đây là chỗ dễ nhầm.

## `resourceVersion` của từng Pod

```json
{
  "kind": "Pod",
  "metadata": {
    "name": "pod-a",
    "resourceVersion": "10010"
  }
}
```

Nó biểu diễn version của object `pod-a`.

## `resourceVersion` của `PodList`

Khi gọi:

```http
GET /api/v1/namespaces/test/pods
```

API Server trả:

```json
{
  "kind": "PodList",
  "metadata": {
    "resourceVersion": "10245"
  },
  "items": [
    {
      "metadata": {
        "name": "pod-a",
        "resourceVersion": "10010"
      }
    },
    {
      "metadata": {
        "name": "pod-b",
        "resourceVersion": "10150"
      }
    }
  ]
}
```

Ở đây có hai loại version:

```text
PodList.metadata.resourceVersion = 10245
Pod A.metadata.resourceVersion    = 10010
Pod B.metadata.resourceVersion    = 10150
```

`PodList.metadata.resourceVersion = 10245` có ý nghĩa:

> Danh sách này là ảnh chụp trạng thái collection Pod tại mốc `10245`.

Client dùng chính giá trị của **list metadata** này để bắt đầu watch:

```http
GET /api/v1/namespaces/test/pods?watch=1&resourceVersion=10245
```

---

# 4. Luồng list rồi watch

## Bước 1: LIST

Client gọi:

```http
GET /api/v1/namespaces/test/pods
```

API Server trả:

```json
{
  "kind": "PodList",
  "metadata": {
    "resourceVersion": "10245"
  },
  "items": [...]
}
```

Client lúc này xây dựng local cache:

```text
Local cache:
- pod-a
- pod-b
- pod-c
```

Đồng thời ghi nhớ:

```text
lastResourceVersion = 10245
```

## Bước 2: WATCH từ mốc đó

Client gửi:

```http
GET /api/v1/namespaces/test/pods?watch=1&resourceVersion=10245
```

Ý nghĩa:

> Hãy gửi cho tôi tất cả thay đổi của Pod trong namespace `test` xảy ra sau mốc `10245`.

API Server không trả xong rồi đóng kết nối như HTTP request bình thường. Nó giữ response mở và gửi event dần dần.

```text
Client                         API Server
   |                               |
   |--- WATCH từ RV 10245 -------->|
   |                               |
   |<---- ADDED pod-d, RV 10596 ---|
   |                               |
   |<-- MODIFIED pod-a, RV 11020 --|
   |                               |
   |<---- DELETED pod-b ----------|
   |                               |
   |       connection vẫn mở       |
```

Điều quan trọng:

> Kết nối TCP do client mở tới API Server, nhưng sau đó API Server liên tục gửi event xuống trên chính response HTTP đó.

---

# 5. Tại sao không bị mất event giữa LIST và WATCH?

Giả sử:

```text
LIST hoàn thành ở resourceVersion 10245.
Trong lúc client chuẩn bị mở WATCH, Pod X được tạo ở RV 10300.
Sau đó client gửi WATCH từ RV 10245.
```

API Server sẽ gửi lại event ở RV `10300`, vì client yêu cầu:

```text
Các thay đổi xảy ra sau 10245.
```

Luồng:

```text
T1: LIST trả về RV 10245
T2: Pod X được tạo, RV 10300
T3: Client mở WATCH từ RV 10245
T4: API Server gửi ADDED Pod X
```

Do đó client không bị khoảng trống:

```text
LIST ---------------- WATCH
        Pod X xuất hiện
```

Đây chính là lợi ích cốt lõi của list-watch:

> Lấy trạng thái hiện tại và đăng ký thay đổi tiếp theo mà không bỏ sót event ở khoảng giữa.

---

# 6. Các loại watch event

Watch response thường chứa các JSON document liên tiếp:

```json
{
  "type": "ADDED",
  "object": {
    "kind": "Pod",
    "metadata": {
      "name": "pod-a",
      "resourceVersion": "10596"
    }
  }
}
```

Các loại phổ biến:

## `ADDED`

```json
{
  "type": "ADDED",
  "object": {...}
}
```

Có object mới xuất hiện trong phạm vi watch.

Ví dụ:

- Pod vừa được tạo
    
- Pod vốn tồn tại nhưng vừa bắt đầu khớp label selector
    
- streaming list gửi trạng thái ban đầu dưới dạng synthetic `ADDED`
    

Client thường xử lý:

```text
thêm object vào local cache
```

## `MODIFIED`

```json
{
  "type": "MODIFIED",
  "object": {...}
}
```

Object đã thay đổi.

Ví dụ Pod:

- thay đổi status
    
- được gán `spec.nodeName`
    
- thêm label
    
- container chuyển trạng thái
    
- Pod IP được cập nhật
    

Client thường:

```text
thay object cũ trong cache bằng object mới
```

## `DELETED`

```json
{
  "type": "DELETED",
  "object": {...}
}
```

Object đã bị xóa hoặc không còn nằm trong phạm vi selector.

Client:

```text
xóa object khỏi local cache
```

## `BOOKMARK`

```json
{
  "type": "BOOKMARK",
  "object": {
    "metadata": {
      "resourceVersion": "12746"
    }
  }
}
```

Đây không phải object mới hay object được sửa.

Nó chỉ thông báo:

> Tôi đã gửi cho bạn đầy đủ tất cả event cho đến ít nhất resourceVersion `12746`.

## `ERROR`

Watch stream cũng có thể gửi event lỗi, chẳng hạn version quá cũ.

---

# 7. HTTP streaming hoạt động thế nào?

Ví dụ response:

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Content-Type: application/json
```

Sau đó body không kết thúc ngay:

```json
{"type":"ADDED","object":{...}}
{"type":"MODIFIED","object":{...}}
{"type":"DELETED","object":{...}}
```

Với HTTP/1.1, `Transfer-Encoding: chunked` cho phép server gửi body thành nhiều phần theo thời gian mà chưa cần biết trước tổng kích thước.

Nó không nhất thiết là WebSocket hay SSE.

Nó vẫn là:

```text
HTTP request
+
HTTP response body được stream kéo dài
```

Có thể hình dung:

```text
TCP connection
└── HTTP response
    ├── JSON event 1
    ├── JSON event 2
    ├── JSON event 3
    └── tiếp tục chờ event mới
```

---

# 8. Client phải lưu `resourceVersion` mới nhất

Ban đầu:

```text
LIST RV = 10245
```

Sau đó nhận:

```text
ADDED pod-a RV = 10596
MODIFIED pod-b RV = 11020
BOOKMARK RV = 12746
```

Client nên cập nhật mốc gần nhất:

```text
lastResourceVersion = 12746
```

Nếu connection bị đứt, client có thể mở lại:

```http
GET /api/v1/namespaces/test/pods?watch=1&resourceVersion=12746
```

API Server tiếp tục gửi các thay đổi sau mốc đó.

```text
Watch cũ:
10596 → 11020 → 12746 → mất mạng

Watch mới:
bắt đầu từ 12746 → tiếp tục
```

---

# 9. Vì sao lịch sử watch không được giữ mãi?

API Server không lưu vô hạn mọi thay đổi từng xảy ra.

Nếu giữ toàn bộ lịch sử:

```text
Pod tạo
Pod sửa
Pod sửa
Pod sửa
Pod xóa
...
```

thì dung lượng và chi phí quản lý sẽ tăng mãi.

Do đó hệ thống chỉ giữ một cửa sổ lịch sử có giới hạn.

Đoạn tài liệu nói rằng cluster dùng etcd 3 mặc định có thể chỉ giữ lịch sử thay đổi gần đây trong một khoảng thời gian nhất định. Giá trị chính xác có thể phụ thuộc cấu hình compact của cluster.

Ví dụ client đang giữ:

```text
resourceVersion = 10245
```

Nhưng API Server giờ chỉ còn lịch sử từ:

```text
resourceVersion = 50000
```

Client gọi:

```http
WATCH từ 10245
```

API Server không còn khả năng tái tạo mọi event từ mốc đó.

Nó trả:

```http
410 Gone
```

---

# 10. `410 Gone` nghĩa là gì?

`410 Gone` trong trường hợp này nghĩa là:

> `resourceVersion` bạn yêu cầu đã quá cũ; lịch sử tương ứng không còn nữa.

Client không được đoán trạng thái hiện tại bằng cách tiếp tục dùng cache cũ.

Nó phải:

```text
1. Bỏ cache cũ.
2. LIST lại toàn bộ trạng thái hiện tại.
3. Lấy resourceVersion mới.
4. WATCH lại từ resourceVersion mới.
```

Ví dụ:

```text
WATCH từ RV 10245
        |
        v
410 Gone
        |
        v
xóa local cache
        |
        v
LIST lại → RV 60000
        |
        v
WATCH từ RV 60000
```

Pseudo-code:

```go
for {
    list, err := listObjects()
    if err != nil {
        retry()
        continue
    }

    cache.replaceAll(list.Items)
    rv := list.ResourceVersion

    err = watchFrom(rv)

    if err == ResourceVersionTooOld {
        cache.clear()
        continue
    }

    if err != nil {
        reconnectFromLastKnownRV()
    }
}
```

---

# 11. BOOKMARK dùng để làm gì?

Giả sử watch đang chạy nhưng rất lâu không có Pod nào thay đổi.

Client vẫn giữ mốc cũ:

```text
lastResourceVersion = 10245
```

Trong khi hệ thống có thể đã tiến đến:

```text
current resourceVersion = 50000
```

Không phải mọi thay đổi từ `10245` đến `50000` đều liên quan đến Pod mà client đang watch. Có thể thay đổi nằm ở:

- ConfigMap
    
- Secret
    
- Lease
    
- Node
    
- object namespace khác
    

Nếu connection bị đứt và client reconnect từ `10245`, mốc này có thể đã quá cũ.

BOOKMARK giúp API Server nói:

```json
{
  "type": "BOOKMARK",
  "object": {
    "metadata": {
      "resourceVersion": "50000"
    }
  }
}
```

Ý nghĩa:

> Dù không có event Pod mới cho bạn, bạn đã đồng bộ đầy đủ đến mốc 50000.

Client có thể cập nhật:

```text
lastResourceVersion = 50000
```

Nếu kết nối đứt:

```http
WATCH từ 50000
```

thay vì `10245`.

## BOOKMARK không được đảm bảo gửi định kỳ

Client gửi:

```http
allowWatchBookmarks=true
```

chỉ có nghĩa:

> Tôi chấp nhận nhận bookmark.

Không có nghĩa:

```text
API Server chắc chắn gửi mỗi 10 giây.
```

Tài liệu nhấn mạnh client không được giả định:

- bookmark sẽ được gửi
    
- bookmark sẽ được gửi theo chu kỳ
    
- luôn có bookmark trước khi connection đóng
    

---

# 12. Reflector là gì?

Kubernetes Go client có thành phần tên `Reflector`.

Nó thực hiện logic:

```text
LIST
  ↓
đổ dữ liệu vào local store
  ↓
WATCH
  ↓
cập nhật store theo event
  ↓
mất kết nối thì reconnect
  ↓
410 Gone thì LIST lại
```

Kiến trúc thường là:

```text
API Server
    |
    | list/watch
    v
Reflector
    |
    v
DeltaFIFO
    |
    v
Informer local cache
    |
    v
Event handlers / Controller
```

Controller không cần tự viết lại toàn bộ logic HTTP list-watch.

Ví dụ một controller muốn theo dõi Pod:

```text
SharedInformer
    ├── List Pods
    ├── Watch Pods
    ├── giữ cache local
    └── gọi callback:
        ├── AddFunc
        ├── UpdateFunc
        └── DeleteFunc
```

---

# 13. Streaming lists là gì?

Mô hình truyền thống cần hai request:

```text
Request 1: LIST trạng thái hiện tại
Request 2: WATCH thay đổi tiếp theo
```

Streaming lists cho phép gộp thành một watch request:

```http
GET /api/v1/namespaces/test/pods
    ?watch=1
    &sendInitialEvents=true
    &allowWatchBookmarks=true
    &resourceVersion=
    &resourceVersionMatch=NotOlderThan
```

API Server trả:

```text
ADDED foo
ADDED bar
BOOKMARK 10245
MODIFIED ...
ADDED ...
DELETED ...
```

## Các `ADDED` ban đầu là synthetic event

Giả sử hiện tại đã có:

```text
foo
bar
```

Hai Pod này không thực sự vừa được tạo.

Nhưng API Server gửi:

```json
{
  "type": "ADDED",
  "object": {
    "metadata": {
      "name": "foo",
      "resourceVersion": "8467"
    }
  }
}
```

và:

```json
{
  "type": "ADDED",
  "object": {
    "metadata": {
      "name": "bar",
      "resourceVersion": "5726"
    }
  }
}
```

để client xây dựng cache ban đầu.

Sau đó:

```json
{
  "type": "BOOKMARK",
  "object": {
    "metadata": {
      "resourceVersion": "10245"
    }
  }
}
```

BOOKMARK ở đây đánh dấu:

> Toàn bộ trạng thái ban đầu đã được gửi đầy đủ và đồng bộ ít nhất đến mốc `10245`.

Từ đây trở đi là event watch bình thường.

```text
Synthetic initial events:
ADDED foo
ADDED bar

Boundary:
BOOKMARK RV 10245

Regular events:
MODIFIED foo
ADDED baz
DELETED bar
```

---

# 14. Streaming list có lợi gì?

Với list thông thường, API Server có thể phải:

```text
1. Thu thập collection rất lớn.
2. Giữ hoặc serialize một response JSON lớn.
3. Gửi toàn bộ list.
4. Sau đó mở watch riêng.
```

Trong cluster lớn, list hàng trăm nghìn object gây áp lực RAM.

Streaming list có thể gửi object dần:

```text
ADDED object 1
ADDED object 2
ADDED object 3
...
BOOKMARK
```

Client cũng có thể xử lý dần thay vì đợi toàn bộ danh sách hoàn tất.

---

# 15. `resourceVersionMatch=NotOlderThan`

Khi dùng:

```text
sendInitialEvents=true
```

tài liệu yêu cầu:

```text
resourceVersionMatch=NotOlderThan
```

Ý nghĩa khái quát:

> Trạng thái trả về phải ít nhất mới bằng mốc resourceVersion mà client yêu cầu, không được cũ hơn.

Ví dụ:

```http
resourceVersion=50000
resourceVersionMatch=NotOlderThan
```

API Server phải đồng bộ đến ít nhất mốc:

```text
>= 50000
```

Nó không được trả trạng thái từ:

```text
49000
```

---

# 16. `resourceVersion=` rỗng có nghĩa gì?

Ví dụ:

```http
resourceVersion=
```

Tức query parameter tồn tại nhưng giá trị rỗng.

Trong ngữ cảnh streaming list ở tài liệu, nó được hiểu là yêu cầu một **consistent read**.

Nói đơn giản:

> Hãy cho tôi trạng thái khởi đầu nhất quán tương ứng với thời điểm request đang được xử lý, sau đó tiếp tục watch.

BOOKMARK được gửi khi quá trình initial state đã đồng bộ đến mốc nhất quán đó.

---

# 17. Tại sao từng Pod trong initial event có version thấp hơn BOOKMARK?

Ví dụ:

```text
foo resourceVersion = 8467
bar resourceVersion = 5726
BOOKMARK             = 10245
```

Điều này hoàn toàn bình thường.

Vì:

- `foo` lần cuối được sửa tại `8467`
    
- `bar` lần cuối được sửa tại `5726`
    
- toàn bộ ảnh chụp collection được xác nhận nhất quán ở `10245`
    

Một Pod không cần có version bằng version hiện tại của toàn collection.

Ví dụ:

```text
RV 5726: bar được sửa lần cuối
RV 8467: foo được sửa lần cuối
RV 9000: Secret nào đó được sửa
RV 9500: Node được sửa
RV 10245: collection snapshot được xác nhận
```

---

# 18. Sharded list and watch giải quyết vấn đề gì?

Trong cluster lớn, có thể có:

```text
500.000 Pods
```

Giả sử controller chạy 4 replica để scale ngang.

Nếu cả 4 replica đều watch toàn bộ Pod:

```text
Replica 0 nhận 500.000 Pod
Replica 1 nhận 500.000 Pod
Replica 2 nhận 500.000 Pod
Replica 3 nhận 500.000 Pod
```

Tổng cộng:

```text
4 × toàn bộ event stream
```

Mỗi replica đều tốn:

- băng thông
    
- CPU deserialize JSON
    
- RAM cache object
    
- chi phí xử lý event
    

Trong khi có thể mỗi replica chỉ cần chịu trách nhiệm cho 1/4 số object.

Sharded list/watch cho phép:

```text
Replica 0 nhận shard 0
Replica 1 nhận shard 1
Replica 2 nhận shard 2
Replica 3 nhận shard 3
```

---

# 19. Sharding ở đây không chia theo thứ tự tên object

Nó dùng hash trên metadata field, hiện tài liệu đề cập:

```text
object.metadata.uid
object.metadata.namespace
```

Ví dụ:

```text
hash(Pod UID) → số 64-bit
```

Sau đó kiểm tra số hash nằm trong khoảng nào.

Toàn bộ không gian hash:

```text
0x0000000000000000
đến
0x10000000000000000
```

Đây tương ứng với khoảng:

```text
0 đến 2^64
```

---

# 20. Ví dụ chia hai replica

## Replica 0

```text
[0, 2^63)
```

Selector:

```text
shardRange(
  object.metadata.uid,
  '0x0000000000000000',
  '0x8000000000000000'
)
```

## Replica 1

```text
[2^63, 2^64)
```

Selector:

```text
shardRange(
  object.metadata.uid,
  '0x8000000000000000',
  '0x10000000000000000'
)
```

Dấu khoảng là:

```text
[start, end)
```

nghĩa là:

- bao gồm `start`
    
- không bao gồm `end`
    

Nhờ đó hai khoảng không bị trùng.

```text
Replica 0:
0 -------------------- 2^63

Replica 1:
                      2^63 -------------------- 2^64
```

Mỗi object có UID được hash và chỉ thuộc một khoảng.

---

# 21. Tại sao dùng hash?

Nếu chia theo tên:

```text
A–M → replica 0
N–Z → replica 1
```

phân bố có thể lệch.

Ví dụ phần lớn object bắt đầu bằng:

```text
pod-
```

Hash giúp phân bố gần đều hơn:

```text
UID A → hash thuộc shard 0
UID B → hash thuộc shard 1
UID C → hash thuộc shard 0
UID D → hash thuộc shard 1
```

FNV-1a được dùng để tạo hash 64-bit xác định.

“Deterministic” nghĩa là:

```text
cùng một UID
→ luôn tạo cùng một hash
→ trên mọi API Server replica
→ luôn thuộc cùng một shard
```

Điều này quan trọng khi control plane có nhiều API Server.

---

# 22. `WithTweakListOptions` làm gì?

Controller thường tạo informer:

```go
factory := informers.NewSharedInformerFactoryWithOptions(
    client,
    resyncPeriod,
    informers.WithTweakListOptions(func(opts *metav1.ListOptions) {
        opts.ShardSelector = shardSelector
    }),
)
```

`WithTweakListOptions` chỉnh `ListOptions` trước mỗi lần:

- LIST
    
- WATCH
    
- relist
    
- reconnect
    

Điều đó bảo đảm cùng shard selector được áp dụng nhất quán.

Nếu chỉ áp dụng cho LIST nhưng không áp dụng WATCH:

```text
LIST: nhận 1/2 object
WATCH: nhận toàn bộ object
```

thì cache sẽ sai.

Nếu chỉ áp dụng WATCH nhưng không áp dụng LIST:

```text
LIST: nhận toàn bộ object
WATCH: chỉ nhận 1/2 thay đổi
```

cũng sai.

Do đó selector phải áp dụng cho cả hai.

---

# 23. `shardInfo` dùng để làm gì?

Khi API Server thực sự áp dụng shard selector, response trả:

```json
{
  "metadata": {
    "resourceVersion": "10245",
    "shardInfo": {
      "selector": "shardRange(...)"
    }
  }
}
```

Nó xác nhận:

> Kết quả này chỉ là một shard, không phải toàn bộ collection.

Nếu không có `shardInfo`, có thể:

- feature gate chưa bật
    
- API Server phiên bản cũ
    
- server không hỗ trợ
    
- selector bị bỏ qua
    

Trong trường hợp đó API Server có thể trả toàn bộ dữ liệu.

Client phải kiểm tra:

```text
Có shardInfo?
├── Có: server đã lọc shard
└── Không: có thể đang nhận toàn bộ collection
```

---

# 24. Tại sao server cũ lại bỏ qua selector thay vì báo lỗi?

Theo đoạn tài liệu, khi không hỗ trợ, trường `ShardSelector` có thể bị ignore và server trả toàn bộ dữ liệu.

Điều này giúp client mới có thể giao tiếp với server cũ, nhưng tạo một yêu cầu quan trọng:

> Client không được mặc định rằng selector chắc chắn đã được áp dụng.

Nó phải nhìn `shardInfo`.

Nếu không có, client có thể tự lọc phía client:

```text
Nhận toàn bộ event
        |
        v
tính hash object
        |
        v
chỉ giữ object thuộc shard của mình
```

Cách này không giảm băng thông từ API Server, nhưng vẫn bảo đảm tính đúng đắn.

---

# 25. Non-contiguous ranges là gì?

Một replica có thể xử lý nhiều khoảng không liền nhau:

```text
[0, 1/4)
OR
[1/2, 3/4)
```

Selector:

```text
shardRange(
  object.metadata.uid,
  '0x0000000000000000',
  '0x4000000000000000'
)
||
shardRange(
  object.metadata.uid,
  '0x8000000000000000',
  '0xc000000000000000'
)
```

Có thể dùng trong trường hợp:

- tái cân bằng shard
    
- một replica tạm thời gánh shard của replica khác
    
- assignment shard động bằng Lease
    
- số replica thay đổi
    

---

# 26. Tổng hợp ba cơ chế

## List + Watch truyền thống

```text
LIST
  ↓
nhận toàn bộ state + RV
  ↓
WATCH từ RV
  ↓
nhận thay đổi
```

## Streaming list

```text
WATCH sendInitialEvents=true
  ↓
ADDED cho toàn bộ state hiện tại
  ↓
BOOKMARK đánh dấu initial sync hoàn tất
  ↓
nhận thay đổi bình thường
```

## Sharded list/watch

```text
Mỗi replica chỉ:
LIST shard của mình
+
WATCH shard của mình
```

Các cơ chế này có thể kết hợp:

```text
Sharded streaming watch
```

Tức mỗi replica mở một streaming list/watch nhưng chỉ cho shard được phân công.

---

# 27. Liên hệ với kubelet

Ví dụ kubelet trên node `wk1` cần theo dõi Pod được schedule vào node đó.

Về logic, kubelet cần collection kiểu:

```text
Pods có:
spec.nodeName = wk1
```

Nó có thể:

```text
1. LIST các Pod thuộc wk1.
2. Nhận resourceVersion.
3. WATCH từ version đó.
4. Khi Pod mới được scheduler gán vào wk1:
   nhận ADDED.
5. Khi Pod spec/status thay đổi:
   nhận MODIFIED.
6. Khi Pod bị xóa:
   nhận DELETED.
```

Luồng khái quát:

```text
Scheduler:
PATCH Pod.spec.nodeName = wk1
        |
        v
API Server lưu thay đổi
        |
        v
watch stream gửi MODIFIED/ADDED
        |
        v
kubelet wk1 nhận event
        |
        v
kubelet sync Pod qua container runtime
```

Kubelet không cần mỗi giây hỏi:

```text
Có Pod mới cho tôi chưa?
Có Pod mới cho tôi chưa?
Có Pod mới cho tôi chưa?
```

Watch giúp API Server gửi thay đổi xuống ngay khi có.

---

# 28. Liên hệ với controller-manager

Ví dụ Deployment controller theo dõi:

- Deployment
    
- ReplicaSet
    
- Pod
    

Nó giữ local cache qua informer.

Khi Deployment thay đổi từ:

```yaml
replicas: 3
```

thành:

```yaml
replicas: 5
```

API Server phát watch event:

```text
MODIFIED Deployment
```

Informer cập nhật cache và gọi event handler.

Controller so sánh:

```text
Desired replicas = 5
Actual replicas  = 3
```

Sau đó tạo thêm ReplicaSet/Pod cần thiết.

```text
API watch event
      ↓
local cache
      ↓
workqueue
      ↓
reconcile
      ↓
ghi thay đổi trở lại API Server
```

---

# 29. Watch event không phải “lệnh”

Một event:

```json
{
  "type": "ADDED",
  "object": {
    "kind": "Pod",
    ...
  }
}
```

không có nghĩa API Server ra lệnh:

```text
Kubelet, hãy chạy container này.
```

Chính xác hơn:

> API Server thông báo rằng trạng thái object đã thay đổi.

Kubelet/controller tự nhìn object rồi reconcile.

Ví dụ kubelet thấy:

```yaml
spec:
  nodeName: wk1
  containers:
    - image: nginx
```

Sau đó kubelet quyết định cần gọi container runtime để biến trạng thái thật thành trạng thái mong muốn.

---

# 30. Sơ đồ tổng thể dễ nhớ

```text
             LIST
Client --------------------> API Server
       <--------------------
       Objects + RV 10245

             WATCH từ 10245
Client --------------------> API Server
       <-------------------- ADDED RV 10596
       <-------------------- MODIFIED RV 11020
       <-------------------- BOOKMARK RV 12746
       <-------------------- DELETED RV 13000

Kết nối bị đứt
Client reconnect từ RV 13000

Nếu RV 13000 còn lịch sử:
    tiếp tục watch

Nếu RV 13000 quá cũ:
    410 Gone
    → clear cache
    → LIST lại
    → WATCH từ RV mới
```

Điểm cốt lõi của toàn bộ đoạn là:

> Kubernetes client thường không liên tục tải lại toàn bộ object. Nó lấy một ảnh chụp ban đầu, ghi nhớ `resourceVersion`, sau đó giữ một HTTP stream để nhận những thay đổi xảy ra sau mốc đó. Streaming list tối ưu bước lấy trạng thái ban đầu, còn sharded list/watch chia event stream giữa nhiều controller replica để giảm tải.