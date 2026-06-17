
## 1. NetworkPolicy là gì?

NetworkPolicy là một resource trong Kubernetes cho phép bạn kiểm soát lưu lượng mạng ở mức IP và port (Layer 3 & 4 của OSI).

Nó dùng để cho phép hoặc chặn kết nối:
-   Từ đâu đến Pod (**Ingress** — inbound traffic).
-   Từ Pod ra đâu (**Egress** — outbound traffic).

Bạn có thể lọc theo:
-   Label của Pod
-   Label của Namespace
-   IP/CIDR
-   Protocol (TCP, UDP, SCTP)
-   Port hoặc range port

Nghĩa là nó giống như một firewall bên trong Kubernetes, nhưng thay vì áp dụng ở node, nó áp dụng ở mức Pod.

---

## 2. Điều kiện để dùng được NetworkPolicy

Bắt buộc cluster của bạn phải cài **CNI plugin** hỗ trợ NetworkPolicy (ví dụ: Calico, Cilium, Kube-router, Weave Net…).
Nếu CNI không hỗ trợ → tạo NetworkPolicy sẽ không có tác dụng.

---

## 3. Hai khái niệm “isolation” quan trọng

-   **Ingress isolation**: Chỉ định ai được phép gửi traffic vào Pod.
-   **Egress isolation**: Chỉ định Pod được phép gửi traffic đi đâu.

**Mặc định** (nếu không có NetworkPolicy):
-   Ingress: Mọi nguồn đều vào được.
-   Egress: Mọi đích đều ra được.

Khi bạn tạo NetworkPolicy mà có `policyTypes: ["Ingress"]` hoặc `["Egress"]`:
-   Pod nào được `podSelector` match sẽ bị giới hạn hướng đó, chỉ cho phép những gì policy khai báo.
-   Các policy không xung đột, chúng cộng dồn quyền truy cập (union).

---

## 4. Ví dụ đơn giản

**Yêu cầu**: Pod có label `role=db` chỉ cho phép:
-   **Ingress**:
    -   Từ Pod có label `role=frontend` trong cùng namespace.
    -   Từ namespace có label `project=myproject`.
    -   Từ IP `172.17.0.0/16` trừ `172.17.1.0/24`.
    -   Chỉ trên port TCP `6379`.
-   **Egress**:
    -   Chỉ ra IP `10.0.0.0/24`.
    -   Chỉ trên port TCP `5978`.


``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

---

## 5. Các kiểu selector trong NetworkPolicy

- `podSelector`: Chọn Pod trong cùng namespace.
    
- `namespaceSelector`: Chọn toàn bộ Pod trong namespace được match label.
    
- `podSelector` + `namespaceSelector`: Chọn Pod cụ thể trong namespace cụ thể.
    
- `ipBlock`: Chọn theo IP/CIDR (thường cho IP ngoài cluster).
    

---

## 6. Mặc định deny/allow

- **Default allow**: Không có policy → tất cả đều allow.
    
- **Muốn default deny**:
    
    YAML
    
    ```
    # Default deny ingress
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: default-deny-ingress
    spec:
      podSelector: {}
      policyTypes:
      - Ingress
    ```
    
    Có thể làm default deny egress hoặc cả hai (Ingress + Egress).
    

---

## 7. Giới hạn của NetworkPolicy

- Không ép tất cả traffic phải đi qua một cổng chung (gateway) → cần service mesh.
    
- Không cấu hình TLS → phải dùng ingress controller hoặc mesh.
    
- Không target theo tên service (chỉ target pod hoặc namespace theo label).
    
- Không log traffic bị chặn.
    
- Không có “deny rule” rõ ràng (chỉ có allow, deny là mặc định khi không match allow nào).
    
- Chưa thể chặn loopback hoặc traffic từ chính node hostNetwork.
    

---

## 💡 Hiểu nhanh

NetworkPolicy = “tường lửa mềm” trong Kubernetes cho Pod, giúp:

- Chỉ cho phép những Pod/Namespace/IP được quyền nói chuyện với nhau.
    
- Mọi cái khác bị im lặng chặn.