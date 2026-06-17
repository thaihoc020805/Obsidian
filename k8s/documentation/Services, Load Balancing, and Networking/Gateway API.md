
# Gateway API là gì?

- Là một nhóm API kinds (CRD) cung cấp:
  - **Dynamic infrastructure provisioning**: Tự động tạo hạ tầng mạng (ví dụ: load balancer cloud).
  - **Advanced traffic routing**: Điều hướng lưu lượng nâng cao (header-based routing, traffic split, rewrite…).
- **Mục tiêu**: thay thế **Ingress** trong tương lai (Ingress hiện khá giới hạn, cần nhiều annotation tùy controller).
- **Ưu điểm lớn**:
  - Mô hình rõ ràng **theo vai trò** (role-oriented).
  - **Mở rộng** được (extensible) — bạn có thể thêm CRD tùy chỉnh.
  - **Portable** — chạy được trên nhiều nền tảng (GKE, EKS, bare-metal…).

---

# Nguyên tắc thiết kế

Gateway API chia quyền và cấu hình theo **3 vai trò**:

1.  **Infrastructure Provider** (nhà cung cấp hạ tầng)
    - Ví dụ: Cloud provider quản lý LB multi-tenant, multi-cluster.
2.  **Cluster Operator** (người quản lý cluster)
    - Cấu hình chính sách, quyền truy cập mạng, firewall, TLS, v.v.
3.  **Application Developer** (lập trình viên app)
    - Chỉ cần khai báo routing app → service (như Ingress trước đây).

Mỗi vai trò chỉ “đụng” vào đúng phần của mình → giảm xung đột và dễ quản lý.

---

# Các API kind chính (3 thành phần cốt lõi)

### a) GatewayClass

-   **Nghĩa**: định nghĩa **kiểu Gateway** (controller nào quản lý).
-   Giống như `IngressClass` nhưng mạnh hơn, chuẩn hơn.
-   Mỗi Gateway phải thuộc về **1 GatewayClass**.
-   **Ví dụ**:
    ```yaml
    apiVersion: gateway.networking.k8s.io/v1
    kind: GatewayClass
    metadata:
      name: example-class
    spec:
      controllerName: [example.com/gateway-controller](https://example.com/gateway-controller)
    ```
    
    > **Ý nghĩa**: bất kỳ Gateway nào dùng `gatewayClassName: example-class` sẽ được quản lý bởi controller `example.com/gateway-controller`.

### b) Gateway

-   **Nghĩa**: một “thực thể” hạ tầng tiếp nhận lưu lượng (LB, proxy).
-   Chứa cấu hình **listener** (giao thức, port, host).
-   **Ví dụ**:
    ```yaml
    apiVersion: gateway.networking.k8s.io/v1
    kind: Gateway
    metadata:
      name: example-gateway
    spec:
      gatewayClassName: example-class
      listeners:
      - name: http
        protocol: HTTP
        port: 80
    ```
    > **Ý nghĩa**: Tạo 1 LB/proxy nghe HTTP port 80, được triển khai bởi controller thuộc `example-class`.

### c) HTTPRoute

-   **Nghĩa**: định nghĩa **quy tắc định tuyến HTTP** từ Gateway → backend (Service).
-   Cho phép match **host**, **path**, **header**, **method**, traffic split...
-   **Ví dụ**:
    ```yaml
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: example-httproute
    spec:
      parentRefs:
      - name: example-gateway
      hostnames:
      - "[www.example.com](https://www.example.com)"
      rules:
      - matches:
        - path:
            type: PathPrefix
            value: /login
        backendRefs:
        - name: example-svc
          port: 8080
    ```
    > **Ý nghĩa**: Nếu request vào Gateway `example-gateway` với:
    > - Host: `www.example.com`
    > - Path bắt đầu `/login`
    >   → Gửi đến service `example-svc:8080`.

---

# Luồng xử lý request (HTTP ví dụ)

1.  Client gõ **http://www.example.com/login**.
2.  DNS trả về IP của Gateway.
3.  Gateway (LB/proxy) nhận request.
4.  Dựa vào **listener** + `HTTPRoute` để quyết định backend.
5.  Có thể match thêm header/path, thêm/bỏ header, split traffic…
6.  Chuyển request đến **Service backend**.

---

# So sánh nhanh với Ingress

| Tính năng | Ingress | Gateway API |
| :--- | :--- | :--- |
| Match theo path/host | ✅ | ✅ (mạnh hơn, nhiều loại match hơn) |
| Match theo header | ❌ | ✅ |
| Traffic splitting | ❌ | ✅ |
| Multi-protocol (TCP/UDP) | ❌ (phải dùng LB/Service) | ✅ |
| Role separation | ❌ | ✅ |
| Cấu hình bằng annotation | ✅ (rất nhiều) | ❌ (field chuẩn hóa) |
| Extensible | Hạn chế | ✅ |

---

# Ví dụ thực tế: Hosting nhiều app chung 1 Gateway

**GatewayClass** (chọn controller):
```yaml
kind: GatewayClass
metadata:
  name: nginx-gateway
spec:
  controllerName: k8s.io/gateway-nginx
````

**Gateway** (nghe port 80):

YAML

```
kind: Gateway
metadata:
  name: shared-gateway
spec:
  gatewayClassName: nginx-gateway
  listeners:
  - name: web
    protocol: HTTP
    port: 80
```

**HTTPRoute app1**:

YAML

```
kind: HTTPRoute
metadata:
  name: app1-route
spec:
  parentRefs:
  - name: shared-gateway
  hostnames: ["app1.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: app1-service
      port: 80
```

**HTTPRoute app2**:

YAML

```
kind: HTTPRoute
metadata:
  name: app2-route
spec:
  parentRefs:
  - name: shared-gateway
  hostnames: ["app2.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: app2-service
      port: 80
```

> Cả 2 app đều dùng chung 1 LB (Gateway) nhưng route theo host khác nhau

---

# So sánh chi tiết Ingress và Gateway API

1. **Về khả năng routing**
    
    - **Ingress**: Chỉ hỗ trợ match theo host và path. Nếu muốn match theo HTTP header hoặc HTTP method, phải dùng annotations (mỗi controller hỗ trợ khác nhau → không portable).
        
    - **Gateway API**: Hỗ trợ match nhiều tiêu chí ngay trong spec chuẩn: Hostnames, Path (Prefix, Exact, Regex), Headers, Query params, HTTP method. Không cần annotations tùy controller → portable.
        
    - **Ví dụ**:
        
        YAML
        
        ```
        rules:
        - matches:
          - headers:
            - name: X-Region
              value: VN
          backendRefs:
          - name: vn-shop-service
            port: 80
        ```
        
        > **Ingress** gần như không làm được chuẩn hóa kiểu này.
        
2. **Về traffic splitting (chia lưu lượng)**
    
    - **Ingress**: Không hỗ trợ chuẩn trong API. Muốn split 80%/20% → phải dùng annotation hoặc hẳn một service mesh.
        
    - **Gateway API**: Hỗ trợ traffic weighting ngay trong API chuẩn:
        
        
        ```
        backendRefs:
        - name: service-v1
          port: 80
          weight: 80
        - name: service-v2
          port: 80
          weight: 20
        ```
        
        > Rất tiện cho canary release hoặc blue/green deployment.
        
3. **Hỗ trợ nhiều giao thức (Multi-protocol)**
    
    - **Ingress**: Chỉ cho HTTP/HTTPS. TCP/UDP phải dùng Service type LoadBalancer hoặc config riêng của controller.
        
    - **Gateway API**: Có thể cấu hình listener cho: HTTP, HTTPS, TCP, UDP trong cùng một Gateway.
        
    - **Ví dụ**: Một Gateway nghe cả HTTP (80) và MySQL (3306):
        
        YAML
        
        ```
        listeners:
        - name: web
          protocol: HTTP
          port: 80
        - name: mysql
          protocol: TCP
          port: 3306
        ```
        
4. **Role separation (chia vai trò quản lý)**
    
    - **Ingress**: Tất cả cấu hình nằm trong 1 resource → team dev có thể vô tình sửa cả phần hạ tầng.
        
    - **Gateway API**: Chia ra:
        
        - `GatewayClass`: do nhà cung cấp hạ tầng định nghĩa.
            
        - `Gateway`: do cluster operator tạo (chỉ định port, LB, TLS...).
            
        - `HTTPRoute`: do dev app tạo (routing tới service).
            
        
        > Giúp tránh xung đột và phân quyền tốt hơn.
        
5. **Extensibility (mở rộng)**
    
    - **Ingress**: Mở rộng = annotations (tùy controller, không chuẩn).
        
    - **Gateway API**: Cho phép liên kết custom resource ở nhiều lớp (Gateway filters, Route filters…). Có thể thêm tính năng mới mà không phá chuẩn.
        
6. **Ví dụ dễ hiểu** Giả sử bạn có 1 LB chung cho 2 app: Shop và Blog. Bạn muốn:
    
    - `shop.example.com` → `shop-service`
        
    - `blog.example.com` → `blog-service`
        
    - Nếu `shop.example.com` và header `X-Region=VN` → route sang `shop-vn-service`
        
    - 10% traffic của blog → `blog-v2-service`
        
    - **Với Ingress**: Có thể làm host-based route (1 & 2) → OK. Còn điều kiện header và traffic split → phải dùng annotation tùy controller → không portable.
        
    - **Với Gateway API**: Làm tất cả trong `HTTPRoute`, portable, không phụ thuộc controller.