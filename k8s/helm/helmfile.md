releases:
  - name: helloworld
    chart: ./helloworld
    installed: true

- **`releases`**: Danh sách các Helm release mà bạn muốn quản lý.  
    Mỗi item là một release.
    
- **`name: helloworld`**  
    Đây là tên của release (tương tự như khi bạn gõ `helm install helloworld ./helloworld`).  
    Helm sẽ dùng `helloworld` làm tên release để track và quản lý.
    
- **`chart: ./helloworld`**  
    Đường dẫn tới chart cần cài. Ở đây là chart local trong thư mục `./helloworld`.  
    Ngoài ra, có thể chỉ định chart từ repo (VD: `bitnami/nginx`).
    
- **`installed: true`**  
    Cờ bật/tắt. Nếu `true` → Helmfile sẽ cài release này.  
    Nếu `false` → Helmfile sẽ không cài hoặc sẽ uninstall nếu nó đang tồn tại.
    

---

### Cách dùng

Với file Helmfile (thường tên `helmfile.yaml`), bạn chỉ cần chạy:

`helmfile apply`

→ Nó sẽ đọc toàn bộ danh sách `releases` và cài/upgrade chart tương ứng.

---

👉 Nói ngắn gọn:

- `name`: tên release trong Helm.
    
- `chart`: chart nào để deploy.
    
- `installed: true`: bật/tắt việc cài chart.

Hai lệnh `helmfile apply` và `helmfile sync` trông giống nhau, nhưng có khác biệt quan trọng:

---

## 🔹 `helmfile apply`

- **Giống như `terraform apply`**.
    
- Quy trình:
    
    1. Helmfile render toàn bộ release (theo `helmfile.yaml` và values).
        
    2. So sánh với trạng thái thực tế trên cluster.
        
    3. Hiển thị diff (giống `helm diff`).
        
    4. Hỏi xác nhận (nếu có).
        
    5. Nếu có thay đổi → mới chạy `helm upgrade/install/uninstall`.
        

👉 Nói ngắn gọn: **chỉ apply khi có diff**.  
Dùng khi bạn muốn chắc chắn rằng Helmfile sẽ thực hiện đúng, và thường trong CI/CD sẽ kết hợp `helmfile diff` trước.

---

## 🔹 `helmfile sync`

- **Giống như “force install/upgrade”**.
    
- Không quan tâm diff. Nó đảm bảo rằng trạng thái cluster sẽ **đồng bộ hoàn toàn** với file Helmfile:
    
    - Nếu chart chưa cài → cài.
        
    - Nếu chart đã cài → upgrade, bất kể có diff hay không.
        
    - Nếu `installed: false` → uninstall.
        

👉 Nói ngắn gọn: **luôn thực thi để cluster khớp với helmfile.yaml**.

---

## 🔑 So sánh nhanh

|Lệnh|Hành vi|
|---|---|
|`helmfile apply`|Tính toán diff, chỉ cài/upgrade/uninstall khi có thay đổi. An toàn hơn.|
|`helmfile sync`|Luôn chạy để đồng bộ chart trên cluster với helmfile.yaml.|

---

## 📌 Ví dụ

File `helmfile.yaml`:
```
releases:
  - name: helloworld
    chart: ./helloworld
    installed: true

```

- Nếu chart `helloworld` đã ở version đúng:
    
    - `helmfile apply` → không làm gì (vì không diff).
        
    - `helmfile sync` → vẫn gọi `helm upgrade` để đồng bộ lại.
        

---

👉 Nói dễ hiểu:

- `apply` = **chỉ chạy khi có khác biệt**.
    
- `sync` = **luôn ép cluster khớp với helmfile.yaml**.