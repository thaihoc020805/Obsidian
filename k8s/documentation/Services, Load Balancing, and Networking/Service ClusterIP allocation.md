## 1. `10.96.0.0/24` nghĩa là gì?

- Đây là **CIDR notation** để mô tả một mạng con (subnet).
    
- `/24` nghĩa là **24 bit đầu** là phần network, còn **8 bit cuối** là phần host.
    
    - IPv4 có 32 bit → 32 - 24 = 8 bit host.
        
    - 8 bit host có thể tạo được 28=2562^8 = 25628=256 địa chỉ IP.
        

---

## 2. Tại sao lại chỉ có **254 usable IP**?

- Trong IPv4, **2 địa chỉ đặc biệt** không dùng cho host:
    
    1. **Network address**: địa chỉ đầu tiên (ở đây là `10.96.0.0`) → dùng để định danh mạng.
        
    2. **Broadcast address**: địa chỉ cuối cùng (ở đây là `10.96.0.255`) → dùng để gửi cho toàn bộ host trong mạng.
        
- Vậy usable IP = tổng số địa chỉ - 2:
    
    256−2=254256 - 2 = 254256−2=254
    
    Các usable IP ở đây là từ `10.96.0.1` → `10.96.0.254`.
    

---

## 3. Minh họa cho `10.96.0.0/24`

|Loại địa chỉ|Giá trị|Ghi chú|
|---|---|---|
|Network address|10.96.0.0|Không gán cho host|
|Usable IP (host)|10.96.0.1 → 10.96.0.254|254 địa chỉ|
|Broadcast address|10.96.0.255|Không gán cho host|

---

💡 **Tóm lại:**

- `/24` → 256 địa chỉ tổng cộng.
    
- Trừ network + broadcast → 254 usable IP cho máy chủ, container, pod, v.v.


CIDR là viết tắt của **Classless Inter-Domain Routing** — tiếng Việt hay gọi là **định tuyến liên miền không phân lớp**.

Nói dễ hiểu, đây là **cách viết và chia mạng IP** giúp:

1. Tiết kiệm địa chỉ IP.
    
2. Linh hoạt hơn so với cách chia theo “class A/B/C” cũ.
    

---

## 1. Cách viết CIDR

CIDR viết dưới dạng:

php-template

CopyEdit

`<Địa chỉ mạng>/<Số bit mạng>`

Ví dụ:

CopyEdit

`10.96.0.0/24`

- `10.96.0.0` → địa chỉ mạng (network address).
    
- `/24` → 24 bit đầu tiên cố định cho phần network, còn lại là phần host.
    

---

## 2. Ý nghĩa của con số sau dấu `/`

- IPv4 = 32 bit.
    
- Số sau `/` = **số bit dành cho network**.
    
- Số bit còn lại = **dành cho host** (máy tính, server… trong mạng).
    

Ví dụ:

|CIDR|Bit network|Bit host|Tổng IP|Usable IP|
|---|---|---|---|---|
|/24|24|8|256|254|
|/16|16|16|65536|65534|
|/30|30|2|4|2|

---

## 3. Lợi ích CIDR

- **Linh hoạt**: Có thể tạo mạng con (subnet) vừa đủ số host cần.
    
- **Tiết kiệm**: Không bị giới hạn bởi Class A/B/C cũ.
    
- **Routing gọn hơn**: ISP có thể gộp nhiều mạng nhỏ thành một route lớn.
    

---

📌 **Tóm tắt**  
CIDR = Cách biểu diễn + chia subnet bằng số bit network.  
Ví dụ `192.168.1.0/24` nghĩa là:

- Network: `192.168.1.0`
    
- Host range: `192.168.1.1 → 192.168.1.254`
    
- Broadcast: `192.168.1.255`



## 1. ClusterIP là gì?

- Mỗi **Service** trong Kubernetes kiểu `ClusterIP` được cấp 1 **IP ảo** trong cluster (không ra ngoài được).
    
- Pod bên trong cluster chỉ cần gọi IP này → K8s sẽ tự load balance tới Pod phù hợp.
    
- Ví dụ: `my-service` có ClusterIP `10.96.0.15`, khi Pod gọi `http://10.96.0.15` thì K8s sẽ điều phối tới Pod backend.
    

---

## 2. IP này được cấp như thế nào?

Có **2 cách**:

1. **Dynamic (tự động)**  
    → Kubernetes chọn 1 IP **chưa ai dùng** trong range đã cấu hình (`--service-cluster-ip-range`).
    
2. **Static (tự đặt)**  
    → Bạn tự chọn IP (phải nằm trong range).  
    Ví dụ:
    
    yaml
    
    CopyEdit
    
    `spec:   clusterIP: 10.96.0.10`
    

> ❗ Mỗi IP này phải **duy nhất toàn cluster**, nếu trùng sẽ báo lỗi.

---

## 3. Vì sao cần IP static?

- Một số service quan trọng cần **IP cố định** để các app khác dễ gọi.
    
- Ví dụ: DNS của cluster (CoreDNS) thường được set `10.96.0.10` → Pod nào cũng biết DNS ở đâu.
    

---

## 4. Nguy cơ khi dùng IP static

- Nếu bạn chưa tạo service đó mà service khác tạo trước → IP bạn muốn có thể bị chiếm → Tạo service sẽ lỗi **"IP conflict"**.
    

---

## 5. Cách Kubernetes tránh xung đột IP

K8s chia range IP dịch vụ ra **2 vùng**:

- **Vùng tĩnh (Static band)**: dành cho bạn đặt IP thủ công → nằm ở **đầu range**.
    
- **Vùng động (Dynamic band)**: kube-apiserver sẽ lấy IP từ **cuối range** trước → giúp giảm nguy cơ đụng vùng tĩnh.
    

Công thức chia:

sql

CopyEdit

`Static band size = min(max(16, tổng_số_IP / 16), 256)`

→ Luôn ít nhất 16 IP tĩnh, nhiều nhất 256 IP tĩnh.

---

## 6. Ví dụ dễ hiểu

### Ví dụ 1:

Range = `10.96.0.0/24` → Có 254 IP usable  
Static band size = `min(max(16, 256/16=16), 256)` = 16

- Static: `10.96.0.1 → 10.96.0.16` (để bạn đặt tay)
    
- Dynamic: `10.96.0.17 → 10.96.0.254` (K8s tự cấp)
    

---

### Ví dụ 2:

Range = `10.96.0.0/20` → Có 4094 IP usable  
Static band size = `min(max(16, 4096/16=256), 256)` = 256

- Static: `10.96.0.1 → 10.96.1.0`
    
- Dynamic: `10.96.1.1 → 10.96.15.254`
    

---

### Ví dụ 3:

Range = `10.96.0.0/16` → Có 65534 IP usable  
Static band size = `min(max(16, 65536/16=4096), 256)` = 256

- Static: `10.96.0.1 → 10.96.1.0`
    
- Dynamic: `10.96.1.1 → 10.96.255.254`
    

---

## 7. Tóm tắt siêu ngắn

- **ClusterIP** = IP ảo cho service bên trong cluster.
    
- Kubernetes **ưu tiên cấp IP từ cuối range** → để bạn thoải mái đặt IP tĩnh ở đầu range.
    
- Static band = vùng “đầu” IP, kích thước từ 16 đến 256 IP.
    
- Nếu cần IP cố định → lấy ở vùng static band để tránh bị chiếm.