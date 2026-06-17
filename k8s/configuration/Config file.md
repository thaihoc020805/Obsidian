### Giới thiệu: File Kubeconfig là gì?

Hãy tưởng tượng bạn là một người đi du lịch công tác thường xuyên. Bạn cần quản lý thông tin của nhiều văn phòng (clusters) ở các thành phố khác nhau. Mỗi văn phòng lại yêu cầu một loại thẻ ra vào (credentials) riêng.

File **kubeconfig** chính là **"cuốn sổ tay du lịch"** của bạn. Bên trong cuốn sổ này có tất cả thông tin bạn cần:

- **Danh sách các văn phòng (Clusters):** Địa chỉ của từng văn phòng, và làm sao để xác thực đó đúng là văn phòng thật (ví dụ: certificate của văn phòng).
    
- **Danh sách các thẻ ra vào (Users):** Mỗi thẻ định danh bạn là ai (ví dụ: thẻ "Nhân viên A", thẻ "Quản lý B").
    
- **Các chuyến đi (Contexts):** Một "chuyến đi" sẽ kết hợp **một thẻ ra vào** với **một văn phòng**. Ví dụ: "Chuyến đi đến văn phòng Hà Nội với tư cách Nhân viên A".
    

Công cụ dòng lệnh `kubectl` sẽ đọc "cuốn sổ tay" này để biết nó phải kết nối đến văn phòng (cluster) nào và dùng thẻ ra vào (user) nào để xác thực.

**Lưu ý:** Tên gọi "kubeconfig file" là một cách gọi chung, không có nghĩa là file đó bắt buộc phải tên là `kubeconfig`. Mặc định, `kubectl` sẽ tìm file tên là `config` trong thư mục `$HOME/.kube`.

---

### ⚠️ Cảnh Báo Bảo Mật

Chỉ sử dụng file kubeconfig từ những nguồn đáng tin cậy. Một file kubeconfig được chế tác đặc biệt có thể chứa mã độc, thực thi lệnh hoặc làm lộ file trên máy của bạn. Nếu bắt buộc phải dùng một file không tin tưởng, hãy kiểm tra kỹ nội dung của nó, giống như cách bạn kiểm tra một đoạn mã shell script lạ.

---

### Ba thành phần chính của Kubeconfig

Một file kubeconfig được cấu trúc từ 3 phần chính:

#### 🏢 1. Clusters (Các Cụm)

Đây là danh sách các cluster Kubernetes mà bạn muốn kết nối tới. Mỗi cluster có:

- `name`: Một cái tên thân thiện do bạn đặt (ví dụ: `dev-cluster`, `prod-cluster`).
    
- `server`: Địa chỉ URL của API Server của cluster đó.
    
- `certificate-authority-data`: Dữ liệu certificate của CA (Certificate Authority) để `kubectl` có thể xác minh danh tính của API Server (tránh bị tấn công man-in-the-middle).
    

#### 👤 2. Users (Người dùng)

Đây là danh sách các thông tin xác thực (credentials). Mỗi user có:

- `name`: Một cái tên thân thiện do bạn đặt (ví dụ: `dev-admin`, `test-user`).
    
- `user`: Chứa thông tin xác thực, ví dụ như:
    
    - `token`: Một bearer token.
        
    - `client-certificate-data` và `client-key-data`: Một cặp certificate/key để xác thực.
        

#### поездка 3. Contexts (Ngữ cảnh)

Đây là thành phần "kết dính" mọi thứ lại với nhau. Một **context** là một bộ ba bao gồm:

- `cluster`: Tên của một cluster đã định nghĩa ở trên.
    
- `user`: Tên của một user đã định nghĩa ở trên.
    
- `namespace` (tùy chọn): Namespace mặc định sẽ được sử dụng khi bạn làm việc trong context này.
    

**Context chính là trái tim của sự linh hoạt.** Nó cho phép bạn tạo ra các "tổ hợp" truy cập tiện lợi. Ví dụ:

- Context `dev`: Dùng user `dev-admin` để kết nối tới `dev-cluster`.
    
- Context `prod-viewer`: Dùng user `readonly-user` để kết nối tới `prod-cluster` ở namespace `monitoring`.
    

---

### Sử dụng Context để chuyển đổi qua lại

`kubectl` luôn làm việc trong một "context hiện tại" (current context). Để xem context hiện tại và tất cả các context khác, bạn dùng lệnh:

Bash

```
kubectl config get-contexts
```

Output sẽ có dạng:

```
CURRENT   NAME           CLUSTER      AUTHINFO   NAMESPACE
* dev-context    dev-cluster  dev-admin
          prod-context   prod-cluster prod-admin
```

Dấu `*` cho biết `dev-context` đang là context hiện tại.

Để **chuyển sang một context khác**, bạn dùng lệnh:

Bash

```
kubectl config use-context prod-context
```

Bây giờ, mọi lệnh `kubectl` bạn gõ sẽ được thực thi trên `prod-cluster` với tư cách là `prod-admin`.

---

### Kubectl tìm và hợp nhất (merge) file Kubeconfig như thế nào?

`kubectl` có một logic rất rõ ràng để xác định cấu hình cuối cùng mà nó sẽ sử dụng.

#### 1. Thứ tự ưu tiên tìm file:

1. **Flag `--kubeconfig`:** Nếu bạn chỉ định flag này (`kubectl get pods --kubeconfig=/path/to/my/config`), `kubectl` sẽ **CHỈ** dùng file này và **KHÔNG** hợp nhất với bất kỳ file nào khác. Đây là ưu tiên cao nhất.
    
2. **Biến môi trường `KUBECONFIG`:** Nếu flag trên không được dùng, `kubectl` sẽ tìm đến biến môi trường `KUBECONFIG`. Biến này có thể chứa một danh sách các đường dẫn đến nhiều file kubeconfig (ngăn cách bởi dấu `:` trên Linux/Mac và `;` trên Windows). `kubectl` sẽ **hợp nhất (merge)** tất cả các file này lại thành một cấu hình hiệu quả.
    
3. **File mặc định:** Nếu cả hai cách trên đều không có, `kubectl` sẽ sử dụng file mặc định tại `$HOME/.kube/config` và không hợp nhất.
    

#### 2. Quy tắc hợp nhất (merge):

Khi `kubectl` hợp nhất các file từ biến `KUBECONFIG`, nó tuân theo quy tắc **"ai đến trước thắng"**:

- Nó sẽ đọc các file theo thứ tự chúng được liệt kê trong biến `KUBECONFIG`.
    
- Khi gặp một giá trị hoặc một map key (ví dụ: context `dev-context`, user `red-user`), nó sẽ lấy thông tin từ file đầu tiên tìm thấy và **bỏ qua hoàn toàn** thông tin của cùng key đó trong các file sau, kể cả khi thông tin đó không xung đột.
    
- Ví dụ: Nếu `file1.yaml` và `file2.yaml` đều định nghĩa một user tên là `red-user`, chỉ có thông tin của `red-user` trong `file1.yaml` được sử dụng.
    

---

### Luồng xác định thông tin cuối cùng (chi tiết)

Đây là cách `kubectl` "suy nghĩ" để quyết định chính xác nó sẽ kết nối đến đâu và với tư cách là ai:

1. **Tìm Context để sử dụng:**
    
    - Ưu tiên 1: Lấy từ flag `--context`.
        
    - Ưu tiên 2: Lấy từ trường `current-context` trong cấu hình đã được hợp nhất.
        
2. **Tìm Cluster và User từ Context:**
    
    - Khi đã có context, `kubectl` sẽ lấy `cluster` và `user` được định nghĩa trong context đó.
        
    - Tuy nhiên, người dùng có thể ghi đè bằng flag: `--cluster` hoặc `--user`.
        
3. **Hoàn thiện thông tin Cluster:**
    
    - Ưu tiên 1: Các flag dòng lệnh như `--server`, `--certificate-authority`.
        
    - Ưu tiên 2: Thông tin `cluster` đã tìm được từ các file kubeconfig đã hợp nhất.
        
    - Nếu không có địa chỉ `server`, lệnh sẽ thất bại.
        
4. **Hoàn thiện thông tin User:**
    
    - Ưu tiên 1: Các flag dòng lệnh như `--client-certificate`, `--token`, `--username`...
        
    - Ưu tiên 2: Thông tin `user` đã tìm được từ các file kubeconfig đã hợp nhất.


### 1. Đường dẫn tương đối (File References)

#### Vấn đề là gì?

Trong file kubeconfig, bạn thường cần trỏ đến các file khác, ví dụ như file certificate (`ca.crt`), client certificate (`client.crt`), và client key (`client.key`). Bạn có hai cách để chỉ định đường dẫn đến các file này:

1. **Đường dẫn tuyệt đối:** Là đường dẫn đầy đủ, bắt đầu từ gốc của ổ đĩa.
    
    - Ví dụ trên Linux/Mac: `/home/an/projects/my-cluster/certs/client.key`
        
    - Ví dụ trên Windows: `C:\Users\An\Projects\my-cluster\certs\client.key`
        
2. **Đường dẫn tương đối:** Là đường dẫn được tính từ một vị trí gốc nào đó.
    

Câu "đường dẫn tương đối so với vị trí của chính file kubeconfig đó" có nghĩa là **vị trí gốc để tính toán chính là thư mục chứa file kubeconfig của bạn.**

#### Ví dụ thực tế:

Hãy tưởng tượng bạn có cấu trúc thư mục cho một dự án như sau:

```
/home/an/project-alpha/
├── kubeconfig.yaml  <-- File kubeconfig của bạn
└── certs/
    ├── ca.crt
    ├── client.crt
    └── client.key
```

Bây giờ, hãy xem nội dung bên trong file `kubeconfig.yaml`:

**Cách làm kém linh hoạt (dùng đường dẫn tuyệt đối):**

YAML

```
apiVersion: v1
clusters:
- name: my-cluster
  cluster:
    server: https://1.2.3.4
    certificate-authority: /home/an/project-alpha/certs/ca.crt # <-- Đường dẫn tuyệt đối
users:
- name: my-user
  user:
    client-certificate: /home/an/project-alpha/certs/client.crt # <-- Đường dẫn tuyệt đối
    client-key: /home/an/project-alpha/certs/client.key       # <-- Đường dẫn tuyệt đối
```

=> **Nhược điểm:** Nếu bạn nén thư mục `project-alpha` này và gửi cho đồng nghiệp tên là "Bình", anh ấy giải nén vào `/home/binh/project-alpha/`. File kubeconfig này sẽ **KHÔNG HOẠT ĐỘNG** trên máy của anh ấy, vì đường dẫn `/home/an/...` không tồn tại.

**Cách làm linh hoạt (dùng đường dẫn tương đối):**

YAML

```
apiVersion: v1
clusters:
- name: my-cluster
  cluster:
    server: https://1.2.3.4
    certificate-authority: certs/ca.crt # <-- Đường dẫn tương đối
users:
- name: my-user
  user:
    client-certificate: certs/client.crt # <-- Đường dẫn tương đối
    client-key: certs/client.key       # <-- Đường dẫn tương đối
```

=> **Ưu điểm (Tính di động):** `kubectl` khi đọc file `kubeconfig.yaml` này, nó sẽ hiểu rằng "à, file này đang nằm ở `/home/an/project-alpha/`, vậy thì `certs/ca.crt` có nghĩa là `/home/an/project-alpha/certs/ca.crt`". Khi anh Bình dùng file này trên máy của anh ấy, `kubectl` cũng sẽ tự động hiểu `certs/ca.crt` nghĩa là `/home/binh/project-alpha/certs/ca.crt`.

**Kết luận:** Sử dụng đường dẫn tương đối làm cho bộ cấu hình của bạn trở nên **di động**. Bạn có thể sao chép cả thư mục dự án đi bất cứ đâu và nó sẽ hoạt động ngay lập tức mà không cần chỉnh sửa lại đường dẫn.

---

### 2. Proxy

#### Vấn đề là gì?

Trong nhiều môi trường doanh nghiệp hoặc tổ chức, máy tính của bạn không được phép kết nối trực tiếp ra Internet. Mọi kết nối phải đi qua một "trạm trung chuyển" gọi là **Proxy Server** để kiểm soát an ninh và tuân thủ chính sách của công ty.

Nếu cluster Kubernetes của bạn nằm ở bên ngoài mạng công ty (ví dụ trên các dịch vụ đám mây như GKE, EKS, AKS), lệnh `kubectl` từ máy bạn sẽ bị chặn, không thể kết nối tới API server của cluster.

#### Giải pháp: `proxy-url`

Trường `proxy-url` trong file kubeconfig cho phép bạn chỉ định cho `kubectl` rằng: "Này `kubectl`, trước khi kết nối tới địa chỉ `server` của cluster, hãy đi qua cái proxy này trước nhé."

#### Ví dụ thực tế:

- **Máy tính của bạn:** Đang ở trong mạng công ty.
    
- **Proxy của công ty:** Có địa chỉ `http://proxy.cong-ty.com:8080`.
    
- **Cluster Kubernetes:** Nằm trên cloud, có địa chỉ `https://my-cluster-api.amazonaws.com`.
    

Đây là cách bạn cấu hình `proxy-url` trong file kubeconfig:

YAML

```
apiVersion: v1
kind: Config
clusters:
- name: my-cloud-cluster
  cluster:
    # Đây là địa chỉ thật của cluster trên cloud
    server: https://my-cluster-api.amazonaws.com
    certificate-authority-data: "DATA..."
    
    # Báo cho kubectl phải đi qua proxy này
    proxy-url: http://proxy.cong-ty.com:8080

contexts:
# ...
users:
# ...
```

**Luồng kết nối sẽ diễn ra như sau:**

1. Bạn gõ lệnh `kubectl get pods`.
    
2. `kubectl` đọc file kubeconfig, thấy có `proxy-url`.
    
3. Thay vì cố gắng kết nối thẳng đến `https://my-cluster-api.amazonaws.com`, `kubectl` sẽ gửi yêu cầu đến `http://proxy.cong-ty.com:8080`.
    
4. Proxy Server của công ty nhận được yêu cầu, kiểm tra và sau đó chuyển tiếp (forward) yêu cầu đó đến `https://my-cluster-api.amazonaws.com`.
    
5. Phản hồi từ API Server cũng sẽ đi ngược lại qua Proxy trước khi về đến máy bạn.
    

**Kết luận:** Tính năng này giúp `kubectl` "tuân thủ" các quy định về mạng của tổ chức, cho phép bạn làm việc với các cluster bên ngoài từ một môi trường mạng bị kiểm soát chặt chẽ.