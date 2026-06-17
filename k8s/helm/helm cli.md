## `helm list -a`

- Mặc định `helm list` chỉ hiển thị **release ở trạng thái “deployed”** (đang chạy).
    
- `-a` = `--all` → hiển thị **tất cả release** bất kể trạng thái, bao gồm:
    
    - `deployed` – đang chạy
        
    - `failed` – cài/upgrade lỗi
        
    - `pending` – đang chờ
        
    - `superseded` – release cũ đã bị thay thế bởi revision mới
        
    - `uninstalled` – đã gỡ bỏ nhưng còn giữ lịch sử (nếu bạn uninstall với cờ `--keep-history`)

## Khác với `helm list -A`

- `helm list -a` → tất cả trạng thái, **nhưng trong 1 namespace** (mặc định `default`, trừ khi bạn thêm `-n`).
    
- `helm list -A` → tất cả namespace, **chỉ release đang deployed**.
    
- Muốn xem tất cả release trong tất cả namespace và mọi trạng thái → kết hợp:
    
    `helm list -A -a`



helm create <tên-chart>
- `tên-chart` = tên bạn muốn đặt cho chart mới.
    
- Sau khi chạy, Helm sẽ tạo ra **một thư mục mới** có cấu trúc chuẩn của một Helm chart.

## 📂 Ví dụ

`helm create myapp`

Kết quả: thư mục `myapp/` sẽ được sinh ra với cấu trúc như sau:

myapp/
├── Chart.yaml          # metadata của chart (tên, version, appVersion, mô tả)
├── values.yaml         # file values mặc định (biến config)
├── charts/             # nơi để các chart phụ thuộc (dependencies)
├── templates/          # chứa các template YAML (Deployment, Service, Ingress…)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # định nghĩa helper template
│   └── NOTES.txt       # hướng dẫn in ra khi cài chart
└── .helmignore         # giống .gitignore, bỏ qua file không cần package


---

## 📝 Ý nghĩa

- `helm create` = **tạo “sườn” một chart mới** để bạn phát triển.
    
- Nó giúp bạn **khỏi phải viết tay** cấu trúc chart từ đầu.
    
- Chart sinh ra có sẵn template cho:
    
    - Deployment
        
    - Service
        
    - Ingress
        
    - ServiceAccount
        
    - Test Pod


helm install
## 🔑 Cú pháp cơ bản

`helm install <release-name> <chart> [flags]`

- `<release-name>`: tên bạn đặt cho **release** (bản cài đặt cụ thể từ chart).
    
- `<chart>`: đường dẫn tới chart local (`./mychart`), hoặc chart trong repo (`bitnami/nginx`).
    
- `[flags]`: tuỳ chọn (namespace, values file, set biến, …).



```bash
# Cài chart "nginx" từ repo Bitnami, đặt release name = my-nginx
helm install my-nginx bitnami/nginx

# Cài chart local trong thư mục ./myapp
helm install demo ./myapp

# Cài chart với values.yaml tùy chỉnh
helm install demo ./myapp -f custom-values.yaml

# Ghi đè trực tiếp giá trị
helm install demo ./myapp --set replicaCount=3 --set image.tag=1.19
```


## 📝 Ý nghĩa khi chạy

1. **Helm đọc chart**
    
    - Lấy `Chart.yaml` (metadata), `values.yaml` (giá trị mặc định), `templates/` (manifest template).
        
2. **Render manifest**
    
    - Helm thay giá trị từ `values.yaml` (hoặc file/--set bạn cung cấp) vào template.
        
    - Tạo ra YAML manifest Kubernetes “thuần” (Deployment, Service…).
        
3. **Apply vào cluster**
    
    - Helm gửi manifest này lên Kubernetes API Server (giống như `kubectl apply`).
        
    - Cluster sinh ra các resource thật.
        
4. **Lưu trạng thái release**
    
    - Helm lưu metadata release (tên, chart version, values…) vào **Secret** trong namespace, để sau này có thể upgrade/rollback/uninstall.
        

---

## 📊 Kết quả

Sau khi chạy `helm install`, bạn sẽ thấy:

- Release mới trong cluster:
    
    `helm ls -n <namespace>`
    
- Các resource thực sự chạy trong Kubernetes:
    
    `kubectl get all -n <namespace>`
    

---

## ⚡ Một số flag hay dùng

- `-n, --namespace <ns>` → cài vào namespace cụ thể
    
- `-f, --values <file>` → chỉ định file values.yaml riêng
    
- `--set key=value` → ghi đè giá trị nhanh
    
- `--dry-run` → render thử, không cài (giúp debug template)
    
- `--debug` → in thêm thông tin chi tiết khi cài


helm uninstall

## 🔑 Cú pháp

`helm uninstall <release-name> [flags]`

- `<release-name>`: tên release bạn muốn gỡ (chính là tên bạn đặt lúc `helm install`).
    
- `[flags]`: tuỳ chọn (namespace, keep-history, …).
    

---

## 📝 Ý nghĩa

- `helm uninstall` = gỡ bỏ toàn bộ **release** ra khỏi cluster.
    
- Nó sẽ:
    
    1. Xoá toàn bộ resource mà release đó đã tạo (Deployment, Service, Ingress, ConfigMap, Secret...).
        
    2. Xoá cả Secret lưu trạng thái release (metadata Helm).
        
    3. Nếu dùng cờ `--keep-history`, Helm sẽ chỉ đánh dấu release là **uninstalled**, chứ không xoá hẳn metadata → bạn có thể xem lại lịch sử bằng `helm list -a`.

📌 Ví dụ

```bash
# Gỡ release my-nginx trong namespace default
helm uninstall my-nginx

# Gỡ release demo trong namespace myns
helm uninstall demo -n myns

# Gỡ release nhưng giữ lại lịch sử (có thể rollback)
helm uninstall demo --keep-history
```

## 📊 Sau khi uninstall

- Release sẽ **biến mất khỏi danh sách**:
    
    `helm list -n myns`
    
- Nếu có dùng `--keep-history`, bạn vẫn thấy nó với trạng thái `uninstalled`:
    
    `helm list -a -n myns`
    
- Các resource Kubernetes (Pod, Service, …) được Helm tạo cũng sẽ bị xoá.
    

---

## ⚡ So sánh nhanh

- `kubectl delete -f manifest.yaml`: xoá resource theo file YAML, nhưng không quản lý “release”.
    
- `helm uninstall`: xoá toàn bộ resource thuộc về release **theo dõi bởi Helm**, rất gọn.
    

---

👉 Tóm gọn: `helm uninstall` = **gỡ release ra khỏi cluster** (xóa hết resource đã tạo, và cả metadata Helm, trừ khi bạn giữ lại bằng `--keep-history`).


helm upgrade

## 🔑 Cú pháp

`helm upgrade <release-name> <chart> [flags]`

- `<release-name>`: tên release đã tồn tại (được cài trước bằng `helm install`).
    
- `<chart>`: chart mới (hoặc cùng chart nhưng với values mới).
    
- `[flags]`: các tùy chọn như file values mới, ghi đè giá trị, namespace...
    

---

## 📝 Ý nghĩa

- `helm upgrade` dùng để **cập nhật release đã có** trong cluster.
    
- Có thể thay đổi:
    
    - **Chart version** (nâng cấp ứng dụng, ví dụ `bitnami/redis 14.5.0` → `15.0.0`).
        
    - **Values** (config khác, ví dụ đổi số replica từ 1 → 3, đổi image tag…).
        
    - **Template** (nếu bạn đã chỉnh sửa chart local).
        
- Helm sẽ:
    
    1. Render manifest mới từ chart + values mới.
        
    2. So sánh với manifest cũ.
        
    3. Apply sự khác biệt vào cluster (giống `kubectl apply`).
        
    4. Ghi thêm một bản revision mới vào history của release.
        

---

## 📌 Ví dụ

```bash
# Nâng cấp release "my-nginx" lên chart nginx version mới
helm upgrade my-nginx bitnami/nginx

# Nâng cấp release "demo" bằng file values mới
helm upgrade demo ./myapp -f values-prod.yaml

# Nâng cấp release và ghi đè số replica trực tiếp
helm upgrade demo ./myapp --set replicaCount=5

# Nâng cấp nhưng cho phép tạo mới nếu chưa có (install + upgrade)
helm upgrade --install demo ./myapp

```


## 📊 Theo dõi upgrade

- Kiểm tra lịch sử release:
    
    `helm history <release-name>`
    
    - Ví dụ:
    
	REVISION    UPDATED                     STATUS     CHART          APP VERSION
	1           2025-08-15 10:21:12 +07:00  superseded nginx-15.2.0   1.21.1
	2           2025-08-19 14:35:00 +07:00  deployed   nginx-15.3.0   1.21.3
    
- Nếu upgrade lỗi, bạn có thể rollback:
    
    `helm rollback <release-name> <revision>`
    

---

## ⚡ Một số flag hay

- `-f, --values <file>` → chỉ định file values mới.
    
- `--set key=value` → ghi đè giá trị nhanh.
    
- `--reuse-values` → giữ nguyên values cũ, chỉ áp dụng cái mới ghi đè.
    
- `--install` → nếu release chưa có thì tự động install (hay dùng trong CI/CD).
    
- `--dry-run --debug` → render thử manifest, không apply.
    

---

👉 Tóm gọn: `helm upgrade` = **cập nhật một release bằng chart/values mới, lưu lịch sử revision để rollback khi cần**.


helm rollback

Ok 👌, tiếp theo là **`helm rollback`** – lệnh “quay ngược” release về một revision trước đó.

---

## 🔑 Cú pháp

`helm rollback <release-name> <revision> [flags]`

- `<release-name>`: tên release bạn muốn rollback.
    
- `<revision>`: số revision trong lịch sử release mà bạn muốn quay lại.
    
- `[flags]`: tùy chọn thêm (namespace, wait, cleanup-on-fail...).
    

---

## 📝 Ý nghĩa

- Mỗi lần bạn `helm install` hoặc `helm upgrade`, Helm sẽ tạo một **revision mới** (giống version history).
    
- `helm rollback` sẽ:
    
    1. Lấy manifest + values của revision cũ.
        
    2. Apply lại chúng lên cluster.
        
    3. Sinh ra một revision mới (ví dụ bạn rollback từ revision 5 → 3, thì Helm sẽ tạo revision 6 giống revision 3).
        

⚠️ Tức là rollback **không xóa lịch sử**, mà chỉ thêm một bản mới “copy” của revision cũ.

---

## 📌 Ví dụ

### 1. Xem lịch sử

`helm history my-nginx`

Output:
``` yaml
REVISION    UPDATED                     STATUS      CHART          APP VERSION    DESCRIPTION
1           2025-08-15 10:21:12 +07:00  superseded  nginx-15.2.0   1.21.1         Install complete
2           2025-08-19 14:35:00 +07:00  deployed    nginx-15.3.0   1.21.3         Upgrade complete
3           2025-08-19 15:10:20 +07:00  deployed    nginx-15.4.0   1.22.0         Upgrade complete
```

### 2. Rollback về revision 1

`helm rollback my-nginx 1`

→ Helm sẽ dùng manifest của revision 1 để apply, và tạo thêm revision 4 (có nội dung giống revision 1).

---

## ⚡ Một số flag hay

- `--wait` → chờ cho đến khi resource triển khai thành công rồi mới báo hoàn tất.
    
- `--cleanup-on-fail` → nếu rollback lỗi, sẽ tự động dọn dẹp.
    
- `-n <namespace>` → rollback release trong namespace cụ thể.
    

---

## 🎯 Khi nào dùng rollback?

- Upgrade chart bị lỗi (Pod crash, config sai).
    
- Muốn nhanh chóng trở lại trạng thái ổn định trước đó.
    

---

👉 Tóm gọn: `helm rollback` = **quay release về một revision cũ (dùng lại manifest + values trước đó), và ghi thêm một revision mới trong lịch sử**.


helm install --debug --dry-run
## 🔑 Cú pháp

`helm install <release-name> <chart> --debug --dry-run [flags khác]`

---

## 📝 Ý nghĩa từng flag

- **`--dry-run`**
    
    - Chạy thử, **không apply gì vào cluster**.
        
    - Helm sẽ render chart → YAML manifest → in ra màn hình, nhưng không tạo release trong cluster.
        
    - Dùng để xem “nếu mình install thật thì Helm sẽ tạo những resource nào”.
        
- **`--debug`**
    
    - In thêm nhiều thông tin chi tiết:
        
        - log render template
            
        - giá trị cuối cùng sau khi merge `values.yaml`, `-f`, `--set`
            
        - lỗi khi parse template
            

Kết hợp `--dry-run --debug` = **render chart và xem rõ ràng mọi thứ Helm sẽ làm**, nhưng không động chạm gì tới cluster.

---

## 📌 Ví dụ

`helm install demo ./mychart --debug --dry-run`

Output (rút gọn):

```yaml 
install.go:178: [debug] Original chart version: ""
install.go:195: [debug] CHART PATH: /home/user/mychart

NAME: demo
LAST DEPLOYED: Tue Aug 19 15:00:00 2025
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: mychart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-mychart
...
---
# Source: mychart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-mychart
...

```

- Bạn thấy toàn bộ manifest YAML sẽ được tạo ra.
    
- Có thể copy dán manifest này chạy trực tiếp với `kubectl apply -f -` để test.
    

---

## ⚡ Khi nào nên dùng?

- Khi viết/chỉnh sửa chart → để test template có render đúng không.
    
- Khi muốn debug giá trị thật sự đang đổ vào template (xem sau merge values).
    
- Khi chuẩn bị cài ở môi trường production → kiểm tra trước Helm sẽ apply gì.
    

---

👉 Tóm gọn:  
`helm install --debug --dry-run` = **giả lập quá trình cài đặt chart, in toàn bộ YAML manifest và log chi tiết ra màn hình, nhưng không deploy vào cluster**.


helm template
lệnh `helm template` chính là “chế độ render chart ra YAML thuần” mà không cài đặt gì vào cluster.

---

## 🔑 Cú pháp

`helm template <release-name> <chart> [flags]`

- `<release-name>`: tên release giả định (sẽ gắn vào labels/metadata trong YAML).
    
- `<chart>`: chart local (`./mychart`) hoặc từ repo (`bitnami/nginx`).
    
- `[flags]`: file values, --set, namespace…
    

---

## 📝 Ý nghĩa

- `helm template` = render chart → sinh ra manifest YAML Kubernetes.
    
- **Khác với `helm install --dry-run`:**
    
    - `helm template` không hề kết nối tới cluster.
        
    - Chỉ render YAML rồi in ra stdout (hoặc ghi file).
        
- Dùng khi bạn muốn:
    
    - Xem chart sinh ra YAML gì.
        
    - Export YAML để deploy thủ công bằng `kubectl apply`.
        
    - Kết hợp Helm chart vào GitOps/CI mà không cài Helm trực tiếp trong cluster.
        

---

## 📌 Ví dụ

```bash
# Render chart local ra YAML
helm template demo ./mychart

# Render chart với values tùy chỉnh
helm template demo ./mychart -f values-prod.yaml

# Ghi YAML ra file
helm template demo ./mychart -f values-prod.yaml > output.yaml

# Render chart trong repo
helm template my-nginx bitnami/nginx --set replicaCount=3

```

Output:

```yaml
---
# Source: mychart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-mychart
  labels:
    app.kubernetes.io/instance: demo
spec:
  type: ClusterIP
  ports:
  - port: 80
---
# Source: mychart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-mychart
...

```

``

---

## ⚡ Khi nào nên dùng `helm template`

- Muốn kiểm tra chart sinh ra YAML gì trước khi deploy.
    
- Muốn commit YAML thuần vào GitOps repo (ArgoCD, Flux…).
    
- Muốn apply YAML bằng `kubectl` mà không dùng Helm trong môi trường target.
    

---

👉 Tóm gọn:  
`helm template` = **biến chart Helm thành YAML Kubernetes thuần**, không cần kết nối cluster, rất tiện cho debug hoặc GitOps.



helm lint
## 🔑 Cú pháp

`helm lint <chart-path> [flags]`

- `<chart-path>`: đường dẫn tới chart local (ví dụ `./mychart`).
    
- `[flags]`: các tùy chọn, như chỉ định values file (`-f`).
    

---

## 📝 Ý nghĩa

- `helm lint` = **kiểm tra chất lượng chart** (giống như “lint code”).
    
- Nó sẽ:
    
    1. Đọc `Chart.yaml` → kiểm tra metadata có đủ field bắt buộc không (name, version...).
        
    2. Đọc `values.yaml` → xem có lỗi YAML hay không.
        
    3. Đọc các template trong `templates/` → render thử với values mặc định.
        
    4. Báo lỗi nếu có:
        
        - Thiếu field bắt buộc.
            
        - YAML invalid.
            
        - Template render lỗi (sai cú pháp Go template).
            
        - Values không hợp lệ.
            

⚠️ Lưu ý: `helm lint` **không kết nối tới cluster**. Nó chỉ kiểm tra chart ở local.

---

## 📌 Ví dụ

`helm lint ./mychart`

Kết quả khi chart OK:

`==> Linting ./mychart [INFO] Chart.yaml: icon is recommended 1 chart(s) linted, no failures`

Khi có lỗi:

`==> Linting ./mychart [ERROR] templates/deployment.yaml: unable to parse YAML         error converting YAML to JSON: yaml: line 10: mapping values are not allowed here Error: 1 chart(s) linted, 1 chart(s) failed`

---

## ⚡ Một số flag hay

- `-f, --values <file>` → test chart với values file khác.
    
- `--set key=value` → ghi đè giá trị nhanh để thử.
    

Ví dụ:

`helm lint ./mychart -f custom-values.yaml`

---

## 🎯 Khi nào dùng `helm lint`

- Trước khi `helm install` để chắc chắn chart không bị lỗi cú pháp.
    
- Trong CI/CD pipeline để tự động validate chart trước khi publish.
    
- Khi bạn phát triển chart mới (`helm create`) và đang chỉnh sửa template.
    

---

👉 Tóm gọn: `helm lint` = **công cụ kiểm tra chart Helm tại local, giúp phát hiện lỗi sớm trước khi deploy**.