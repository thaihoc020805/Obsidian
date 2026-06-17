tools (khi list_tools)

```json
{
  "meta": null,
  "nextCursor": null,
  "tools": [
    {
      "name": "add",
      "title": null,
      "description": "Add two numbers",
      "inputSchema": {
        "properties": {
          "a": {
            "title": "A",
            "type": "integer"
          },
          "b": {
            "title": "B",
            "type": "integer"
          }
        },
        "required": [
          "a",
          "b"
        ],
        "title": "addArguments",
        "type": "object"
      },
      "outputSchema": {
        "properties": {
          "result": {
            "title": "Result",
            "type": "integer"
          }
        },
        "required": [
          "result"
        ],
        "title": "addOutput",
        "type": "object"
      },
      "annotations": null,
      "meta": null
    }
  ]
}
```

- `"name": "add"`: Tên định danh **duy nhất** cho công cụ. Đây là cách AI sẽ gọi nó.
    
- `"description": "Add two numbers"`: **Phần quan trọng nhất**. AI dựa vào mô tả này để hiểu công cụ làm gì và khi nào nên sử dụng nó. Mô tả càng rõ ràng, AI càng thông minh.
    
- `"inputSchema"`: Định nghĩa các tham số (dữ liệu đầu vào) mà công cụ cần.
    
    - `"properties"`: Liệt kê các tham số (`a` và `b`).
        
    - `"type": "integer"`: Khai báo kiểu dữ liệu cho từng tham số.
        
    - `"required": ["a", "b"]`: Báo rằng cả hai tham số `a` và `b` đều là **bắt buộc**.
        
- `"outputSchema"`: Định nghĩa dữ liệu đầu ra mà công cụ sẽ trả về (ở đây là một đối tượng có chứa `"result"` là một số nguyên).

convert_to_llm_tool

```json
{
  "type": "function",
  "function": {
    "name": tool.name,
    "description": tool.description,
    "type": "function",
    "parameters": {
      "type": "object",
      "properties":tool.inputSchema["properties"]
    }
  }
}
```

gửi tới llm
```json
{
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant."
    },
    {
      "role": "user",
      "content": prompt (from user)
    }
  ],
  "model": model_name,
  "tools": functions(list các cái convert_to_llm_tool ở trên),
  "temperature": 1.0,
  "max_tokens": 1000,
  "top_p": 1.0
}
```

temperature=1. - Độ sáng tạo:

- 0.0 = rất chính xác, ít sáng tạo

- 1.0 = cân bằng

- 2.0 = rất sáng tạo, có thể "bịa đặt"

max_tokens=1000 - Giới hạn độ dài phản hồi:

- Tối đa 1000 từ/token trong câu trả lời

top_p=1. - Độ đa dạng từ vựng:

- 1.0 = sử dụng toàn bộ từ vựng

- 0.1 = chỉ dùng từ có xác suất cao


```json
{
  "choices": [
    {
      "content_filter_results": {
        "hate": {
          "filtered": false,
          "severity": "safe"
        },
        "protected_material_code": {
          "filtered": false,
          "detected": false
        },
        "protected_material_text": {
          "filtered": false,
          "detected": false
        },
        "self_harm": {
          "filtered": false,
          "severity": "safe"
        },
        "sexual": {
          "filtered": false,
          "severity": "safe"
        },
        "violence": {
          "filtered": false,
          "severity": "safe"
        }
      },
      "finish_reason": "tool_calls",
      "index": 0,
      "logprobs": null,
      "message": {
        "annotations": [],
        "content": "Here are some popular cities in the world:\n\n1. New York\n2. London\n3. Paris\n4. Tokyo\n5. Sydney\n\nIf we consider each city's count as \"1,\" the total number of cities would be **5**. Let's now add this to **100**. Calculating...",
        "refusal": null,
        "role": "assistant",
        "tool_calls": [
          {
            "function": {
              "arguments": "{\"a\":5,\"b\":100}",
              "name": "add"
            },
            "id": "call_QwxslqD9V1R36jTQHLxClT9D",
            "type": "function"
          }
        ]
      }
    }
  ],
  "created": 1752925858,
  "id": "chatcmpl-Bv0NeOHX4giM4o0eahjD2yqsf326Z",
  "model": "gpt-4o-2024-11-20",
  "object": "chat.completion",
  "prompt_filter_results": [
    {
      "prompt_index": 0,
      "content_filter_results": {
        "hate": {
          "filtered": false,
          "severity": "safe"
        },
        "jailbreak": {
          "filtered": false,
          "detected": false
        },
        "self_harm": {
          "filtered": false,
          "severity": "safe"
        },
        "sexual": {
          "filtered": false,
          "severity": "safe"
        },
        "violence": {
          "filtered": false,
          "severity": "safe"
        }
      }
    }
  ],
  "system_fingerprint": "fp_ee1d74bde0",
  "usage": {
    "completion_tokens": 83,
    "completion_tokens_details": {
      "accepted_prediction_tokens": 0,
      "audio_tokens": 0,
      "reasoning_tokens": 0,
      "rejected_prediction_tokens": 0
    },
    "prompt_tokens": 74,
    "prompt_tokens_details": {
      "audio_tokens": 0,
      "cached_tokens": 0
    },
    "total_tokens": 157
  }
}
```

### ## Tổng quan 🤖

Đây là các trường thông tin chung về phản hồi:

- **`id`**: Một chuỗi định danh duy nhất cho yêu cầu này. Rất hữu ích để theo dõi và gỡ lỗi.
    
- **`object`**: Loại đối tượng, ở đây là `"chat.completion"`, xác nhận đây là kết quả từ một cuộc hội thoại.
    
- **`created`**: Dấu thời gian (timestamp) khi phản hồi được tạo ra.
    
- **`model`**: Tên của mô hình AI đã xử lý yêu cầu (ví dụ: `"gpt-4o-2024-11-20"`).
    
- **`system_fingerprint`**: Một mã định danh cho cấu hình hệ thống backend đã tạo ra phản hồi này, giúp việc tái tạo và báo lỗi dễ dàng hơn.
    

---

### ## Nội dung chính (`choices`)

Đây là phần quan trọng nhất, chứa câu trả lời của AI. Nó là một danh sách (array), nhưng thường chỉ có một phần tử.

- **`index: 0`**: Chỉ số của lựa chọn này trong danh sách (bắt đầu từ 0).
    
- **`finish_reason: "tool_calls"`** 🔑: **Đây là trường quyết định**. Nó cho biết lý do AI dừng tạo văn bản.
    
    - `"stop"`: AI đã trả lời xong.
        
    - `"length"`: AI dừng vì đã đạt đến giới hạn token.
        
    - `"tool_calls"`: AI muốn gọi một hoặc nhiều công cụ bên ngoài.
        
- **`message`**: Đối tượng chứa nội dung do AI tạo ra.
    
    - **`role: "assistant"`**: Xác nhận đây là tin nhắn từ AI.
        
    - **`content`**: Đây là phần "suy nghĩ" của AI. Nó giải thích cho người dùng biết nó đang làm gì trước khi gọi tool. Trong ví dụ này, nó giải thích logic: đếm 5 thành phố và chuẩn bị cộng với 100.
        
    - **`tool_calls`** 💡: Đây là chỉ thị hành động cho hệ thống của bạn. Nó là một danh sách các công cụ cần gọi.
        
        - **`id`**: ID duy nhất cho lệnh gọi công cụ này (`call_...`). Bạn sẽ dùng ID này để gửi kết quả trở lại cho AI.
            
        - **`type: "function"`**: Loại công cụ là một hàm.
            
        - **`function`**: Chi tiết về hàm cần gọi.
            
            - **`name: "add"`**: Tên của hàm mà AI muốn bạn thực thi.
                
            - **`arguments: "{\"a\":5,\"b\":100}"`**: Các đối số mà AI đã tự suy luận ra. Nó được định dạng dưới dạng một chuỗi JSON. Hệ thống của bạn cần phân tích (parse) chuỗi này để lấy ra `{ a: 5, b: 100 }` và truyền vào hàm `add`.
                

---

### ## Bộ lọc an toàn 

`content_filter_results`: kết quả từ ai
`prompt_filter_results`: đầu vào từ user

Các khối này cho thấy kết quả kiểm duyệt nội dung của cả đầu vào (`prompt`) và đầu ra (`content`).

- **`filtered: false`**: Nội dung không bị chặn.
    
- **`severity: "safe"`**: Mức độ an toàn, cho thấy nội dung không chứa các yếu tố độc hại (thù ghét, tự hại, bạo lực, v.v.).
    

---

### ## Thống kê sử dụng (`usage`) 📊

Phần này rất quan trọng để quản lý chi phí và hiệu suất.

- **`prompt_tokens: 74`**: Số lượng token trong yêu cầu bạn gửi lên (bao gồm cả lịch sử trò chuyện và định nghĩa tool).
    
- **`completion_tokens: 83`**: Số lượng token trong phần `message` mà AI đã tạo ra.
    
- **`total_tokens: 157`**: Tổng số token đã sử dụng cho yêu cầu này. Đây là con số được dùng để tính phí.

gọi tool call

```json
{
    "name": "add",
    "args": {"a": 5, "b": 100} (dict)
  }
```

response khi gọi

```json
{
  "meta": null,
  "content": [
    {
      "type": "text",
      "text": "1100",
      "annotations": null,
      "meta": null
    }
  ],
  "structuredContent": {
    "result": 1100
  },
  "isError": false
}
```
`"structuredContent"`: Chứa kết quả dưới dạng cấu trúc dữ liệu