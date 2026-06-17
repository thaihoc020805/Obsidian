### Tại sao chúng ta cần các loại "Probe" (Đầu dò)?

Hãy tưởng tượng bạn có một container đang chạy. Kubernetes rất giỏi trong việc biết được process chính của container đó có đang tồn tại hay không. Nếu process đó bị "chết" (crash), Kubernetes sẽ tự động khởi động lại container.

**Nhưng vấn đề là:** Đôi khi, ứng dụng của bạn vẫn "chạy" (process vẫn tồn tại) nhưng lại bị treo (deadlock), không thể xử lý yêu cầu, hoặc đang trong quá trình khởi tạo rất lâu và chưa sẵn sàng nhận traffic. Kubernetes không thể tự biết được những trạng thái này.

Đó là lúc các **Probe** ra đời. Probes là cách để Kubernetes "hỏi thăm sức khỏe" của ứng dụng bên trong container.

Hãy hình dung ứng dụng của bạn là một **quán cà phê**:

---

### ❤️ Liveness Probe (Đầu dò "Sống")

- **Câu hỏi nó đặt ra:** "Này anh bạn, bạn có còn 'sống' không hay đã bị treo rồi?"
    
- **Analogy quán cà phê:** Liệu người pha chế (barista) có đang bị đứng hình, bất động không? Dù anh ta vẫn đang đứng đó (process đang chạy), nhưng lại không thể làm gì cả.
    
- **Mục đích:** Dùng để phát hiện các tình huống ứng dụng bị treo, deadlock, hoặc rơi vào một trạng thái lỗi không thể phục hồi dù process vẫn còn.
    
- **Hành động khi thất bại:** Nếu Liveness Probe thất bại liên tục, Kubelet sẽ cho rằng container này không còn cứu vãn được nữa và sẽ **KHỞI ĐỘNG LẠI (RESTART)** container đó. Đây là một cơ chế tự chữa lành (self-healing) cực kỳ mạnh mẽ.
    

**Điểm quan trọng:** Liveness Probe không đợi Readiness Probe thành công. Để tránh việc nó khởi động lại container quá sớm, bạn nên dùng `initialDelaySeconds` (đợi một lúc rồi mới bắt đầu dò) hoặc tốt hơn là dùng Startup Probe.

---

### ✅ Readiness Probe (Đầu dò "Sẵn sàng")

- **Câu hỏi nó đặt ra:** "Bạn đã sẵn sàng để nhận khách (traffic) chưa?"
    
- **Analogy quán cà phê:** Máy pha cà phê đã đủ nóng chưa? Nguyên liệu đã chuẩn bị xong chưa? Nếu chưa, chúng ta không nên mở cửa cho khách vào vội, kẻo họ phải chờ và có trải nghiệm tồi tệ.
    
- **Mục đích:**
    
    1. **Khi khởi động:** Đảm bảo ứng dụng đã hoàn thành các tác vụ tốn thời gian (như kết nối database, tải file, làm nóng cache) trước khi bắt đầu nhận traffic.
        
    2. **Trong lúc chạy:** Báo hiệu rằng ứng dụng đang tạm thời quá tải hoặc đang trong quá trình bảo trì và không nên nhận thêm traffic mới.
        
- **Hành động khi thất bại:** Nếu Readiness Probe thất bại, Kubernetes sẽ **GỠ BỎ POD RA KHỎI DANH SÁCH ENDPOINT CỦA SERVICE**. Điều này có nghĩa là Service sẽ tạm thời **ngừng gửi traffic mới** đến Pod này. Pod **KHÔNG** bị khởi động lại. Khi Readiness Probe thành công trở lại, Pod sẽ được thêm vào lại và tiếp tục nhận traffic.
    

**Điểm quan trọng:** Readiness Probe chạy trong suốt vòng đời của container.

---

### 🚀 Startup Probe (Đầu dò "Khởi động")

- **Câu hỏi nó đặt ra:** "Bạn có phải là một ứng dụng cần rất nhiều thời gian để khởi động không? Hãy cho tôi biết khi nào bạn khởi động xong nhé."
    
- **Analogy quán cà phê:** Hôm nay là ngày khai trương! Quán cần 5-10 phút để sắp xếp bàn ghế, cắm điện, chuẩn bị mọi thứ. Trong thời gian này, đừng hỏi "anh có bị đứng hình không?" (Liveness) hay "máy pha cà phê nóng chưa?" (Readiness) vội, hãy cho chúng tôi thời gian.
    
- **Mục đích:** Dành riêng cho các ứng dụng khởi động chậm. Nó sẽ "vô hiệu hóa" Liveness và Readiness Probe cho đến khi nó thành công. Điều này giúp các ứng dụng "nặng" không bị Liveness Probe "giết nhầm" trước khi chúng kịp sẵn sàng.
    
- **Hành động khi thất bại:** Nếu Startup Probe thất bại liên tục (vượt quá ngưỡng cho phép), Kubelet sẽ cho rằng container đã khởi động lỗi và sẽ **KHỞI ĐỘNG LẠI (RESTART)** container.
    
- **Khi nào nó chạy?** Loại probe này **CHỈ** chạy lúc container khởi động. Một khi nó thành công, nó sẽ không bao giờ chạy lại nữa, và quyền kiểm soát sẽ được giao lại cho Liveness và Readiness Probe.
    

---

### Bảng so sánh tổng hợp

|Tiêu chí|Liveness Probe (Sống)|Readiness Probe (Sẵn sàng)|Startup Probe (Khởi động)|
|---|---|---|---|
|**Mục đích**|Kiểm tra container có bị treo/đơ không?|Kiểm tra container có sẵn sàng nhận traffic không?|Kiểm tra ứng dụng khởi động chậm đã xong chưa?|
|**Hành động khi thất bại**|**Khởi động lại (Restart)** container.|**Ngừng gửi traffic mới** đến Pod.|**Khởi động lại (Restart)** container.|
|**Khi nào chạy?**|Trong suốt vòng đời của container.|Trong suốt vòng đời của container.|**Chỉ** lúc container khởi động.|
|**Analogy**|Người pha chế có bị đứng hình không?|Máy pha cà phê đã nóng chưa?|Quán đã set up xong cho ngày khai trương chưa?|