![[Pasted image 20250804213932.png]]

![[Pasted image 20250804214048.png]]


![[Pasted image 20250804214601.png]]


http attribute is the protocol that the incoming request gets forwarded to internal service and it is different to http in browser, each of them stay on two differences step

## 1. **myapp.com trong ingress là gì?**

- Trong manifest Ingress, bạn sẽ thấy khai báo:
    
    text
    
    `rules:   - host: myapp.com    http:      paths:        ...`
    
    - **`myapp.com` ở đây là tên miền (domain) mà Ingress sẽ lọc/áp dụng rule.**
        
    - Bất kỳ HTTP request nào có Host header là `myapp.com` mới được Ingress controller kiểm tra rule này.
        

## 2. **Người dùng nhập domain myapp.com ở đâu?**

- Khi truy cập ứng dụng, **user sẽ gõ trên trình duyệt hoặc app là:**
    
    text
    
    `http://myapp.com`
    
- Hệ điều hành sẽ tìm cách “resolve” (tra cứu) domain `myapp.com` thành một địa chỉ IP thông qua hệ thống DNS (Domain Name System).
    

## 3. **myapp.com trỏ về đâu?**

- **Bạn (người quản trị) phải cấu hình DNS cho domain myapp.com** để trỏ về IP public của entrypoint của ứng dụng:
    
    - Nếu dùng cloud load balancer (ví dụ AWS ELB, GCP LB): DNS trỏ về IP/DNS của load balancer.
        
    - Nếu dùng bare-metal/proxy server: DNS trỏ về IP public của proxy server đó.
        

## 4. **Luồng hoạt động tổng quát:**

1. **User gõ myapp.com trên trình duyệt.**
    
2. **DNS (do bạn cấu hình trên nhà cung cấp tên miền) trả về IP của load balancer hoặc proxy.**
    
3. **Request đến load balancer/proxy → forward vào Ingress controller pod.**
    
4. **Ingress controller kiểm tra Host header**:
    
    - Nếu là `myapp.com` và đúng path rule → route về Service backend tương ứng.
        
5. **Ứng dụng trả về phản hồi cho user.**
    

## **Tóm lại:**

- `myapp.com` trong ingress là host/domain mà user truy cập cũng như để Ingress nhận diện và định tuyến traffic chính xác.
    
- Bạn phải trỏ DNS domain này về đúng IP public của LB/proxy server phía trước Ingress Controller.
    
- Người dùng nhập domain này trên trình duyệt hoặc app — từ đó traffic mới luân chuyển vào hệ thống của bạn thông qua Ingress.
    

**Nếu không trỏ DNS về đúng IP của entrypoint (LB hoặc proxy), user sẽ không truy cập được vào app dù bạn cấu hình Ingress đúng!**



**Ý nghĩa của thuộc tính `http` trong Ingress:**

- Nó khai báo rằng các rule bên trong (ví dụ: `paths`, `backend`) chỉ áp dụng cho các kết nối HTTP/HTTPS.
    
- Bên trong `http`, bạn định nghĩa danh sách các `paths` mà Ingress sẽ listen và phân phối (route) các HTTP request dựa trên đường dẫn, tới Service backend tương ứng.
    
- Khi một request tới domain/host này (ở ví dụ: myapp.com), Ingress controller sẽ dựa trên rule trong mục `http` để xác định chuyển tiếp request tới đâu (service backend nào, port nào).
    

**Ví dụ ý nghĩa từng phần trong attribute `http`:**

text

```yaml
rules:
  - host: myapp.com
    http:                  # Đây là nơi bạn định nghĩa các rule cho HTTP request
      paths:
        - backend:
            serviceName: myapp-internal-service
            servicePort: 8080

```

- Mọi HTTP request gửi tới `myapp.com` sẽ được route (chuyển tiếp) đến service tên `myapp-internal-service` ở port 8080.
    

**Tóm lại:**  
`http` là attribute để khai báo các rule route cho các HTTP request trong cấu hình Ingress, tách biệt rõ với các loại traffic khác như TCP/UDP, và phù hợp với mục tiêu của Ingress là điều phối HTTP/HTTPS traffic đến các service backend phía sau.

Trong Kubernetes Ingress, thuộc tính `paths` trong phần `http` có ý nghĩa như sau:

- `paths` là một danh sách, mỗi phần tử xác định một đường dẫn URL (path) mà Ingress controller sẽ dựa vào để route (chuyển tiếp) request HTTP tới backend service tương ứng.
    
- Mỗi mục trong `paths` xác định:
    
    - `path`: đường dẫn URL tương ứng. Ví dụ `/api`, `/products` hoặc `/`.
        
    - `backend`: service Kubernetes và port mà request sẽ được chuyển tới nếu URL path khớp.
        

Nói cách khác, `paths` giúp định tuyến HTTP request dựa theo đường dẫn URL tới các service khác nhau trong cluster.

Ví dụ:

text

```yaml
http:
  paths:
    - path: /api
      backend:
        serviceName: my-api-service
        servicePort: 8080
    - path: /web
      backend:
        serviceName: my-web-service
        servicePort: 80

```

Ý nghĩa:

- Request với URL bắt đầu bằng `/api` sẽ được Ingress forward đến service `my-api-service` port 8080.
    
- Request với URL bắt đầu bằng `/web` sẽ được forward đến service `my-web-service` port 80.
    

Ngoài ra, còn có các kiểu `pathType` để xác định cách so khớp path (Exact, Prefix, ImplementationSpecific) nhưng trong ví dụ thiết kế cơ bản thì `paths` định nghĩa các đường dẫn phục vụ cho routing request HTTP dựa trên URL.

Tóm lại:

- `paths` trong Ingress là tập các rule mapping URL path đến các backend services tương ứng.
    
- Định tuyến traffic HTTP dựa vào path để gửi request đúng service nội bộ trong cluster.
    

Thông tin chi tiết có thể tham khảo từ tài liệu chính thức Kubernetes Ingress[](https://kubernetes.io/docs/concepts/services-networking/ingress/).


![[Pasted image 20250804215427.png]]
![[Pasted image 20250804215855.png]]

Trong Kubernetes, apiVersion là một trường trong file cấu hình YAML đại diện cho phiên bản API mà resource đó sẽ sử dụng.

Cụ thể, `apiVersion: networking.k8s.io/v1beta1` có nghĩa là:

- Đây là resource sử dụng nhóm API (`API Group`) là `networking.k8s.io`.
    
- Phiên bản API là `v1beta1`, là phiên bản beta (thử nghiệm, chưa chính thức stable) của API networking.
    
- Phiên bản này được sử dụng trong một số tài nguyên quản lý mạng như Ingress trước đây.
    

Tuy nhiên, kể từ Kubernetes phiên bản 1.19 trở lên, phiên bản API chính thức (stable) cho Ingress và các tài nguyên networking được chuyển sang `networking.k8s.io/v1`. Phiên bản v1beta1 đã bị deprecated (không khuyến khích dùng) và chính thức loại bỏ trong các bản Kubernetes mới hơn.

Điều này đồng nghĩa rằng:

- Nếu bạn thấy apiVersion `networking.k8s.io/v1beta1` trong file YAML, đó là phiên bản API cũ, chỉ phù hợp với các cluster Kubernetes phiên bản thấp hơn.
    
- Với Kubernetes mới, bạn nên dùng `networking.k8s.io/v1` để đảm bảo tương thích và hỗ trợ đầy đủ tính năng.
    

Tóm lại, `apiVersion: networking.k8s.io/v1beta1` là phiên bản API cũ của các tài nguyên mạng như Ingress trong Kubernetes, đang được thay thế dần bằng phiên bản stable `networking.k8s.io/v1`. Bạn nên cập nhật file cấu hình lên phiên bản mới để tránh lỗi và tận dụng tính năng mới nhất của Kubernetes.

![[Pasted image 20250804220135.png]]

![[Pasted image 20250804220331.png]]


![[Pasted image 20250804220536.png]]


![[Pasted image 20250804224918.png]]

![[Pasted image 20250804224928.png]]

![[Pasted image 20250804225110.png]]

![[Pasted image 20250804225159.png]]

## 1. **Không có cloud, không có LoadBalancer tự động**

- Nếu bạn cài Kubernetes ở local, trên server vật lý (bare metal) hoặc môi trường không cloud, **không có dịch vụ nào tự động tạo entrypoint (lối vào) cho cluster của bạn**.
    
- Mặc định: toàn bộ các node trong cluster sẽ **không có IP public, không mở port public**, không ai bên ngoài truy cập được vào cluster.
    

## 2. **Bạn phải tự cấu hình entrypoint**

- Để người ngoài (ngoài mạng/văn phòng/laptop) truy cập được vào app của bạn (chạy trong cluster), **bạn PHẢI tự tạo một điểm vào cluster** gọi là "entrypoint".
    
- Có hai dạng chính:
    
    - **Bên trong cluster:** chọn một node của cluster, mở public IP cho nó, cấu hình route.
        
    - **Bên ngoài cluster:** dùng một server riêng biệt (proxy server), đặt nó ngoài cluster, gán IP public và route toàn bộ truy cập vào đây.
        

## 3. **Proxy Server (Load Balancer tự dựng)**

- Proxy server này có thể là **máy riêng biệt** hoặc một VM, cài một phần mềm như: Nginx, HAProxy, Traefik, hoặc thiết bị phần cứng load balancer.
    
- Bạn cấu hình nó:
    
    - Gán public IP (hoặc domain).
        
    - Open port (80, 443...).
        
    - Forward mọi request nhận được vào cluster (cụ thể là vào Ingress Controller pod).
        

**→ Proxy server này sẽ là máy DUY NHẤT public ra outside, tất cả node pod service khác trong cluster vẫn kín, giúp tăng bảo mật!**

## 4. **Luồng traffic:**

1. Client (người dùng) gửi request đến domain/public IP (trỏ vào proxy server).
    
2. Proxy server nhận request, chuyển (proxy) vào Ingress Controller pod trong cluster.
    
3. Ingress Controller kiểm tra ingress rule, xác định app backend phù hợp (theo đường dẫn, domain, v.v).
    
4. Request được route nội bộ đến đúng service, đến đúng pod.
    

## 5. **Vì sao làm vậy?**

- **Không có cloud:** Không có cloud LB, IP, DNS, bạn phải tự làm.
    
- **Giúp bảo mật:** Chỉ public 1 entrypoint, không node/pod nào khác lộ IP.
    
- **Đa dạng giải pháp:** Có thể chọn phần mềm hoặc phần cứng, tùy ý.
    

**Tóm lại:**  
Nếu bạn dùng Kubernetes ở local/bare-metal, bạn phải tự triển khai/mua/cấu hình một “entrypoint” (proxy server/load balancer) để nhận traffic từ ngoài và chuyển vào cluster. Nếu không làm vậy, app trong cluster sẽ KHÔNG truy cập được từ ngoài mạng!


![[Pasted image 20250804234457.png]]

Ingress Controller trong Kubernetes có tác dụng rất quan trọng như sau:

- Nó là một ứng dụng chạy bên trong cluster Kubernetes, có nhiệm vụ tiếp nhận lưu lượng HTTP/HTTPS từ bên ngoài (thường thông qua một LoadBalancer hoặc NodePort) rồi dựa vào các quy tắc (Ingress rules) mà bạn định nghĩa để định tuyến, chuyển hướng lưu lượng đó đến các service và pod nội bộ tương ứng.
    
- Ingress Controller sẽ "theo dõi" các tài nguyên Ingress bạn tạo ra, đọc các rule trong đó để biết cách xử lý các yêu cầu (request) đến, ví dụ như dựa trên đường dẫn URL, hostname, hoặc các điều kiện khác để phân luồng traffic đến đúng backend service.
    
- Nhờ có Ingress Controller, bạn có thể dễ dàng quản lý định tuyến truy cập HTTP/S với nhiều quy tắc phức tạp trong một nơi duy nhất, thay vì phải tạo nhiều LoadBalancer hoặc NodePort riêng lẻ cho từng service.
    
- Nó giúp tối ưu chi phí khi chỉ cần một LoadBalancer chung cho nhiều dịch vụ, đồng thời cho phép cấu hình linh hoạt với các chức năng như rewrite URL, TLS termination, authentication, rate limiting thông qua các annotation hoặc cấu hình riêng.
    

Tóm lại, Ingress Controller là thành phần "điều phối viên" chịu trách nhiệm thực thi các quy tắc định tuyến lưu lượng từ bên ngoài vào trong Kubernetes dựa trên tài nguyên Ingress mà bạn khai báo, giúp tổ chức và quản lý lưu lượng truy cập hiệu quả hơn trong cụm Kubernetes.


![[Pasted image 20250805000055.png]]

 ![[Pasted image 20250805001314.png]]

![[Pasted image 20250805001425.png]]


![[Pasted image 20250805001613.png]]

![[Pasted image 20250805001759.png]]

