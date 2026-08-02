# Slide 1 — Vấn đề khi nhiều nhóm dùng chung một cụm Kubernetes

## Nội dung trên slide

### **Vấn đề khi nhiều nhóm dùng chung một cụm Kubernetes**

Kubernetes Namespace cung cấp một phạm vi cách ly cơ bản, nhưng:

- **Không có cấu trúc phân cấp**  
    Namespace không thể chứa Namespace khác và không thể biểu diễn cấu trúc công ty → phòng ban → nhóm → môi trường.
- **Bùng nổ Namespace**  
    Mỗi nhóm cần nhiều Namespace như `dev`, `test`, `staging`, `prod`; quản trị viên phải tạo và cấu hình lặp lại.
- **Khó áp dụng quản trị đồng nhất**  
    RBAC, quota, NetworkPolicy, LimitRange và các chính sách phải được quản lý trên nhiều Namespace riêng lẻ.
- **Bùng nổ Cluster**  
    Để có mức cách ly rõ ràng hơn, tổ chức thường tạo một Cluster cho mỗi nhóm, làm tăng mạnh chi phí vận hành.

### Sơ đồ nên đặt trên slide

```
Công ty ACME

Backend       Frontend       Data          AI
├── dev       ├── dev        ├── dev       ├── dev
├── test      ├── test       ├── test      ├── test
└── prod      └── prod       └── prod      └── prod

Kubernetes chỉ nhìn thấy một danh sách Namespace phẳng:
backend-dev, backend-test, backend-prod, frontend-dev...
```

Kubernetes xác nhận Namespace không thể lồng vào nhau; mỗi tài nguyên chỉ thuộc một Namespace. Việc dùng chung Cluster giúp giảm chi phí nhưng đồng thời tạo ra các vấn đề về bảo mật, công bằng và noisy neighbor.

## Lời nói

> Giả sử một công ty có 10 nhóm phát triển. Mỗi nhóm cần ít nhất ba môi trường là dev, test và prod. Như vậy chúng ta đã có 30 Namespace.
> 
> Khi số lượng nhóm tăng lên 50, chúng ta có thể có 150 Namespace, chưa tính các Namespace dành cho CI/CD, preview environment hay các dịch vụ riêng.
> 
> Vấn đề là Kubernetes chỉ nhìn thấy một danh sách Namespace phẳng. Nó không biết Namespace `backend-dev` và `backend-prod` cùng thuộc nhóm Backend; nhóm Backend lại thuộc phòng Engineering; và phòng Engineering thuộc công ty ACME.
> 
> Vì không có cấu trúc đó, quản trị viên phải tự quản lý RoleBinding, ResourceQuota, LimitRange, NetworkPolicy và nhiều cấu hình khác cho từng Namespace.
> 
> Kubernetes có Namespace để chia phạm vi tài nguyên, nhưng không có một đơn vị quản trị cao hơn để đại diện cho một nhóm hay một phòng ban.
> 
> Khi việc quản lý Namespace trở nên quá phức tạp, một lựa chọn phổ biến là tạo riêng một Cluster cho từng nhóm. Tuy nhiên, cách này lại dẫn đến một vấn đề khác là bùng nổ số lượng Cluster.
> 
> Đây là mâu thuẫn chính: dùng chung Cluster thì khó quản trị và cách ly; còn tách Cluster thì tốn tài nguyên và công sức vận hành.

### Câu kết nối sang slide sau

> Vậy tại sao chúng ta không chọn cách đơn giản nhất là mỗi nhóm sử dụng một Cluster riêng?

---

# Slide 2 — Vì sao không triển khai một Cluster cho mỗi nhóm?

## Nội dung trên slide

### **Vì sao không triển khai một Cluster cho mỗi nhóm?**

Chi phí của một Cluster có thể được nhìn đơn giản thành:

```
Chi phí Cluster
=
Chi phí cố định
+
Chi phí phụ thuộc vào Workload
```

### Dùng chung một Cluster

```
1 × chi phí cố định
+
Tổng Workload của 10 nhóm
```

### Tách thành 10 Cluster

```
10 × chi phí cố định
+
Tổng Workload của 10 nhóm
```

### Điểm quan trọng

> Phần Workload có thể gần tương đương, nhưng phần chi phí cố định bị nhân lên theo số lượng Cluster.

### Dòng ghi chú nhỏ trên slide

```
Cluster riêng phù hợp khi Workload lớn hoặc yêu cầu cách ly cao.
Shared Cluster phù hợp khi nhiều nhóm có Workload nhỏ và trung bình.
```

## Lời nói

> Việc tạo Cluster riêng không phải lúc nào cũng sai.
> 
> Nếu một nhóm có 10.000 Pod, cần một lượng lớn tài nguyên hoặc có yêu cầu bảo mật rất cao, việc dành riêng một Cluster cho nhóm đó là hoàn toàn hợp lý.
> 
> Nhưng giả sử chúng ta có 10 nhóm và mỗi nhóm chỉ chạy khoảng 100 Pod. Tổng cộng chỉ có khoảng 1.000 Pod.
> 
> Nếu dùng chung Cluster, chúng ta chỉ cần một Control Plane đủ lớn để quản lý 1.000 Pod.
> 
> Nếu tách riêng, chúng ta phải có 10 Control Plane, mỗi Control Plane chỉ quản lý khoảng 100 Pod.
> 
> Có thể có ý kiến cho rằng thay vì một Control Plane lớn, chúng ta chia thành 10 Control Plane nhỏ thì tổng tài nguyên vẫn bằng nhau.
> 
> Nhưng thực tế không hoàn toàn như vậy. Mỗi Cluster đều có một phần chi phí cố định: API Server, Scheduler, Controller Manager, etcd, endpoint, certificate và các quy trình vận hành đi kèm.
> 
> Những thành phần này vẫn phải tồn tại kể cả khi Cluster chỉ có vài chục Pod.
> 
> Control Plane lớn hơn chắc chắn cần thêm tài nguyên khi Workload tăng, nhưng chi phí đó không tăng tuyến tính theo số lượng nhóm. Một Control Plane quản lý 10 nhóm không nhất thiết cần tài nguyên gấp 10 lần một Control Plane quản lý một nhóm.

---

# Slide 3 — Chi phí bị nhân bản khi tách nhiều Cluster

## Nội dung trên slide

### **Chi phí bị nhân bản khi tách nhiều Cluster**

### Mỗi nhóm một Cluster có tính sẵn sàng cao

```
10 nhóm × 3 máy Control Plane
=
30 máy Control Plane
```

Theo mức tối thiểu để minh họa:

```
30 × 2 vCPU  = 60 vCPU
30 × 2 GiB   = 60 GiB RAM
```

Ngoài ra còn có:

```
10 cụm etcd
10 API Server endpoint
10 Scheduler
10 Controller Manager
10 bộ Certificate
10 quy trình Backup
10 quy trình Upgrade
10 hệ thống Monitoring
```

### 10 nhóm dùng chung một Cluster

```
3 hoặc 5 máy Control Plane lớn hơn
```

Ví dụ minh họa:

```
3 × 8 vCPU  = 24 vCPU
3 × 16 GiB  = 48 GiB RAM
```

> Kích thước thực tế phải được xác định bằng Benchmark.

Tài liệu kubeadm yêu cầu tối thiểu 2 CPU cho máy Control Plane và ít nhất 2 GiB RAM trên mỗi máy. Đây chỉ là ngưỡng cài đặt tối thiểu, không phải cấu hình khuyến nghị cho môi trường sản xuất.

## Lời nói

> Để minh họa, giả sử mỗi Cluster cần ba máy Control Plane nhằm tăng khả năng chịu lỗi.
> 
> Với 10 nhóm, chúng ta cần 30 máy Control Plane.
> 
> Nếu chỉ tính theo mức tối thiểu của kubeadm là hai CPU và hai GiB RAM trên mỗi máy, chúng ta đã cần 60 vCPU và 60 GiB RAM.
> 
> Đây mới chỉ là mức tối thiểu để minh họa, chưa phải cấu hình phù hợp cho môi trường sản xuất.
> 
> Quan trọng hơn, tài nguyên không phải chi phí duy nhất. Chúng ta còn có 10 cụm etcd, 10 API endpoint, 10 bộ certificate, 10 quy trình backup, 10 lần nâng cấp và 10 nơi phải theo dõi.
> 
> Khi dùng chung một Cluster, chúng ta có thể sử dụng ba hoặc năm máy Control Plane lớn hơn. Ví dụ, ba máy với tám vCPU và 16 GiB RAM.
> 
> Con số cụ thể phải được đo bằng benchmark. Tuy nhiên, điểm cần nhấn mạnh là một Shared Control Plane không cần lớn gấp 10 chỉ vì nó phục vụ 10 nhóm.
> 
> Vì vậy Shared Cluster vẫn có giá trị lớn đối với các tổ chức có nhiều nhóm nhỏ và trung bình.

### Câu chuyển

> Shared Cluster có lợi về chi phí và vận hành. Nhưng Kubernetes gốc chưa cung cấp một mô hình quản lý nhiều nhóm đủ thuận tiện. Một số giải pháp đã được xây dựng để xử lý vấn đề đó.

---

# Slide 4 — Các giải pháp hiện có

## Nội dung trên slide

### **Các giải pháp hiện có**

## Rancher Project

- Gom nhiều Namespace thành một Project.
- Phân quyền theo Project.
- Quota ở cấp Project và Namespace.
- Giao diện quản trị người dùng và nhiều Cluster.
- Phù hợp khi tổ chức đã sử dụng Rancher.

**Hạn chế:**

- Project vẫn là mô hình phẳng.
- Không có Project cha – con.
- Phụ thuộc vào nền tảng Rancher.
- Chính sách chuyên sâu thường cần thêm công cụ khác.

## Capsule Tenant

- Gom nhiều Namespace thành một Tenant.
- Tenant Owner tự tạo và quản lý Namespace.
- Quota tổng trên nhiều Namespace.
- Giới hạn Registry, StorageClass, Ingress và Node.
- Tự động phân phối tài nguyên giữa các Namespace.
- Capsule Proxy lọc tài nguyên Cluster-scoped.

**Hạn chế:**

- Tất cả Tenant nằm ngang hàng.
- Tenant Owner không quản lý chính đối tượng Tenant.
- Không có SubTenant hoặc phân quyền theo cây tổ chức.

Rancher định nghĩa cấu trúc Cluster → Project → Namespace và hỗ trợ vai trò cùng quota ở cấp Project. Capsule tập trung sâu hơn vào multi-tenancy trong một Cluster, bao gồm quota, enforcement, resource replication và proxy.

## Lời nói

> Hai giải pháp tiêu biểu cho bài toán này là Rancher Project và Capsule.
> 
> Rancher là một nền tảng quản lý Kubernetes lớn. Project chỉ là một chức năng trong Rancher, giúp gom nhiều Namespace, phân quyền người dùng và đặt quota.
> 
> Rancher có lợi thế rất mạnh về giao diện, quản lý danh tính và quản lý nhiều Cluster. Tuy nhiên, Project vẫn chỉ là một tầng phẳng nằm giữa Cluster và Namespace.
> 
> Capsule tập trung chuyên sâu hơn vào multi-tenancy trong một Cluster. Capsule tạo ra khái niệm Tenant. Mỗi Tenant có thể sở hữu nhiều Namespace, có quota tổng, chính sách riêng và Tenant Owner có thể tự tạo Namespace.
> 
> Capsule còn có thể giới hạn Tenant được sử dụng StorageClass, IngressClass, Registry hoặc nhóm Node nào. Nó cũng có thể tự động phân phối NetworkPolicy, ConfigMap, Secret hoặc LimitRange xuống các Namespace.
> 
> Capsule Proxy giải quyết vấn đề người dùng cần xem một số tài nguyên Cluster-scoped nhưng không được nhìn thấy toàn bộ Cluster.
> 
> Tuy nhiên, Rancher Project và Capsule Tenant đều chỉ hỗ trợ một tầng quản lý phẳng. Project không chứa Project con và Tenant không chứa Tenant con.

---

# Slide 5 — Khoảng trống: Tenant phẳng không phản ánh tổ chức thực tế

## Nội dung trên slide

### **Khoảng trống: Tenant phẳng không phản ánh tổ chức thực tế**

### Capsule hiện tại

```
Tenant Backend
├── backend-dev
└── backend-prod

Tenant Frontend
├── frontend-dev
└── frontend-prod

Tenant AI
Tenant Analytics
```

Tất cả Tenant nằm ngang hàng.

### Tổ chức thực tế

```
Công ty ACME
├── Phòng Kỹ thuật
│   ├── Nhóm Backend
│   │   ├── dev
│   │   └── prod
│   └── Nhóm Frontend
│       ├── dev
│       └── prod
│
└── Phòng Dữ liệu
    ├── Nhóm AI
    └── Nhóm Phân tích
```

### Những khả năng còn thiếu

- Không thể tạo Tenant cha – con.
- Không thể ủy quyền tạo SubTenant.
- Không có quota tổng theo phòng ban hoặc tổ chức.
- Không có chính sách kế thừa qua nhiều cấp.
- Không có quản trị viên theo từng nhánh của cây.

Capsule ghi rõ rằng các Tenant được thiết kế ở cùng một cấp và không hỗ trợ hierarchy; một người chỉ có thể được gán làm owner của nhiều Tenant độc lập.

### Ghi chú nhỏ nên đặt cuối slide

> HNC từng hỗ trợ Namespace phân cấp, nhưng tập trung vào quan hệ giữa các Namespace, không cung cấp một nền tảng Tenant đầy đủ như Capsule; dự án hiện nằm trong tổ chức `kubernetes-retired`.

HNC từng hỗ trợ delegated namespace creation, policy propagation và hierarchical quota, vì vậy phần trình bày đồ án cần phân biệt rõ giải pháp của bạn với HNC.

## Lời nói

> Capsule giải quyết rất tốt việc gom nhiều Namespace thành một Tenant.
> 
> Nhưng tất cả Tenant của Capsule đều nằm ngang hàng. Capsule không biết Tenant Backend và Tenant Frontend cùng thuộc phòng Kỹ thuật, hoặc Tenant AI và Tenant Analytics cùng thuộc phòng Dữ liệu.
> 
> Điều này tạo ra một vấn đề mới. Kubernetes ban đầu có Namespace sprawl. Capsule gom Namespace thành Tenant, nhưng khi số lượng Tenant tăng lên, chúng ta lại bắt đầu có Tenant sprawl.
> 
> Ví dụ công ty ACME cấp cho phòng Kỹ thuật 70 CPU. Phòng Kỹ thuật muốn chia 40 CPU cho Backend và 30 CPU cho Frontend.
> 
> Với Capsule, Cluster Admin phải tạo hai Tenant độc lập và cấu hình quota cho từng Tenant. Capsule không biết tổng quota của Backend và Frontend phải nằm trong ngân sách 70 CPU của phòng Kỹ thuật.
> 
> Ngoài ra, trưởng phòng Kỹ thuật không thể tự tạo thêm Tenant Platform hoặc Mobile. Việc tạo và quản lý Tenant vẫn thuộc về Cluster Admin.
> 
> Đã từng có HNC hỗ trợ Namespace cha – con, kế thừa chính sách và quota phân cấp. Tuy nhiên HNC tập trung vào Namespace, không có khái niệm Tenant hoàn chỉnh với owner, hạ tầng được phép sử dụng, proxy hoặc các chính sách multi-tenancy chuyên sâu.
> 
> Vì vậy, khoảng trống của đồ án không đơn thuần là thêm cha – con vào Namespace. Mục tiêu là kết hợp khả năng quản trị Tenant của Capsule với một mô hình phân cấp hoàn chỉnh.

---

# Slide 6 — Giải pháp đề xuất: HieraTenant

## Nội dung trên slide

### **Giải pháp đề xuất: HieraTenant**

> Một Kubernetes Operator cung cấp mô hình Tenant phân cấp cho Shared Cluster.

```
Công ty ACME
├── Phòng Kỹ thuật
│   ├── Tenant Backend
│   │   ├── backend-dev
│   │   └── backend-prod
│   └── Tenant Frontend
│       ├── frontend-dev
│       └── frontend-prod
│
└── Phòng Dữ liệu
    ├── Tenant AI
    └── Tenant Phân tích
```

### Mục tiêu

- Biểu diễn cấu trúc tổ chức trong Kubernetes.
- Cho phép ủy quyền quản trị theo từng cấp.
- Kế thừa quyền, quota và chính sách.
- Cho phép các nhóm tự phục vụ trong giới hạn được giao.
- Duy trì một Shared Control Plane duy nhất.

## Lời nói

> Giải pháp được đề xuất có tên tạm thời là HieraTenant.
> 
> HieraTenant là một Kubernetes Operator bổ sung mô hình Tenant phân cấp cho Shared Cluster.
> 
> Thay vì tất cả Tenant nằm ngang hàng, mỗi Tenant có thể có một Tenant cha. Từ đó chúng ta có thể biểu diễn công ty, phòng ban, nhóm và các môi trường triển khai.
> 
> Cluster Admin chỉ cần tạo Tenant gốc là ACME và giao cho ACME Admin.
> 
> ACME Admin có thể tạo phòng Kỹ thuật và phòng Dữ liệu. Trưởng phòng Kỹ thuật có thể tiếp tục tạo Backend và Frontend. Mỗi nhóm cuối cùng có thể tự tạo các Namespace dev, test và prod.
> 
> HieraTenant không tạo thêm Control Plane và không biến mỗi Tenant thành một Cluster ảo. Tất cả vẫn sử dụng chung Kubernetes Control Plane.
> 
> Mục tiêu của hệ thống là quản lý quyền, tài nguyên và chính sách theo cây tổ chức, đồng thời giữ được lợi ích chi phí của Shared Cluster.

---

# Slide 7 — Tổng quan chức năng

## Nội dung trên slide

### **Các chức năng chính**

### Tổ chức và ủy quyền

- Tenant cha – con.
- Tạo SubTenant theo quyền được giao.
- Quản trị theo từng nhánh của cây.

### Quản lý tài nguyên

- Quota phân cấp.
- Tổng hợp mức sử dụng theo Subtree.
- Giới hạn Node, Storage và Ingress.

### Bảo mật và cách ly

- RBAC kế thừa.
- Chính sách kế thừa và cưỡng chế.
- NetworkPolicy và Scheduling Isolation.

### Trải nghiệm người dùng

- Tự tạo và quản lý Namespace.
- Tự động phân phối tài nguyên.
- API Proxy theo phạm vi Tenant.

## Lời nói

> HieraTenant sẽ không chỉ cung cấp trường `parentRef`.
> 
> Hệ thống phải triển khai lại các chức năng cơ bản của một nền tảng multi-tenancy như tự tạo Namespace, RBAC, quota và policy enforcement.
> 
> Điểm khác biệt là tất cả các chức năng này hoạt động trên một cây Tenant.
> 
> Quyền của quản trị viên được áp dụng cho toàn bộ nhánh bên dưới. Quota của Tenant con phải nằm trong ngân sách của Tenant cha. Chính sách của Tenant cha được kế thừa xuống các Tenant con và Namespace.
> 
> Ngoài ra, hệ thống còn cung cấp API Proxy để người dùng chỉ nhìn thấy tài nguyên thuộc nhánh của mình.

---

# Slide 8 — Tenant phân cấp và ủy quyền quản trị

## Nội dung trên slide

### **Tenant phân cấp và ủy quyền quản trị**

```
ACME Admin
└── Có quyền trên toàn bộ ACME

Kỹ thuật Admin
└── Chỉ có quyền trên nhánh Kỹ thuật

Backend Admin
└── Chỉ có quyền trên Backend

Developer
└── Chỉ có quyền trên Namespace được giao
```

### Quy tắc

- Một Tenant có tối đa một Tenant cha.
- Không cho phép tạo vòng lặp.
- Tenant con không thể tự thoát khỏi Tenant cha.
- Parent Admin quản lý toàn bộ Subtree.
- Child Admin chỉ quản lý phạm vi được ủy quyền.

## Lời nói

> Thành phần đầu tiên là mô hình Tenant phân cấp.
> 
> Mỗi Tenant có thể tham chiếu tới một Tenant cha. Operator phải kiểm tra Tenant cha tồn tại và không được phép tạo vòng lặp.
> 
> Ví dụ không thể để ACME là cha của Engineering, trong khi Engineering lại là cha của ACME.
> 
> Quản trị viên của Tenant cha có quyền quản lý toàn bộ cây bên dưới. Tuy nhiên, quản trị viên của Tenant con không được phép thay đổi Tenant cha hoặc truy cập vào nhánh khác.
> 
> Đây là delegated administration. Cluster Admin không cần trực tiếp tạo mọi Tenant và Namespace.
> 
> Cluster Admin quản lý Tenant gốc; quản trị viên tổ chức quản lý phòng ban; trưởng phòng quản lý các nhóm; còn nhóm tự quản lý Namespace của mình.

---

# Slide 9 — Namespace tự phục vụ và quản lý vòng đời

## Nội dung trên slide

### **Namespace tự phục vụ**

```
Backend Admin
        ↓
Tạo Namespace backend-dev
        ↓
HieraTenant tự động:
- Gán Namespace vào Tenant Backend
- Tạo RoleBinding
- Áp ResourceQuota và LimitRange
- Áp NetworkPolicy
- Gắn Label và Annotation bắt buộc
- Phân phối cấu hình mặc định
```

### Quản lý vòng đời

- Giới hạn số lượng Namespace.
- Quy tắc đặt tên hoặc Prefix.
- Ngăn Namespace bị chiếm bởi Tenant khác.
- Xóa Tenant an toàn khi vẫn còn Tenant con hoặc Namespace.

## Lời nói

> Sau khi được giao quyền, Backend Admin có thể tự tạo Namespace mà không cần gửi yêu cầu cho Cluster Admin.
> 
> Khi Namespace được tạo, Admission Webhook xác định người tạo thuộc Tenant nào và kiểm tra Tenant đó còn quota Namespace hay không.
> 
> Controller sau đó gắn Namespace vào Tenant, tạo RoleBinding, quota, NetworkPolicy và các cấu hình mặc định.
> 
> Hệ thống cũng có thể bắt buộc Namespace sử dụng Prefix. Ví dụ Tenant Backend chỉ được tạo Namespace bắt đầu bằng `backend-`.
> 
> Khi xóa một Tenant, hệ thống không được xóa ngay nếu Tenant vẫn còn SubTenant hoặc Namespace. Người dùng phải di chuyển hoặc xóa các tài nguyên con trước, hoặc sử dụng một chính sách xóa theo tầng có kiểm soát.

---

# Slide 10 — Phân quyền kế thừa theo cây Tenant

## Nội dung trên slide

### **Phân quyền kế thừa theo cây Tenant**

```
Alice — ACME Admin
├── Kỹ thuật
│   ├── Backend
│   └── Frontend
└── Dữ liệu
    ├── AI
    └── Phân tích

Bob — Kỹ thuật Admin
└── Kỹ thuật
    ├── Backend
    └── Frontend

Carol — Backend Admin
└── Backend
    ├── backend-dev
    └── backend-prod
```

### Nguyên tắc

- Quyền đi từ Tenant cha xuống Tenant con.
- Không truyền quyền sang nhánh ngang hàng.
- Tenant con có thể bổ sung thành viên riêng.
- Không cho Tenant con tự cấp quyền vượt quá Tenant cha.

## Lời nói

> RBAC của Kubernetes hoạt động trên từng Namespace và ClusterRoleBinding.
> 
> HieraTenant sử dụng Controller để chuyển mô hình quyền theo cây thành các RoleBinding và ClusterRole phù hợp.
> 
> Alice là ACME Admin nên có thể quản lý toàn bộ nhánh ACME.
> 
> Bob là Kỹ thuật Admin nên có thể quản lý Backend và Frontend nhưng không được truy cập phòng Dữ liệu.
> 
> Carol chỉ quản lý Backend và các Namespace của Backend.
> 
> Một vấn đề quan trọng là ngăn privilege escalation. Tenant con có thể cấp quyền cho thành viên trong phạm vi của mình, nhưng không được cấp quyền cao hơn quyền mà Tenant cha cho phép.

---

# Slide 11 — Quota phân cấp

## Nội dung trên slide

### **Quota phân cấp**

```
ACME: 100 CPU
├── Kỹ thuật: 70 CPU
│   ├── Backend: 40 CPU
│   └── Frontend: 30 CPU
│
└── Dữ liệu: 30 CPU
    ├── AI: 20 CPU
    └── Phân tích: 10 CPU
```

### Quy tắc

```
Tổng quota Tenant con
≤
Quota Tenant cha
```

```
Tổng mức sử dụng Subtree
≤
Quota của Tenant gốc Subtree
```

### Hệ thống cung cấp

- Quota CPU, RAM, Storage và số lượng đối tượng.
- Tổng hợp usage của toàn bộ nhánh.
- Trạng thái còn lại của từng Tenant.
- Từ chối cấp quota vượt ngân sách Tenant cha.

## Lời nói

> Quota phân cấp là một trong những chức năng quan trọng nhất.
> 
> ACME được cấp tổng cộng 100 CPU. ACME chia 70 CPU cho phòng Kỹ thuật và 30 CPU cho phòng Dữ liệu.
> 
> Phòng Kỹ thuật tiếp tục chia 40 CPU cho Backend và 30 CPU cho Frontend.
> 
> Nếu Backend yêu cầu tăng lên 50 CPU, tổng quota của Backend và Frontend sẽ là 80 CPU, vượt ngân sách 70 CPU của phòng Kỹ thuật. Admission Webhook sẽ từ chối thay đổi đó.
> 
> Hệ thống cũng tổng hợp mức sử dụng thực tế của tất cả Namespace trong Subtree.
> 
> Lưu ý quota ở đây là giới hạn quản trị. Nó không tự động bảo đảm rằng Cluster luôn có đủ capacity vật lý. Đây sẽ là một hạn chế được trình bày ở phần cuối.

---

# Slide 12 — Kế thừa chính sách và phân phối tài nguyên

## Nội dung trên slide

### **Kế thừa chính sách**

```
ACME:
- Cấm Privileged Container
- Cấm HostPath
- Bắt buộc khai báo CPU và RAM

Kỹ thuật:
- Kế thừa chính sách ACME
- Chỉ cho phép Registry nội bộ

Backend:
- Kế thừa ACME và Kỹ thuật
- Chỉ cho phép Image đã được phê duyệt
```

### Nguyên tắc

> Tenant con được phép siết chặt, nhưng không được nới lỏng chính sách của Tenant cha.

### Phân phối tài nguyên

Tự động tạo trong các Namespace:

- NetworkPolicy.
- LimitRange.
- ResourceQuota.
- ConfigMap.
- ImagePullSecret.
- Các tài nguyên tùy chỉnh của nền tảng.

Capsule hiện hỗ trợ `TenantResource` và `GlobalTenantResource` để tự động phân phối tài nguyên qua các Namespace. HieraTenant có thể mở rộng cơ chế này theo toàn bộ Subtree.

## Lời nói

> Chính sách của Tenant cha được kế thừa xuống toàn bộ cây bên dưới.
> 
> Ví dụ ACME cấm Privileged Container và HostPath. Phòng Kỹ thuật và tất cả các nhóm bên dưới không thể tắt các chính sách này.
> 
> Phòng Kỹ thuật có thể thêm yêu cầu chỉ sử dụng Registry nội bộ. Backend có thể tiếp tục siết chặt hơn bằng cách chỉ cho phép một tập Image cụ thể.
> 
> Nguyên tắc quan trọng là Tenant con được phép chặt hơn, nhưng không được phép nới lỏng giới hạn của Tenant cha.
> 
> Ngoài admission policy, hệ thống còn phải tự động phân phối các tài nguyên chung. Cluster Admin có thể định nghĩa một NetworkPolicy hoặc LimitRange ở cấp ACME và tài nguyên đó được tạo trong tất cả Namespace bên dưới.
> 
> Quá trình này tránh việc cấu hình thủ công và bảo đảm mọi Namespace mới đều nhận được bộ chính sách cơ bản.

---

# Slide 13 — Cách ly trong Shared Cluster

## Nội dung trên slide

### **Các lớp cách ly**

### Quyền truy cập

- Tenant chỉ quản lý tài nguyên trong Subtree của mình.
- Ngăn đọc Secret và Workload của Tenant khác.

### Mạng

- Mặc định chặn giao tiếp giữa các Tenant.
- Cho phép giao tiếp có kiểm soát trong cùng Subtree.

### Workload

- Cấm `privileged`, `hostPID`, `hostNetwork`, `hostPath`.
- Giới hạn loại Service và cấu hình nguy hiểm.

### Hạ tầng

- Giới hạn Node Pool.
- Giới hạn StorageClass.
- Giới hạn IngressClass và tên miền.
- Giới hạn Container Registry.

Capsule đã cung cấp nhiều loại enforcement ở cấp Tenant như node selector, StorageClass, IngressClass, hostname, registry và Service type.

## Lời nói

> Hierarchy không có ý nghĩa nếu các Tenant vẫn có thể truy cập hoặc ảnh hưởng trực tiếp lẫn nhau.
> 
> HieraTenant cần triển khai nhiều lớp cách ly.
> 
> Lớp đầu tiên là RBAC, ngăn người dùng đọc hoặc sửa tài nguyên của nhánh khác.
> 
> Lớp thứ hai là NetworkPolicy. Mặc định traffic giữa hai Tenant khác nhau bị chặn. Các Namespace trong cùng Tenant hoặc cùng Subtree có thể được cho phép giao tiếp theo chính sách.
> 
> Lớp thứ ba là bảo vệ Workload. Admission Webhook chặn các cấu hình như Privileged Container, HostPID, HostNetwork hoặc HostPath.
> 
> Lớp cuối cùng là hạ tầng. Tenant Free có thể chỉ được chạy trên Node Pool Free và sử dụng StorageClass tiêu chuẩn. Tenant Premium có thể được sử dụng SSD và Node Pool riêng.
> 
> HieraTenant không cần tự viết CNI hoặc Scheduler. Nó sinh NetworkPolicy và chèn các yêu cầu scheduling để Kubernetes cùng CNI hiện tại thực thi.

---

# Slide 14 — API Proxy theo phạm vi Tenant

## Nội dung trên slide

### **Vấn đề với tài nguyên Cluster-scoped**

RBAC Kubernetes thường có hai lựa chọn:

```
Không cấp quyền LIST
→ Người dùng không xem được tài nguyên của mình.

Cấp quyền LIST
→ Người dùng nhìn thấy tài nguyên của toàn Cluster.
```

### Tenant-aware API Proxy

```
Cluster thực tế:
backend-dev
backend-prod
frontend-dev
data-prod

Backend Admin nhìn thấy:
backend-dev
backend-prod
```

### Có thể lọc

- Namespace.
- Node phù hợp với Tenant.
- StorageClass được phép.
- IngressClass được phép.
- PersistentVolume thuộc Tenant.
- Các tài nguyên Cluster-scoped được chỉ định.

Capsule Proxy được xây dựng để xử lý trường hợp Tenant Owner không thể list các tài nguyên Cluster-scoped thuộc phạm vi của mình.

## Lời nói

> Một vấn đề khó của Kubernetes RBAC là thao tác `list` trên tài nguyên Cluster-scoped.
> 
> Ví dụ Namespace là một tài nguyên Cluster-scoped. Nếu không cấp quyền list Namespace, Backend Admin không thể chạy `kubectl get namespaces`.
> 
> Nhưng nếu cấp quyền list trực tiếp, người dùng có thể nhìn thấy tên Namespace của toàn Cluster.
> 
> API Proxy đứng giữa người dùng và Kubernetes API Server. Proxy xác định người dùng thuộc Tenant nào rồi lọc kết quả theo toàn bộ Subtree của Tenant đó.
> 
> Backend Admin chỉ nhìn thấy các Namespace của Backend. Kỹ thuật Admin nhìn thấy Backend và Frontend. ACME Admin nhìn thấy tất cả Namespace của ACME.
> 
> Proxy cũng có thể lọc Node, StorageClass, IngressClass và các tài nguyên Cluster-scoped khác dựa trên chính sách của Tenant.

---

# Slide 15 — Kiến trúc hệ thống

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

## Lời nói

> Kiến trúc của hệ thống gồm ba thành phần chính.
> 
> Thành phần đầu tiên là Controller Manager. Nó theo dõi các Custom Resource như HierarchicalTenant và liên tục đồng bộ trạng thái mong muốn xuống Kubernetes.
> 
> Tenant Tree Controller quản lý quan hệ cha – con và phát hiện vòng lặp.
> 
> Namespace Controller quản lý ownership và lifecycle của Namespace.
> 
> RBAC Controller tạo các RoleBinding phù hợp. Quota Controller tổng hợp usage và kiểm tra ngân sách của từng Subtree.
> 
> Policy và Resource Propagation Controller phân phối chính sách cùng các tài nguyên mặc định xuống Tenant con và Namespace.
> 
> Thành phần thứ hai là Admission Webhook. Validating Webhook kiểm tra các thao tác có hợp lệ hay không, ví dụ quota có vượt Tenant cha hoặc người dùng có quyền tạo SubTenant không.
> 
> Mutating Webhook bổ sung label, annotation, node selector hoặc giá trị mặc định vào tài nguyên.
> 
> Thành phần thứ ba là API Proxy. Proxy lọc các tài nguyên Cluster-scoped theo phạm vi của người dùng.
> 
> Hệ thống không cần tự viết Authentication, Scheduler, Network Plugin hoặc Storage Driver. Những chức năng đó tiếp tục sử dụng Kubernetes và hạ tầng hiện tại.

### Các CRD nên có trong MVP

```
HierarchicalTenant
TenantPolicy
TenantResourceTemplate
ProxyPolicy
```

Có thể đặt quota, owner và parent trực tiếp trong `HierarchicalTenant` để tránh tạo quá nhiều CRD ở phiên bản đầu.

---

# Slide 16 — Kiểm thử và đánh giá bằng KWOK

## Nội dung trên slide

### **Kiểm thử khả năng mở rộng bằng KWOK**

Mô phỏng:

```
500–1.000 Node
500 Tenant
2.000 SubTenant
5.000 Namespace
100.000 Pod
```

### Kịch bản đánh giá

- Tạo hàng loạt Tenant và SubTenant.
- Thay đổi policy ở Tenant gốc.
- Tổng hợp quota trên cây lớn.
- Tạo hàng nghìn Namespace đồng thời.
- Kiểm tra Scheduling Isolation.
- Mô phỏng Node chuyển sang `NotReady`.
- Đo độ trễ Admission Webhook và Controller.

### Chỉ số

- Thời gian reconcile.
- Thời gian truyền policy.
- Thời gian cập nhật quota.
- Độ trễ tạo Namespace.
- CPU và RAM của Controller.
- Số lỗi hoặc trạng thái không đồng bộ.

## Lời nói

> KWOK không phải một thành phần chạy trong HieraTenant. Nó được sử dụng để kiểm thử khả năng mở rộng của hệ thống.
> 
> KWOK cho phép tạo hàng trăm hoặc hàng nghìn Node giả mà không cần tương ứng từng máy thật.
> 
> Tôi có thể tạo 500 Tenant, 2.000 SubTenant, 5.000 Namespace và 100.000 Pod để kiểm tra Controller.
> 
> Một kịch bản là thay đổi chính sách ở Tenant ACME và đo thời gian để chính sách được truyền xuống toàn bộ cây.
> 
> Một kịch bản khác là làm 100 Node chuyển sang trạng thái NotReady rồi kiểm tra các Pod của từng Tenant có còn tuân thủ Node Pool hay không.
> 
> Tôi cũng sẽ đo độ trễ của Admission Webhook, thời gian tổng hợp quota và tài nguyên CPU, RAM mà Controller sử dụng.
> 
> Tuy nhiên, KWOK không chạy container thật nên không thể đánh giá chính xác hiệu năng mạng, Storage, CPU contention hay mức cách ly Kernel.

---

# Slide 17 — Hạn chế và phạm vi sử dụng

## Nội dung trên slide

### **Hạn chế**

- Các Tenant vẫn dùng chung API Server, etcd và Scheduler.
- Nếu dùng chung Worker Node, các Workload vẫn có thể dùng chung Kernel.
- CRD và các tài nguyên Cluster-scoped vẫn thuộc toàn Cluster.
- Hierarchy chỉ biểu diễn một chiều sở hữu.
- Quota không đồng nghĩa với bảo đảm Capacity vật lý.
- Proxy, Webhook và Controller trở thành thành phần quan trọng cần tính sẵn sàng cao.
- Không phù hợp để cho người dùng hoàn toàn không tin cậy chạy mã tùy ý.

### Phù hợp với

- Nền tảng phát triển nội bộ.
- Công ty có nhiều phòng ban và nhóm.
- Trường đại học, phòng thí nghiệm.
- Nền tảng CI/CD và môi trường thử nghiệm.
- Khách hàng doanh nghiệp có kiểm soát.

### Không thay thế

```
vCluster
Dedicated Cluster
Runtime Sandbox
```

## Lời nói

> HieraTenant cải thiện quản trị Shared Cluster nhưng không loại bỏ hoàn toàn các nhược điểm của Shared Cluster.
> 
> Tất cả Tenant vẫn dùng chung API Server, etcd và Scheduler. Vì vậy sự cố Control Plane vẫn có thể ảnh hưởng đến toàn bộ Tenant.
> 
> Nếu các Pod chạy chung Worker Node bằng runtime thông thường, chúng vẫn sử dụng chung Kernel. HieraTenant không thay thế Kata Containers, gVisor hoặc Node riêng.
> 
> Các CRD vẫn là tài nguyên Cluster-scoped. Hai Tenant không thể tự cài hai phiên bản xung đột của cùng một CRD như trong vCluster.
> 
> Quota cũng chỉ là giới hạn quản trị. Việc cấp 100 CPU quota không bảo đảm Cluster thực tế luôn có 100 CPU sẵn sàng.
> 
> Vì vậy giải pháp phù hợp nhất với các nhóm nội bộ hoặc khách hàng đã được kiểm soát, chứ không phải nền tảng công khai cho bất kỳ ai chạy container tùy ý.
> 
> Khi cần Control Plane riêng, CRD riêng hoặc mức cách ly mạnh hơn, vCluster hoặc Dedicated Cluster vẫn là lựa chọn phù hợp hơn.