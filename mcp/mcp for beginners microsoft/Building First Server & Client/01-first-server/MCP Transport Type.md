## 1. STDIO Transport

**STDIO** là viết tắt của "Standard Input/Output" - đây là transport đơn giản nhất trong MCP.

**Cách hoạt động:**

- MCP client và server giao tiếp qua stdin (đầu vào chuẩn) và stdout (đầu ra chuẩn)
- Server được khởi chạy như một subprocess của client
- Client gửi JSON-RPC messages qua stdin của server
- Server phản hồi bằng cách gửi messages qua stdout của nó

**Ưu điểm:**

- Rất đơn giản để implement và debug
- Không cần network configuration
- Bảo mật cao vì chỉ hoạt động locally
- Phù hợp cho development và testing

**Nhược điểm:**

- Chỉ hoạt động trên cùng một máy
- Server phải được khởi chạy mỗi khi cần sử dụng
- Không thể share server giữa nhiều clients

**Ví dụ sử dụng:**

json

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "python",
      "args": ["-m", "mcp_server_filesystem"],
      "transport": "stdio"
    }
  }
}
```

## 2. SSE Transport (Server-Sent Events)

**SSE** cho phép MCP hoạt động qua HTTP với khả năng server push messages đến client.

**Cách hoạt động:**

- Client kết nối đến HTTP endpoint của MCP server
- Client gửi messages qua HTTP POST requests
- Server có thể push messages đến client qua SSE stream
- Sử dụng hai endpoints: một cho messages, một cho SSE stream

**Ưu điểm:**

- Hoạt động qua network, có thể remote
- Tương thích tốt với web browsers
- Server có thể push notifications
- Dễ dàng load balance và scale

**Nhược điểm:**

- Phức tạp hơn STDIO
- Cần handle HTTP authentication và security
- Yêu cầu network configuration

**Ví dụ cấu trúc:**

```
POST /messages - Để gửi JSON-RPC messages
GET /events - SSE endpoint để nhận server-pushed events
```


INFO:     Started server process [13084]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)     
INFO:     127.0.0.1:59401 - "GET /sse HTTP/1.1" 200 OK
INFO:     127.0.0.1:59404 - "POST /messages/?session_id=10e6e3991ab54805b33dad041e477197 HTTP/1.1" 202 Accepted
INFO:     127.0.0.1:59404 - "POST /messages/?session_id=10e6e3991ab54805b33dad041e477197 HTTP/1.1" 202 Accepted
## 3. Streamable HTTP Transport

**Streamable HTTP** là phiên bản nâng cao của HTTP transport, hỗ trợ streaming data.

**Cách hoạt động:**

- Tương tự SSE nhưng với khả năng streaming bidirectional
- Hỗ trợ streaming lớn data chunks
- Có thể handle long-running operations hiệu quả hơn
- Sử dụng HTTP/2 features như multiplexing

**Ưu điểm:**

- Hiệu suất cao cho large data transfers
- Hỗ trợ streaming bidirectional
- Tối ưu cho real-time applications
- Better resource utilization

**Nhược điểm:**

- Phức tạp nhất trong 3 loại transport
- Yêu cầu HTTP/2 support
- Khó debug hơn so với STDIO

INFO:     Waiting for application startup.
[07/19/25 13:31:31] INFO     StreamableHTTP     streamable_http_manager.py:111
                             session manager                                  
                             started                                          
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit) 
INFO:     127.0.0.1:59451 - "POST /mcp HTTP/1.1" 200 OK
INFO:     127.0.0.1:59449 - "POST /mcp HTTP/1.1" 202 Accepted
INFO:     127.0.0.1:59451 - "GET /mcp HTTP/1.1" 200 OK
## So sánh và Lựa chọn

**Khi nào dùng STDIO:**

- Development và testing local
- Simple tools và scripts
- Khi bảo mật là ưu tiên hàng đầu
- Không cần network access

**Khi nào dùng SSE:**

- Web applications
- Cần remote access đến MCP server
- Multiple clients cần connect đến cùng server
- Cần server-side notifications

**Khi nào dùng Streamable HTTP:**

- High-performance applications
- Large data processing
- Real-time applications
- Khi cần optimize network usage


Mỗi transport đều có vai trò riêng trong ecosystem MCP, tùy thuộc vào use case cụ thể mà bạn có thể chọn transport phù hợp nhất.