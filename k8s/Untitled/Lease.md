Mấy object `Lease` “đang chạy” đó **không phải Pod/process đang chạy** đâu. Nó chỉ là **record trong etcd** để Kubernetes ghi trạng thái “ai còn sống / ai đang giữ quyền leader”.

Thường bạn thấy kiểu:

```
kubectl get lease -A
```

sẽ có 2 nhóm chính:

## 1. Lease trong `kube-node-lease`

Ví dụ:

```
kube-node-lease   fdp-k8s-wk1kube-node-lease   fdp-k8s-wk2kube-node-lease   fdp-k8s-cp1
```

Mỗi Node có 1 Lease.

Ý nghĩa:

```
kubelet trên node đó→ cứ vài giây update Lease→ báo với control plane: "tao còn sống"
```

Nên nếu có 4 node thì thường có 4 Lease trong namespace `kube-node-lease`.

Bạn xem chi tiết:

```
kubectl get lease -n kube-node-lease
```

hoặc:

```
kubectl describe lease -n kube-node-lease fdp-k8s-wk1
```

Bạn sẽ thấy mấy field kiểu:

```
HolderIdentity: fdp-k8s-wk1RenewTime: ...LeaseDurationSeconds: 40
```

`renewTime` cứ đổi liên tục nghĩa là node còn sống.

---

## 2. Lease trong `kube-system`

Ví dụ:

```
kube-system   kube-controller-managerkube-system   kube-scheduler
```

Cái này dùng cho **leader election**.

Ví dụ cluster HA có nhiều control plane:

```
cp1 có kube-schedulercp2 có kube-schedulercp3 có kube-scheduler
```

Nhưng chỉ được **1 thằng scheduler chính thức làm việc**.

Nó dùng Lease để ghi:

```
HolderIdentity: cp2_xxx
```

Tức là:

```
Hiện tại cp2 đang giữ quyền leader scheduler
```

Nếu cp2 chết, lease không được renew nữa, cp1/cp3 tranh nhau lấy lease, rồi một thằng lên leader.

---

## Tại sao gọi là “đang chạy”?

Vì `kubectl get lease` hiển thị object còn tồn tại, giống như `kubectl get pod`.

Nhưng thực ra Lease chỉ là dữ liệu kiểu:

```
kind: Leasemetadata:  name: fdp-k8s-wk1spec:  holderIdentity: fdp-k8s-wk1  renewTime: "2026-07-03T..."
```

Nó không tự chạy gì cả.

Process thật sự đang chạy là:

```
kubeletkube-schedulerkube-controller-manager
```

Còn Lease chỉ là “tờ giấy check-in” mà mấy process này liên tục ký lại.

---

Nói cực ngắn:

```
Lease object = tờ giấy điểm danhkubelet/scheduler/controller-manager = người điểm danhrenewTime = lần điểm danh mới nhất
```

Nếu `renewTime` ngừng cập nhật quá lâu, Kubernetes hiểu là thằng đó có vấn đề.


Luồng thực tế là:

```
kubelet
    │
    │ PATCH Lease
    ▼
kube-apiserver
    │
    ▼
etcd
    │
    ▼
Object Lease được cập nhật
```

Cụ thể:

1. Kubelet trên `worker-1` cứ khoảng 10 giây sẽ gửi request lên API Server:

```
PATCH /apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases/worker-1
```

Nội dung đại loại:

```
spec:  holderIdentity: worker-1  renewTime: 2026-07-03T10:30:15Z
```

2. API Server xác thực request.
3. API Server ghi object Lease mới vào etcd.
4. Node Controller (trong `kube-controller-manager`) dùng **watch** để theo dõi các Lease này.

---

## Nếu kubelet chết thì sao?

Giả sử:

```
10:00:00 renew
10:00:10 renew
10:00:20 renew
```

Đến 10:00:20 thì máy bị tắt.

Không còn ai update nữa.

Object Lease vẫn nằm trong etcd:

```
spec:
  renewTime: 10:00:20
```

Nó **không tự biến mất**.

Sau khoảng thời gian quy định (mặc định khoảng 40–50 giây), Node Controller thấy:

> "Ủa, Lease này lâu rồi không renew."

Nó sẽ đánh dấu Node:

```
Ready=False
```

hoặc

```
Ready=Unknown
```

rồi thực hiện các bước như thêm taint `node.kubernetes.io/unreachable` và cuối cùng evict Pod nếu cần.

---

## Lease có bị xóa không?

Thông thường:

**Không.**

Ví dụ Node `worker-2` chết.

Bạn vẫn có thể thấy:

```
kubectl get lease -n kube-node-lease
```

```
NAMEworker-1worker-2worker-3
```

`worker-2` vẫn còn.

Chỉ có điều:

```
kubectl describe lease worker-2
```

sẽ thấy

```
RenewTime:2026-07-03T10:00:20Z
```

không thay đổi nữa.

---

## Khi nào Lease bị xóa?

Chỉ khi:

- bạn xóa Node

```
kubectl delete node worker-2
```

thì Lease tương ứng cũng sẽ bị dọn dẹp,

hoặc

- một controller nào đó chủ động xóa nó.

Lease **không có TTL tự động**.

---

## Một điểm rất quan trọng

Bạn nói:

> "nếu hết hạn và pod chết"

Thực ra phải là:

```
Lease hết hạn
        │
        ▼
Node bị coi là chết
        │
        ▼
Các Pod trên Node đó bị evict (xóa khỏi API)
        │
        ▼
Scheduler tạo Pod mới ở Node khác
```

Pod **không có Lease riêng**.

Lease là của **Node** (heartbeat) hoặc của **leader election** cho các component như scheduler, controller-manager, v.v.

---

### Toàn bộ luồng

```
          kubelet
             │
     PATCH Lease mỗi 10s
             │
             ▼
        kube-apiserver
             │
             ▼
            etcd
             │
             ▼
        Lease object

             │
             ▼
     Node Controller (watch)

    renewTime mới?
      │
      ├── Có
      │     Node Ready
      │
      └── Không
            │
            ▼
      Node NotReady/Unknown
            │
            ▼
      Thêm taint
            │
            ▼
      Evict Pod
```

Nên có thể coi **Lease giống như một "heartbeat record" trong etcd**, còn **Node Controller là thành phần liên tục đọc record đó để quyết định Node còn sống hay không**.