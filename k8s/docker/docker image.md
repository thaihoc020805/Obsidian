	![[Pasted image 20250727230423.png]]

the right is how to build an application in traditional way
and the left is how to build an image with dockerfile, then build with -t flag (tag name) (this will build your image locally on your system)
final, push it to docker hub registry

mmumshad/my-custom-app meaning account-name/image-name:lastest (in dockerhub, if don't specific the tag of the image, it will automately use lastest)


Dockerfile
![[Pasted image 20250727231259.png]]

from: start from a base OS or another image, Dockerfile required start with FROM instruction
run: run the command in those base image, you can run for install dependencies
copy: copy file in local system to docker image, the above copy source code
entrypoint: specify the command that will be run when the image is run as a container

layer
![[Pasted image 20250727233528.png]]

each layer only stores the changes from the previous layer

![[Pasted image 20250727233854.png]]


when face failure when build image, i will reus the previous layeres from cache and continue to build the remaining layers
it also useful when we change or create another step. For example, if we change the source code, it will re built this layers and others next layers 


cmd vs entrypoint
![[Pasted image 20250728000520.png]]

![[Pasted image 20250728000747.png]]


![[Pasted image 20250728000815.png]]

others instructions

### **ARG vs. ENV**

Cả `ARG` và `ENV` đều dùng để định nghĩa biến, nhưng chúng có mục đích và phạm vi sử dụng khác nhau.

#### **ARG (Argument - Tham số lúc build)**

- **Tác dụng:** `ARG` dùng để khai báo các biến **chỉ tồn tại trong quá trình build image**. Sau khi image được build xong, các biến `ARG` sẽ không còn tồn tại bên trong container nữa.
    
- **Dễ hiểu:** Hãy tưởng tượng `ARG` như là một "tham số đầu vào" cho bản thiết kế (Dockerfile). Bạn truyền giá trị cho nó khi bắt đầu "xây nhà" (build image), và nó chỉ được dùng bởi "thợ xây" (quá trình build). Khi "ngôi nhà" (image) hoàn thành, không ai trong nhà biết về các tham số đó nữa.
    
- **Ví dụ sử dụng:**
    
    - Chỉ định phiên bản của một phần mềm cần cài đặt mà không cần sửa trực tiếp Dockerfile.
        
    - Truyền vào các thông tin build-time như mã phiên bản (version tag).
        

**Dockerfile ví dụ:**

Dockerfile

```
ARG APP_VERSION=1.0
FROM alpine
LABEL version=${APP_VERSION} # Sử dụng biến ARG
RUN echo "Building version ${APP_VERSION}" # Biến này có thể dùng trong quá trình build
```

Khi build, bạn có thể truyền giá trị khác: `docker build --build-arg APP_VERSION=2.0 -t my-app .`

---

#### **ENV (Environment - Biến môi trường)**

- **Tác dụng:** `ENV` dùng để thiết lập các **biến môi trường** bên trong image và container. Các biến này sẽ **luôn tồn tại** khi container được chạy từ image đó.
    
- **Dễ hiểu:** `ENV` giống như việc bạn "treo một tờ ghi chú" bên trong ngôi nhà. Bất cứ ai vào ở (chạy container) đều có thể đọc và sử dụng thông tin trên tờ ghi chú đó.
    
- **Ví dụ sử dụng:**
    
    - Cấu hình thông tin kết nối database (username, password, host).
        
    - Thiết lập cổng (port) mà ứng dụng sẽ lắng nghe.
        
    - Cấu hình môi trường hoạt động (development, staging, production).
        

**Dockerfile ví dụ:**

Dockerfile

```
# Sử dụng ARG để thiết lập giá trị mặc định cho ENV
ARG DB_HOST=localhost
ENV DATABASE_HOST=${DB_HOST}

FROM alpine
# Hoặc gán trực tiếp
ENV APP_PORT=8080

CMD ["echo", "Ứng dụng đang chạy trên cổng ${APP_PORT} và kết nối tới ${DATABASE_HOST}"]
```

Khi chạy trong Docker:
- Sử dụng `ENV` trong `Dockerfile` hoặc truyền biến môi trường qua cờ `-e` của lệnh `docker run` (ví dụ: `docker run -e MY_VAR=value ...`).
- Trong code Python, chỉ cần dùng `os.environ.get()` để đọc các biến này.

|Tiêu chí|`ARG`|`ENV`|
|---|---|---|
|**Phạm vi**|Chỉ tồn tại lúc build image|Tồn tại cả lúc build và lúc chạy container|
|**Mục đích**|Tùy biến quá trình build|Cấu hình cho ứng dụng bên trong container|
|**Giá trị**|Có thể được truyền từ dòng lệnh `docker build`|Có thể được ghi đè lúc chạy container (`docker run -e`)|

---

### ## **WORKDIR**

- **Tác dụng:** Chỉ thị `WORKDIR` dùng để **thiết lập thư mục làm việc hiện tại** cho các chỉ thị theo sau nó trong Dockerfile (`RUN`, `CMD`, `ENTRYPOINT`, `COPY`, `ADD`).
    
- **Dễ hiểu:** Thay vì phải gõ đường dẫn đầy đủ cho mỗi lệnh, bạn dùng `WORKDIR` để "di chuyển" vào một thư mục. Mọi lệnh sau đó sẽ được thực thi từ bên trong thư mục này. Nếu thư mục chưa tồn tại, Docker sẽ tự động tạo ra nó.
    
- **Ví dụ:**
    

**Cách không nên làm (lặp lại đường dẫn):**

Dockerfile

```
FROM ubuntu
RUN mkdir /app
COPY . /app
RUN cd /app && npm install
CMD ["/app/start.sh"]
```

**Cách tốt hơn với `WORKDIR`:**

Dockerfile

```
FROM ubuntu
WORKDIR /app  # Thiết lập thư mục làm việc là /app

COPY . .      # Copy file vào /app
RUN npm install # Chạy lệnh trong /app
CMD ["./start.sh"] # Thực thi file trong /app
```

---

### ## **HEALTHCHECK**

- **Tác dụng:** `HEALTHCHECK` cung cấp một cách để **kiểm tra "sức khỏe" của container**, xem ứng dụng bên trong nó có đang hoạt động bình thường hay không. Docker sẽ chạy một lệnh kiểm tra định kỳ bên trong container.
    
- **Dễ hiểu:** Tưởng tượng bạn có một "bác sĩ" tự động kiểm tra xem ứng dụng của bạn có còn "khỏe" không. Nếu ứng dụng không phản hồi đúng cách sau vài lần kiểm tra, nó sẽ được đánh dấu là `unhealthy` (không khỏe mạnh). Điều này rất hữu ích cho các hệ thống tự động như Docker Swarm hay Kubernetes để khởi động lại container bị lỗi.
    
- **Trạng thái của container:**
    
    - `starting`: Trạng thái ban đầu, đang chờ kết quả kiểm tra đầu tiên.
        
    - `healthy`: Lệnh kiểm tra trả về kết quả thành công.
        
    - `unhealthy`: Lệnh kiểm tra thất bại liên tiếp quá số lần cho phép.
        

**Dockerfile ví dụ (cho một web server):**

Dockerfile

```
FROM nginx
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost/ || exit 1
```

- `--interval=30s`: Cứ 30 giây kiểm tra một lần.
    
- `--timeout=3s`: Nếu lệnh kiểm tra chạy quá 3 giây thì coi như thất bại.
    
- `--retries=3`: Nếu thất bại 3 lần liên tiếp, container sẽ bị đánh dấu là `unhealthy`.
    
- `CMD curl ...`: Lệnh để kiểm tra. `curl -f` sẽ trả về mã lỗi nếu HTTP status code là lỗi (ví dụ: 404, 500).

có thể check bằng docker ps ( ở cột STATUS ) hoặc xem chi tiết với docker inspect.

Lưu ý: `CMD` bên trong `HEALTHCHECK` **không phải và không liên quan** đến chỉ thị `CMD` chính của Dockerfile. Nó chỉ là một từ khóa thuộc về cú pháp của `HEALTHCHECK`.

---

### ## **USER**

- **Tác dụng:** Chỉ thị `USER` dùng để **thiết lập người dùng (user)** sẽ thực thi các lệnh `RUN`, `CMD`, `ENTRYPOINT` tiếp theo.
    
- **Dễ hiểu & Tầm quan trọng:** Mặc định, các lệnh trong container được chạy với quyền **root** (quản trị viên cao nhất). Điều này tiềm ẩn rủi ro bảo mật, vì nếu một kẻ tấn công chiếm được quyền điều khiển ứng dụng, họ sẽ có toàn quyền trên container. `USER` cho phép bạn chuyển sang một người dùng có quyền hạn thấp hơn, giúp giảm thiểu rủi ro này. Đây là một **thực hành bảo mật rất tốt**.
    
- **Ví dụ:**
    

Dockerfile

```
FROM alpine

# Tạo một user mới tên là 'appuser' không có quyền root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Chuyển sang user vừa tạo
USER appuser

# Từ đây, các lệnh sau sẽ được chạy bởi 'appuser'
WORKDIR /home/appuser
COPY --chown=appuser:appgroup . .
CMD ["./my-app"]
```

COPY --chown=appuser:appgroup . .  sẽ tương đương 

COPY . . 
RUN chown -R appuser:appgroup .