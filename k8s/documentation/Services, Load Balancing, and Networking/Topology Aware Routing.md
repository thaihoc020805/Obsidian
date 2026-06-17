## 1. Topology Aware Routing là gì?

- Đây là tính năng giúp **ưu tiên route traffic trong cùng một zone** (vùng) thay vì gửi đi zone khác.
    
- Mục tiêu:
    
    - Giảm **latency** (độ trễ) vì dữ liệu không phải đi xa.
        
    - Tiết kiệm chi phí (nếu cloud tính phí cross-zone).
        
    - Tăng độ tin cậy (ít phụ thuộc mạng giữa các zone).
        

💡 **Zone**: Vùng vật lý trong cloud (ví dụ AWS `us-east-1a`, `us-east-1b`).

---

## 2. Ví dụ dễ hiểu

Giả sử bạn có:

- **Cluster 3 zone**: zone-a, zone-b, zone-c.
    
- **Service** có Pods ở cả 3 zone.
    

Nếu **Pod ở zone-a** gọi Service:

- Bình thường (default) → kube-proxy có thể gửi request sang **Pod ở zone-b** hoặc **zone-c**.
    
- Với **Topology Aware Routing** → kube-proxy sẽ **ưu tiên chọn Pod ở zone-a** (gần nhất).
    

---

## 3. Cách bật

Gắn annotation vào Service:

yaml

CopyEdit

`metadata:   annotations:     service.kubernetes.io/topology-mode: Auto`

- Khi bật, Kubernetes sẽ thêm **hints** (gợi ý) vào EndpointSlice để báo zone nào nên nhận traffic.
    

---

## 4. Hoạt động bên trong

1. **EndpointSlice Controller**:
    
    - Nhìn topology của node (`topology.kubernetes.io/zone`).
        
    - Chia endpoint cho từng zone theo tỷ lệ CPU khả dụng.
        
    - Ghi vào `hints.forZones`.
        
2. **kube-proxy**:
    
    - Đọc các hints này.
        
    - Nếu Pod đang ở zone-a → chỉ route tới endpoints có hint `zone-a`.
        

---

## 5. Khi nào dùng tốt nhất

- Traffic đến từ các zone **tương đối cân bằng**.
    
- Mỗi zone có **≥ 3 endpoints** (Pods).
    
- Cluster multi-zone.
    

Nếu một zone nhận 90% traffic, zone đó dễ **quá tải** → tính năng này **không nên bật**.

---

## 6. Ví dụ EndpointSlice có hint

yaml

CopyEdit

`endpoints:   - addresses: ["10.1.2.3"]     zone: zone-a     hints:       forZones:         - name: "zone-a"`

Nghĩa là endpoint này được ưu tiên phục vụ Pod trong **zone-a**.

---

## 7. Safeguards (các điều kiện an toàn)

Kube-proxy **sẽ bỏ qua** Topology Aware Hints và route toàn cluster nếu:

- Số endpoint < số zone.
    
- Không thể chia đều endpoints giữa các zone.
    
- Node thiếu thông tin zone.
    
- Endpoint không có hint.
    
- Zone hiện tại không có endpoint nào có hint.
    

---

## 8. Hạn chế

- Không dùng chung với `internalTrafficPolicy: Local` cho cùng Service.
    
- Không tính đến **tolerations** hay **autoscaling** chính xác → có thể mất cân bằng.
    
- Không hợp khi traffic tập trung mạnh vào một vài zone.
    

---

## 9. Tóm tắt siêu ngắn

- **Tính năng**: Route traffic trong cùng zone để giảm latency & chi phí.
    
- **Bật**: `service.kubernetes.io/topology-mode: Auto`.
    
- **Yêu cầu**: Multi-zone, nhiều endpoint, traffic phân bố đều.
    
- **Hoạt động**: EndpointSlice Controller gán hint → kube-proxy lọc endpoint theo zone.
    
- **Fallback**: Nếu thiếu dữ liệu hoặc mất cân bằng, sẽ route toàn cluster.



## 1. Zone có từ đâu?

### a) **Cluster chạy trên Cloud (AWS, GCP, Azure, v.v.)**

- Cloud Controller Manager sẽ **tự động** gán label zone cho node.
    
- Label này thường là:
    
    yaml
    
    CopyEdit
    
    `topology.kubernetes.io/zone=us-east-1a topology.kubernetes.io/region=us-east-1`
    
- Bạn không cần làm gì thêm, miễn là cluster được cấu hình dùng cloud provider.
    

---

### b) **Cluster chạy Bare-metal / On-prem**

- Kubernetes **không tự đoán** được zone → Bạn phải **tự gán label** cho node.
    
- Ví dụ:
    
    bash
    
    CopyEdit
    
    `kubectl label node my-node-1 topology.kubernetes.io/zone=zone-a kubectl label node my-node-1 topology.kubernetes.io/region=region-1`
    
- Có thể dùng bất kỳ tên zone/region nào, miễn consistent giữa các node.
    

---

## 2. Cách kiểm tra node có zone chưa

Chạy:

bash

CopyEdit

`kubectl get nodes --show-labels`

Tìm các label:

bash

CopyEdit

`topology.kubernetes.io/zone topology.kubernetes.io/region`

Ví dụ trên AWS:

bash

CopyEdit

`ip-192-168-1-10   topology.kubernetes.io/region=us-east-1,topology.kubernetes.io/zone=us-east-1a ip-192-168-1-11   topology.kubernetes.io/region=us-east-1,topology.kubernetes.io/zone=us-east-1b`

---

## 3. Nếu chưa có zone label

Tự gán bằng:

bash

CopyEdit

`kubectl label node <node-name> topology.kubernetes.io/zone=zone-a kubectl label node <node-name> topology.kubernetes.io/region=region-1`

> Lưu ý: Nếu bạn dùng multi-zone topology aware routing, mà không gán đúng các label này → kube-proxy sẽ bỏ qua tính năng.