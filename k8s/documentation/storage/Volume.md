Ok, mình gom phần **Kubernetes Volumes** bạn gửi thành bản giải thích **siêu dễ hiểu + chi tiết**, kèm **đầy đủ ví dụ YAML** đúng như trong tài liệu. Cứ nhớ: _volume = một thư mục_ được gắn (mount) vào container; “thư mục” đó lấy dữ liệu từ đâu tùy **loại volume**.

---

# 1) Vì sao cần Volumes?

- **Persist dữ liệu**: file trong container là _tạm_; container crash/restart là mất. Volume giúp giữ dữ liệu qua restart.
    
- **Chia sẻ dữ liệu**: giữa các container trong **cùng Pod**, giữa **các Pod**, hay với **hệ thống bên ngoài** (NFS, iSCSI…).
    
- **Cấp config/secret** cho app qua file, hoặc _metadata_ của chính Pod (downward API).
    

---

# 2) Cách Volumes hoạt động (rất ngắn gọn)

- Khai báo volume tại **`.spec.volumes`** của Pod, và mount nó vào container tại **`.spec.containers[*].volumeMounts`**.
    
- **Ephemeral volumes**: sống cùng Pod (Pod xóa → volume mất).  
    **Persistent volumes**: sống độc lập với Pod (Pod xóa → volume vẫn còn).
    
- Volume **không mount lồng trong volume khác**; muốn dùng thư mục con → **`subPath`**.
    

---

# 3) Các loại Volume & ví dụ

## A. Cấp cấu hình/metadata (ephemeral)

### 3.1 `configMap`

- Mount dữ liệu cấu hình dạng file **readOnly**.
    
- Tạo ConfigMap trước, rồi mount vào Pod.
    

**Ví dụ (đã cho):**

yaml

CopyEdit

`apiVersion: v1 kind: Pod metadata:   name: configmap-pod spec:   containers:     - name: test       image: busybox:1.28       command: ['sh', '-c', 'echo "The app is running!" && tail -f /dev/null']       volumeMounts:         - name: config-vol           mountPath: /etc/config   volumes:     - name: config-vol       configMap:         name: log-config         items:           - key: log_level             path: log_level.conf`

**Lưu ý:** `ConfigMap` luôn readOnly; nếu mount bằng **subPath** thì **không cập nhật nóng** khi ConfigMap đổi.

---

### 3.2 `downwardAPI`

- Mount **thông tin của Pod/Container** (tên, namespace, resource limits…) thành file text.
    
- Nếu dùng **subPath** cũng không nhận update ngay khi giá trị thay đổi.
    

_(Ví dụ chi tiết thuộc phần “Expose Pod Information…”, nên ở đây chỉ nhắc cách dùng.)_

---

## B. Lưu tạm / chia sẻ trong Pod (ephemeral)

### 3.3 `emptyDir`

- Tạo ra khi Pod được schedule lên node, ban đầu **trống**; _xóa_ khi Pod bị remove khỏi node.
    
- Dùng làm **scratch/cache**, chia sẻ giữa containers.
    
- `medium: Memory` → dùng **tmpfs (RAM)**, nhanh nhưng **tính vào limit RAM**.
    

**Ví dụ `emptyDir` (đã cho):**

yaml

CopyEdit

`apiVersion: v1 kind: Pod metadata:   name: test-pd spec:   containers:   - image: registry.k8s.io/test-webserver     name: test-container     volumeMounts:     - mountPath: /cache       name: cache-volume   volumes:   - name: cache-volume     emptyDir:       sizeLimit: 500Mi`

**Ví dụ `emptyDir` dùng RAM (đã cho):**

yaml

CopyEdit

`apiVersion: v1 kind: Pod metadata:   name: test-pd spec:   containers:   - image: registry.k8s.io/test-webserver     name: test-container     volumeMounts:     - mountPath: /cache       name: cache-volume   volumes:   - name: cache-volume     emptyDir:       sizeLimit: 500Mi       medium: Memory`

---

## C. Tương tác trực tiếp với host (cẩn trọng)

### 3.4 `hostPath`

- Mount **file/thư mục** từ filesystem của **node** vào Pod.
    
- **Nguy cơ bảo mật cao** – chỉ dùng khi bắt buộc. Có thể chỉ định `type` để kiểm tra tồn tại/kiểu đường dẫn.
    

**Ví dụ `hostPath` Linux (đã cho):**

yaml

CopyEdit

`apiVersion: v1 kind: Pod metadata:   name: hostpath-example-linux spec:   os: { name: linux }   nodeSelector:     kubernetes.io/os: linux   containers:   - name: example-container     image: registry.k8s.io/test-webserver     volumeMounts:     - mountPath: /foo       name: example-volume       readOnly: true   volumes:   - name: example-volume     hostPath:       path: /data/foo       type: Directory`

**Ví dụ `hostPath` với `FileOrCreate` (đã cho):**

yaml

CopyEdit

`apiVersion: v1 kind: Pod metadata:   name: test-webserver spec:   os: { name: linux }   nodeSelector:     kubernetes.io/os: linux   containers:   - name: test-webserver     image: registry.k8s.io/test-webserver:latest     volumeMounts:     - mountPath: /var/local/aaa       name: mydir     - mountPath: /var/local/aaa/1.txt       name: myfile   volumes:   - name: mydir     hostPath:       path: /var/local/aaa       type: DirectoryOrCreate   - name: myfile     hostPath:       path: /var/local/aaa/1.txt       type: FileOrCreate`

---

## D. Dữ liệu tĩnh đóng gói theo image/artifact (ephemeral – v1.33 beta)

### 3.5 `image` (volume kiểu OCI object)

- Mount nội dung của **một OCI image/artifact** như một volume **read-only**.
    
- `pullPolicy`: `Always` / `IfNotPresent` / `Never`.
    
- Thường được mount **noexec** trên Linux.
    

**Ví dụ (đã cho):**

yaml

CopyEdit

`apiVersion: v1 kind: Pod metadata:   name: image-volume spec:   containers:   - name: shell     command: ["sleep", "infinity"]     image: debian     volumeMounts:     - name: volume       mountPath: /volume   volumes:   - name: volume     image:       reference: quay.io/crio/artifact:v2       pullPolicy: IfNotPresent`

---

## E. Lưu trữ bền vững / chia sẻ giữa Pods (persistent)

### 3.6 `persistentVolumeClaim`

- Cách Pod **gắn** một **PersistentVolume (PV)** thông qua **PVC** – không cần biết backend là gì.
    

_(Phần PV/PVC chi tiết nằm trong mục PersistentVolumes/PersistentVolumeClaims.)_

---

### 3.7 `local` (PV local – ràng buộc node)

- PV trỏ tới **đĩa/thư mục cục bộ** trên **một node**; cần `nodeAffinity`.
    
- Hiệu năng cao nhưng **phụ thuộc node** (node chết → volume inaccessible).
    

**Ví dụ PV local (đã cho):**

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolume metadata:   name: example-pv spec:   capacity:     storage: 100Gi   volumeMode: Filesystem   accessModes:   - ReadWriteOnce   persistentVolumeReclaimPolicy: Delete   storageClassName: local-storage   local:     path: /mnt/disks/ssd1   nodeAffinity:     required:       nodeSelectorTerms:       - matchExpressions:         - key: kubernetes.io/hostname           operator: In           values:           - example-node`

---

### 3.8 `nfs`

- Mount **NFS share** → _nhiều Pod cùng đọc/ghi_.
    
- Dữ liệu giữ nguyên khi Pod xóa.
    

**Ví dụ (đã cho):**

yaml

CopyEdit

`apiVersion: v1 kind: Pod metadata:   name: test-pd spec:   containers:   - image: registry.k8s.io/test-webserver     name: test-container     volumeMounts:     - mountPath: /my-nfs-data       name: test-volume   volumes:   - name: test-volume     nfs:       server: my-nfs-server.example.com       path: /my-nfs-volume       readOnly: true`

---

### 3.9 `iscsi`

- Mount iSCSI LUN (đã có sẵn trên server).
    
- **RO nhiều consumer** ok, nhưng **RW chỉ một**.
    

_(Ví dụ chi tiết nằm trong phần iSCSI example – ở đây nêu cách dùng.)_

---

## F. Gộp nhiều nguồn thành một

### 3.10 `projected`

- Gộp **configMap + secret + downwardAPI**… vào **một** đường dẫn mount.
    

---

## G. Các loại đã deprecated/removed → dùng **CSI driver** thay thế

- **awsElasticBlockStore**, **gcePersistentDisk**, **azureDisk**, **azureFile**, **cinder**, **vsphereVolume** → đã chuyển sang **CSI** tương ứng; in-tree driver cũ bị redirect/remove.
    
- **glusterfs**, **rbd**, **cephfs** (in-tree) → **đã remove**.
    
- **gitRepo** → deprecated (dùng initContainer + emptyDir).
    
- **portworxVolume** → đã có **Portworx CSI**.
    

---

# 4) subPath & subPathExpr (dùng thư mục con)

### 4.1 `subPath`

- Mount **một thư mục con** của volume vào đường dẫn trong container.
    

**Ví dụ LAMP (đã cho):**

yaml

CopyEdit

`apiVersion: v1 kind: Pod metadata:   name: my-lamp-site spec:     containers:     - name: mysql       image: mysql       env:       - name: MYSQL_ROOT_PASSWORD         value: "rootpasswd"       volumeMounts:       - mountPath: /var/lib/mysql         name: site-data         subPath: mysql     - name: php       image: php:7.0-apache       volumeMounts:       - mountPath: /var/www/html         name: site-data         subPath: html     volumes:     - name: site-data       persistentVolumeClaim:         claimName: my-lamp-site-data`

### 4.2 `subPathExpr`

- Dùng **biến môi trường** (downward API) để tạo đường dẫn con.
    

**Ví dụ (đã cho):**

yaml

CopyEdit

`apiVersion: v1 kind: Pod metadata:   name: pod1 spec:   containers:   - name: container1     env:     - name: POD_NAME       valueFrom:         fieldRef:           apiVersion: v1           fieldPath: metadata.name     image: busybox:1.28     command: [ "sh", "-c", "while [ true ]; do echo 'Hello'; sleep 10; done | tee -a /logs/hello.txt" ]     volumeMounts:     - name: workdir1       mountPath: /logs       subPathExpr: $(POD_NAME)   restartPolicy: Never   volumes:   - name: workdir1     hostPath:       path: /var/log/pods`

---

# 5) Mount propagation (nâng cao – cẩn trọng)

- Cho phép lan truyền “mount” giữa host ↔ container ↔ Pod khác.
    
    - `None` (mặc định) – không lan truyền.
        
    - `HostToContainer` – host mount gì thì container thấy.
        
    - `Bidirectional` – 2 chiều (chỉ dùng trong **privileged**; rủi ro cao).
        
- Khuyến nghị: chỉ dùng với **hostPath** hoặc **emptyDir (Memory)**.
    

---

# 6) Read-only & Recursive Read-only (RRO – v1.33 stable)

- `readOnly: true` làm mount **RO** _nhưng_ trên Linux **không đệ quy** (submount vẫn có thể RW).
    
- Bật đệ quy bằng `recursiveReadOnly`:
    
    - `Enabled` (yêu cầu: `readOnly: true`, `mountPropagation: None`, kernel ≥ 5.12, runtime hỗ trợ).
        
    - `IfPossible` (cố bật, không được thì tắt).
        

**Ví dụ RRO (đã cho):**

yaml

CopyEdit

`apiVersion: v1 kind: Pod metadata:   name: rro spec:   volumes:     - name: mnt       hostPath:         path: /mnt   containers:     - name: busybox       image: busybox       args: ["sleep", "infinity"]       volumeMounts:         - name: mnt           mountPath: /mnt-rro           readOnly: true           mountPropagation: None           recursiveReadOnly: Enabled         - name: mnt           mountPath: /mnt-ro           readOnly: true         - name: mnt           mountPath: /mnt-rw`

---

# 7) CSI & di trú từ in-tree

- **CSI** là chuẩn giúp vendors phát hành **driver lưu trữ** bên ngoài core K8s.
    
- **CSIMigration** tự động chuyển thao tác từ driver cũ (in-tree) sang **driver CSI** tương ứng (bạn cần **cài driver CSI**).
    
- Dùng CSI qua 3 cách:
    
    1. **PVC/PV** (thông dụng nhất)
        
    2. **Generic Ephemeral Volume**
        
    3. **CSI Ephemeral Volume** (nếu driver hỗ trợ)
        

---

# 8) Mẹo chọn nhanh volume cho use case

- **Config thường** → `configMap`; **bí mật** → `secret`.
    
- **Metadata Pod** → `downwardAPI`.
    
- **Tạm/Cache/Chia sẻ trong Pod** → `emptyDir` (RAM nếu cần nhanh).
    
- **Dữ liệu tĩnh đóng gói** → `image` volume.
    
- **Chia sẻ giữa nhiều Pod** → `nfs` (hoặc PV hỗ trợ RWX qua CSI).
    
- **Block/Persistent cloud/on-prem** → **PVC + CSI driver**.
    
- **Đụng host thật sự** → `hostPath` (hạn chế, ưu tiên PV `local` nếu được).
    
- **Muốn mount thư mục con** → `subPath`/`subPathExpr`.