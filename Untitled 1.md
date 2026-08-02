
slide 1
# Vấn đề

Kubernetes Namespace cung cấp một mức độ cách ly cơ bản, nhưng không có cấu trúc phân cấp. Khi nhiều nhóm hoặc khách hàng cùng sử dụng chung một cụm Kubernetes, tổ chức sẽ phải đối mặt với những lựa chọn khó khăn:

- **Isolation is all-or-nothing** — Kubernetes không cung cấp sẵn cơ chế để gom các Namespace theo từng nhóm hoặc áp dụng chính sách nhất quán trên toàn bộ các Namespace của nhóm đó.
    
- **Namespace sprawl** — Khi số lượng Namespace tăng lên, quản trị viên cụm trở thành điểm nghẽn vì phải tự tạo và cấu hình từng Namespace.
    
- **Cluster sprawl** — Để đạt được mức cách ly phù hợp, các tổ chức thường triển khai một cụm Kubernetes riêng cho mỗi nhóm, làm gia tăng đáng kể chi phí tài nguyên và công sức vận hành.



slide 2
# Vì sao không triển khai một Cluster cho mỗi nhóm?

## Chi phí bị nhân bản khi tách nhiều Cluster


Một Cluster HA thường cần tối thiểu 3 máy Control Plane:

```text
10 nhóm × 3 máy Control Plane
=
30 máy Control Plane
```

Ngoài tài nguyên máy chủ, tổ chức còn phải duy trì:

```text
10 cụm etcd
10 API Server endpoint
10 bộ chứng chỉ
10 quy trình Backup
10 quy trình Upgrade
10 hệ thống Monitoring
```

### Nhiều nhóm sử dụng chung một Cluster

Có thể tập trung tài nguyên vào một Control Plane 

- Các thành phần Control Plane chỉ cần triển khai một lần.
    
- Tài nguyên dự phòng được sử dụng chung.
    
- Backup, Monitoring và Upgrade được quản lý tập trung.
    
- Không cần nhân bản toàn bộ hệ thống theo số lượng nhóm.
    



# Các giải pháp hiện có

## Rancher Project

- Gom nhiều Namespace thành một Project.
    
- Quản lý User, RBAC và Resource Quota theo Project.
    
- Cung cấp giao diện quản trị trực quan.
    
- Hỗ trợ quản lý Multi-cluster trong cùng một nền tảng.
    
- Phù hợp với tổ chức đang sử dụng Rancher.
    

**Hạn chế:**

- Không hỗ trợ Project cha – con hoặc cấu trúc phân cấp.
    
- Project chủ yếu tập trung vào User, RBAC, Quota và Namespace management.
    
- Khả năng thiết lập Policies chuyên sâu cho từng Project còn hạn chế.
    
- Các yêu cầu như giới hạn Registry, StorageClass, Node Pool hoặc Workload configuration thường phải kết hợp thêm Admission Policy, Kubewarden, Kyverno hoặc công cụ khác.
    


# Các giải pháp hiện có
## Capsule Tenant

- Gom nhiều Namespace thành một Tenant.
    
- Cho phép Tenant Owner tự tạo và quản lý Namespace.
    
- Quản lý Resource Quota trên toàn bộ Tenant.
    
- Cung cấp hệ thống Tenant Policies chi tiết hơn.
    
- Dễ dàng giới hạn Registry, StorageClass, IngressClass, Service Type và Node Pool mà Tenant được phép sử dụng.
    
- Replicate các Resource như NetworkPolicy, LimitRange, Secret và ConfigMap xuống nhiều Namespace.
    
- Capsule Proxy lọc Cluster-scoped resources theo phạm vi của Tenant.
    

**Hạn chế:**

- Không hỗ trợ Tenant cha – con hoặc SubTenant.
    
- Không thể biểu diễn cấu trúc Organization → Department → Team.
    
- Không hỗ trợ phân quyền quản trị theo từng nhánh của tổ chức.