
slide 1
# Vấn đề

Kubernetes Namespace cung cấp một mức độ cách ly cơ bản, nhưng không có cấu trúc phân cấp. Khi nhiều nhóm hoặc khách hàng cùng sử dụng chung một cụm Kubernetes, tổ chức sẽ phải đối mặt với những lựa chọn khó khăn:

- **Isolation is all-or-nothing** — Kubernetes không cung cấp sẵn cơ chế để gom các Namespace theo từng nhóm hoặc áp dụng chính sách nhất quán trên toàn bộ các Namespace của nhóm đó.
    
- **Namespace sprawl** — Khi số lượng Namespace tăng lên, quản trị viên cụm trở thành điểm nghẽn vì phải tự tạo và cấu hình từng Namespace.
    
- **Cluster sprawl** — Để đạt được mức cách ly phù hợp, các tổ chức thường triển khai một cụm Kubernetes riêng cho mỗi nhóm, làm gia tăng đáng kể chi phí tài nguyên và công sức vận hành.



slide 2 Vì sao không triển khai một Cluster cho mỗi nhóm?

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

Có thể tập trung tài nguyên vào một Control Plane làm tăng HA

- Các thành phần Control Plane chỉ cần triển khai một lần.
    
- Backup, Monitoring và Upgrade được quản lý tập trung.
    
    


slide 3 Các giải pháp hiện có

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
    

slide 4 Các giải pháp hiện có
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


slide 5 Khoảng trống: Flat Tenant 

## Capsule hiện tại

```text
Tenant Team A
├── product A
└── product B

Tenant Team b
├── product C
└── product D

Tenant Team C
...
...
```

Tất cả Tenant đều nằm ngang hàng trong một **Flat Tenant Model**.

## Cấu trúc tổ chức thực tế

```text
Organization 
├── Department A
│   ├── Team A
│   │   ├── product A
│   │       ├── product A dev
│   │       ├── product A stg
│   │   └── product B
│   │       ├── product B dev
│   │       ├── product B stg
│   └── ....
│       ├── ....
│
└── ....

```

## Những khả năng còn thiếu

- Không hỗ trợ Parent Tenant và Child Tenant.
    
- Không thể ủy quyền,  phân Quota cho từng cấp.
    
- Không hỗ trợ Policy Inheritance qua nhiều cấp.
- Không thể phân role  theo từng nhánh của tree.



slide 6 **Giải pháp đề xuất: 

HierarchyTenant
 
 cung cấp mô hình HirTenant phân cấp cho Shared Cluster.

```
Organization 
├── Department A
│   ├── Team A
│   │   ├── product A
│   │       ├── product A dev
│   │       ├── product A stg
│   │   └── product B
│   │       ├── product B dev
│   │       ├── product B stg
│   └── ....
│       ├── ....
│
└── ....

```

### Mục tiêu

- Biểu diễn cấu trúc tổ chức trong Kubernetes.
- Cho phép ủy quyền quản trị theo từng cấp.
- Kế thừa quyền, quota và chính sách.
- Cho phép các nhóm tự phục vụ trong giới hạn được giao.
- Duy trì một Shared Control Plane duy nhất.


Slide 7  Các Role trong hệ thống

|Role|Trách nhiệm|
|---|---|
|**Cluster Admin**|Cài đặt và quản trị HieraTenant, tạo Root Tenant, thiết lập Global Policies, Resource Budget và các giới hạn hạ tầng. Sau khi ủy quyền, không cần tham gia vào công việc quản lý Tenant và Namespace hằng ngày.|
|**Tenant Owner**|Quản lý Tenant được giao và phạm vi Subtree bên dưới. Có thể tạo Child Tenant, tạo Namespace, phân bổ Quota, thiết lập Policies chặt hơn và cấp quyền cho các thành viên. Không có quyền quản trị toàn Cluster.|
|**Tenant User**|Triển khai và quản lý Workload trong các Namespace được cấp quyền, trong phạm vi Quota và Policies đã được Cluster Admin và Tenant Owner thiết lập.|
# Slide 8 — Tổng quan chức năng

# Các chức năng chính

## Organization & Delegation

- Hỗ trợ Parent Tenant, Child Tenant và SubTenant.
    
- Delegated Administration theo từng cấp tổ chức.
    
- Quản trị User, Role và Permission theo từng Subtree.
    

## Resource Management

- Hierarchical Quota giữa Parent Tenant và Child Tenant.
    
- Aggregate Resource Usage trên toàn bộ Subtree.
    
- Giới hạn Node Pool, StorageClass, IngressClass và Resource usage.
    

## Security & Isolation

- Hierarchical RBAC và Permission Inheritance.
    
- Policy Inheritance và Policy Enforcement qua nhiều cấp.
    
- Network Isolation giữa các Tenant.
    

## Tenant Self-service

- Tự tạo và quản lý Namespace.
    
- Tự động Resource Replication xuống các Namespace.
    
- API Proxy lọc Cluster-scoped resources.
    
- Cung cấp trạng thái, Quota và Resource Usage theo Tenant.


slide 9
# Organization & Delegation

## Hierarchical Tenant Model

```text
Organization 
├── Department A
│   ├── Team A
│   │   ├── product A
│   │       ├── product A dev
│   │       ├── product A stg
│   │   └── product B
...
```

- Mỗi Tenant có thể có một Parent Tenant và nhiều Child Tenant.
    
- Child Tenant có thể tiếp tục tạo các SubTenant bên dưới.
    
- Toàn bộ Tenant và SubTenant tạo thành một Tenant Tree.
    

## Delegated Administration

- Cluster Admin không cần trực tiếp xử lý mọi Tenant và Namespace.
    
- Mỗi Tenant Owner chỉ quản lý Tenant được giao và toàn bộ Subtree bên dưới.
    
- Tenant Owner không thể truy cập hoặc quản lý các nhánh Tenant khác.
    

## Subtree-scoped Access Management

- Quản lý User, Role và Permission trong phạm vi Subtree.
    
- Permission có thể được kế thừa từ Parent Tenant xuống Child Tenant.
    
- Tenant Owner có thể cấp quyền cho thành viên nhưng không được vượt quá quyền đã được Parent Tenant cho phép.
    

slide 10 Resource Management

## Hierarchical Quota

```text
Organization — 200 CPU
├── Department A — 120 CPU
│   ├── Team A — 80 CPU
│   │   ├── Product A — 50 CPU
│   │   │   ├── product-a-dev
│   │   │   ├── product-a-stg
│   │   │
│   │   └── Product B — 30 CPU
│   │       ├── product-b-dev
│   │       └── product-b-prod
│
└── Department B — 80 CPU
```

- Resource Budget được phân bổ từ Parent Tenant xuống Child Tenant.
    
- Tenant Owner chỉ được phân bổ tài nguyên trong Resource Budget được giao.
    
- Quota của Product được sử dụng chung bởi các Namespace môi trường bên dưới.
    

slide 11 Resource Management
## Aggregate Resource Usage

Resource Usage được tổng hợp theo chiều từ dưới lên:

Ví dụ:

```text
product-a-dev:        5 CPU
product-a-stg:       10 CPU
──────────────────────────
Product A Usage:     15/50 CPU

Product B Usage:     20/30 CPU
──────────────────────────
Team A Usage:        60/80 CPU
```

Mỗi Tenant Owner có thể theo dõi:

- Resource Budget được cấp, Resource Usage hiện tại, Remaining Resources.
- Usage của từng Child Tenant và toàn bộ Subtree.
    

## Infrastructure Restrictions

Mỗi Tenant hoặc Subtree chỉ được sử dụng các tài nguyên hạ tầng đã được cho phép:

- **Node Pool** — giới hạn vị trí Workload được Scheduling.
    
- **StorageClass** — giới hạn loại Storage được sử dụng.
    
- **Resource Usage** — giới hạn CPU, Memory, Storage và số lượng Kubernetes Resources.


slide 12 Security & Isolation

# Security & Isolation

## Hierarchical RBAC & Permission Inheritance

```text
Organization Owner
├── Department A Owner
│   ├── Team A Owner
│   │   ├── Product A Owner
│   │   └── Product B Owner
│   └── Team B Owner
└── Department B Owner
```

- Permission được kế thừa từ Parent Tenant xuống toàn bộ Child Tenant và Namespace bên dưới.
    
- Child Tenant có thể bổ sung User và Role riêng nhưng không được cấp quyền vượt quá Parent Tenant.
    

## Network Isolation

- Mặc định chặn Network Traffic giữa các Tenant độc lập.
    
- Các Namespace trong cùng Product hoặc Tenant có thể giao tiếp theo Policy được cấu hình.
    
- Giao tiếp giữa các Subtree chỉ được phép khi có Network Policy rõ ràng.
    
- Network Isolation được thực thi thông qua Kubernetes NetworkPolicy và CNI Plugin.
    


slide 13 Security & Isolation
## Policy Inheritance & Enforcement

Policy được kế thừa qua nhiều cấp trong Tenant Tree:

```text
Organization
└── Department A
    └── Team A
        └── Product A
            ├── product-a-dev
            ├── product-a-stg
            └── product-a-prod
```

Ví dụ:

```text
Organization:
- Cấm Privileged Container
- Cấm HostPath

Department A:
- Chỉ cho phép Internal Registry

Team A:
- Bắt buộc khai báo CPU và Memory Request

Product A:
- Chỉ được sử dụng các Image đã được phê duyệt
```

Workload của Product A phải tuân thủ toàn bộ Policy từ các cấp phía trên.

- Child Tenant có thể bổ sung Policy chặt hơn.
    
- Child Tenant không được loại bỏ hoặc nới lỏng Policy của Parent Tenant.
    
- Admission Webhook kiểm tra và từ chối các Resource vi phạm Policy.
    

slide 14 Tenant Self-service

## Namespace Self-service

```text
Product A Owner
├── Tạo product-a-dev
├── Tạo product-a-stg
└── Tạo product-a-prod
```

- Tenant Owner có thể tự tạo và quản lý Namespace trong Tenant được giao.
    
- Namespace mới được tự động gắn vào đúng Tenant.
    
- Hệ thống kiểm tra Namespace Quota, Naming Policy và Permission trước khi cho phép tạo.
    
- Cluster Admin không cần trực tiếp tạo và cấu hình từng Namespace.


slide 15  Tenant Self-service
## Resource Replication

Các Resource dùng chung được tự động Replicate xuống các Namespace:

```text
Product A
├── product-a-dev
│   ├── NetworkPolicy
│   ├── LimitRange
│   └── ImagePullSecret
├── product-a-stg
│   ├── NetworkPolicy
│   ├── LimitRange
│   └── ImagePullSecret
└── product-a-prod
    ├── NetworkPolicy
    ├── LimitRange
    └── ImagePullSecret
```

Có thể Replicate:

- NetworkPolicy.
    
- LimitRange và ResourceQuota.
    
- ConfigMap và Secret.
    
- ServiceAccount.
    
- Các Custom Resource được cho phép.
    

Resource có thể được khai báo ở Parent Tenant và tự động truyền xuống toàn bộ Subtree.


slide 16  Tenant Self-service
## Tenant-aware API Proxy

API Proxy lọc các Cluster-scoped resources theo phạm vi của Tenant:

```text
Toàn Cluster:
product-a-dev
product-a-stg
product-a-prod
product-b-dev
team-b-prod

Product A Owner nhìn thấy:
product-a-dev
product-a-stg
product-a-prod
```

- User chỉ nhìn thấy Namespace và Cluster-scoped resources được phép sử dụng.
    
- Có thể lọc Namespace, Node, StorageClass, IngressClass và PersistentVolume.
    
- Giảm nguy cơ làm lộ thông tin của Tenant khác.

- Dễ dàng tích hợp với các UI tool


# Slide 17 — Kiến trúc hệ thống

## Nội dung trên slide

### **Kiến trúc HieraTenant**

```
Người dùng / kubectl / Portal
              │
              ▼
      HieraTenant API Proxy
              │
              ▼
      Kubernetes API Server
        │             │
        │             └── Admission Webhook
        │                    ├── Kiểm tra cây Tenant
        │                    ├── Kiểm tra quyền
        │                    ├── Kiểm tra quota
        │                    └── Cưỡng chế policy
        │
        ▼
  HieraTenant Controller Manager
        ├── Tenant Tree Controller
        ├── Namespace Controller
        ├── RBAC Controller
        ├── Quota Controller
        ├── Policy Controller
        ├── Resource Propagation Controller
        └── Status Aggregator
              │
              ▼
      Kubernetes Resources / etcd
```

### Các thành phần cần phát triển

1. Custom Resource Definition.
2. Controller Manager.
3. Validating Admission Webhook.
4. Mutating Admission Webhook.
5. Tenant-aware API Proxy.
6. Metrics và công cụ dòng lệnh tùy chọn.


# Slide 18 — Hạn chế và phạm vi sử dụng

## Nội dung trên slide

### **Hạn chế**

- Các Tenant vẫn dùng chung API Server, etcd và Scheduler.
- Nếu dùng chung Worker Node, các Workload vẫn có thể dùng chung Kernel.
- CRD và các tài nguyên Cluster-scoped vẫn thuộc toàn Cluster.
- Proxy, Webhook và Controller trở thành thành phần quan trọng cần tính sẵn sàng cao.
- Không phù hợp để cho người dùng hoàn toàn không tin cậy chạy mã tùy ý.

### Phù hợp với

- Nền tảng phát triển nội bộ.
- Công ty có nhiều phòng ban và nhóm.
- Trường đại học, phòng thí nghiệm.
- Khách hàng doanh nghiệp có kiểm soát.

### Không thay thế

```
vCluster
Dedicated Cluster
Runtime Sandbox
```