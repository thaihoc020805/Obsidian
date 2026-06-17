## 1. Dual-stack là gì?

- Bình thường cluster Kubernetes dùng **IPv4** _hoặc_ **IPv6** (single-stack).
    
- **Dual-stack** = **mỗi Pod và Service có cả IPv4 và IPv6** cùng lúc.
    
- Mục đích:
    
    - Tương thích với cả môi trường IPv4 và IPv6.
        
    - Giúp kết nối Internet và nội bộ qua cả 2 loại địa chỉ.
        

Ví dụ:

- Pod có thể có:
    
    `IPv4: 10.244.1.5 IPv6: fd12:3456:789a:1::5`
    
- Service có thể có:

    `IPv4: 10.96.0.12 IPv6: fd12:3456:789a::c`
    

---

## 2. Điều kiện để dùng dual-stack

- Kubernetes **>= 1.20** (tốt nhất 1.23 trở lên).
    
- Hạ tầng mạng (cloud/bare metal) hỗ trợ IPv6.
    
- CNI plugin (Calico, Cilium, Flannel,...) phải support dual-stack.
    
- Node phải có cả interface IPv4 và IPv6.
    

---

## 3. Cấu hình dual-stack

Cần set CIDR cho cả IPv4 và IPv6 ở các component:

bash

CopyEdit

`# kube-apiserver --service-cluster-ip-range=10.96.0.0/16,fd12:3456:789a::/108  # kube-controller-manager --cluster-cidr=10.244.0.0/16,fd12:3456:789a:1::/64 --service-cluster-ip-range=10.96.0.0/16,fd12:3456:789a::/108  # kube-proxy --cluster-cidr=10.244.0.0/16,fd12:3456:789a:1::/64  # kubelet (nếu bare metal) --node-ip=192.168.1.10,fd12:3456:789a:1::10`

---

## 4. Cách Service hoạt động với dual-stack

Có 3 chế độ `.spec.ipFamilyPolicy`:

1. **SingleStack** → chỉ 1 loại IP (IPv4 hoặc IPv6).
    
2. **PreferDualStack** → lấy cả IPv4 và IPv6 nếu cluster hỗ trợ, nếu không thì fallback về SingleStack.
    
3. **RequireDualStack** → bắt buộc phải có cả IPv4 và IPv6, nếu không thì tạo Service thất bại.
    

### Chỉ định thứ tự IP

- `.spec.ipFamilies` để chọn thứ tự:
    
    yaml
    
    CopyEdit
    
    `ipFamilies: - IPv6 - IPv4`
    
    → `clusterIP` sẽ là IPv6 vì IPv6 đứng trước.
    

---

## 5. Ví dụ

**PreferDualStack + IPv6 ưu tiên:**

yaml

CopyEdit

`apiVersion: v1 kind: Service metadata:   name: my-service spec:   ipFamilyPolicy: PreferDualStack   ipFamilies:   - IPv6   - IPv4   selector:     app: my-app   ports:     - protocol: TCP       port: 80`

Khi tạo trên cluster dual-stack:

yaml

CopyEdit

`clusterIPs:   - fd12:3456:789a::15   # IPv6   - 10.96.0.15           # IPv4 clusterIP: fd12:3456:789a::15`

---

## 6. Khi upgrade cluster từ single-stack lên dual-stack

- Service cũ sẽ tự set:
    
    yaml
    
    CopyEdit
    
    `ipFamilyPolicy: SingleStack ipFamilies: [IPv4]`
    
- Bạn có thể đổi thành `PreferDualStack` để thêm IP còn lại.
    

---

## 7. LoadBalancer & Headless Service

- LoadBalancer cũng có thể cấp cả IPv4 và IPv6 nếu cloud provider hỗ trợ.
    
- Headless Service không selector → mặc định `RequireDualStack`.
    

---

## 8. Egress traffic

- Nếu Pod có IPv6 private muốn ra Internet IPv6, cần NAT (IP masquerading) hoặc proxy.
    
- Phải đảm bảo CNI hỗ trợ IPv6.
    

---

## 9. Windows support

- Windows **không hỗ trợ IPv6-only**.
    
- Có thể dùng dual-stack nhưng chỉ với L2bridge, không hỗ trợ VXLAN overlay.
    

---

💡 **Tóm tắt siêu ngắn gọn**:

- Dual-stack = mỗi Pod/Service có cả IPv4 + IPv6.
    
- `.ipFamilyPolicy` điều khiển kiểu: Single, Prefer, Require dual-stack.
    
- `.ipFamilies` chọn thứ tự ưu tiên.
    
- Cần CNI và hạ tầng hỗ trợ IPv6.
    
- Lợi ích: linh hoạt, tương thích cả IPv4 & IPv6.