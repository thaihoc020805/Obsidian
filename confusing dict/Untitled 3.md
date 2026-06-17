**SemVer 2.0.0 (Semantic Versioning 2.0.0 standard)** là một chuẩn đặt tên phiên bản phần mềm để đảm bảo tính **dự đoán, rõ ràng và nhất quán**. Nó được dùng rộng rãi trong open-source lẫn enterprise (ví dụ: npm, Maven, Docker tags).

---

## 📌 Cấu trúc phiên bản

Một phiên bản hợp lệ có dạng:

`MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]`

Ví dụ:  
`1.4.2-alpha.3+001`

Trong đó:

1. **MAJOR** → thay đổi **không tương thích** (breaking changes).  
    Ví dụ: `1.x.x → 2.0.0`
    
2. **MINOR** → thêm tính năng mới nhưng **giữ backward compatibility**.  
    Ví dụ: `1.2.x → 1.3.0`
    
3. **PATCH** → sửa lỗi, tối ưu, **không thay đổi API**.  
    Ví dụ: `1.3.0 → 1.3.1`
    
4. **PRERELEASE (tuỳ chọn)** → gắn nhãn cho phiên bản chưa stable (alpha, beta, rc).  
    Ví dụ: `1.3.0-alpha.1`
    
5. **BUILD METADATA (tuỳ chọn)** → thêm thông tin build/CI, không ảnh hưởng thứ tự sắp xếp phiên bản.  
    Ví dụ: `1.3.0+20230821`
    

---

## 📌 Quy tắc sắp xếp thứ tự (precedence)

- So sánh lần lượt **MAJOR → MINOR → PATCH** (theo số nguyên, không padding).
    
- `1.10.0` > `1.2.9`
    
- Nếu có **PRERELEASE**, thì coi là **nhỏ hơn bản stable** cùng version:  
    `1.0.0-alpha < 1.0.0`
    
- **BUILD metadata** chỉ để tham khảo, **không ảnh hưởng thứ tự**:  
    `1.0.0+abc = 1.0.0+xyz`
    

---

## 📌 Lợi ích khi dùng SemVer

- Dễ dự đoán khi nâng cấp: nhìn số version là biết có breaking change hay không.
    
- Hỗ trợ tốt dependency management trong package managers (npm, pip, go mod...).
    
- Tạo chuẩn chung giữa developer, tester, DevOps và end-user.
    

---

👉 Tóm gọn:

- **MAJOR** = phá API cũ.
    
- **MINOR** = thêm tính năng mới, vẫn tương thích.
    
- **PATCH** = fix bug, giữ nguyên API.
    
- Có thể kèm **alpha/beta/rc** và **build metadata**.