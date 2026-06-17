![[Pasted image 20250728120659.png]]

nginx = docker.io/nginx/nginx


![[Pasted image 20250728120742.png]]


![[Pasted image 20250728120902.png]]


Lệnh `docker image tag` dùng để **tạo một tên mới (bí danh - alias) cho một image đã tồn tại**.

Nó không tạo ra một bản sao mới của image, mà chỉ đơn giản là gắn thêm một cái nhãn khác vào cùng một image đó. Việc này cực kỳ hữu ích và là một thao tác không thể thiếu khi làm việc với Docker.

---

### Các tác dụng chính

#### 1. Quản lý phiên bản (Versioning) 🏷️

Đây là tác dụng phổ biến nhất. Giống như phiên bản của phần mềm (ví dụ: `v1.0`, `v1.1`), tag giúp bạn phân biệt các phiên bản khác nhau của image.

- Khi bạn xây dựng một image mới, bạn có thể tag nó là `my-app:1.0`.
    
- Sau khi sửa lỗi và xây dựng lại, bạn tag phiên bản mới là `my-app:1.1`.
    
- Bạn cũng có thể tag phiên bản mới nhất là `my-app:latest`.
    

Điều này cho phép bạn hoặc người khác có thể chọn chính xác phiên bản image cần dùng, ví dụ `docker run my-app:1.0`, hoặc luôn dùng phiên bản mới nhất với `docker run my-app:latest`.

#### 2. Đẩy image lên Registry (Pushing to a Registry) ☁️

Các dịch vụ lưu trữ image như Docker Hub yêu cầu image phải có tên theo định dạng `<tên_người_dùng>/<tên_image>:<tag>`.

Ví dụ, bạn vừa build một image có tên là `my-app`. Để đẩy nó lên Docker Hub, bạn cần:

1. **Tag lại image:** `docker image tag my-app:latest your-username/my-app:latest`
    
2. **Đẩy image đã tag:** `docker push your-username/my-app:latest`
    

Lệnh tag ở đây tạo ra một cái tên mới phù hợp với yêu cầu của Docker Hub, trỏ đến image gốc của bạn.

---

**Ví dụ dễ hiểu:** Hãy tưởng tượng một người có nhiều tên gọi (tên khai sinh, tên ở nhà, biệt danh), nhưng tất cả đều chỉ cùng một người. Tương tự, một Docker image có thể có nhiều tag (`my-app:1.0`, `my-app:latest`, `your-username/my-app:latest`), nhưng tất cả đều trỏ đến cùng một Image ID duy nhất.