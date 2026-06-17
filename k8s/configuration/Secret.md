### Secret là gì?

Hãy tưởng tượng ConfigMap là một "bảng ghi chú" công khai ai cũng có thể đọc. Còn **Secret** chính là một **"chiếc két sắt"** hay một **"hộp có khóa"**.

**Nói đơn giản:** Secret là một đối tượng của Kubernetes được thiết kế đặc biệt để lưu trữ một lượng nhỏ dữ liệu nhạy cảm. Ví dụ:

- Mật khẩu (password)
    
- Token truy cập (API token)
    
- Khóa SSH (SSH key)
    

Giống như ConfigMap, việc dùng Secret giúp bạn tách biệt thông tin bí mật ra khỏi code ứng dụng. Bạn sẽ không bao giờ phải viết cứng mật khẩu vào trong mã nguồn hay container image của mình.

---

### ⚠️ Cảnh Báo An Ninh CỰC KỲ QUAN TRỌNG

Đây là điều quan trọng nhất bạn phải nhớ về Secret:

**Mặc định, dữ liệu trong Secret được lưu trữ trên etcd (cơ sở dữ liệu của Kubernetes) dưới dạng KHÔNG MÃ HÓA.** Nó chỉ được mã hóa bằng base64, vốn chỉ là một cách che giấu (obscure) chứ **hoàn toàn không phải là mã hóa bảo mật**. Bất kỳ ai có quyền truy cập vào API server hoặc vào etcd đều có thể đọc được nội dung của Secret.

**Để sử dụng Secret một cách an toàn, bạn BẮT BUỘC phải thực hiện ít nhất các bước sau:**

1. **Bật Mã Hóa Lúc Lưu Trữ (Encryption at Rest):** Cấu hình Kubernetes để mã hóa dữ liệu Secret khi nó được lưu vào etcd.
    
2. **Phân Quyền RBAC chặt chẽ:** Sử dụng RBAC (Role-Based Access Control) để giới hạn quyền truy cập vào Secret. Chỉ cấp quyền đọc Secret cho những tài khoản/ứng dụng thực sự cần nó (nguyên tắc đặc quyền tối thiểu).
    
3. **Giới hạn Truy cập cho Container:** Chỉ cho phép những container thực sự cần thiết được quyền đọc Secret, thay vì cho cả Pod.
    
4. **Cân nhắc dùng giải pháp bên ngoài:** Sử dụng các công cụ chuyên dụng như HashiCorp Vault, AWS Secrets Manager, Google Secret Manager... để quản lý và tiêm (inject) bí mật vào Pod một cách an toàn hơn.
    

---

### Các loại Secret và Công dụng

Kubernetes định nghĩa sẵn một vài "loại" (type) cho Secret để dễ dàng quản lý và giúp các công cụ khác hiểu được mục đích của nó. `type` giống như một cái "nhãn" dán lên chiếc két sắt của bạn.

|Loại (type)|Công dụng|
|---|---|
|`Opaque`|**Mặc định**. Dùng cho dữ liệu tùy ý, không theo một khuôn mẫu nào.|
|`kubernetes.io/service-account-token`|Chứa token của một ServiceAccount (cơ chế cũ).|
|`kubernetes.io/dockercfg`|Chứa thông tin xác thực registry Docker (định dạng cũ `~/.dockercfg`).|
|`kubernetes.io/dockerconfigjson`|Chứa thông tin xác thực registry Docker (định dạng mới `~/.docker/config.json`).|
|`kubernetes.io/basic-auth`|Chứa username và password cho xác thực cơ bản (Basic Authentication).|
|`kubernetes.io/ssh-auth`|Chứa thông tin xác thực SSH (khóa riêng tư).|
|`kubernetes.io/tls`|Chứa cặp certificate và private key cho TLS (HTTPS).|
|`bootstrap.kubernetes.io/token`|Chứa token dùng trong quá trình khởi tạo (bootstrap) một node mới.|

Xuất sang Trang tính

#### 1. Secret loại `Opaque`

Đây là loại mặc định nếu bạn không chỉ định `type`. Nó linh hoạt nhất.

Bash

```
# Tạo một Secret Opaque rỗng
kubectl create secret generic empty-secret
```

#### 2. Secret loại `kubernetes.io/service-account-token`

Đây là cơ chế cũ để cung cấp token tồn tại lâu dài cho Pod. **Kubernetes khuyến cáo không nên dùng cách này nữa** mà thay vào đó hãy dùng `TokenRequest` API để lấy token ngắn hạn, tự động xoay vòng, an toàn hơn nhiều.

#### 3. Secret loại `kubernetes.io/dockerconfigjson` (Rất phổ biến)

Dùng để lưu trữ thông tin đăng nhập vào các private container registry (như Docker Hub, GCR, ECR...). Khi Pod của bạn cần kéo một image từ private registry, Kubelet sẽ dùng Secret này để xác thực.

Bash

```
# Tạo một secret để kéo image từ private registry
kubectl create secret docker-registry my-registry-secret \
  --docker-server=my.private-registry.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com
```

#### 4. Secret loại `kubernetes.io/basic-auth`

Dùng cho xác thực cơ bản, yêu cầu phải có 2 key: `username` và `password`.

YAML

```
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: "admin"
  password: "t0p-S3cret-Pa$$w0rd"
```

#### 5. Secret loại `kubernetes.io/ssh-auth`

Dùng để xác thực SSH, yêu cầu phải có key `ssh-privatekey`.

#### 6. Secret loại `kubernetes.io/tls` (Rất phổ biến)

Dùng để cung cấp certificate cho Ingress Controller (để chạy HTTPS) hoặc các ứng dụng khác. Yêu cầu phải có 2 key: `tls.crt` (public certificate) và `tls.key` (private key).

Bash

```
# Tạo một TLS secret từ file cert và key có sẵn
kubectl create secret tls my-tls-secret --cert=path/to/tls.crt --key=path/to/tls.key
```

#### 7. Secret loại `bootstrap.kubernetes.io/token`

Đây là loại chuyên dụng cho hệ thống, dùng trong quá trình một node mới gia nhập vào cluster. Bạn thường sẽ không cần tạo nó bằng tay trừ khi đang cấu hình cluster thủ công.

---

### Cách Sử Dụng Secret trong Pod

Cách sử dụng Secret trong Pod gần như **giống hệt** với ConfigMap.

#### 1. Dùng Secret làm file trong Volume (Cách khuyến khích)

Đây là cách an toàn nhất. Secret sẽ được mount vào container dưới dạng các file trong một volume chỉ đọc (read-only) và được lưu trên một `tmpfs` (hệ thống file trong bộ nhớ RAM), nghĩa là nó không bị ghi xuống đĩa.

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mycontainer
    image: redis
    volumeMounts:
    - name: foo-secret-volume
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo-secret-volume
    secret:
      secretName: mysecret
```

- **Cập nhật tự động:** Giống hệt ConfigMap, nếu bạn cập nhật Secret, các file trong volume này sẽ được **tự động cập nhật**.
    
- **Ngoại lệ `subPath`:** Cũng giống hệt ConfigMap, nếu bạn dùng `subPath` để mount một file cụ thể từ Secret, nó sẽ **KHÔNG** được cập nhật tự động.
    

#### 2. Dùng Secret làm Biến Môi Trường (Environment Variables)

Bạn cũng có thể "bơm" giá trị từ Secret vào các biến môi trường của container.

YAML

```
# ... spec của container ...
env:
  - name: SECRET_USERNAME
    valueFrom:
      secretKeyRef:
        name: mysecret # Tên Secret
        key: username  # Key trong Secret
```

**Cảnh báo:** Dùng biến môi trường sẽ kém an toàn hơn vì bất kỳ ai có thể xem định nghĩa Pod (`kubectl describe pod ...`) hoặc có quyền `exec` vào container đều có thể thấy các biến môi trường này. Hơn nữa, các ứng dụng con hoặc thư viện có thể vô tình ghi log các biến môi trường ra ngoài.

- **Cập nhật tự động:** Giống hệt ConfigMap, biến môi trường lấy từ Secret sẽ **KHÔNG** được cập nhật tự động. Pod phải được khởi động lại.
    

#### 3. Dùng để Kéo Image (imagePullSecrets)

Đây là một công dụng đặc biệt của Secret. Bạn khai báo nó ở cấp `spec` của Pod để Kubelet có thể dùng nó kéo image từ private registry.

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: private-reg-pod
spec:
  containers:
  - name: my-private-container
    image: my.private-registry.com/my-app:1.0
  imagePullSecrets:
  - name: my-registry-secret # Tên của Secret loại docker-registry đã tạo ở trên
```

---

### Các khái niệm quan trọng khác

#### `data` vs `stringData`

Khi tạo Secret bằng file YAML, bạn có 2 lựa chọn:

- `data`: Yêu cầu giá trị phải được **mã hóa base64** sẵn.
    
    YAML
    
    ```
    data:
      username: YWRtaW4= # "admin" đã được mã hóa base64
    ```
    
- `stringData`: Cho phép bạn viết thẳng giá trị dạng **văn bản thuần túy (plain text)**. Kubernetes sẽ tự động mã hóa base64 giúp bạn khi tạo Secret. **Cách này dễ đọc và dễ quản lý hơn.**
    
    YAML
    
    ```
    stringData:
      username: "admin"
    ```
    

Nếu một key xuất hiện ở cả `data` và `stringData`, giá trị trong `stringData` sẽ được ưu tiên.

#### Secret Tùy chọn (Optional Secrets)

Mặc định, nếu Pod tham chiếu đến một Secret không tồn tại, Pod sẽ không thể khởi động. Bạn có thể đánh dấu nó là tùy chọn (`optional: true`), khi đó Pod sẽ vẫn chạy bình thường dù Secret không có.

YAML

```
# ... trong phần volumes ...
    secret:
      secretName: maybe-mysecret
      optional: true # Nếu secret này không tồn tại, Pod vẫn chạy
```

#### Secret Bất Biến (Immutable Secrets)

Hoạt động **y hệt** như Immutable ConfigMap. Thêm `immutable: true` vào Secret để ngăn chặn mọi thay đổi, giúp tăng cường an toàn và hiệu suất (giảm tải cho API server). Một khi đã đặt là bất biến, bạn chỉ có thể xóa đi và tạo lại.

#### "Dotfiles" (File ẩn)

Nếu một `key` trong Secret bắt đầu bằng dấu chấm (`.`), ví dụ `.secret-file`, khi được mount thành volume, nó sẽ tạo ra một file ẩn tương ứng. Bạn phải dùng `ls -la` mới thấy được file này.

#### Giới hạn và ràng buộc

- **Tên Secret:** Phải là một tên miền phụ DNS hợp lệ.
    
- **Kích thước:** Giống ConfigMap, một Secret bị giới hạn kích thước tối đa là **1MiB**.
    

#### Các giải pháp thay thế Secret

Như đã đề cập ở phần cảnh báo, đôi khi Secret của Kubernetes là chưa đủ an toàn. Các lựa chọn thay thế bao gồm:

- Dùng **ServiceAccount token** để xác thực giữa các dịch vụ trong cluster.
    
- Dùng các công cụ của bên thứ ba như **Vault, AWS/Google Secret Manager**.
    
- Các cơ chế nâng cao khác như dùng plugin thiết bị để truy cập phần cứng mã hóa (TPM)...
    

Tóm lại, Secret là công cụ nền tảng để quản lý dữ liệu nhạy cảm, nhưng bạn phải luôn ý thức về các rủi ro bảo mật mặc định của nó và áp dụng các biện pháp tăng cường để bảo vệ hệ thống của mình.