### ConfigMap là gì?

Hãy tưởng tượng bạn đang viết một ứng dụng. Ứng dụng này cần một vài thông tin cài đặt để hoạt động, ví dụ như:

- Địa chỉ kết nối đến cơ sở dữ liệu (database).
    
- Số mạng sống ban đầu của người chơi trong game.
    
- Tên của một file cấu hình giao diện.
    

Thay vì "viết cứng" những giá trị này trực tiếp vào code của ứng dụng, bạn có thể lưu chúng vào một đối tượng riêng của Kubernetes gọi là **ConfigMap**.

**Nói đơn giản:** ConfigMap giống như một "bảng ghi chú" hoặc một "tệp cài đặt" bên ngoài, chứa các cặp thông tin dạng **khóa - giá trị** (key-value). Ứng dụng của bạn (chạy trong một Pod) có thể "đọc" cái bảng ghi chú này để lấy thông tin cấu hình cần thiết.

**Ví dụ:**

- **Khóa:** `DATABASE_HOST`, **Giá trị:** `mysql-service.prod`
    
- **Khóa:** `player_initial_lives`, **Giá trị:** `"3"`
    

---

### Tại sao chúng ta cần ConfigMap? (Động lực)

Lợi ích lớn nhất của ConfigMap là **tách biệt cấu hình ra khỏi mã nguồn ứng dụng**.

Hãy xem xét ví dụ thực tế:

1. **Khi bạn code trên máy tính (môi trường phát triển):** Bạn muốn ứng dụng kết nối tới database tại địa chỉ `localhost`.
    
2. **Khi bạn triển khai ứng dụng lên "đám mây" Kubernetes (môi trường sản phẩm):** Ứng dụng phải kết nối tới một Service database có địa chỉ là `my-database-service`.
    

Nếu không có ConfigMap, bạn sẽ phải sửa code, build lại container image cho mỗi môi trường khác nhau. Rất mất công!

Với ConfigMap, bạn chỉ cần:

- Giữ nguyên code ứng dụng, lúc nào cũng đọc cấu hình từ một biến môi trường tên là `DATABASE_HOST`.
    
- Tạo một ConfigMap cho môi trường phát triển với `DATABASE_HOST: localhost`.
    
- Tạo một ConfigMap khác cho môi trường sản phẩm với `DATABASE_HOST: my-database-service`.
    

Như vậy, ứng dụng của bạn trở nên "di động", có thể chạy ở bất cứ đâu mà không cần sửa code.

---

### Những điều cần lưu ý (Cảnh báo & Giới hạn)

- **⚠️ Cảnh báo Bảo mật:** ConfigMap **KHÔNG** mã hóa dữ liệu. Nó được lưu dưới dạng văn bản thuần túy (plain text). Vì vậy, **tuyệt đối không lưu các thông tin nhạy cảm** như mật khẩu, API key, token... vào ConfigMap. Đối với dữ liệu bí mật, hãy dùng **Secret**.
    
- **Giới hạn Kích thước:** ConfigMap không dùng để lưu trữ những khối dữ liệu lớn. Tổng kích thước dữ liệu trong một ConfigMap **không thể vượt quá 1 MiB**. Nếu cần lưu trữ nhiều hơn, bạn nên dùng một volume (ổ đĩa mạng) hoặc một dịch vụ database/file riêng.
    

---

### Cấu trúc của một ConfigMap

Một ConfigMap có 2 trường chính để chứa dữ liệu:

- `data`: Dành cho dữ liệu dạng chuỗi ký tự (văn bản) theo chuẩn UTF-8. Đây là trường phổ biến nhất.
    
- `binaryData`: Dành cho dữ liệu nhị phân (binary), được lưu dưới dạng chuỗi mã hóa base64.
    

Các key (tên của khóa) trong `data` và `binaryData` không được trùng nhau.

Dưới đây là một ví dụ về file YAML định nghĩa một ConfigMap tên là `game-demo`:

YAML

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # Các khóa dạng thuộc tính: mỗi khóa ứng với một giá trị đơn giản
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # Các khóa dạng file: giá trị là nội dung của cả một file cấu hình
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```

---

### 4 Cách để Pod sử dụng ConfigMap

Pod và ConfigMap phải ở trong cùng một **namespace** (không gian tên) để có thể làm việc với nhau. Có 4 cách chính để một container trong Pod "tiêu thụ" dữ liệu từ ConfigMap:

1. **Làm đối số cho dòng lệnh (command & args) của container.**
    
2. **Làm biến môi trường (environment variables) cho container.**
    
3. **Tạo thành các file trong một volume chỉ đọc (read-only volume).**
    
4. **Viết code bên trong Pod để gọi trực tiếp Kubernetes API và đọc ConfigMap.** (Cách này linh hoạt hơn, cho phép ứng dụng tự nhận biết khi ConfigMap thay đổi và có thể đọc ConfigMap từ namespace khác).
    

Hai cách phổ biến và dễ dùng nhất là cách 2 và 3. Chúng ta sẽ đi sâu vào hai cách này.

#### Cách 1: Sử dụng ConfigMap làm Biến Môi Trường

Đây là cách bạn "bơm" các giá trị từ ConfigMap vào các biến môi trường của container.

- **`envFrom`:** Dùng để lấy **tất cả** các cặp key-value trong ConfigMap và biến chúng thành các biến môi trường. Tên biến môi trường sẽ chính là tên key.
    
    YAML
    
    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod
    spec:
      containers:
      - name: myapp
        image: busybox
        command: ["sleep", "3600"]
        envFrom:
          - configMapRef:
              name: game-demo # Tên của ConfigMap
    ```
    
    Trong container `myapp` sẽ có các biến môi trường: `player_initial_lives`, `ui_properties_file_name`, `game.properties`, v.v.
    
- **`env` và `valueFrom`:** Dùng khi bạn chỉ muốn lấy **một vài key cụ thể** và có thể đặt tên biến môi trường khác với tên key.
    
    YAML
    
    ```
    # ...spec của Pod...
      containers:
      - name: myapp
        image: alpine
        command: ["sleep", "3600"]
        env:
          # Định nghĩa biến môi trường
          - name: PLAYER_INITIAL_LIVES # Tên biến môi trường trong container
            valueFrom:
              configMapKeyRef:
                name: game-demo           # Tên ConfigMap
                key: player_initial_lives  # Tên key cần lấy giá trị
    ```
    

**Quan trọng:** Biến môi trường được tạo theo cách này **KHÔNG tự động cập nhật** nếu ConfigMap gốc thay đổi. Bạn phải khởi động lại Pod để nhận giá trị mới.

#### Cách 2: Sử dụng ConfigMap làm File trong Volume

Đây là cách bạn biến mỗi key trong ConfigMap thành một file riêng biệt bên trong container.

**Các bước thực hiện:**

1. Trong `spec` của Pod, định nghĩa một `volumes` và trỏ nó đến ConfigMap của bạn.
    
2. Trong `spec` của container, tạo một `volumeMounts` để "gắn" (mount) volume đó vào một thư mục bên trong container.
    

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      volumeMounts:
      - name: config-volume      # Tên phải khớp với volume ở dưới
        mountPath: /config        # Thư mục bên trong container
        readOnly: true
  
  # Bạn định nghĩa volume ở cấp Pod
  volumes:
  - name: config-volume         # Đặt một cái tên cho volume
    configMap:
      name: game-demo           # Tên ConfigMap bạn muốn sử dụng
```

**Kết quả:**

- Nếu bạn để như trên, Kubernetes sẽ tạo ra một thư mục `/config` trong container `demo`.
    
- Bên trong thư mục `/config`, sẽ có 4 file tương ứng với 4 key trong ConfigMap `game-demo`:
    
    - `player_initial_lives` (nội dung là "3")
        
    - `ui_properties_file_name` (nội dung là "user-interface.properties")
        
    - `game.properties` (nội dung là multi-line như đã định nghĩa)
        
    - `user-interface.properties` (nội dung là multi-line)
        

**Tùy chọn `items`:** Nếu bạn chỉ muốn mount một vài key nhất định và có thể đổi tên file:

YAML

```
# ...
  volumes:
  - name: config-volume
    configMap:
      name: game-demo
      items:
      - key: "game.properties"
        path: "game.properties" # Tên file trong thư mục mount
      - key: "user-interface.properties"
        path: "user-interface.properties"
```

Với cấu hình này, chỉ có 2 file `game.properties` và `user-interface.properties` được tạo ra trong thư mục `/config`.

**Cập nhật tự động:**

- **Điểm cộng lớn:** Khi ConfigMap được mount dưới dạng volume, nếu bạn cập nhật ConfigMap gốc, các file bên trong Pod sẽ **được cập nhật tự động** sau một khoảng thời gian ngắn (thường là khoảng 1 phút). Ứng dụng của bạn có thể đọc lại file để lấy cấu hình mới mà không cần khởi động lại Pod.
    
- **Ngoại lệ:** Một container sử dụng ConfigMap với `subPath` trong volume mount sẽ **KHÔNG** nhận được cập nhật tự động.
    

---

### ConfigMap Bất Biến (Immutable ConfigMaps)

Từ phiên bản Kubernetes v1.21, bạn có thể tạo ra một ConfigMap "bất biến" bằng cách thêm `immutable: true`.

YAML

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-immutable-config
data:
  my-key: my-value
immutable: true
```

**Tại sao lại dùng nó?**

1. **An toàn:** Bảo vệ bạn khỏi những cập nhật vô tình hoặc không mong muốn có thể làm sập ứng dụng.
    
2. **Hiệu suất:** Cải thiện đáng kể hiệu suất của cluster. Khi Kubernetes biết một ConfigMap là bất biến, nó sẽ không cần phải "theo dõi" (watch) sự thay đổi của nó nữa, giúp giảm tải cho hệ thống (kube-apiserver).
    

**Lưu ý:** Một khi đã được đánh dấu là `immutable`, bạn **KHÔNG** thể thay đổi lại trạng thái này hay chỉnh sửa nội dung `data` của nó. Cách duy nhất để cập nhật là **xóa ConfigMap đi và tạo lại một cái mới**.