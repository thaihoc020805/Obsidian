## 1. StorageClass là gì?

- **StorageClass** = “mẫu/loại dịch vụ lưu trữ” mà admin định nghĩa sẵn cho cluster.
    
- Mỗi loại có thể khác nhau về:
    
    - Loại storage (SSD, HDD, NFS, EBS…)
        
    - Tốc độ (low-latency, high-throughput…)
        
    - Chính sách backup
        
    - Chính sách reclaim khi xóa PVC
        
- Kubernetes **không quan tâm** StorageClass nghĩa gì, admin tự định nghĩa.
    

📌 Ví dụ:

- `fast-ssd`: SSD nhanh, tự xóa khi PVC bị xóa.
    
- `nfs-shared`: NFS cho phép nhiều node dùng chung.
    

---

## 2. Thành phần chính của một StorageClass

Một StorageClass thường có:

|Trường|Ý nghĩa|
|---|---|
|**name**|Tên StorageClass (PVC sẽ dùng tên này để yêu cầu)|
|**provisioner**|Ai (driver CSI) sẽ tạo PV mới khi PVC yêu cầu|
|**parameters**|Thông số riêng cho driver (loại disk, IOPS, zone…)|
|**reclaimPolicy**|PVC xóa thì PV sẽ bị Delete hay Retain|
|**allowVolumeExpansion**|Cho phép tăng dung lượng PVC hay không|
|**mountOptions**|Các tùy chọn khi mount (vd: nfsvers=4.1, noatime)|
|**volumeBindingMode**|Khi nào bind volume (Immediate / WaitForFirstConsumer)|

---

## 3. Default StorageClass

- Nếu PVC **không chỉ định** `storageClassName` → Kubernetes dùng StorageClass default.
    
- Đánh dấu default bằng annotation:
    

yaml

CopyEdit

`storageclass.kubernetes.io/is-default-class: "true"`

- Nếu có nhiều default → dùng cái tạo gần nhất.
    
- Có thể tồn tại cluster **không có default**, khi đó PVC phải chỉ rõ `storageClassName`.
    

---

## 4. Provisioner

- Là **tên driver** chịu trách nhiệm tạo PV.
    
- Ví dụ:
    
    - AWS EBS: `ebs.csi.aws.com`
        
    - GCP PD: `pd.csi.storage.gke.io`
        
    - NFS CSI: `nfs.csi.k8s.io`
        
- Một số plugin “internal” (`kubernetes.io/...`) đã deprecated, giờ đa số dùng CSI driver.
    

---

## 5. Reclaim Policy

- Quy định PV sẽ làm gì sau khi PVC bị xóa:
    
    - **Delete**: Xóa luôn PV + storage backend (mặc định).
        
    - **Retain**: Giữ lại PV (cần admin cleanup).
        
- StorageClass tạo PV mới sẽ dùng reclaimPolicy được chỉ định trong nó.
    

---

## 6. Volume Expansion

- Nếu `allowVolumeExpansion: true` → có thể **tăng size PVC** bằng `kubectl edit pvc`.
    
- Chỉ tăng, không giảm.
    
- Driver phải hỗ trợ volume expansion (vd: CSI, Azure File, Portworx…).
    

---

## 7. Mount Options

- StorageClass có thể quy định mountOptions áp dụng cho PV tạo ra.
    
- Nếu driver không hỗ trợ mountOptions → provisioning sẽ fail.
    

---

## 8. Volume Binding Mode

- **Immediate** (mặc định): PV tạo ngay khi PVC được tạo.
    
- **WaitForFirstConsumer**: Chờ đến khi có Pod dùng PVC → mới tạo PV và bind, để đảm bảo PV được tạo đúng zone/node.
    
    - Quan trọng cho storage có ràng buộc topology (local, zonal disks).
        

---

## 9. Allowed Topologies

- Dùng khi muốn hạn chế PV được tạo ở zone/region cụ thể.
    

yaml

CopyEdit

`allowedTopologies: - matchLabelExpressions:   - key: topology.kubernetes.io/zone     values:       - us-central-1a       - us-central-1b`

---

## 10. Parameters

- Là các tham số driver-specific, ví dụ AWS EBS:
    

yaml

CopyEdit

`parameters:   type: io1   iopsPerGB: "50"   encrypted: "true"`

- Mỗi driver CSI sẽ có bộ parameters riêng.
    

---

## 11. Ví dụ StorageClass đầy đủ

yaml

CopyEdit

`apiVersion: storage.k8s.io/v1 kind: StorageClass metadata:   name: low-latency   annotations:     storageclass.kubernetes.io/is-default-class: "false" provisioner: csi-driver.example.com reclaimPolicy: Retain allowVolumeExpansion: true mountOptions:   - discard volumeBindingMode: WaitForFirstConsumer parameters:   guaranteedReadWriteLatency: "true"`

PVC dùng StorageClass này:

yaml

CopyEdit

`apiVersion: v1 kind: PersistentVolumeClaim metadata:   name: mypvc spec:   accessModes: [ReadWriteOnce]   resources:     requests:       storage: 10Gi   storageClassName: low-latency`