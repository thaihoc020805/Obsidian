### **Bước 1 – Cài đặt cert-manager**

- **cert-manager** là một “người quản lý chứng chỉ” trong Kubernetes.
    
- Nó biết cách nói chuyện với các CA (Let's Encrypt, Vault, …) để:
    
    - Xin cấp chứng chỉ.
        
    - Lưu chứng chỉ vào Kubernetes Secret.
        
    - Tự gia hạn chứng chỉ khi sắp hết hạn.
        
- Giống như có “robot” chuyên đi xin và quản lý giấy tờ cho bạn.
    

**Bạn đã có sẵn cert-manager** trong cluster → bỏ qua bước này.

---

### **Bước 2 – Tạo Issuer / ClusterIssuer**

- Issuer là “hồ sơ cấu hình” để cert-manager biết:
    
    - Xin chứng chỉ ở đâu (CA nào: Let's Encrypt, Vault,…).
        
    - Dùng email nào để đăng ký.
        
    - Dùng phương thức xác thực nào để chứng minh bạn sở hữu domain (HTTP-01, DNS-01…).
        
- Có 2 loại:
    
    - **Issuer**: áp dụng cho một namespace.
        
    - **ClusterIssuer**: áp dụng cho toàn cluster.
        
- Ở đây bạn đã có sẵn:
    
    - `letsencrypt-staging` → môi trường test.
        
    - `letsencrypt-prod` → môi trường thật.
        

---

### **Bước 3 – Cập nhật Ingress**

- Ingress là “cổng vào” ứng dụng trong Kubernetes.
    
- Khi bạn thêm annotation:
    
    yaml
    
    CopyEdit
    
    `cert-manager.io/cluster-issuer: "letsencrypt-prod"`
    
    nghĩa là: “Hãy dùng issuer này để xin chứng chỉ cho domain này”.
    
- Trong Ingress cũng phải khai báo:
    
    yaml
    
    CopyEdit
    
    `tls:   - hosts:       - hoc-lab-test.ceternity.com     secretName: hoc-lab-test-tls`
    
    → cert-manager sẽ lưu chứng chỉ vào Secret `hoc-lab-test-tls`, Nginx Ingress sẽ lấy ra để phục vụ HTTPS.
    

---

### **Bước 4 – cert-manager xin chứng chỉ**

- cert-manager sẽ:
    
    1. Gửi yêu cầu ACME tới Let's Encrypt.
        
    2. Let's Encrypt yêu cầu bạn “chứng minh” sở hữu domain bằng **HTTP-01 challenge**:
        
        - cert-manager tạm thời tạo 1 đường dẫn đặc biệt trên domain (`/.well-known/acme-challenge/...`) qua Ingress.
            
        - Let's Encrypt truy cập đường dẫn đó để kiểm tra.
            
    3. Nếu thành công → Let's Encrypt cấp chứng chỉ.
        
    4. cert-manager lưu chứng chỉ vào Secret trong Kubernetes.
        
    5. Ingress Nginx tự động dùng chứng chỉ đó để bật HTTPS.
        

---

### **Bước 5 – Gia hạn tự động**

- cert-manager sẽ tự kiểm tra thời hạn chứng chỉ.
    
- Khi còn khoảng 30 ngày trước khi hết hạn, nó sẽ tự xin chứng chỉ mới và cập nhật Secret → bạn không cần làm gì thêm.