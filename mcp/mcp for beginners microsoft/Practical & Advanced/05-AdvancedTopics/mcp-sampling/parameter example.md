## **1. Temperature (Nhiệt độ)**

**Là gì?** Kiểm soát mức độ "sáng tạo" hay "ngẫu nhiên" của AI

**Ví dụ thực tế:**

- **Temperature = 0.0**: AI sẽ luôn chọn từ có xác suất cao nhất
    - Input: "Thủ đô của Việt Nam là..."
    - Output: "Hà Nội" (luôn luôn)
- **Temperature = 1.0**: AI có thể chọn nhiều từ khác nhau
    - Input: "Hôm nay tôi muốn ăn..."
    - Output có thể là: "phở", "bún", "cơm", "bánh mì"...

## **2. MaxTokens (Số token tối đa)**

**Là gì?** Giới hạn độ dài câu trả lời (1 token ≈ 1 từ tiếng Anh hoặc vài ký tự)

**Ví dụ:**

- `maxTokens = 10`: "Xin chào, tôi là AI trợ lý"
- `maxTokens = 100`: Có thể viết cả một đoạn văn dài

## **3. StopSequences (Chuỗi dừng)**

**Là gì?** Những từ/ký tự khi gặp sẽ dừng sinh văn bản

**Ví dụ:**

```
stopSequences = [".", "END"]
Input: "Viết một câu về Hà Nội"
Output: "Hà Nội là thủ đô xinh đẹp." (dừng khi gặp dấu chấm)
```

## **4. Top_p (Nucleus Sampling)**

**Là gì?** Chỉ chọn từ trong nhóm có tổng xác suất = top_p

**Ví dụ đơn giản:** Khi AI nghĩ từ tiếp theo cho "Tôi thích ăn..."

- "phở": 40% xác suất
- "bún": 30% xác suất
- "cơm": 20% xác suất
- "bánh": 5% xác suất
- "cỏ": 5% xác suất

Với `top_p = 0.9`: AI chỉ xem xét "phở", "bún", "cơm" (tổng = 90%) → Loại bỏ những lựa chọn kỳ lạ như "cỏ"

## **5. Top_k**

**Là gì?** Chỉ xem xét K từ có xác suất cao nhất

**Ví dụ:** Với `top_k = 3`: AI chỉ chọn trong 3 từ đầu ("phở", "bún", "cơm") → Đơn giản hơn top_p, chỉ đếm số lượng

## **6. Presence_penalty (Phạt sự xuất hiện)**

**Là gì?** Phạt những từ đã xuất hiện (dù chỉ 1 lần)

**Ví dụ:**

- `presence_penalty = 0`: "Tôi thích phở. Phở rất ngon. Phở Hà Nội..."
- `presence_penalty = 2`: "Tôi thích phở. Món này rất ngon. Đặc sản Hà Nội..." → Tránh lặp lại ý tưởng

## **7. Frequency_penalty (Phạt tần suất)**

**Là gì?** Phạt theo số lần xuất hiện (càng xuất hiện nhiều càng bị phạt nặng)

**Ví dụ:**

- `frequency_penalty = 0`: "rất rất rất ngon"
- `frequency_penalty = 2`: "cực kỳ ngon" (tránh lặp từ "rất")

## **8. Seed (Hạt giống ngẫu nhiên)**

**Là gì?** Số để tạo kết quả giống nhau mỗi lần chạy

**Ví dụ:**

```
seed = 12345
Lần 1: "Hôm nay trời đẹp quá"
Lần 2: "Hôm nay trời đẹp quá" (giống hệt)

Không có seed:
Lần 1: "Hôm nay trời đẹp quá"  
Lần 2: "Thời tiết hôm nay thật tuyệt"
```

## **Kết hợp các tham số:**

python

```python
# Viết email chuyên nghiệp
temperature = 0.3      # Ít sáng tạo, chính xác
maxTokens = 200       # Đủ cho 1 email
presence_penalty = 0.5 # Tránh lặp ý

# Viết truyện sáng tạo  
temperature = 0.9      # Rất sáng tạo
maxTokens = 1000      # Viết dài
frequency_penalty = 1  # Tránh lặp từ
```