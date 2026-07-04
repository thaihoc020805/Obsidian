

Điểm quan trọng nhất:

> Kubeconfig chỉ là file cấu hình mà phía client tự đọc.  
> Phía server không “đọc kubeconfig của client để kiểm tra”.

TLS chỉ trao đổi certificate/public information cần thiết; private key không bao giờ được gửi qua mạng.

---

# 1. Chiều kubelet → API server

Trong chiều này:

```text
kubelet = TLS client
API server = TLS server
```

Luồng:

```text
kubelet trên worker
    │
    │ HTTPS tới k8s-api:6443
    ▼
kube-apiserver
```

## Certificate của API server nằm ở đâu?

Certificate mà API server trình cho kubelet thường nằm trên **control-plane node**:

```text
/etc/kubernetes/pki/apiserver.crt
/etc/kubernetes/pki/apiserver.key
```

Đây là:

```text
apiserver.crt = server certificate
apiserver.key = private key tương ứng
```

API server static Pod được cấu hình sử dụng chúng trong:

```text
/etc/kubernetes/manifests/kube-apiserver.yaml
```

Bạn thường thấy:

```yaml
- --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
- --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

Khi kubelet mở kết nối, API server gửi:

```text
apiserver.crt
```

cho kubelet trong TLS handshake.

API server **không gửi private key**.

---

# 2. Kubelet dùng cái gì để kiểm tra `apiserver.crt`?

Kubelet cần public certificate của cluster CA:

```text
ca.crt
```

CA đó đã ký `apiserver.crt`.

Trong kubeconfig của kubelet thường có:

```yaml
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTi...
    server: https://k8s-api:6443
```

Hoặc có thể tham chiếu file:

```yaml
certificate-authority: /etc/kubernetes/pki/ca.crt
```

Trong cluster kubeadm, kubelet thường đọc:

```text
/etc/kubernetes/kubelet.conf
```

Và bên trong file này có `certificate-authority-data` được nhúng trực tiếp bằng base64.

Do đó trên worker, kubelet có thể không cần đọc trực tiếp:

```text
/etc/kubernetes/pki/ca.crt
```

mỗi lần kết nối, vì CA đã được nhúng trong `kubelet.conf`.

Bạn kiểm tra bằng:

```bash
sudo grep -A6 'clusters:' /etc/kubernetes/kubelet.conf
```

Có thể thấy:

```yaml
clusters:
- cluster:
    certificate-authority-data: ...
    server: https://k8s-api:6443
```

Kubernetes ghi nhận rằng trong TLS bootstrap, sau khi xác định được cluster information, kubeadm lưu `ca.crt` và tạo `kubelet.conf` để kubelet sử dụng. ([Kubernetes](https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/ "Implementation details | Kubernetes"))

## Không phải `/var/lib/kubelet/pki` trên control plane

Bạn hỏi:

> Cert API server gửi có nằm ở `/var/lib/kubelet/pki/` trên control plane không?

**Không.**

Certificate API server nằm ở:

```text
/etc/kubernetes/pki/apiserver.crt
```

trên control-plane node.

Còn:

```text
/var/lib/kubelet/pki/
```

là thư mục certificate của **kubelet trên chính node đó**.

Mỗi node, kể cả worker và control-plane node, đều có kubelet riêng:

```text
worker wk1:
/var/lib/kubelet/pki/

control-plane cp1:
/var/lib/kubelet/pki/
```

Thư mục trên cp1 thuộc về kubelet cp1, không phải kube-apiserver.

---

# 3. Kubelet xác minh API server như thế nào?

Kubelet đọc từ `/etc/kubernetes/kubelet.conf`:

```yaml
server: https://k8s-api:6443
certificate-authority-data: ...
```

Sau đó:

```text
API server gửi apiserver.crt
             │
             ▼
kubelet kiểm tra:
- cert có được cluster CA ký không?
- cert còn hạn không?
- DNS/IP k8s-api có nằm trong SAN không?
- chữ ký có hợp lệ không?
```

Luồng:

```text
API server
  server cert:
  /etc/kubernetes/pki/apiserver.crt
        │
        │ gửi public cert
        ▼
kubelet
  kiểm tra bằng CA lấy từ:
  /etc/kubernetes/kubelet.conf
```

Kubernetes phân loại rõ `apiserver.crt` là server certificate của API server và kubelet có client certificate riêng để xác thực với API server. ([Kubernetes](https://kubernetes.io/docs/setup/best-practices/certificates/ "PKI certificates and requirements | Kubernetes"))

---

# 4. Sau đó kubelet chứng minh danh tính với API server

Đây là chiều xác thực ngược lại trong cùng TLS connection.

Kubelet đọc client credential từ:

```text
/etc/kubernetes/kubelet.conf
```

Ví dụ:

```yaml
users:
- name: system:node:fdp-k8s-wk1
  user:
    client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
    client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
```

Ở kubeadm, cùng một PEM file có thể chứa:

```text
client certificate
private key
```

nên cả `client-certificate` và `client-key` cùng trỏ tới:

```text
/var/lib/kubelet/pki/kubelet-client-current.pem
```

Kubernetes xác nhận kubeadm mặc định sử dụng symlink này trong `/etc/kubernetes/kubelet.conf` để hỗ trợ rotation. ([Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/ "Troubleshooting kubeadm | Kubernetes"))

Luồng TLS:

```text
kubelet                              API server
   │                                     │
   │←── apiserver.crt ───────────────────│
   │                                     │
   │ kiểm tra bằng cluster CA            │
   │                                     │
   │── kubelet client certificate ──────→│
   │                                     │
   │                       API server kiểm tra
```

Nhưng chú ý:

```text
client certificate được gửi
private key không được gửi
```

Private key chỉ được kubelet sử dụng cục bộ để ký dữ liệu trong TLS handshake, chứng minh rằng kubelet thật sự sở hữu private key tương ứng.

---

# 5. API server kiểm tra client certificate của kubelet bằng gì?

API server có flag:

```text
--client-ca-file=/etc/kubernetes/pki/ca.crt
```

Trong:

```text
/etc/kubernetes/manifests/kube-apiserver.yaml
```

API server dùng public CA đó để kiểm tra client certificate do kubelet gửi.

Ví dụ certificate kubelet chứa:

```text
Subject:
  CN = system:node:fdp-k8s-wk1
  O  = system:nodes
```

Sau khi xác minh chữ ký certificate, API server nhận danh tính:

```text
username: system:node:fdp-k8s-wk1
groups:
- system:nodes
```

Sau đó mới tới authorization:

```text
Authentication:
Certificate này có hợp lệ không?
Danh tính là ai?

Authorization:
Danh tính này có được phép làm request này không?
```

Authorization có thể do:

```text
Node Authorizer
RBAC
```

xử lý tùy request.

Authorization luôn diễn ra sau authentication. ([Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/authorization/?utm_source=chatgpt.com "Authorization"))

---

# 6. API server không kiểm tra `kubelet.conf`

Đoạn bạn nói:

> API server sẽ check kubeconf và xem quyền rồi accept

Chỗ này không đúng.

`kubelet.conf` chỉ tồn tại cục bộ trên node và được kubelet đọc.

API server không nhìn thấy file này.

Nó chỉ nhận được qua mạng:

```text
- TLS ClientHello
- kubelet client certificate
- chữ ký chứng minh kubelet có private key
- HTTP request sau khi TLS hoàn thành
```

Nó không nhận:

```text
/etc/kubernetes/kubelet.conf
private key
toàn bộ kubeconfig
```

Có thể hình dung:

```text
/etc/kubernetes/kubelet.conf
           │
           │ kubelet tự đọc
           ▼
kubelet biết:
- kết nối server nào
- tin CA nào
- dùng client cert/key nào
           │
           ▼
TLS handshake với API server
```

Kubeconfig là chỉ dẫn cho client, không phải giấy tờ được gửi nguyên file cho server.

---

# 7. Client certificate có phải cái được gửi lúc bootstrap không?

Cần tách hai giai đoạn.

## Giai đoạn bootstrap ban đầu

Ban đầu node chưa có:

```text
kubelet-client-current.pem
```

Kubeadm tạo tạm:

```text
/etc/kubernetes/bootstrap-kubelet.conf
```

File này chứa:

```text
- API server address
- cluster CA
- bootstrap token
```

Kubelet dùng bootstrap token để gọi API server và tạo CSR.

Luồng:

```text
kubelet tạo local key pair
        │
        ├── private key: giữ tại node
        │
        └── public key: đưa vào CSR
                       │
                       ▼
            gửi CSR tới API server
```

CSR chứa public key, không chứa private key.

Kubernetes mô tả TLS bootstrap là dùng shared bootstrap token để tạm thời xác thực, sau đó gửi CSR cho một key pair được tạo cục bộ. ([Kubernetes](https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/ "Implementation details | Kubernetes"))

Sau khi CSR được approve và ký:

```text
certificate được trả lại kubelet
        │
        ▼
/var/lib/kubelet/pki/kubelet-client-<timestamp>.pem
        │
        ▼
kubelet-client-current.pem
```

Sau đó:

```text
bootstrap-kubelet.conf bị xóa
kubelet.conf được dùng lâu dài
```

## Giai đoạn hoạt động bình thường

Mỗi khi tạo TLS connection tới API server, kubelet sử dụng certificate này:

```text
/var/lib/kubelet/pki/kubelet-client-current.pem
```

Certificate được gửi trong TLS handshake.

Không nên nói:

> File này chính là cái được gửi lúc bootstrap.

Chính xác hơn:

> Trong bootstrap, kubelet gửi CSR chứa public key. Sau khi CSR được ký, kubelet nhận client certificate. Từ đó về sau, kubelet gửi client certificate đó trong các TLS handshake tới API server.

---

# 8. Tổng kết chiều kubelet → API server

File trên worker:

```text
/etc/kubernetes/kubelet.conf
/var/lib/kubelet/pki/kubelet-client-current.pem
```

File trên control plane:

```text
/etc/kubernetes/pki/apiserver.crt
/etc/kubernetes/pki/apiserver.key
/etc/kubernetes/pki/ca.crt
```

Luồng:

```text
KUBELET                                             API SERVER
worker node                                         control plane

/etc/kubernetes/kubelet.conf
  server: k8s-api:6443
  CA: cluster CA
  client cert/key
       │
       │ TCP/TLS
       ▼
                                             /etc/kubernetes/pki/apiserver.crt
                                             /etc/kubernetes/pki/apiserver.key

1. API server gửi apiserver.crt
2. Kubelet kiểm tra bằng CA trong kubelet.conf
3. Kubelet gửi client certificate
4. API server kiểm tra bằng /etc/kubernetes/pki/ca.crt
5. API server xác định system:node:<node-name>
6. API server chạy authorization
7. HTTPS request được xử lý
```

---

# 9. Chiều API server → kubelet

Bây giờ đảo vai:

```text
API server = TLS client
kubelet = TLS server
```

Luồng mạng:

```text
kube-apiserver
      │
      │ HTTPS
      ▼
worker-node-ip:10250
      │
      ▼
kubelet HTTPS server
```

Dùng cho:

```text
kubectl logs
kubectl exec
kubectl attach
kubectl port-forward
stats và một số kubelet APIs
```

Trong chiều này cần hai nhóm certificate khác hoàn toàn:

```text
1. Kubelet serving certificate
   Kubelet chứng minh danh tính server với API server

2. API server kubelet-client certificate
   API server chứng minh danh tính client với kubelet
```

---

# 10. Kubelet serving certificate nằm ở đâu?

Kubelet là HTTPS server trên `10250`, nên nó cần:

```text
serving certificate
serving private key
```

Thường nằm trên chính node chạy kubelet, dưới:

```text
/var/lib/kubelet/pki/
```

Trong cấu hình mặc định kubeadm, bạn có thể thấy:

```text
/var/lib/kubelet/pki/kubelet.crt
/var/lib/kubelet/pki/kubelet.key
```

Kiểm tra:

```bash
sudo ls -l /var/lib/kubelet/pki/
```

Và xem kubelet config:

```bash
sudo grep -E 'tlsCertFile|tlsPrivateKeyFile|serverTLSBootstrap' \
  /var/lib/kubelet/config.yaml
```

Các field liên quan là:

```yaml
tlsCertFile: /var/lib/kubelet/pki/kubelet.crt
tlsPrivateKeyFile: /var/lib/kubelet/pki/kubelet.key
```

Nếu không chỉ định serving certificate hợp lệ, kubelet có thể tự tạo self-signed serving certificate.

Đây là nguyên nhân tài liệu nói mặc định API server có thể không xác minh kubelet serving certificate.

Nếu bật:

```yaml
serverTLSBootstrap: true
```

kubelet có thể tạo CSR để xin serving certificate được CA ký. Tuy nhiên CSR serving certificate không được default signer tự động approve; cần administrator hoặc controller riêng approve. ([Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/?utm_source=chatgpt.com "Certificate Management with kubeadm"))

---

# 11. API server kiểm tra kubelet serving certificate bằng gì?

API server cần CA đã ký serving certificate của kubelet.

Flag:

```text
--kubelet-certificate-authority=/path/to/ca.crt
```

Trong cluster dùng chung cluster CA, có thể là:

```text
--kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt
```

Luồng:

```text
kubelet trên wk1
serving cert:
  /var/lib/kubelet/pki/kubelet.crt
       │
       │ gửi public cert
       ▼
API server
kiểm tra bằng:
  --kubelet-certificate-authority
```

API server kiểm tra:

```text
- certificate có được CA tin cậy ký không?
- certificate còn hạn không?
- Node IP hoặc hostname có nằm trong SAN không?
- signature có hợp lệ không?
```

Nếu API server không có:

```text
--kubelet-certificate-authority
```

thì nó có thể mở TLS connection nhưng bỏ qua việc xác minh danh tính server kubelet.

Nghĩa là:

```text
có mã hóa
nhưng không chắc đang nói chuyện đúng kubelet
```

Kubernetes khuyến nghị cung cấp CA cho API server để xác minh kubelet serving certificate. Kubelet server certificate là một certificate riêng trên mỗi node. ([Kubernetes](https://kubernetes.io/docs/setup/best-practices/certificates/ "PKI certificates and requirements | Kubernetes"))

---

# 12. API server dùng certificate nào để xác thực với kubelet?

API server là client trong chiều này.

Nó dùng:

```text
/etc/kubernetes/pki/apiserver-kubelet-client.crt
/etc/kubernetes/pki/apiserver-kubelet-client.key
```

Các flag trong:

```text
/etc/kubernetes/manifests/kube-apiserver.yaml
```

thường là:

```yaml
- --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
- --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
```

Kubernetes yêu cầu API server có client certificate/key để xác thực với kubelet server. ([Kubernetes](https://kubernetes.io/docs/setup/best-practices/certificates/ "PKI certificates and requirements | Kubernetes"))

Luồng:

```text
API server                                 kubelet
    │                                        │
    │←──── kubelet serving certificate ──────│
    │                                        │
    │ kiểm tra bằng                          │
    │ --kubelet-certificate-authority        │
    │                                        │
    │──── apiserver-kubelet-client.crt ─────→│
    │                                        │
    │                    kubelet kiểm tra cert
```

Private key:

```text
apiserver-kubelet-client.key
```

không được gửi.

API server chỉ sử dụng nó để chứng minh quyền sở hữu certificate.

---

# 13. Kubelet kiểm tra client certificate của API server bằng gì?

Kubelet cần một CA bundle dùng để xác minh các client gọi vào cổng `10250`.

Trong kubelet config thường có:

```yaml
authentication:
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
```

Hoặc tương đương flag cũ:

```text
--client-ca-file=/etc/kubernetes/pki/ca.crt
```

Kubelet dùng CA đó để xác minh:

```text
/etc/kubernetes/pki/apiserver-kubelet-client.crt
```

mà API server gửi.

Kubernetes mô tả cấu hình X.509 authentication cho kubelet như sau:

```text
kubelet:
  --client-ca-file

API server:
  --kubelet-client-certificate
  --kubelet-client-key
```

([Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/ "Kubelet authentication/authorization | Kubernetes"))

Chú ý tên flag rất dễ lẫn:

```text
API server:
--kubelet-certificate-authority
    dùng để kiểm tra server cert của kubelet

Kubelet:
--client-ca-file
    dùng để kiểm tra client cert của API server
```

---

# 14. Sau khi kubelet xác thực API server, ai kiểm tra quyền?

Giả sử kubelet nhận client certificate:

```text
apiserver-kubelet-client.crt
```

Certificate thường có Subject dạng:

```text
CN = kube-apiserver-kubelet-client
O  = system:masters
```

Kubelet xác thực xong sẽ có danh tính người gọi.

Sau đó kubelet chạy authorization.

Trong kubeadm thường dùng:

```yaml
authorization:
  mode: Webhook
```

Điểm thú vị là:

> Kubelet nhận request từ API server, nhưng để kiểm tra API server có quyền gọi endpoint đó hay không, kubelet có thể lại gọi API server qua chiều kubelet → API server.

Ví dụ:

```text
API server ── GET /containerLogs/... ──→ kubelet:10250
                                           │
                                           │ SubjectAccessReview
                                           ▼
                                      API server:6443
```

Kubelet gửi SubjectAccessReview:

```text
User kube-apiserver-kubelet-client
có quyền:
verb=get
resource=nodes
subresource=log
resourceName=fdp-k8s-wk1
không?
```

API server trả:

```text
allowed: true
```

Kubelet mới thực hiện request.

Kubernetes ghi rõ khi kubelet chạy authorization mode `Webhook`, kubelet gọi SubjectAccessReview API để xác định request có được phép hay không. ([Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/ "Kubelet authentication/authorization | Kubernetes"))

---

# 15. Toàn bộ chiều API server → kubelet

Files trên control plane:

```text
/etc/kubernetes/pki/apiserver-kubelet-client.crt
/etc/kubernetes/pki/apiserver-kubelet-client.key
/etc/kubernetes/pki/ca.crt
/etc/kubernetes/manifests/kube-apiserver.yaml
```

Files trên worker:

```text
/var/lib/kubelet/pki/kubelet.crt
/var/lib/kubelet/pki/kubelet.key
/var/lib/kubelet/config.yaml
/etc/kubernetes/kubelet.conf
/etc/kubernetes/pki/ca.crt
```

Luồng:

```text
API SERVER                                      KUBELET
TLS client                                      TLS server
control plane                                   worker

1. API server mở TCP đến worker-IP:10250
2. Kubelet gửi kubelet serving certificate
3. API server kiểm tra bằng --kubelet-certificate-authority
4. API server gửi apiserver-kubelet-client.crt
5. Kubelet kiểm tra bằng authentication.x509.clientCAFile
6. Kubelet xác định danh tính của API server
7. Kubelet authorization Webhook gọi SubjectAccessReview
8. Nếu allowed, kubelet xử lý logs/exec/port-forward...
```

---

# 16. Hai chiều đặt cạnh nhau

|Chiều|TLS client|TLS server|Server certificate|Client certificate|
|---|---|---|---|---|
|Kubelet → API server|Kubelet|API server|`apiserver.crt`|`kubelet-client-current.pem`|
|API server → kubelet|API server|Kubelet|`kubelet.crt` hoặc serving cert được bootstrap|`apiserver-kubelet-client.crt`|

## Chiều kubelet → API server

```text
API server chứng minh:
  /etc/kubernetes/pki/apiserver.crt

Kubelet kiểm tra bằng:
  CA trong /etc/kubernetes/kubelet.conf

Kubelet chứng minh:
  /var/lib/kubelet/pki/kubelet-client-current.pem

API server kiểm tra bằng:
  /etc/kubernetes/pki/ca.crt
  thông qua --client-ca-file
```

## Chiều API server → kubelet

```text
Kubelet chứng minh:
  /var/lib/kubelet/pki/kubelet.crt

API server kiểm tra bằng:
  --kubelet-certificate-authority

API server chứng minh:
  /etc/kubernetes/pki/apiserver-kubelet-client.crt

Kubelet kiểm tra bằng:
  authentication.x509.clientCAFile
```

---

# 17. Vì sao cần tới bốn certificate?

Vì một certificate thường có vai trò cụ thể:

```text
Server certificate:
“Tôi là server mà bạn muốn kết nối.”

Client certificate:
“Tôi là client được phép gọi bạn.”
```

Kubelet và API server đổi vai tùy chiều:

```text
Kubelet → API server:
API server là server
Kubelet là client

API server → Kubelet:
Kubelet là server
API server là client
```

Cho nên cần:

```text
API server serving cert
Kubelet client cert
Kubelet serving cert
API server kubelet-client cert
```

Không thể chỉ nói chung chung “cert của kubelet” vì kubelet có ít nhất hai vai trò certificate khác nhau:

```text
kubelet client certificate
kubelet serving certificate
```

Đây chính là chỗ dễ gây nhầm nhất.


```
/var/lib/kubelet/pki/kubelet.crt
/var/lib/kubelet/pki/kubelet.key
```

Bạn không thấy `tlsCertFile` và `tlsPrivateKeyFile` trong `config.yaml` vì **hai field đó đang để giá trị mặc định rỗng**.

Khi chúng không được cấu hình rõ, kubelet tự tạo một cặp certificate self-signed và lưu trong thư mục certificate mặc định. Kubernetes mô tả chính xác: nếu `tlsCertFile` và `tlsPrivateKeyFile` không được cung cấp, kubelet tạo certificate và key self-signed rồi lưu trong thư mục được chỉ định bởi `--cert-dir`.

# Trạng thái hiện tại của kubelet bạn

Bạn có:

```
/var/lib/kubelet/pki/├── kubelet-client-2026-06-23-22-51-03.pem├── kubelet-client-current.pem├── kubelet.crt└── kubelet.key
```

Vai trò từng file:

```
kubelet-client-current.pem→ client certificate + private key→ dùng khi kubelet chủ động kết nối API server:6443kubelet.crt→ server/serving certificate→ kubelet trình ra khi API server kết nối kubelet:10250kubelet.key→ private key của kubelet serving certificate
```

Tức là có hai cặp credential hoàn toàn khác nhau:

```
Kubelet → API server:    kubelet-client-current.pemAPI server → kubelet:    kubelet.crt + kubelet.key
```

# Tại sao không khai báo trong `config.yaml`?

Cấu hình về mặt logic hiện tại là:

```
tlsCertFile: ""tlsPrivateKeyFile: ""serverTLSBootstrap: false
```

Các field có giá trị mặc định nên khi kubeadm sinh file hoặc khi cấu hình được serialize, chúng có thể không xuất hiện.

Kubelet xử lý như sau:

```
tlsCertFile có được cấu hình không?        │        ├── Có        │    → dùng file được chỉ định        │        └── Không             → dùng cert-dir             → nếu chưa có cert/key thì tự sinh:                  kubelet.crt                  kubelet.key
```

Thư mục mặc định thường là:

```
/var/lib/kubelet/pki
```

Do đó dù không thấy:

```
tlsCertFile: /var/lib/kubelet/pki/kubelet.crttlsPrivateKeyFile: /var/lib/kubelet/pki/kubelet.key
```

kubelet vẫn đang dùng hai file đó theo cơ chế mặc định.