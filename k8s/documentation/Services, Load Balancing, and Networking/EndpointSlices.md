
## 1. EndpointSlices là gì?

- Là **cơ chế mới** trong Kubernetes (thay thế API `Endpoints` cũ) để lưu trữ và quản lý danh sách **network endpoints** của một Service.
- Mỗi EndpointSlice chứa danh sách các địa chỉ **IP:Port** của Pod backend mà Service trỏ tới.
- Mục tiêu:
    1.  **Scale tốt hơn** — không bị giới hạn như resource `Endpoints` cũ (một object duy nhất có thể quá to khi Service có nhiều Pod).
    2.  **Cập nhật hiệu quả hơn** — chia endpoints thành nhiều "slice" nhỏ để khi thay đổi chỉ update slice liên quan.

---

## 2. Cách hoạt động cơ bản

- Khi bạn tạo một Service có selector (ví dụ: `app=web`), control plane sẽ:
    - Tìm tất cả Pod match selector.
    - Gom nhóm chúng vào **EndpointSlices** (mỗi slice tối đa **100 endpoints** mặc định).
    - Gán label `kubernetes.io/service-name: <service-name>` để biết slice này thuộc Service nào.
- **kube-proxy** trên mỗi Node sẽ **watch** các EndpointSlices → biết phải route traffic nội bộ như thế nào.

---

## 3. Ví dụ một EndpointSlice

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: example-abc
  labels:
    kubernetes.io/service-name: example
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 80
endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
    hostname: pod-1
    nodeName: node-1
    zone: us-west2-a
````

**Ý nghĩa**:

- **addressType: IPv4** → slice này chỉ chứa IPv4 (nếu service dual-stack sẽ có slice IPv6 riêng).
    
- **ports** → danh sách port áp dụng cho toàn bộ endpoints trong slice.
    
- **endpoints** → mỗi item là một Pod backend:
    
    - `addresses`: IP Pod
        
    - `ready: true` → kube-proxy có thể route traffic đến đây.
        
    - `nodeName` và `zone` → thông tin topology.
        

---

## 4. Các condition quan trọng

- **ready**: = `serving && !terminating` (Pod sẵn sàng nhận request).
    
- **serving**: endpoint đang xử lý request (tương ứng Pod Ready condition).
    
- **terminating**: endpoint đang bị xóa (Pod có deletionTimestamp).
    
    - kube-proxy thường bỏ qua endpoints `terminating`, nhưng nếu tất cả đều `terminating` thì nó vẫn route để tránh downtime.
        

---

## 5. Quản lý và phân phối EndpointSlices

- Mỗi slice có tối đa **100 endpoints** (config qua `--max-endpoints-per-slice` ở kube-controller-manager, tối đa 1000).
    
- Khi Pod thay đổi:
    
    1. Xóa endpoints cũ khỏi slice.
        
    2. Update endpoints đã thay đổi.
        
    3. Nếu còn endpoints mới → thêm vào slice chưa full hoặc tạo slice mới.
        
- Controller ưu tiên **giảm số lần update slice** hơn là tối ưu full slot → có thể sẽ tạo slice mới thay vì lấp đầy slice cũ.
    
- Trong rolling update, các slice sẽ tự "repack" vì Pod mới thay thế Pod cũ.
    

---

## 6. Lợi ích so với `Endpoints` cũ

|Tiêu chí|Endpoints cũ|EndpointSlices|
|---|---|---|
|Giới hạn 1 object|✅|❌ (chia nhiều object)|
|Update lớn gây load API|✅|❌ (chỉ update slice liên quan)|
|Hỗ trợ dual-stack|❌|✅|
|Có metadata topology|❌|✅|
|Scale lớn (nghìn Pods)|❌|✅|

Xuất sang Trang tính

---

## 7. Các lưu ý khi dùng

- Một Service có thể có nhiều EndpointSlices (IPv4, IPv6, nhiều port, hoặc nhiều slice do nhiều Pod).
    
- Khi đọc từ API, bạn **phải gộp** tất cả EndpointSlices của một Service và **loại bỏ trùng lặp** endpoints.
    
- Nếu bạn viết controller hoặc app đọc endpoints → đừng giả định mỗi Service chỉ có 1 slice.
    

---

## 8. Liên hệ thực tế

Giả sử bạn có Service `web-svc` trỏ tới 500 Pod:

- `Endpoints` cũ → 1 object JSON cực to chứa 500 entry, mỗi thay đổi gửi tới **tất cả kube-proxy** → tốn băng thông và CPU.
    
- `EndpointSlices` mới → 5 object (mỗi cái 100 Pod). Khi 1 Pod chết:
    
    - Chỉ slice chứa Pod đó update → giảm dữ liệu truyền xuống kube-proxy.