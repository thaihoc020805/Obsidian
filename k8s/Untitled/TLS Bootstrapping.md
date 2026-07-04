# 1. TLS bootstrapping

## Bản dịch

Trong một Kubernetes cluster, các thành phần trên worker node — `kubelet` và `kube-proxy` — cần giao tiếp với các thành phần của control plane, cụ thể là `kube-apiserver`.

Để đảm bảo việc giao tiếp:

- được giữ bí mật;
    
- không bị can thiệp;
    
- và mỗi thành phần biết rằng nó đang nói chuyện với một thành phần đáng tin cậy khác;
    

Kubernetes đặc biệt khuyến nghị sử dụng **TLS client certificate** trên các node.

Quy trình thông thường để khởi tạo các thành phần này, đặc biệt là worker node cần certificate để giao tiếp an toàn với `kube-apiserver`, có thể khá phức tạp. Công việc này thường nằm ngoài phạm vi tự động hóa cơ bản của Kubernetes và đòi hỏi thêm nhiều thao tác.

Điều đó làm cho việc khởi tạo hoặc mở rộng cluster trở nên khó khăn.

Để đơn giản hóa quy trình, từ Kubernetes 1.4, Kubernetes giới thiệu API dùng để:

- yêu cầu certificate;
    
- phê duyệt yêu cầu;
    
- ký và cấp certificate.
    

Tài liệu này mô tả:

- quy trình khởi tạo node;
    
- cách cấu hình TLS client certificate bootstrapping cho kubelet;
    
- và cơ chế đó hoạt động như thế nào.
    

## Giải thích

Ở đây đang nói đến chiều giao tiếp:

```text
kubelet trên worker
        |
        | HTTPS request
        v
kube-apiserver trên control plane
```

Kubelet là **TLS client**, còn API server là **TLS server**.

Nhưng trong mutual TLS, hai bên đều có thể xuất trình certificate:

```text
API server đưa server certificate
→ kubelet kiểm tra certificate của API server

kubelet đưa client certificate
→ API server kiểm tra certificate của kubelet
```

Hai certificate có mục đích khác nhau:

```text
API server serving certificate
Dùng để chứng minh: "Tôi đúng là API server"

Kubelet client certificate
Dùng để chứng minh: "Tôi đúng là kubelet của node X"
```

TLS bootstrapping chủ yếu giải quyết vấn đề:

> Một kubelet mới chưa có client certificate thì làm cách nào xin được client certificate đầu tiên?

Đây là bài toán kiểu “con gà và quả trứng”:

```text
Muốn gọi API server an toàn
→ kubelet cần certificate

Muốn xin certificate
→ kubelet phải gọi API server
```

Kubernetes giải quyết bằng một credential tạm thời gọi là **bootstrap token**.

---

# 2. Initialization process

## Bản dịch

Khi một worker node khởi động, kubelet thực hiện các bước sau:

1. Tìm file kubeconfig của nó.
    
2. Lấy URL của API server và thông tin xác thực từ kubeconfig; thông thường là TLS private key và certificate đã được ký.
    
3. Thử giao tiếp với API server bằng các thông tin xác thực đó.
    

Giả sử `kube-apiserver` xác thực thành công credential của kubelet, API server sẽ coi kubelet là một node hợp lệ và bắt đầu giao Pod cho node đó.

Quy trình trên phụ thuộc vào:

- private key và certificate phải tồn tại trên máy local và được tham chiếu trong kubeconfig;
    
- certificate phải được ký bởi một Certificate Authority mà `kube-apiserver` tin tưởng.
    

Các công việc sau thuộc trách nhiệm của người thiết lập và quản lý cluster:

1. Tạo CA private key và CA certificate.
    
2. Phân phối CA certificate đến các control-plane node, nơi `kube-apiserver` đang chạy.
    
3. Tạo private key và certificate cho từng kubelet. Kubernetes đặc biệt khuyến nghị mỗi kubelet có certificate riêng và Common Name riêng.
    
4. Dùng CA private key để ký certificate của kubelet.
    
5. Phân phối kubelet private key và certificate đã ký đến đúng node chạy kubelet đó.
    

TLS bootstrapping được mô tả trong tài liệu này nhằm đơn giản hóa và tự động hóa một phần hoặc toàn bộ các bước từ bước 3 trở đi. Đây là những bước phải thực hiện thường xuyên khi khởi tạo hoặc mở rộng cluster.

## Giải thích

Không có TLS bootstrapping, admin phải tự làm gần như thế này:

```text
Trên control plane:

1. Tạo private key cho wk1
2. Tạo CSR cho wk1
3. Dùng Kubernetes CA ký CSR
4. Tạo certificate cho wk1
5. Copy private key + certificate sang wk1
6. Tạo kubeconfig cho kubelet wk1
```

Sau đó làm lại cho:

```text
wk2
wk3
wk4
...
```

Đó là lý do khi cluster có nhiều node thì thao tác thủ công rất phiền và dễ lộ private key.

Một kubelet client certificate thường có danh tính dạng:

```text
Subject:
  CN = system:node:fdp-k8s-wk1
  O  = system:nodes
```

Trong đó:

```text
CN = username khi gọi API
O  = group của username
```

Vì vậy, sau khi xác thực certificate, API server coi kubelet là:

```text
User:  system:node:fdp-k8s-wk1
Group: system:nodes
```

Lưu ý câu:

> API server xác thực thành công thì bắt đầu giao Pod cho node.

Cách diễn đạt này hơi đơn giản hóa. Thực tế:

```text
kubelet đăng ký/tạo hoặc cập nhật Node object
scheduler chọn node cho Pod
scheduler ghi spec.nodeName vào Pod
kubelet watch các Pod có spec.nodeName bằng node của nó
```

Không phải API server tự quyết định Pod chạy trên node nào. API server chủ yếu lưu trữ và cung cấp API; scheduler mới là thành phần chọn node.

---

# 3. Bootstrap initialization

## Bản dịch

Trong quá trình bootstrap initialization, các bước sau xảy ra:

1. Kubelet bắt đầu chạy.
    
2. Kubelet phát hiện rằng nó chưa có kubeconfig thông thường.
    
3. Kubelet tìm và thấy một file `bootstrap-kubeconfig`.
    
4. Kubelet đọc bootstrap kubeconfig để lấy:
    
    - URL của API server;
        
    - một token có quyền sử dụng hạn chế.
        
5. Kubelet kết nối đến API server và xác thực bằng token.
    
6. Kubelet lúc này có credential giới hạn, chỉ đủ để tạo và truy xuất CertificateSigningRequest.
    
7. Kubelet tạo một CSR cho chính nó, với:
    

```text
signerName: kubernetes.io/kube-apiserver-client-kubelet
```

8. CSR được phê duyệt bằng một trong hai cách:
    
    - `kube-controller-manager` tự động phê duyệt, nếu đã cấu hình;
        
    - một quy trình bên ngoài, có thể là con người, phê duyệt CSR thông qua Kubernetes API hoặc `kubectl`.
        
9. Certificate được tạo cho kubelet.
    
10. Certificate được cấp cho kubelet.
    
11. Kubelet lấy certificate về.
    
12. Kubelet tạo kubeconfig chính thức, chứa hoặc tham chiếu private key và certificate đã ký.
    
13. Kubelet bắt đầu hoạt động bình thường.
    
14. Tùy chọn: nếu được cấu hình, kubelet tự động yêu cầu gia hạn certificate khi certificate sắp hết hạn.
    
15. Certificate gia hạn được phê duyệt và cấp tự động hoặc thủ công, tùy cấu hình.
    

Phần còn lại của tài liệu mô tả các bước cần thiết để cấu hình TLS bootstrapping và các giới hạn của nó.

## Giải thích toàn bộ luồng

Có thể hình dung thành ba giai đoạn.

### Giai đoạn 1: kubelet chưa có danh tính chính thức

Kubelet chỉ có:

```text
bootstrap token
CA certificate để kiểm tra API server
địa chỉ API server
```

Ví dụ bootstrap kubeconfig:

```yaml
server: https://k8s-api:6443
certificate-authority: /etc/kubernetes/pki/ca.crt
token: abcdef.0123456789abcdef
```

Bootstrap token không chứng minh:

> Tôi là node `fdp-k8s-wk1`.

Nó chỉ chứng minh đại khái:

> Tôi là một client được phép tham gia quy trình bootstrap.

Khi dùng bootstrap token, danh tính thường là:

```text
Username: system:bootstrap:<token-id>
Groups:
  - system:bootstrappers
  - system:bootstrappers:kubeadm:default-node-token
```

### Giai đoạn 2: kubelet tự tạo key và CSR

Kubelet tự tạo private key ngay trên worker node:

```text
worker node:
  kubelet private key
```

Private key không cần gửi đến control plane.

Kubelet chỉ gửi CSR, trong đó chứa:

```text
public key
subject mong muốn
key usages
chữ ký chứng minh nó giữ private key tương ứng
```

Luồng:

```text
kubelet
  |
  | POST CertificateSigningRequest
  v
kube-apiserver
  |
  | lưu CSR object
  v
etcd
```

CSR là một Kubernetes API object, ví dụ:

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: node-csr-abc
spec:
  username: system:bootstrap:abcdef
  groups:
  - system:bootstrappers
  signerName: kubernetes.io/kube-apiserver-client-kubelet
  request: <base64 encoded PKCS#10 CSR>
  usages:
  - digital signature
  - client auth
```

### Giai đoạn 3: duyệt và ký

Có hai bước độc lập:

```text
Approve CSR
Sign CSR
```

Đây là chỗ rất dễ nhầm.

Một controller có thể xác định:

```text
CSR này hợp lệ → đánh dấu Approved
```

Sau đó signing controller mới:

```text
thấy CSR đã Approved
→ dùng CA private key ký
→ ghi certificate vào status.certificate
```

Luồng đầy đủ:

```text
kubelet
   |
   | tạo CSR
   v
API server / etcd
   |
   | CSR approver kiểm tra
   v
CSR condition = Approved
   |
   | CSR signing controller dùng CA key
   v
status.certificate = signed certificate
   |
   | kubelet GET/WATCH CSR
   v
kubelet tải certificate về
```

Sau đó kubelet không cần bootstrap token cho hoạt động bình thường nữa.

Nó chuyển sang client certificate:

```text
system:node:fdp-k8s-wk1
```

---

# 4. Configuration

## Bản dịch

Để cấu hình TLS bootstrapping và tùy chọn tự động phê duyệt, bạn phải cấu hình các thành phần sau:

- `kube-apiserver`;
    
- `kube-controller-manager`;
    
- `kubelet`;
    
- các resource trong cluster:
    
    - `ClusterRoleBinding`;
        
    - và có thể cả `ClusterRole`.
        

Ngoài ra, bạn cần Kubernetes Certificate Authority.

## Giải thích

Mỗi thành phần đảm nhiệm một phần khác nhau:

```text
kubelet
→ tạo private key và CSR

kube-apiserver
→ xác thực bootstrap token
→ kiểm tra RBAC
→ nhận và lưu CSR object

CSR approving controller
→ quyết định CSR có được Approved không

CSR signing controller trong kube-controller-manager
→ dùng CA private key để ký CSR

RBAC
→ quy định ai được tạo CSR
→ ai đủ điều kiện được auto-approve
```

Không có một thành phần duy nhất làm toàn bộ TLS bootstrapping.

---

# 5. Certificate Authority

## Bản dịch

Giống như trường hợp không sử dụng bootstrapping, bạn vẫn cần một CA private key và CA certificate.

Chúng sẽ được sử dụng để ký certificate của kubelet. Bạn vẫn chịu trách nhiệm phân phối chúng đến các control-plane node.

Trong phạm vi tài liệu này, giả sử chúng được phân phối đến control-plane node tại:

```text
/var/lib/kubernetes/ca.pem
/var/lib/kubernetes/ca-key.pem
```

Trong đó:

```text
ca.pem      là CA certificate
ca-key.pem  là CA private key
```

Tài liệu gọi chúng là “Kubernetes CA certificate and key”.

Tất cả các thành phần Kubernetes sử dụng các certificate này — kubelet, kube-apiserver và kube-controller-manager — đều giả định private key và certificate được mã hóa ở định dạng PEM.

## Giải thích

Hai file này hoàn toàn khác nhau:

```text
CA certificate
- chứa public key
- có thể phân phối rộng hơn
- dùng để kiểm tra certificate được CA ký

CA private key
- bí mật cực kỳ quan trọng
- dùng để ký certificate mới
- không được copy xuống worker node
```

Trong cluster được tạo bằng `kubeadm`, chúng thường là:

```text
/etc/kubernetes/pki/ca.crt
/etc/kubernetes/pki/ca.key
```

Worker node thường chỉ cần:

```text
ca.crt
```

Worker không cần và không được giữ:

```text
ca.key
```

Khi kubelet xin certificate:

```text
kube-controller-manager dùng ca.key để ký
kube-apiserver dùng ca.crt để kiểm tra certificate kubelet
```

---

# 6. kube-apiserver configuration

## Bản dịch

`kube-apiserver` có một số yêu cầu để bật TLS bootstrapping:

1. Nhận biết CA đã ký client certificate.
    
2. Xác thực kubelet đang bootstrap thành thành viên của nhóm `system:bootstrappers`.
    
3. Cấp quyền cho kubelet đang bootstrap tạo CertificateSigningRequest.
    

## Giải thích

Ba việc tương ứng với ba khái niệm riêng biệt:

```text
Authentication:
Bạn là ai?

Authorization:
Bạn có được làm việc này không?

Certificate trust:
Certificate của bạn có được CA đáng tin cậy ký không?
```

Ví dụ:

```text
Bootstrap token hợp lệ
→ Authentication thành công
→ user thuộc system:bootstrappers

User gửi POST /apis/certificates.k8s.io/.../certificatesigningrequests
→ RBAC kiểm tra authorization

Sau này kubelet dùng client certificate
→ API server kiểm tra certificate bằng client CA
```

---

# 7. Recognizing client certificates

## Bản dịch

Đây là cấu hình thông thường cho mọi cơ chế client certificate authentication.

Nếu chưa thiết lập, thêm cờ sau vào lệnh khởi chạy `kube-apiserver`:

```text
--client-ca-file=FILENAME
```

Cờ này bật client certificate authentication và trỏ đến một CA bundle chứa certificate của CA đã ký client certificate.

Ví dụ:

```text
--client-ca-file=/var/lib/kubernetes/ca.pem
```

## Giải thích

Khi kubelet gửi client certificate, API server cần trả lời câu hỏi:

> Certificate này có được một CA mà tôi tin tưởng ký không?

API server dùng file được cấu hình tại:

```text
--client-ca-file
```

để kiểm tra chữ ký.

Ví dụ kubelet gửi:

```text
Subject:
  CN = system:node:fdp-k8s-wk1
  O  = system:nodes

Issuer:
  CN = kubernetes
```

API server sẽ:

1. Đọc CA certificate trong `--client-ca-file`.
    
2. Xác minh chữ ký certificate.
    
3. Kiểm tra thời hạn.
    
4. Kiểm tra key usage có cho phép `client auth` không.
    
5. Lấy CN làm username.
    
6. Lấy Organization làm group.
    

Sau đó authentication result là:

```text
username = system:node:fdp-k8s-wk1
groups   = system:nodes
```

Cần phân biệt:

```text
--client-ca-file
```

với certificate HTTPS của chính API server:

```text
--tls-cert-file
--tls-private-key-file
```

Chúng phục vụ hai chiều khác nhau:

```text
API server certificate:
kubelet xác minh API server

Client CA:
API server xác minh kubelet
```

---

# 8. Initial bootstrap authentication

## Bản dịch

Để kubelet đang bootstrap có thể kết nối đến `kube-apiserver` và yêu cầu certificate, trước tiên nó phải được xác thực với server.

Bạn có thể dùng bất kỳ authenticator nào có khả năng xác thực kubelet.

Mặc dù có thể sử dụng bất kỳ chiến lược xác thực nào cho credential bootstrap ban đầu của kubelet, hai cơ chế sau được khuyến nghị vì dễ cấp phát:

1. Bootstrap Tokens.
    
2. Token authentication file.
    

Dùng bootstrap token là phương pháp đơn giản hơn, dễ quản lý hơn và không yêu cầu thêm cờ đặc biệt để khởi động `kube-apiserver`, ngoài việc bật bootstrap token authentication khi cần.

Dù chọn phương pháp nào, yêu cầu là kubelet phải xác thực thành một user có quyền:

- tạo và truy xuất CSR;
    
- đủ điều kiện được tự động phê duyệt khi yêu cầu node client certificate, nếu bật auto-approval.
    

Kubelet xác thực bằng bootstrap token sẽ được xác thực là một user thuộc group:

```text
system:bootstrappers
```

Đây là phương pháp tiêu chuẩn.

Khi tính năng phát triển hoàn thiện hơn, bạn nên đảm bảo token được ràng buộc với chính sách RBAC, giới hạn request dùng bootstrap token chỉ trong các thao tác liên quan đến cấp certificate.

Việc giới hạn token theo group cho phép quản lý linh hoạt. Ví dụ, bạn có thể vô hiệu hóa quyền truy cập của một bootstrap group sau khi đã hoàn tất cấp phát node.

## Giải thích

Bootstrap token chỉ nên có quyền cực nhỏ:

```text
được tạo CSR
được đọc CSR của mình
có thể đủ điều kiện auto-approve
```

Nó không nên có các quyền như:

```text
tạo Pod
đọc Secret
xóa Deployment
đọc toàn bộ Node
```

Mô hình đúng là:

```text
token → authentication → group system:bootstrappers
group → ClusterRoleBinding → quyền tạo CSR
```

Token không trực tiếp “chứa quyền”.

Quyền đến từ RBAC:

```text
Identity
   |
   v
Group membership
   |
   v
ClusterRoleBinding
   |
   v
ClusterRole permissions
```

---

# 9. Bootstrap tokens

## Bản dịch

Bootstrap token được lưu dưới dạng Secret trong Kubernetes cluster, sau đó được cấp cho từng kubelet.

Bạn có thể:

- dùng một token cho toàn bộ cluster;
    
- hoặc cấp một token riêng cho mỗi worker node.
    

Quy trình gồm hai bước:

1. Tạo Kubernetes Secret chứa token ID, token secret và các scope.
    
2. Cấp token cho kubelet.
    

Từ góc nhìn của kubelet, token này không có ý nghĩa đặc biệt so với token khác.

Tuy nhiên, từ góc nhìn của `kube-apiserver`, bootstrap token là đặc biệt. Dựa vào:

- loại Secret;
    
- namespace;
    
- tên Secret;
    

API server nhận biết đây là bootstrap token và coi người xác thực bằng token đó là thành viên của group:

```text
system:bootstrappers
```

Điều này đáp ứng yêu cầu cơ bản của TLS bootstrapping.

Nếu muốn sử dụng bootstrap token, bạn phải bật trên `kube-apiserver` bằng cờ:

```text
--enable-bootstrap-token-auth=true
```

## Giải thích

Trong cluster kubeadm, bootstrap token thường có dạng:

```text
abcdef.0123456789abcdef
```

Trong đó:

```text
abcdef            = token ID
0123456789abcdef  = token secret
```

Secret tương ứng thường có tên:

```text
bootstrap-token-abcdef
```

và nằm trong namespace:

```text
kube-system
```

Ví dụ:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-abcdef
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  token-id: abcdef
  token-secret: 0123456789abcdef
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"
  auth-extra-groups: system:bootstrappers:kubeadm:default-node-token
```

Khi kubelet gửi:

```http
Authorization: Bearer abcdef.0123456789abcdef
```

API server tách token ID:

```text
abcdef
```

rồi tìm Secret:

```text
kube-system/bootstrap-token-abcdef
```

Nếu token secret khớp và chưa hết hạn, authentication thành công.

Trong cluster kubeadm, bạn có thể xem token bằng:

```bash
kubeadm token list
```

Tạo token mới:

```bash
kubeadm token create
```

Tạo token và in luôn lệnh join:

```bash
kubeadm token create --print-join-command
```

---

# 10. Token authentication file

## Bản dịch

`kube-apiserver` cũng có khả năng chấp nhận token từ một authentication file.

Các token này có thể là chuỗi tùy ý, nhưng nên chứa ít nhất 128 bit entropy được sinh bởi một bộ sinh số ngẫu nhiên an toàn, ví dụ `/dev/urandom`.

Có nhiều cách sinh token, ví dụ:

```bash
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```

Lệnh này tạo token dạng:

```text
02b50b05283e98dd0fd71db496ef01e8
```

File token có thể có dạng:

```text
02b50b05283e98dd0fd71db496ef01e8,kubelet-bootstrap,10001,"system:bootstrappers"
```

Ba giá trị đầu có thể tùy chỉnh, còn group trong dấu ngoặc kép nên là:

```text
system:bootstrappers
```

Thêm cờ sau vào lệnh chạy `kube-apiserver`:

```text
--token-auth-file=FILENAME
```

để bật token authentication file.

## Giải thích

Định dạng một dòng token file là:

```text
token,username,userUID,"group1,group2"
```

Ví dụ:

```text
02b50...,kubelet-bootstrap,10001,"system:bootstrappers"
```

Khi client gửi token:

```http
Authorization: Bearer 02b50...
```

API server xác thực nó thành:

```text
username: kubelet-bootstrap
uid:      10001
groups:
- system:bootstrappers
```

Tuy nhiên đây là cơ chế cũ và kém linh hoạt hơn bootstrap Secret:

```text
Muốn thêm/xóa token
→ phải sửa file trên control plane
→ thường phải quản lý đồng bộ trên nhiều API server
```

Trong khi bootstrap token Secret được lưu qua Kubernetes API, nên quản lý dễ hơn.

Với cluster kubeadm, bạn chủ yếu gặp **Bootstrap Tokens**, không phải static token file.

---

# 11. Authorize kubelet to create CSR

## Bản dịch

Sau khi node đang bootstrap đã được xác thực là thành viên của group:

```text
system:bootstrappers
```

nó cần được cấp quyền tạo CSR và lấy CSR sau khi hoàn tất.

Kubernetes cung cấp sẵn một ClusterRole có đúng các quyền đó:

```text
system:node-bootstrapper
```

Để sử dụng, bạn chỉ cần tạo ClusterRoleBinding ràng buộc group:

```text
system:bootstrappers
```

với ClusterRole:

```text
system:node-bootstrapper
```

Manifest:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: create-csrs-for-bootstrapping
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:node-bootstrapper
  apiGroup: rbac.authorization.k8s.io
```

## Giải thích

Authentication thành công chưa có nghĩa là request được phép.

Kubelet hiện có danh tính:

```text
system:bootstrap:abcdef
```

và group:

```text
system:bootstrappers
```

Nhưng khi nó gửi:

```http
POST /apis/certificates.k8s.io/v1/certificatesigningrequests
```

API server vẫn gọi authorization layer:

```text
User system:bootstrap:abcdef
có quyền create certificatesigningrequests không?
```

ClusterRoleBinding ở trên nối:

```text
Group system:bootstrappers
          |
          v
ClusterRole system:node-bootstrapper
```

Bạn có thể xem quyền của ClusterRole bằng:

```bash
kubectl get clusterrole system:node-bootstrapper -o yaml
```

Kiểm tra trực tiếp:

```bash
kubectl auth can-i create certificatesigningrequests \
  --as=system:bootstrap:abcdef \
  --as-group=system:bootstrappers
```

Vai trò của binding này chỉ là:

```text
cho phép tạo và làm việc với CSR
```

Nó chưa có nghĩa CSR sẽ tự động được duyệt.

Hai thứ riêng biệt:

```text
Quyền tạo CSR
≠
Quyền được auto-approve CSR
```

---

# 12. kube-controller-manager configuration

## Bản dịch

Trong khi API server tiếp nhận certificate request từ kubelet và xác thực các request đó, `kube-controller-manager` chịu trách nhiệm cấp certificate thực tế đã được ký.

Controller-manager thực hiện chức năng này thông qua certificate-issuing control loop.

Nó hoạt động như một local signer, sử dụng các tài nguyên nằm trên disk.

Theo nội dung tài liệu, các certificate được cấp có thời hạn một năm và một tập key usage mặc định.

Để controller-manager có thể ký certificate, nó cần:

- truy cập được Kubernetes CA private key và CA certificate;
    
- bật chức năng ký CSR.
    

## Giải thích

Phân chia trách nhiệm:

```text
kube-apiserver:
- nhận HTTP request
- authenticate
- authorize
- lưu CSR object

kube-controller-manager:
- watch CSR objects
- approve trong một số trường hợp
- ký CSR đã được Approved
```

API server không tự cầm CA private key để ký certificate.

Signing controller chạy trong `kube-controller-manager` mới cần CA private key.

Luồng watch:

```text
kube-controller-manager
        |
        | watch CertificateSigningRequest
        v
kube-apiserver
```

Khi thấy CSR:

```text
Approved = true
signerName phù hợp
```

signing controller ký và cập nhật:

```yaml
status:
  certificate: <base64-certificate>
```

---

# 13. Access to key and certificate

## Bản dịch

Như đã mô tả trước đó, bạn cần tạo Kubernetes CA private key và CA certificate, rồi phân phối chúng đến control-plane node.

Chúng được controller-manager sử dụng để ký certificate của kubelet.

Các certificate đã ký sau đó sẽ được kubelet sử dụng để xác thực với `kube-apiserver`.

Vì vậy, CA được cung cấp cho controller-manager cũng phải là CA mà API server tin tưởng để xác thực client certificate.

CA này được cấu hình cho API server bằng:

```text
--client-ca-file=FILENAME
```

Để cung cấp CA certificate và private key cho `kube-controller-manager`, dùng các cờ:

```text
--cluster-signing-cert-file="/etc/path/to/kubernetes/ca/ca.crt"
--cluster-signing-key-file="/etc/path/to/kubernetes/ca/ca.key"
```

Ví dụ:

```text
--cluster-signing-cert-file="/var/lib/kubernetes/ca.pem"
--cluster-signing-key-file="/var/lib/kubernetes/ca-key.pem"
```

Thời hạn của certificate đã ký có thể được cấu hình bằng:

```text
--cluster-signing-duration
```

## Giải thích

Hai bên phải thống nhất cùng một trust chain:

```text
kube-controller-manager:
dùng CA private key A để ký kubelet certificate

kube-apiserver:
dùng CA certificate A để xác minh kubelet certificate
```

Nếu dùng nhầm hai CA khác nhau:

```text
controller-manager ký bằng CA A
API server chỉ tin CA B
```

thì kubelet có certificate nhưng khi kết nối sẽ gặp lỗi như:

```text
x509: certificate signed by unknown authority
```

Trong kubeadm control plane, bạn có thể kiểm tra manifest:

```bash
sudo grep -E 'cluster-signing|client-ca' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml \
  /etc/kubernetes/manifests/kube-apiserver.yaml
```

Bạn thường thấy:

```text
kube-apiserver:
--client-ca-file=/etc/kubernetes/pki/ca.crt

kube-controller-manager:
--cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
--cluster-signing-key-file=/etc/kubernetes/pki/ca.key
```

---

# 14. Approval

## Bản dịch

Để tự động phê duyệt CSR, bạn cần cho controller-manager biết rằng các request đó được phép phê duyệt.

Điều này được thực hiện bằng cách cấp quyền RBAC cho đúng group.

Có hai tập quyền riêng biệt.

### `nodeclient`

Khi một node yêu cầu certificate mới lần đầu, nó chưa có certificate.

Nó đang xác thực bằng bootstrap token và do đó thuộc group:

```text
system:bootstrappers
```

### `selfnodeclient`

Khi một node gia hạn certificate, theo định nghĩa nó đã có certificate.

Nó dùng certificate hiện tại để xác thực và thuộc group:

```text
system:nodes
```

Để kubelet yêu cầu và nhận certificate mới lần đầu, tạo ClusterRoleBinding ràng buộc:

```text
system:bootstrappers
```

với ClusterRole:

```text
system:certificates.k8s.io:certificatesigningrequests:nodeclient
```

Manifest:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-csrs-for-group
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
  apiGroup: rbac.authorization.k8s.io
```

Để kubelet tự gia hạn client certificate, tạo ClusterRoleBinding ràng buộc:

```text
system:nodes
```

với ClusterRole:

```text
system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
```

Manifest:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-renewals-for-nodes
subjects:
- kind: Group
  name: system:nodes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
  apiGroup: rbac.authorization.k8s.io
```

Controller `csrapproving`, chạy trong `kube-controller-manager`, được bật mặc định.

Controller sử dụng SubjectAccessReview API để xác định một user có được phép yêu cầu loại CSR đó hay không, sau đó phê duyệt dựa trên kết quả authorization.

Để tránh xung đột với các approver khác, built-in approver không chủ động từ chối CSR trái phép. Nó chỉ bỏ qua các request không được cấp quyền.

Controller cũng dọn dẹp certificate hết hạn trong quá trình garbage collection.

## Giải thích sâu

Đây là phần khó nhất của tài liệu.

ClusterRole tên dài như:

```text
system:certificates.k8s.io:certificatesigningrequests:nodeclient
```

không đơn thuần cấp quyền:

```text
update CSR approval
```

Nó cho phép SubjectAccessReview trả lời rằng user đó đủ điều kiện xin đúng loại node client certificate.

### Lần đầu bootstrap

Danh tính:

```text
User: system:bootstrap:abcdef
Group: system:bootstrappers
```

CSR mong muốn:

```text
CN: system:node:fdp-k8s-wk1
O:  system:nodes
Signer: kubernetes.io/kube-apiserver-client-kubelet
```

Approver kiểm tra:

```text
Người đang xin là bootstrapper?
CSR có đúng dạng node client certificate?
Tên node có hợp lệ?
RBAC có cấp nodeclient không?
```

Nếu hợp lệ:

```yaml
status:
  conditions:
  - type: Approved
```

### Lúc gia hạn

Kubelet đã có certificate, nên authentication là:

```text
User: system:node:fdp-k8s-wk1
Group: system:nodes
```

Nó xin certificate mới cho chính nó.

Approver kiểm tra quyền:

```text
selfnodeclient
```

Tức là:

> Node này có được phép tự gia hạn certificate của chính nó không?

Nó không có nghĩa node `wk1` được tự do xin certificate cho `wk2`.

### Approver và signer là hai controller khác nhau về logic

```text
Approver:
CSR có đáng được cấp không?

Signer:
CSR đã Approved chưa?
Nếu rồi, dùng CA key ký.
```

Nếu CSR chỉ được tạo mà chưa được approve:

```bash
kubectl get csr
```

có thể thấy:

```text
CONDITION: Pending
```

Sau khi approve:

```text
CONDITION: Approved,Issued
```

`Approved` và `Issued` phản ánh hai bước khác nhau.

---

# 15. kubelet configuration

## Bản dịch

Sau khi control-plane node được cấu hình đúng và toàn bộ authentication, authorization cần thiết đã sẵn sàng, ta có thể cấu hình kubelet.

Kubelet cần các thông tin sau để bootstrap:

1. Đường dẫn lưu private key và certificate mà nó sinh ra; có thể dùng mặc định.
    
2. Đường dẫn đến kubeconfig chính thức hiện chưa tồn tại. Kubelet sẽ ghi kubeconfig đã bootstrap vào đây.
    
3. Đường dẫn đến bootstrap kubeconfig, cung cấp:
    
    - URL của API server;
        
    - bootstrap credential, ví dụ bootstrap token.
        
4. Tùy chọn: cấu hình xoay vòng certificate.
    

Bootstrap kubeconfig phải nằm tại đường dẫn kubelet có thể truy cập, ví dụ:

```text
/var/lib/kubelet/bootstrap-kubeconfig
```

Nó có định dạng giống kubeconfig thông thường:

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /var/lib/kubernetes/ca.pem
    server: https://my.server.example.com:6443
  name: bootstrap
contexts:
- context:
    cluster: bootstrap
    user: kubelet-bootstrap
  name: bootstrap
current-context: bootstrap
preferences: {}
users:
- name: kubelet-bootstrap
  user:
    token: 07401b.f395accd246ae52d
```

Các trường quan trọng:

```text
certificate-authority:
đường dẫn đến CA dùng để kiểm tra server certificate của kube-apiserver

server:
URL của kube-apiserver

token:
bootstrap token
```

Định dạng token không quan trọng, miễn là khớp với cơ chế mà API server mong đợi.

Vì bootstrap kubeconfig là kubeconfig chuẩn, bạn có thể dùng `kubectl config` để tạo:

```bash
kubectl config \
  --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig \
  set-cluster bootstrap \
  --server='https://my.server.example.com:6443' \
  --certificate-authority=/var/lib/kubernetes/ca.pem

kubectl config \
  --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig \
  set-credentials kubelet-bootstrap \
  --token=07401b.f395accd246ae52d

kubectl config \
  --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig \
  set-context bootstrap \
  --user=kubelet-bootstrap \
  --cluster=bootstrap

kubectl config \
  --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig \
  use-context bootstrap
```

Để kubelet sử dụng bootstrap kubeconfig, cấu hình:

```text
--bootstrap-kubeconfig="/var/lib/kubelet/bootstrap-kubeconfig"
--kubeconfig="/var/lib/kubelet/kubeconfig"
```

Khi kubelet khởi động:

- nếu file chỉ định bởi `--kubeconfig` chưa tồn tại;
    
- kubelet sử dụng file chỉ định bởi `--bootstrap-kubeconfig`;
    
- kubelet yêu cầu client certificate từ API server;
    
- sau khi CSR được phê duyệt và certificate được trả về;
    
- kubelet ghi kubeconfig chính thức vào đường dẫn `--kubeconfig`;
    
- certificate và private key được đặt trong thư mục `--cert-dir`.
    

## Giải thích

Có hai kubeconfig khác nhau.

### Bootstrap kubeconfig

Credential tạm thời:

```text
token
```

Mục đích duy nhất:

```text
xin client certificate đầu tiên
```

### Kubeconfig chính thức

Credential lâu dài:

```text
client private key
client certificate
```

Trong kubeadm hiện đại, bạn thường thấy:

```text
/etc/kubernetes/bootstrap-kubelet.conf
/etc/kubernetes/kubelet.conf
```

Sau khi bootstrap thành công, file bootstrap có thể bị xóa.

Kubelet thường sử dụng:

```text
/etc/kubernetes/kubelet.conf
```

Trong đó client certificate có thể trỏ đến:

```text
/var/lib/kubelet/pki/kubelet-client-current.pem
```

Ví dụ:

```yaml
users:
- name: system:node:fdp-k8s-wk1
  user:
    client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
    client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
```

Một file PEM có thể chứa cả:

```text
private key
certificate
```

Symlink:

```text
kubelet-client-current.pem
```

thường trỏ đến certificate hiện tại:

```text
kubelet-client-2026-06-23-22-51-03.pem
```

Khi rotate certificate, kubelet tạo file mới rồi cập nhật symlink. Nhờ vậy kubeconfig không phải thay đổi đường dẫn mỗi lần.

Điều kiện quan trọng:

```text
Nếu kubeconfig chính thức đã tồn tại
→ kubelet sử dụng nó
→ không quay lại bootstrap token
```

Bootstrap kubeconfig chỉ là phương án khi credential chính thức chưa tồn tại.

---

# 16. Client and serving certificates

## Bản dịch

Toàn bộ nội dung phía trên liên quan đến **kubelet client certificate**, tức certificate mà kubelet dùng để xác thực với `kube-apiserver`.

Kubelet cũng có thể sử dụng **serving certificate**.

Bản thân kubelet mở một HTTPS endpoint cho một số tính năng. Để bảo vệ endpoint đó, kubelet có thể:

1. Sử dụng private key và certificate được cung cấp qua:
    

```text
--tls-private-key-file
--tls-cert-file
```

2. Tạo self-signed private key và certificate nếu không được cung cấp.
    
3. Yêu cầu serving certificate từ cluster thông qua CSR API.
    

Client certificate được cấp bởi TLS bootstrapping mặc định chỉ được ký cho mục đích client authentication.

Do đó nó không thể được sử dụng làm serving certificate hoặc cho server authentication.

Tuy nhiên, có thể bật cơ chế yêu cầu và xoay vòng server certificate thông qua certificate rotation.

## Giải thích

Kubelet đóng hai vai trò cùng lúc.

### Vai trò client

```text
kubelet → kube-apiserver:6443
```

Dùng:

```text
kubelet client certificate
Extended Key Usage: TLS Web Client Authentication
```

### Vai trò server

Kubelet mở HTTPS port, thường là:

```text
10250
```

Các thành phần như API server có thể kết nối:

```text
kube-apiserver → kubelet:10250
```

Ví dụ cho:

```text
kubectl logs
kubectl exec
kubectl attach
port-forward trong một số luồng
metrics/resource access
```

Lúc đó kubelet cần serving certificate:

```text
Extended Key Usage: TLS Web Server Authentication
```

Không thể lấy một certificate chỉ có:

```text
Client Auth
```

rồi dùng làm server certificate.

Có thể kiểm tra certificate:

```bash
openssl x509 \
  -in /var/lib/kubelet/pki/kubelet-client-current.pem \
  -text -noout
```

Tìm:

```text
X509v3 Extended Key Usage:
    TLS Web Client Authentication
```

Serving certificate cần:

```text
TLS Web Server Authentication
```

---

# 17. Certificate rotation

## Bản dịch

Từ Kubernetes 1.8 trở lên, kubelet có các tính năng cho phép xoay vòng client certificate và serving certificate.

Để kubelet tự xoay vòng client certificate, kubelet có thể tạo CSR mới khi credential hiện tại sắp hết hạn.

Bật bằng trường:

```yaml
rotateCertificates: true
```

trong kubelet configuration file.

Cờ command line cũ, hiện không còn được khuyến nghị, là:

```text
--rotate-certificates
```

Bật `RotateKubeletServerCertificate` khiến kubelet:

- yêu cầu serving certificate sau khi đã bootstrap client credential;
    
- và tự xoay vòng serving certificate đó.
    

Bật bằng trường:

```yaml
serverTLSBootstrap: true
```

Hoặc cờ command line cũ:

```text
--rotate-server-certificates
```

Lưu ý:

Các CSR approving controller tích hợp trong Kubernetes không tự động phê duyệt node serving certificate vì lý do bảo mật.

Để sử dụng `RotateKubeletServerCertificate`, operator cần:

- chạy custom approving controller;
    
- hoặc tự phê duyệt serving CSR thủ công.
    

Quy trình duyệt serving certificate của kubelet thường chỉ nên phê duyệt CSR đáp ứng các điều kiện:

1. Request được gửi bởi node:
    
    - `spec.username` có dạng `system:node:<nodeName>`;
        
    - `spec.groups` chứa `system:nodes`.
        
2. CSR yêu cầu usage dành cho serving certificate:
    
    - chứa `server auth`;
        
    - có thể chứa `digital signature`;
        
    - có thể chứa `key encipherment`;
        
    - không chứa usage khác.
        
3. Subject Alternative Names chỉ chứa IP và DNS thực sự thuộc về node yêu cầu.
    
4. Không có URI SAN hoặc Email SAN.
    

## Giải thích

Client certificate rotation dễ tự động phê duyệt hơn vì phạm vi tương đối rõ:

```text
Node wk1 xin client certificate
cho danh tính system:node:wk1
```

Serving certificate nguy hiểm hơn vì SAN quyết định certificate hợp lệ cho hostname/IP nào.

Giả sử node `wk1` gửi CSR xin serving certificate chứa:

```text
DNS:
  kubernetes.default.svc

IP:
  172.21.92.163
```

Trong khi đó là địa chỉ hoặc tên của API server.

Nếu hệ thống tự động approve mù quáng, wk1 có thể có certificate giả danh API server hoặc một node khác.

Vì thế approver phải xác minh:

```text
Requester là wk1
SAN chỉ chứa:
- fdp-k8s-wk1
- IP thực sự của wk1
```

Trong cluster của bạn, kiểm tra kubelet config:

```bash
grep -E 'rotateCertificates|serverTLSBootstrap' \
  /var/lib/kubelet/config.yaml
```

Bạn có thể thấy:

```yaml
rotateCertificates: true
serverTLSBootstrap: true
```

Xem CSR:

```bash
kubectl get csr
```

CSR client thường có signer:

```text
kubernetes.io/kube-apiserver-client-kubelet
```

CSR serving thường có signer:

```text
kubernetes.io/kubelet-serving
```

---

# 18. Other authenticating components

## Bản dịch

Tất cả TLS bootstrapping mô tả trong tài liệu này liên quan đến kubelet.

Tuy nhiên, các thành phần khác cũng có thể cần giao tiếp trực tiếp với `kube-apiserver`.

Một ví dụ đáng chú ý là `kube-proxy`, thành phần node chạy trên mọi node. Ngoài ra còn có thể có các thành phần monitoring hoặc networking.

Giống kubelet, những thành phần này cũng cần phương pháp xác thực với `kube-apiserver`.

Có một số lựa chọn để tạo credential cho chúng:

### Cách cũ

Tự tạo và phân phối certificate giống cách đã làm cho kubelet trước khi có TLS bootstrapping.

### DaemonSet

Do kubelet đã chạy trên từng node và đủ khả năng khởi động các dịch vụ cơ bản, bạn có thể chạy:

- kube-proxy;
    
- và các dịch vụ đặc thù theo node;
    

dưới dạng DaemonSet trong namespace `kube-system`, thay vì chạy như process độc lập.

Vì chạy bên trong cluster, chúng có thể sử dụng ServiceAccount với các quyền phù hợp.

Đây có thể là phương pháp cấu hình đơn giản nhất cho các dịch vụ như vậy.

## Giải thích

Trong cluster kubeadm hiện đại, `kube-proxy` thường chạy dưới dạng:

```text
DaemonSet/kube-proxy
```

Bạn có thể kiểm tra:

```bash
kubectl -n kube-system get daemonset kube-proxy
```

Mỗi Pod kube-proxy dùng ServiceAccount:

```text
system:serviceaccount:kube-system:kube-proxy
```

Credential thường là projected ServiceAccount token, không nhất thiết là một client certificate riêng cho từng node.

Luồng:

```text
kube-proxy Pod
   |
   | ServiceAccount token
   v
kube-apiserver
```

Trong khi kubelet là process host-level:

```text
systemd → kubelet process
```

Kubelet không thể dựa vào ServiceAccount của một Pod để khởi động chính nó, vì:

```text
Muốn chạy Pod
→ trước tiên phải có kubelet hoạt động
```

Vì vậy kubelet cần cơ chế bootstrap đặc biệt.

---

# 19. kubectl approval

## Bản dịch

CSR có thể được phê duyệt bên ngoài các luồng approval tích hợp trong controller-manager.

Signing controller không ký mọi certificate request ngay lập tức.

Thay vào đó, nó chờ cho đến khi CSR được đánh dấu trạng thái:

```text
Approved
```

bởi một user có đủ đặc quyền.

Luồng này cho phép:

- automated approval bởi một external approval controller;
    
- approval controller tích hợp trong core controller-manager;
    
- hoặc cluster administrator phê duyệt thủ công.
    

Administrator có thể liệt kê CSR:

```bash
kubectl get csr
```

Xem chi tiết:

```bash
kubectl describe csr <name>
```

Phê duyệt:

```bash
kubectl certificate approve <name>
```

Từ chối:

```bash
kubectl certificate deny <name>
```

## Giải thích

Giả sử một worker join cluster nhưng auto-approval chưa được cấu hình đúng:

```bash
kubectl get csr
```

có thể hiện:

```text
NAME        AGE   SIGNERNAME                                    REQUESTOR                 CONDITION
csr-abc     2m    kubernetes.io/kube-apiserver-client-kubelet   system:bootstrap:abcdef   Pending
```

Bạn xem kỹ:

```bash
kubectl describe csr csr-abc
```

Trước khi approve, cần kiểm tra:

```text
Requester có đúng bootstrap user không?
Subject có phải system:node:<đúng-node-name> không?
Organization có phải system:nodes không?
SignerName có đúng không?
Usages có phải client auth không?
```

Sau đó:

```bash
kubectl certificate approve csr-abc
```

Kết quả:

```text
Approved,Issued
```

Nếu request có dấu hiệu giả mạo:

```bash
kubectl certificate deny csr-abc
```

`deny` không đồng nghĩa xóa CSR. Nó thêm condition từ chối vào CSR object.

---

# 20. Toàn bộ luồng TLS bootstrapping cô đọng

## Trước khi join

Control plane có:

```text
Kubernetes CA:
  ca.crt
  ca.key

API server:
  tin ca.crt để xác minh kubelet client certificate

Controller manager:
  giữ ca.key để ký kubelet certificate

Cluster:
  có bootstrap token Secret
  có RBAC cho system:bootstrappers
```

Worker có:

```text
bootstrap token
CA certificate
địa chỉ API server
kubelet
```

## Khi kubelet khởi động

```text
1. kubelet tìm kubeconfig chính thức
2. chưa có → đọc bootstrap kubeconfig
3. kiểm tra server certificate của API server bằng CA certificate
4. gửi bootstrap token
5. API server xác thực:
   system:bootstrap:<token-id>
   group system:bootstrappers
6. kubelet tự tạo private key
7. kubelet tạo CSR object
8. API server dùng RBAC cho phép tạo CSR
9. CSR approver kiểm tra và đánh dấu Approved
10. signing controller dùng CA private key ký
11. certificate được ghi vào CSR status
12. kubelet tải certificate về
13. kubelet tạo kubeconfig chính thức
14. kubelet dùng client certificate từ đây
```

## Sau bootstrap

Kubelet xác thực với danh tính:

```text
User:
system:node:fdp-k8s-wk1

Group:
system:nodes
```

Không còn xác thực bình thường bằng:

```text
system:bootstrap:<token-id>
```

## Khi certificate sắp hết hạn

```text
1. kubelet dùng certificate hiện tại để xác thực
2. tạo CSR mới
3. requester là system:node:<nodeName>
4. group là system:nodes
5. selfnodeclient approver phê duyệt
6. signer cấp certificate mới
7. kubelet cập nhật kubelet-client-current.pem
```

---

# 21. Sơ đồ tổng thể

```text
┌──────────────────── WORKER NODE ────────────────────┐
│                                                     │
│  bootstrap kubeconfig                               │
│  ├── API server URL                                 │
│  ├── CA certificate                                 │
│  └── bootstrap token                                │
│                                                     │
│  kubelet                                            │
│    ├── tạo private key                              │
│    └── tạo CSR                                      │
└───────────────┬─────────────────────────────────────┘
                │
                │ HTTPS + bootstrap token
                │ POST CSR
                ▼
┌────────────────── KUBE-APISERVER ───────────────────┐
│ Authenticate token                                  │
│ → system:bootstrap:<id>                             │
│ → system:bootstrappers                              │
│                                                     │
│ Authorize bằng RBAC                                 │
│ → được create CSR                                   │
│                                                     │
│ Lưu CSR vào etcd                                    │
└───────────────┬─────────────────────────────────────┘
                │ watch CSR
                ▼
┌────────── KUBE-CONTROLLER-MANAGER ──────────────────┐
│ CSR approving controller                            │
│ → kiểm tra request                                  │
│ → đánh dấu Approved                                 │
│                                                     │
│ CSR signing controller                              │
│ → dùng Kubernetes CA private key                    │
│ → ký certificate                                    │
│ → ghi status.certificate                            │
└───────────────┬─────────────────────────────────────┘
                │
                │ kubelet GET/WATCH CSR
                ▼
┌──────────────────── WORKER NODE ────────────────────┐
│ kubelet nhận certificate                            │
│                                                     │
│ Tạo kubeconfig chính thức                           │
│                                                     │
│ Sau đó xác thực thành:                              │
│ user  = system:node:<nodeName>                      │
│ group = system:nodes                                │
└─────────────────────────────────────────────────────┘
```

# 22. Điểm quan trọng nhất cần nhớ

TLS bootstrapping không phải là:

```text
API server tự gửi private key và certificate cho kubelet
```

Mà là:

```text
kubelet tự tạo private key trên worker
→ chỉ gửi public-key CSR lên
→ control plane duyệt và ký
→ kubelet lấy certificate đã ký về
```

Private key của kubelet không cần rời khỏi worker node.

Bootstrap token chỉ là credential tạm thời để xin certificate đầu tiên.

Sau khi bootstrap thành công, kubelet dùng client certificate riêng của node.

Certificate client và certificate serving của kubelet là hai loại khác nhau:

```text
client certificate:
kubelet xác thực tới API server

serving certificate:
kubelet chứng minh danh tính khi thành phần khác kết nối vào port 10250
```

Việc được phép tạo CSR, CSR được phê duyệt và CSR được ký là ba bước khác nhau:

```text
Create CSR
→ Approve CSR
→ Sign/Issue certificate
```