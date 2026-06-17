# 1) Cặp đôi PV & PVC (nhà kho & phiếu kho)

- **PV (PersistentVolume)**: kho thật, do admin tạo hoặc hệ thống tự tạo. Sống độc lập với Pod.
    
- **PVC (PersistentVolumeClaim)**: phiếu yêu cầu kho (dung lượng + cách truy cập).
    

**Pod** không chạm PV trực tiếp → Pod chỉ “cầm” **PVC**, còn PVC “chỉ vào” **PV** đã ghép.

### Ví dụ: Pod dùng PVC

yaml

CopyEdit

`apiVersion: v1 kind: Pod metadata:   name: web spec:   containers:     - name: app       image: nginx       volumeMounts:         - name: data           mountPath: /var/www/html   # Thư mục trong container   volumes:     - name: data       persistentVolumeClaim:         claimName: myclaim           # PVC đã tạo`

---

# 2) Hai kiểu tạo PV: Static & Dynamic

## 2.1 Static (admin tự tạo kho trước)

Admin tạo PV thủ công, dev chỉ cần tạo PVC trỏ vào.

**PV (ví dụ NFS):**

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolume metadata:   name: pv-nfs-10gi spec:   capacity:     storage: 10Gi   accessModes: [ReadWriteMany]      # NFS hỗ trợ RWX   storageClassName: nfs-sc   nfs:     server: 10.0.0.10     path: /export/app-data   persistentVolumeReclaimPolicy: Retain`

**PVC:**

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolumeClaim metadata:   name: myclaim spec:   accessModes: [ReadWriteMany]   resources:     requests:       storage: 10Gi   storageClassName: nfs-sc`

**Khi nào dùng**: có sẵn NFS/ổ đĩa cụ thể, muốn kiểm soát “kho nào cho ai”.

---

## 2.2 Dynamic (hệ thống tự tạo kho theo StorageClass)

Bạn chỉ cần làm PVC + chỉ ra `storageClassName`; controller sẽ tự tạo PV phù hợp.

**StorageClass (ví dụ SSD mặc định + cho phép mở rộng):**

yaml

CopyEdit

`apiVersion: storage.k8s.io/v1 kind: StorageClass metadata:   name: premium-ssd   annotations:     storageclass.kubernetes.io/is-default-class: "true"  # mặc định provisioner: csi.example.cloud.com                       # CSI driver allowVolumeExpansion: true reclaimPolicy: Delete                                    # xóa cả kho khi xóa PVC parameters:   type: ssd`

**PVC (không chỉ class → sẽ dùng default `premium-ssd`):**

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolumeClaim metadata:   name: db-data spec:   accessModes: [ReadWriteOnce]    # hầu hết block storage chỉ RWO   resources:     requests:       storage: 20Gi   # storageClassName: premium-ssd  # có thể để trống để dùng default`

**Khi nào dùng**: 99% trường hợp trong cloud/K8s production.

> Lưu ý: đặt `storageClassName: ""` (chuỗi rỗng) sẽ **tắt** dynamic provisioning cho PVC đó.

---

# 3) AccessMode (quyền gắn kho) – hiểu thật nhanh

- **RWO (ReadWriteOnce)**: 1 **node** gắn đọc/ghi (nhiều Pod cùng node vẫn dùng chung được).
    
- **ROX (ReadOnlyMany)**: nhiều node gắn **chỉ đọc**.
    
- **RWX (ReadWriteMany)**: nhiều node gắn đọc/ghi (cần NFS/FS dùng chung hoặc CSI hỗ trợ RWX).
    
- **RWOP (ReadWriteOncePod)**: đúng 1 **Pod** được gắn đọc/ghi (cứng hơn RWO).
    

**Mẹo nhớ**: block storage (EBS/GPD/Azure Disk…) thường **RWO**; muốn **RWX** thường dùng **NFS**/file storage/CSI hỗ trợ RWX.

---

# 4) VolumeMode (Filesystem vs Block)

- **Filesystem** (mặc định): K8s format (nếu trống) và mount như thư mục.
    
- **Block**: raw disk (/dev/xxx), app tự quản lý.
    

**Ví dụ dùng Block:**

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolumeClaim metadata:   name: block-pvc spec:   accessModes: [ReadWriteOnce]   volumeMode: Block   resources:     requests:       storage: 10Gi --- apiVersion: v1 kind: Pod metadata:   name: uses-block spec:   containers:     - name: app       image: alpine       command: ["sh","-c","sleep 3600"]       volumeDevices:         - name: data              # LƯU Ý: volumeDevices (không phải volumeMounts)           devicePath: /dev/xvda   volumes:     - name: data       persistentVolumeClaim:         claimName: block-pvc`

---

# 5) Reclaim Policy (khi xóa PVC thì kho làm gì?)

- **Delete**: xóa cả PV và “kho” trên hạ tầng. (mặc định cho dynamic)
    
- **Retain**: giữ lại dữ liệu (cần admin dọn tay trước khi cấp lại).
    
- **Recycle**: _đã deprecated_.
    

**Tình huống với Retain – tái sử dụng kho cũ an toàn:**

1. Đổi reclaimPolicy PV thành `Retain`.
    
2. Xóa PVC cũ → PV vào trạng thái `Released` (dữ liệu còn).
    
3. Admin xóa `spec.claimRef` của PV → PV thành `Available`.
    
4. Tạo PVC mới **chỉ định** đúng `volumeName` của PV cũ → bind lại.
    

---

# 6) Bảo vệ đang sử dụng (finalizers/protection)

- PVC đang được Pod dùng → xóa sẽ bị giữ ở trạng thái **Terminating** với finalizer `kubernetes.io/pvc-protection`.
    
- PV đang bound → cũng không xóa ngay (`kubernetes.io/pv-protection`).
    
- Với **Delete** reclaim policy, còn có finalizer của provisioner để đảm bảo **xóa backend trước** rồi mới xóa PV.
    

---

# 7) Pre-bind (đặt cọc để ghép đúng kho mong muốn)

**PVC chỉ định PV cụ thể:**

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolumeClaim metadata:   name: foo-pvc spec:   storageClassName: ""   # tránh default class   volumeName: foo-pv     # chỉ định PV đích   accessModes: [ReadWriteOnce]   resources:     requests:       storage: 5Gi`

**PV “đặt chỗ” cho PVC cụ thể:**

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolume metadata:   name: foo-pv spec:   storageClassName: ""   capacity: { storage: 5Gi }   accessModes: [ReadWriteOnce]   claimRef:     name: foo-pvc     namespace: default   hostPath:     path: /tmp/data   # ví dụ single-node test`

> Dùng khi muốn **chắc chắn** PVC đó phải dính đúng PV (vd: kho cũ `Retain`).

---

# 8) Mở rộng dung lượng (Resize PVC)

Điều kiện: `allowVolumeExpansion: true` trong StorageClass **và** driver hỗ trợ.

**Tăng size**: chỉ cần `kubectl edit pvc myclaim` và tăng `spec.resources.requests.storage` (ví dụ 10Gi → 20Gi).

- Với **Filesystem**, việc nới FS thường diễn ra khi Pod khởi động lại hoặc online nếu FS hỗ trợ (Ext4/XFS).
    
- **Không** chỉnh trực tiếp `PV.spec.capacity` (sẽ làm controller nghĩ đã tự tăng rồi → không resize).
    

---

# 9) Mount options & Node affinity

**Mount options (ví dụ NFS):**

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolume metadata:   name: pv-nfs-opts spec:   capacity: { storage: 5Gi }   accessModes: [ReadWriteMany]   storageClassName: nfs-sc   mountOptions:     - nfsvers=4.1     - hard   nfs:     server: 10.0.0.10     path: /export/opt`

**Node affinity** (thường dùng với `local` volume):

yaml

CopyEdit

`spec:   nodeAffinity:     required:       nodeSelectorTerms:         - matchExpressions:           - key: kubernetes.io/hostname             operator: In             values: ["node-12"]`

→ Pod dùng PV này chỉ được schedule lên **node-12**.

---

# 10) Snapshot, Clone, Populator (nhanh gọn)

- **Snapshot**: chụp ảnh volume (chỉ với CSI). Tạo PVC mới từ ảnh:
    

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolumeClaim metadata:   name: restore-pvc spec:   storageClassName: csi-hostpath-sc   dataSource:     kind: VolumeSnapshot     name: snapshot-2025-08-01     apiGroup: snapshot.storage.k8s.io   accessModes: [ReadWriteOnce]   resources:     requests: { storage: 10Gi }`

- **Clone**: tạo PVC mới từ **PVC cũ** (CSI):
    

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolumeClaim metadata:   name: cloned-pvc spec:   storageClassName: my-csi   dataSource:     kind: PersistentVolumeClaim     name: src-pvc   accessModes: [ReadWriteOnce]   resources:     requests: { storage: 10Gi }`

- **Populator**: tạo volume có sẵn dữ liệu từ **CRD** tùy chỉnh (dùng `dataSourceRef`).
    

---

# 11) Namespaces & “Many” mode

- **PV** là **cluster-scoped** (không thuộc namespace).
    
- **PVC** là **namespaced**.
    
- RWX/ROX cho **nhiều Pod** nhưng vẫn **trong cùng namespace** của PVC (muốn chia NS → dùng cơ chế khác, vd: ReferenceGrant, nhưng đừng đụng nếu chưa cần).
    

---

# 12) Khi nào **không cần** PV/PVC?

- Dữ liệu tạm, **mất không sao** → dùng `emptyDir`:
    

yaml

CopyEdit

`volumes:   - name: cache     emptyDir: {}`

- Container chỉ cần mount **ConfigMap/Secret** (file cấu hình), không phải dữ liệu runtime.
    

> Nếu bạn nói “Redis của mình chỉ TTL, restart mất cũng được”, thì **không cần PV** (đừng bật AOF). Ngược lại, cần **giữ** dữ liệu qua restart → dùng PVC + AOF/RDB.

---

# 13) Debug nhanh (PVC/PV Pending, Pod không mount được)

1. **PVC Pending** quá lâu:
    
    - Thiếu **StorageClass** hoặc sai `storageClassName`.
        
    - Yêu cầu **RWX** nhưng class chỉ hỗ trợ **RWO**.
        
    - Kích thước yêu cầu (vd 200Gi) > quota/không có.
        
    - Dùng **selector** label nhưng không có PV khớp.
        
2. **Pod Pending** / **ContainerCrashLoop** do mount fail:
    
    - Driver/CSI chưa chạy, node không hỗ trợ.
        
    - Với `local`/`hostPath`, Pod bị schedule sai node (không khớp nodeAffinity).
        
    - NFS mount options sai → mount lỗi.
        

**Lệnh hữu ích:**

bash

CopyEdit

`kubectl get sc kubectl get pvc -A kubectl describe pvc myclaim kubectl get pv kubectl describe pv <pv-name> kubectl describe pod web kubectl get events --sort-by=.lastTimestamp`

---

# 14) Bộ “công thức” ngắn gọn cho 3 nhu cầu phổ biến

## A) App chỉ 1 replica, lưu DB/volume cục bộ (RWO)

- PVC: `accessModes: [ReadWriteOnce]`
    
- StorageClass: block storage (mặc định), `allowVolumeExpansion: true`, `reclaimPolicy: Delete`
    
- Pod: mount PVC vào `/var/lib/...`
    

## B) Nhiều Pod trên nhiều node cùng ghi đọc (RWX)

- Dùng NFS/FS share hoặc CSI hỗ trợ **RWX**.
    
- PVC: `accessModes: [ReadWriteMany]`
    
- StorageClass: loại hỗ trợ RWX.
    

## C) Cần dữ liệu tồn tại sau khi xóa PVC (backup thủ công)

- PV/StorageClass: `reclaimPolicy: Retain`
    
- Khi xoá PVC: làm quy trình “Retain → remove claimRef → pre-bind PVC mới”.
    

---

# 15) Một ví dụ **trọn bộ** (Dynamic + mở rộng + Delete)

**StorageClass (mặc định, cho phép resize):**

yaml

CopyEdit

`apiVersion: storage.k8s.io/v1 kind: StorageClass metadata:   name: standard-ssd   annotations:     storageclass.kubernetes.io/is-default-class: "true" provisioner: csi.example.cloud.com allowVolumeExpansion: true reclaimPolicy: Delete`

**PVC (20Gi RWO):**

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolumeClaim metadata:   name: app-data spec:   accessModes: [ReadWriteOnce]   resources:     requests:       storage: 20Gi   # storageClassName: standard-ssd   # để trống sẽ dùng default`

**Deployment dùng PVC:**

yaml

CopyEdit

`apiVersion: apps/v1 kind: Deployment metadata:   name: app spec:   replicas: 2   selector: { matchLabels: { app: app } }   template:     metadata: { labels: { app: app } }     spec:       containers:         - name: api           image: nginx           volumeMounts:             - name: data               mountPath: /data       volumes:         - name: data           persistentVolumeClaim:             claimName: app-data`

> Nhớ: RWO → tại một thời điểm, volume chỉ attach được vào **một node**. Với 2 replica, scheduler sẽ cố đặt cả hai replica lên **cùng node** để cùng đọc/ghi. Nếu HPA scale sang node khác, PVC RWO có thể làm replica kia Pending (đó là bản chất RWO). Muốn scale tự do nhiều node → hãy dùng **RWX**.

**Resize lên 50Gi**:

bash

CopyEdit

`kubectl edit pvc app-data # đổi storage: 50Gi # chờ controller + (có thể) restart Pod để FS nới kích thước`

---

# 16) Checklist chọn nhanh

- **Cần chia sẻ giữa nhiều node?** → RWX (NFS/CSI RWX).
    
- **Chỉ 1 node?** → RWO (block storage nhanh & rẻ).
    
- **Dữ liệu quan trọng?** → ReclaimPolicy = Delete (tự dọn) hoặc Retain (giữ để kiểm soát).
    
- **Cần tăng dung lượng sau này?** → StorageClass `allowVolumeExpansion: true`.
    
- **Dùng tạm/không cần lưu?** → `emptyDir` (khỏi PVC).

## 1. Bình thường ta dùng **`volumeMounts`**

- Dùng khi volume ở dạng **Filesystem**.
    
- Kubernetes sẽ **format** (nếu cần) và mount volume vào một **thư mục** trong container.
    
- Ví dụ:
    

yaml

CopyEdit

`volumeMounts:   - name: mydata     mountPath: /app/data  # Volume xuất hiện như thư mục /app/data`

→ App đọc/ghi file trong thư mục này như bình thường.

---

## 2. Nhưng nếu muốn dùng volume ở dạng **raw block device**

- Lúc này volume **không format**, không có filesystem.
    
- Bạn muốn container thấy nó như một **thiết bị /dev/sdX**.
    
- App trong container sẽ **tự xử lý block device** này (vd: format, đọc raw sector…).
    

Để làm điều đó → Kubernetes cung cấp **`volumeDevices`**.

---

## 3. Cách hoạt động của `volumeDevices`

- **Chỉ dùng khi `volumeMode: Block` trong PVC/PV**.
    
- Thay vì mount vào một **đường dẫn thư mục**, bạn chỉ định **`devicePath`** (vị trí xuất hiện trong container).
    
- Ví dụ:
    

yaml

CopyEdit

`volumeDevices:   - name: myblock     devicePath: /dev/xvda`

→ Trong container, sẽ có một file thiết bị `/dev/xvda` gắn trực tiếp vào storage backend.

---

## 4. Ví dụ trọn bộ

**PVC dạng Block:**

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolumeClaim metadata:   name: block-pvc spec:   accessModes: [ReadWriteOnce]   volumeMode: Block            # BẮT BUỘC để dùng volumeDevices   resources:     requests:       storage: 10Gi`

**Pod dùng block device:**

yaml

CopyEdit

`apiVersion: v1 kind: Pod metadata:   name: pod-with-block spec:   containers:     - name: app       image: ubuntu       command: ["sh", "-c", "ls -l /dev/xvda && sleep 3600"]       volumeDevices:         - name: myblock           devicePath: /dev/xvda   volumes:     - name: myblock       persistentVolumeClaim:         claimName: block-pvc`

Khi chạy:

- Trong container sẽ thấy `/dev/xvda` là một block device thật.
    
- Bạn có thể `fdisk`, `mkfs`, hoặc chạy database yêu cầu raw device.
    

---

## 5. Khác nhau giữa `volumeMounts` và `volumeDevices`

|Đặc điểm|volumeMounts|volumeDevices|
|---|---|---|
|VolumeMode yêu cầu|Filesystem (mặc định)|Block|
|Mount vào|Thư mục trong container|File thiết bị trong `/dev`|
|Kubernetes format volume?|Có thể (nếu trống)|Không|
|Ai quản lý FS?|Kubernetes|Ứng dụng trong container|
|Use case|Lưu file, log, config…|DB hoặc app cần raw device|


## 1. **Bảo vệ đang sử dụng** (Finalizers / Protection)

Mục đích: **tránh mất dữ liệu do lỡ tay xóa PV/PVC khi nó đang được dùng**.

### a) PVC đang được Pod dùng

- Nếu một **Pod** đang mount PVC đó, bạn gõ:
    

bash

CopyEdit

`kubectl delete pvc my-data`

- PVC **không bị xóa ngay**. Nó sẽ:
    
    - Vào trạng thái **`Terminating`**.
        
    - Trong spec sẽ có **`finalizers: [kubernetes.io/pvc-protection]`**.
        
- Kubernetes giữ lại cho đến khi **không còn Pod nào dùng** → mới xóa PVC.
    

**Lý do**: Nếu xóa ngay → Pod sẽ crash, dữ liệu có thể hỏng.

---

### b) PV đang bound

- Nếu PV đang **bound** với một PVC, bạn xóa PV:
    

bash

CopyEdit

`kubectl delete pv my-pv`

- PV **không biến mất ngay**:
    
    - Trạng thái **`Terminating`**.
        
    - Có **`finalizers: [kubernetes.io/pv-protection]`**.
        
- Chỉ khi **PVC đã release** (không bound nữa) → PV mới được xóa.
    

**Lý do**: Tránh mất volume backend khi PVC còn dùng.

---

### c) Với **Delete reclaimPolicy**

- Khi PVC bị xóa, PV sẽ bị xóa cùng (Delete policy).
    
- Nhưng trước khi xóa PV, **Kubernetes sẽ yêu cầu CSI driver hoặc provisioner** xóa volume ở backend (AWS EBS, GCP PD, NFS share…).
    
- Để đảm bảo điều này, PV có thêm finalizer **của provisioner**, ví dụ:
    
    - `external-provisioner.volume.kubernetes.io/finalizer` (CSI).
        
    - `kubernetes.io/pv-controller` (in-tree plugin).
        
- Chỉ khi backend xóa thành công → finalizer được gỡ → PV biến mất.


## 1. **Mount Options** – cách mount volume vào node

### Hiểu đơn giản

Khi Kubernetes gắn một PV vào node, nó sẽ chạy một lệnh mount (giống Linux `mount`), và bạn có thể thêm **tùy chọn mount** để điều chỉnh cách hoạt động.

### Tại sao cần?

- Điều chỉnh hiệu năng (vd: `noatime` để tránh ghi thời gian truy cập).
    
- Cấu hình giao thức (vd: NFS dùng phiên bản 4.1).
    
- Bật/tắt tính năng (vd: `ro` chỉ đọc).
    

### Ví dụ PV với mount options

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolume metadata:   name: pv-nfs spec:   capacity:     storage: 10Gi   accessModes:     - ReadWriteMany   storageClassName: nfs-sc   mountOptions:     - hard     - nfsvers=4.1     - noatime   nfs:     path: /export/data     server: 10.0.0.10`

Khi Pod dùng PVC trỏ đến PV này, Kubernetes sẽ mount NFS với tùy chọn:

ruby

CopyEdit

`mount -t nfs -o hard,nfsvers=4.1,noatime 10.0.0.10:/export/data /var/lib/kubelet/pods/...`

**Lưu ý**:

- Không phải loại volume nào cũng hỗ trợ mountOptions (vd: `hostPath` không hỗ trợ).
    
- Nếu viết sai option → mount sẽ thất bại, Pod không chạy được.
    

---

## 2. **Node Affinity** – giới hạn volume chỉ gắn được vào node cụ thể

### Hiểu đơn giản

Một số PV **chỉ tồn tại trên một node nhất định** (vd: `local` volume, `hostPath`, ổ SSD gắn trực tiếp vào node).  
→ Kubernetes cần biết **volume này chỉ dùng được trên node nào** để scheduler đặt Pod lên đúng node.

### Ví dụ PV với Node Affinity

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolume metadata:   name: local-pv spec:   capacity:     storage: 50Gi   accessModes:     - ReadWriteOnce   storageClassName: local-sc   local:     path: /mnt/disks/ssd1   nodeAffinity:     required:       nodeSelectorTerms:         - matchExpressions:             - key: kubernetes.io/hostname               operator: In               values:                 - node-1`

**Ý nghĩa**:

- PV này là ổ đĩa cục bộ `/mnt/disks/ssd1` **chỉ có ở node-1**.
    
- Nếu một PVC bind PV này → Pod đó sẽ **luôn được schedule** vào node-1.
    

---

## 3. So sánh nhanh

|Đặc điểm|Mount Options|Node Affinity|
|---|---|---|
|Tác dụng|Điều chỉnh cách mount volume|Giới hạn node có thể mount volume|
|Khi áp dụng|Lúc Kubernetes mount PV vào node|Lúc scheduler quyết định đặt Pod ở đâu|
|Ví dụ|`nfsvers=4.1`, `noatime`, `hard`|`hostname=node-1`|
|Liên quan đến|Driver / hệ thống file|Scheduling / Topology|




yaml

CopyEdit

`nfs:   server: 10.0.0.10   path: /export/app-data`

là đang khai báo **volume kiểu NFS** (Network File System) cho PersistentVolume trong Kubernetes.  
Mình giải thích từng ý nhé:

---

## 1. NFS là gì?

- **NFS (Network File System)** là một giao thức cho phép nhiều máy tính truy cập và chia sẻ file/thư mục qua mạng giống như file nằm trên máy local.
    
- Server NFS lưu trữ dữ liệu; các client (node Kubernetes) mount thư mục được chia sẻ này vào hệ thống file của mình.
    
- Ưu điểm: nhiều node đọc/ghi cùng lúc (**RWX**).
    
- Nhược điểm: phụ thuộc mạng → mạng chậm hoặc NFS server down thì ảnh hưởng ngay.
    

---

## 2. Ý nghĩa các trường

- `server: 10.0.0.10` → Địa chỉ IP của máy NFS server (nơi chứa dữ liệu).
    
- `path: /export/app-data` → Đường dẫn thư mục trên NFS server đang được chia sẻ (export).
    

Khi Pod dùng PVC bind tới PV này, Kubernetes sẽ mount NFS:

bash

CopyEdit

`mount -t nfs 10.0.0.10:/export/app-data /var/lib/kubelet/pods/...`

---

## 3. Ví dụ PV dùng NFS

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolume metadata:   name: pv-nfs spec:   capacity:     storage: 5Gi   accessModes:     - ReadWriteMany         # NFS hỗ trợ nhiều node đọc/ghi   persistentVolumeReclaimPolicy: Retain   nfs:     server: 10.0.0.10     path: /export/app-data`

PVC yêu cầu:

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolumeClaim metadata:   name: pvc-nfs spec:   accessModes:     - ReadWriteMany   resources:     requests:       storage: 5Gi`

Pod dùng:

yaml

CopyEdit

`volumes:   - name: data     persistentVolumeClaim:       claimName: pvc-nfs`

---

## 4. Khi nào nên dùng NFS trong Kubernetes

- Cần **RWX** (nhiều Pod trên nhiều node cùng đọc/ghi).
    
- Dữ liệu không yêu cầu hiệu năng cao như block storage.
    
- Muốn chia sẻ thư mục cấu hình, ảnh, file upload giữa nhiều service.


:

yaml

CopyEdit

`hostPath:   path: /tmp/data`

là đang khai báo **volume kiểu hostPath** trong PersistentVolume (hoặc trong Pod spec) của Kubernetes.

---

## 1. hostPath là gì?

- **hostPath** cho phép bạn dùng **một thư mục hoặc file trên ổ đĩa của node** (máy host) như một volume cho Pod.
    
- Nói cách khác: nó **gắn (mount)** đường dẫn trên máy host → vào container trong Pod.
    

---

## 2. Ý nghĩa của `path`

- `path: /tmp/data` nghĩa là Kubernetes sẽ lấy thư mục `/tmp/data` trên **node** nơi Pod chạy.
    
- Thư mục này sẽ được mount vào container theo `mountPath` bạn cấu hình trong `volumeMounts`.
    

---

## 3. Ví dụ Pod dùng hostPath

yaml

CopyEdit

`apiVersion: v1 kind: Pod metadata:   name: mypod spec:   containers:     - name: app       image: nginx       volumeMounts:         - name: myvol           mountPath: /usr/share/nginx/html   # mount vào trong container   volumes:     - name: myvol       hostPath:         path: /tmp/data   # đường dẫn trên node`

📌 Nếu node có `/tmp/data/index.html` → container sẽ thấy file này ở `/usr/share/nginx/html/index.html`.

---

## 4. Khi nào dùng hostPath?

- Test nhanh trên môi trường local (vd: Minikube, single-node).
    
- Dùng file đặc biệt của node (vd: `/var/run/docker.sock` để Pod tương tác Docker daemon).
    
- **Không nên dùng** trong môi trường multi-node production vì:
    
    - Mỗi node có filesystem riêng → Pod chạy trên node khác sẽ **không thấy dữ liệu**.
        
    - Dễ mất dữ liệu khi node bị xóa hoặc Pod di chuyển sang node khác.


## 1. `provisioner: csi.example.cloud.com` trong StorageClass

### a) StorageClass làm gì?

- **StorageClass** định nghĩa “cách” mà Kubernetes sẽ tạo (provision) một PersistentVolume mới khi PVC yêu cầu **dynamic provisioning**.
    
- Để dynamic provisioning hoạt động, Kubernetes phải biết **ai** sẽ tạo volume → cái này chính là **provisioner**.
    

### b) `provisioner` là gì?

- `provisioner` là **tên driver** (thường là CSI – Container Storage Interface) chịu trách nhiệm tạo, xóa, quản lý volume trên hạ tầng lưu trữ.
    
- Ví dụ:
    
    - AWS EBS CSI driver: `ebs.csi.aws.com`
        
    - GCP PD CSI driver: `pd.csi.storage.gke.io`
        
    - Azure Disk CSI driver: `disk.csi.azure.com`
        
    - NFS CSI driver: `nfs.csi.k8s.io`
        
- Trong doc bạn đọc, `csi.example.cloud.com` chỉ là **tên giả** để minh họa.
    
- Khi PVC tạo mới → Kubernetes gọi CSI driver tương ứng với `provisioner` này để tạo volume.
    

📌 Nếu viết sai hoặc driver không cài → PVC sẽ bị **Pending** mãi vì không có ai tạo PV.



**Container Storage Interface (CSI)** là một **chuẩn giao tiếp chung** giữa Kubernetes và các hệ thống lưu trữ bên ngoài (block storage, file storage, v.v.) để tạo, gắn, tháo và quản lý volume cho Pod.

Mình giải thích dễ hiểu nhé:

---

## 1. Vì sao cần CSI?

- Trước đây, Kubernetes tích hợp trực tiếp các loại storage (EBS, NFS, Ceph, v.v.) vào core code → mỗi khi muốn hỗ trợ loại storage mới phải chỉnh sửa và build lại Kubernetes.
    
- CSI tách phần quản lý storage thành **plugin bên ngoài** → Kubernetes chỉ cần gọi theo một **API chuẩn**, còn driver CSI sẽ tự xử lý.
    
- Giúp:
    
    - Thêm loại storage mới mà **không phải sửa code Kubernetes**.
        
    - Cập nhật/triển khai driver dễ dàng hơn.
        
    - Một driver có thể dùng cho nhiều hệ thống (K8s, Mesos, Docker…).
        

---

## 2. CSI gồm những phần nào?

Một driver CSI thường triển khai 3 nhóm API:

1. **Controller Service**
    
    - Tạo/Xóa volume (Create/Delete)
        
    - Attach/Detach volume vào node
        
2. **Node Service**
    
    - Mount/Unmount volume vào node
        
3. **Identity Service**
    
    - Cung cấp thông tin về driver (tên, version, capability…)
        

Kubernetes sẽ gọi driver CSI qua gRPC để thực hiện các thao tác này.

---

## 3. Ví dụ luồng hoạt động

Khi bạn tạo PVC:

1. Kubernetes đọc `provisioner` trong StorageClass (vd: `ebs.csi.aws.com`).
    
2. Gọi **CreateVolume** của driver CSI để tạo volume ở backend (AWS EBS).
    
3. Khi Pod chạy:
    
    - Gọi **ControllerPublishVolume** để attach volume vào node.
        
    - Gọi **NodeStageVolume** + **NodePublishVolume** để mount vào container.
        
4. Khi Pod/PVC xóa:
    
    - Gọi **DeleteVolume** để xóa volume (nếu reclaimPolicy là `Delete`).
        

---

## 4. Một số CSI driver phổ biến

- AWS EBS: `ebs.csi.aws.com`
    
- GCP Persistent Disk: `pd.csi.storage.gke.io`
    
- Azure Disk: `disk.csi.azure.com`
    
- NFS: `nfs.csi.k8s.io`
    
- Ceph RBD: `rbd.csi.ceph.com`
    
- S3 (qua FUSE/driver): `s3.csi.k8s.io`
    

---

## 5. Tóm gọn bằng ví dụ

Khi bạn tạo PVC:

yaml

CopyEdit

`apiVersion: storage.k8s.io/v1 kind: StorageClass metadata:   name: my-sc provisioner: ebs.csi.aws.com`

→ Kubernetes sẽ nhờ **AWS EBS CSI driver** tạo volume trong AWS và gắn vào node, thay vì tự thao tác.


## 1. **Mount** là gì?

- **Mount** = gắn một filesystem (thiết bị lưu trữ, thư mục chia sẻ, NFS, v.v.) vào một **điểm gắn** (mount point) trong cây thư mục Linux.
    
- Sau khi mount, nội dung của volume/thiết bị sẽ **xuất hiện** tại mount point đó.
    
- Ví dụ Linux:
    

bash

CopyEdit

`mount /dev/sdb1 /mnt/data`

→ Ổ `/dev/sdb1` sẽ hiện nội dung ở `/mnt/data`.

Trong Kubernetes:

- Khi bạn mount PVC vào Pod qua `volumeMounts`, kubelet sẽ **mount volume vào node** trước, sau đó mount tiếp vào container.
    

---

## 2. **Bind mount** là gì?

- **Bind mount** là một loại mount đặc biệt:  
    Thay vì mount **một filesystem mới**, nó **gắn lại (bind)** một thư mục/đường dẫn **đã tồn tại** vào vị trí khác.
    
- Giống như tạo “cửa thứ hai” dẫn vào cùng một chỗ dữ liệu.
    
- Ví dụ:
    

bash

CopyEdit

`mount --bind /var/log /mnt/logs`

→ `/mnt/logs` và `/var/log` thực ra cùng trỏ tới một thư mục gốc; thay đổi ở một bên sẽ thấy ở bên kia.

Trong Kubernetes:

- Khi kubelet đã mount PV vào `/var/lib/kubelet/pods/<id>/volumes/...` trên node, nó **bind mount** thư mục đó vào container tại `mountPath` bạn định nghĩa trong Pod spec.
    
- Nhờ bind mount, container nhìn thấy dữ liệu mà thực chất nằm trên node (hoặc storage backend).
    

---

## 3. Minh họa quy trình trong Kubernetes

Giả sử Pod dùng PVC:

1. PVC bind với PV (tìm được storage thật).
    
2. Node nơi Pod chạy sẽ **mount** volume thật (EBS, NFS, hostPath…) vào một thư mục tạm trên node.
    
3. Kubelet **bind mount** thư mục tạm này vào trong container ở `mountPath` bạn chỉ định.
    

Ví dụ Pod spec:

yaml

CopyEdit

`volumeMounts:   - name: mydata     mountPath: /app/data`

→ Thư mục `/app/data` trong container thực chất là bind mount của thư mục volume trên node.

---

## 4. Tóm gọn

| Thuật ngữ      | Ý nghĩa                                    | Ví dụ                       |
| -------------- | ------------------------------------------ | --------------------------- |
| **Mount**      | Gắn filesystem vào một điểm mount mới      | Mount ổ đĩa vào `/mnt/data` |
| **Bind mount** | Gắn một thư mục đã tồn tại vào vị trí khác | `/var/log` → `/mnt/logs`    |
