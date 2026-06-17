# Giải thích “serverless” thật dễ hiểu

Hãy tưởng tượng mở quán nước:

- Cách cũ: Tự thuê mặt bằng, mua bàn ghế, tuyển nhân viên, lo điện nước… Dù có khách hay không vẫn phải trả tiền cố định.
    
- Serverless: Thuê “dịch vụ quán” trọn gói theo lượt khách. Mỗi khi có khách, dịch vụ tự mở quầy, phục vụ xong thì đóng lại. Không có khách thì không tốn tiền.
    

Trong công nghệ cũng vậy:

- “Serverless” không phải là “không có server”, mà là nhà cung cấp đám mây lo hết chuyện máy chủ cho mình (mua máy, cài đặt, bảo trì, mở rộng).
    
- Lập trình viên chỉ viết “hàm” làm một việc cụ thể (ví dụ: xử lý một request, resize ảnh, gửi email). Khi có sự kiện xảy ra, đám mây tự “gọi” hàm đó chạy. Xong việc thì thôi.
    

## Điểm chính cần nhớ

- Không quản lý máy chủ: Không phải cài đặt Linux, không phải vá lỗi bảo mật, không lo hỏng ổ cứng.
    
- Tự động mở rộng: Nhiều người dùng cùng lúc thì hệ thống tự nhân bản để phục vụ; ít người dùng thì tự co lại.
    
- Trả tiền theo dùng: Chỉ trả cho thời gian hàm chạy và số lần gọi, không phải trả 24/7.
    
- Kích hoạt theo sự kiện: HTTP request, file được upload, message từ hàng đợi, lịch cron… đều có thể “gọi” hàm.
    

## Ví dụ đời thường

- Khi có người tải ảnh lên, một “hàm” tự động chạy để tạo ảnh thumbnail, rồi lưu lại.
    
- API “/order” được gọi thì một hàm xử lý đơn hàng, ghi vào cơ sở dữ liệu, gửi email xác nhận.
    
- Hàng đêm 2:00, một hàm chạy tự động để tổng hợp báo cáo.
    

## Ưu và nhược điểm (nói ngắn gọn)

- Ưu:
    
    - Nhanh ra sản phẩm vì khỏi lo hạ tầng.
        
    - Tiết kiệm chi phí khi lưu lượng không đều.
        
    - Tự động co giãn theo tải.
        
- Nhược:
    
    - Thỉnh thoảng “khởi động lạnh” làm chậm yêu cầu đầu tiên.
        
    - Bị giới hạn thời gian chạy, bộ nhớ; không hợp tác vụ dài.
        
    - Dễ “dính” vào dịch vụ của một nhà cung cấp (khó chuyển đổi).
        

## Khi nào nên dùng

- Ứng dụng có lưu lượng lên xuống thất thường (MVP, chiến dịch marketing).
    
- Xử lý theo sự kiện, theo lô: resize ảnh, gửi thông báo, ETL dữ liệu.
    
- API nhỏ, microservice, webhook.
    

## Khi nào chưa phù hợp

- Nhu cầu chạy liên tục dài giờ, kết nối lâu (streaming nặng, game server).
    
- Yêu cầu độ trễ cực thấp, ổn định từng mili-giây.
    
- Cần kiểm soát hạ tầng rất sâu (compliance đặc thù).
    

Nếu muốn, có thể mô tả use case cụ thể (ví dụ: web bán hàng, app chia sẻ ảnh…) và mình sẽ phác thảo cách làm kiểu serverless từng bước cho dễ hình dung.