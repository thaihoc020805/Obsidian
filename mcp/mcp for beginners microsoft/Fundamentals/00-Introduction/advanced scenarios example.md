### **Ví dụ Kịch bản A: Gọi Công cụ Trực tiếp (Direct Tool Calling)**

Kịch bản này giống như ra một mệnh lệnh rõ ràng và nhận lại kết quả ngay lập tức.

- **Ngữ cảnh:** Bạn đang sử dụng một ứng dụng trợ lý ảo có tích hợp công cụ tìm kiếm.
    
- **Bước 7 - Lời nhắc của người dùng:**
    
    > **"Thời tiết ở Hà Nội ngày mai thế nào?"**
    
- **Luồng hoạt động diễn ra như sau:**
    
    1. **Client LLM (LLM phía Client)** nhận ra đây là một câu hỏi cần dữ liệu thời gian thực.
        
    2. Nó quét qua danh sách công cụ và tìm thấy một công cụ phù hợp, ví dụ như `get_weather_forecast`.
        
    3. **Client LLM** yêu cầu **Client App**: "Hãy thực thi công cụ `get_weather_forecast` với tham số `location='Hanoi'` và `date='tomorrow'`".
        
    4. **Client App** gửi yêu cầu này đến **MCP Server 1** (máy chủ chứa công cụ thời tiết).
        
    5. **Server 1** thực thi và trả về dữ liệu thô, ví dụ: `{ "date": "2025-07-18", "temp_high": 34, "temp_low": 26, "condition": "Nắng gián đoạn" }`.
        
    6. **Client LLM** nhận dữ liệu này và chuyển nó thành một câu trả lời tự nhiên.
        
    7. **Client App** hiển thị cho bạn:
        
        > **"Dự báo thời tiết ngày mai tại Hà Nội: Nhiệt độ cao nhất 34°C, thấp nhất 26°C, trời có nắng gián đoạn."**
        

➡️ **Kết luận:** Yêu cầu rõ ràng (`thời tiết`) được ánh xạ trực tiếp tới một công cụ có sẵn (`get_weather_forecast`).

---

### **Ví dụ Kịch bản B: Đàm phán Tính năng (Feature Negotiation)**

Kịch bản này phức tạp hơn, giống như bạn đưa ra một yêu cầu mở và hệ thống cần "thảo luận" để tìm ra cách tốt nhất để thực hiện.

- **Ngữ cảnh:** Bạn đang sử dụng một trình soạn thảo mã (như VS Code) tích hợp AI và bạn vừa viết một đoạn mã Python phức tạp.
    
- **Bước 7 - Lời nhắc của người dùng:**
    
    > **"Tối ưu hóa đoạn mã này cho tôi."**
    
- **Luồng hoạt động diễn ra như sau:**
    
    1. **Client LLM** nhận ra yêu cầu "tối ưu hóa mã" rất mơ hồ. Nó không tương ứng với một công cụ đơn giản nào như `tìm_lỗi_chính_tả`. Nó cần một _khả năng_ phân tích và viết lại mã.
        
    2. **Client LLM** thông báo cho **Client App**: "Tôi cần khả năng phân tích và tối ưu hóa mã Python."
        
    3. **Client App** gửi yêu cầu này đến **MCP Server 2** (máy chủ chuyên về phân tích mã): "Client cần tối ưu hóa mã Python, bạn có những tính năng nào?"
        
    4. **Server 2** có thể hỏi **Server LLM** để hiểu rõ hơn. **Server LLM** gợi ý: "Hãy cung cấp cho client các công cụ để kiểm tra hiệu năng (profiling), phân tích độ phức tạp thuật toán, và gợi ý cách viết lại mã."
        
    5. **Server 2** trả lời **Client App**: "Tôi có các tính năng sau: `run_code_profiler`, `analyze_complexity`, và `suggest_refactoring`."
        
    6. Bây giờ **Client App** (dưới sự chỉ dẫn của **Client LLM**) biết phải làm gì. Nó lần lượt gọi các công cụ đã được "đàm phán":
        
        - Gọi `run_code_profiler` để xem hàm nào chạy chậm nhất.
            
        - Gọi `analyze_complexity` để kiểm tra độ phức tạp của thuật toán.
            
        - Dựa trên kết quả trên, gọi `suggest_refactoring` để lấy phiên bản mã tốt hơn.
            
    7. **Client LLM** nhận lại phiên bản mã đã được tối ưu hóa cùng với giải thích.
        
    8. **Client App** hiển thị cho bạn:
        
        > **"Đã tối ưu hóa mã của bạn. Vòng lặp lồng nhau ban đầu đã được thay thế bằng phương pháp tra cứu dictionary để cải thiện hiệu năng từ O(n²) xuống O(n). Đây là phiên bản mã mới..."**
        

➡️ **Kết luận:** Yêu cầu mơ hồ (`tối ưu hóa`) đã khởi động một quá trình "đàm phán" để khám phá và sử dụng một tập hợp các công cụ chuyên sâu, thay vì chỉ gọi một lệnh duy nhất.