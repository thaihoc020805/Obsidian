## 1. Vấn đề trước khi có Internal Traffic Policy

- Bạn có 2 Pod chạy trên **cùng một node**.
    
- Chúng nói chuyện với nhau qua **Service**.
    
- Bình thường, **kube-proxy** sẽ chọn **bất kỳ Pod nào trong cluster** làm endpoint (có thể ở node khác).
    
- Nếu chọn Pod ở **node khác**:
    
    1. Request phải **đi qua network giữa các node**.
        
    2. Tốn băng thông, tốn thời gian (latency), dễ bị ảnh hưởng nếu network giữa các node bị chậm hoặc lỗi.
        

📌 Tức là… dù **2 Pod đứng cạnh nhau**, chúng vẫn… gửi thư qua bưu điện toàn quốc.

---

## 2. Giải pháp: Internal Traffic Policy = Local

- Khi bạn set:
    

yaml

CopyEdit

`internalTrafficPolicy: Local`

- **Kube-proxy** chỉ chọn **Pod trên cùng node** với nơi request bắt đầu.
    
- Nếu trên node đó **không có Pod nào** cho Service đó ⇒ Service coi như **không có endpoint** (cho Pod ở node đó).
    

📌 Giống như việc **ship hàng nội quận** — chỉ nhận đơn từ quận mình. Không có hàng ở quận ⇒ báo “hết hàng”.

---

## 3. Lợi ích

- **Tốc độ**: Không đi qua cross-node network ⇒ latency thấp hơn.
    
- **Tiết kiệm**: Không tiêu tốn băng thông mạng giữa các node.
    
- **Ổn định**: Ít phụ thuộc vào network giữa các node ⇒ giảm rủi ro nếu network cluster bị trục trặc.
    

---

## 4. Hạn chế

- Nếu node không có endpoint ⇒ Pod trên node đó **không truy cập được Service**, dù node khác vẫn có Pod.
    
- Không phù hợp với Service mà **Pod phân bố không đều** hoặc **ít replica**.
    
- Không áp dụng cho traffic từ bên ngoài cluster (chỉ áp dụng **internal traffic** giữa Pod → Service).
    

---

## 5. Ví dụ dễ hiểu

Giả sử có **Service my-service** với 4 Pod:

less

CopyEdit

`Node A: pod1, pod2 Node B: pod3, pod4`

- **internalTrafficPolicy: Cluster** (mặc định):  
    Pod1 có thể gọi đến pod3 hoặc pod4 (Node B), hoặc pod2 (Node A).
    
- **internalTrafficPolicy: Local**:  
    Pod1 **chỉ** gọi đến pod2 (cùng Node A). Không bao giờ chọn pod3, pod4.