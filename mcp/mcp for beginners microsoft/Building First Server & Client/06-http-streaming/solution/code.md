```python
# client.py
from mcp.client.streamable_http import streamablehttp_client
from mcp import ClientSession
import asyncio
import mcp.types as types
from mcp.shared.session import RequestResponder
import requests
import logging

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)
logger = logging.getLogger('mcp_client')

class LoggingCollector:
    def __init__(self):
        self.log_messages: list[types.LoggingMessageNotificationParams] = []
    async def __call__(self, params: types.LoggingMessageNotificationParams) -> None:
        self.log_messages.append(params)
        logger.info("MCP Log: %s - %s", params.level, params.data)

logging_collector = LoggingCollector()
port = 8000

async def message_handler(
    message: RequestResponder[types.ServerRequest, types.ClientResult]
    | types.ServerNotification
    | Exception,
) -> None:
    logger.info("Received message: %s", message)
    if isinstance(message, Exception):
        logger.error("Exception received!")
        raise message
    elif isinstance(message, types.ServerNotification):
        logger.info("NOTIFICATION: %s", message)
    elif isinstance(message, RequestResponder):
        logger.info("REQUEST_RESPONDER: %s", message)
    else:
        logger.info("SERVER_MESSAGE: %s", message)

async def main():
    logger.info("Starting client...")
    async with streamablehttp_client(f"http://localhost:{port}/mcp") as (
        read_stream,
        write_stream,
        session_callback,
    ):
        async with ClientSession(
            read_stream,
            write_stream,
            logging_callback=logging_collector,
            message_handler=message_handler,
        ) as session:
            id_before = session_callback()
            logger.info("Session ID before init: %s", id_before)
            await session.initialize()
            id_after = session_callback()
            logger.info("Session ID after init: %s", id_after)
            logger.info("Session initialized, ready to call tools.")
            tool_result = await session.call_tool("process_files", {"message": "hello from client"})
            logger.info("Tool result: %s", tool_result)
            if logging_collector.log_messages:
                logger.info("Collected log messages:")
                for log in logging_collector.log_messages:
                    logger.info("Log: %s", log)

def stream_progress(message="hello", url="http://localhost:8000/stream"):
    params = {"message": message}
    logger.info("Connecting to %s with message: %s", url, message)
    try:
        with requests.get(url, params=params, stream=True, timeout=10) as r:
            r.raise_for_status()
            logger.info("--- Streaming Progress ---")
            for line in r.iter_lines():
                if line:
                    # Still print the streamed content to stdout for visibility
                    decoded_line = line.decode().strip()
                    print(decoded_line)
                    logger.debug("Stream content: %s", decoded_line)
            logger.info("--- Stream Ended ---")
    except requests.RequestException as e:
        logger.error("Error during streaming: %s", e)

if __name__ == "__main__":
    import sys
    
    if len(sys.argv) > 1 and sys.argv[1] == "mcp":
        # MCP client mode
        logger.info("Running MCP client...")
        asyncio.run(main())
    else:
        # Classic HTTP streaming client mode
        logger.info("Running classic HTTP streaming client...")
        stream_progress()
        
    # Don't run both by default, let the user choose the mode
```

## 🔍 r.raise_for_status():

### Tác dụng: Kiểm tra HTTP status code và báo lỗi nếu có vấn đề

python

Apply to client.py

# Ví dụ:

response = requests.get("http://example.com/api")

# ✅ Nếu status 200, 201, 202... → Không làm gì

# ❌ Nếu status 404, 500, 403... → Throw exception

response.raise_for_status()  # Sẽ raise HTTPError nếu status >= 400

### So sánh:

python

Apply to client.py

# Không dùng raise_for_status()

if response.status_code != 200:

    print("Có lỗi!")

    return

# Dùng raise_for_status() (ngắn gọn hơn)

response.raise_for_status()  # Tự động check và báo lỗi

## 📡 stream=True:

### Tác dụng: Nhận data từng chunk thay vì download hết một lúc

python

Apply to client.py

# stream=False (mặc định) - Download hết rồi mới xử lý

response = requests.get(url)

data = response.content  # Chờ download xong hết

# stream=True - Nhận data từng phần

response = requests.get(url, stream=True)

for line in response.iter_lines():  # Xử lý từng dòng ngay

    print(line)  # In ra ngay khi nhận được

### Lợi ích:

- Memory tiết kiệm: Không cần lưu hết data trong RAM

- Real-time: Xử lý data ngay khi nhận được

- Phù hợp: Streaming, progress tracking, large files

## ⏱️ timeout=10:

### Tác dụng: Đặt thời gian chờ tối đa 10 giây

python

Apply to client.py

# Không timeout → Có thể chờ mãi mãi

response = requests.get(url)  # Nguy hiểm!

# Có timeout → Chờ tối đa 10s

response = requests.get(url, timeout=10)  # An toàn hơn

### Điều gì xảy ra:

- < 10s: Request thành công → OK

- > 10s: Throw requests.exceptions.Timeout → Dừng request



logger = logging.getLogger('mcp_client')

### 📝 Tác dụng:

- Tạo/lấy một logger có tên 'mcp_client'

- Nếu logger này đã tồn tại → Lấy ra dùng lại

- Nếu chưa tồn tại → Tạo mới

### 🏷️ Tên logger:

- 'mcp_client' = namespace cho log messages

- Giúp phân biệt log từ module khác nhau

- Hiển thị trong log output


## 🔄 1. RequestResponder[[types.ServerRequest, types.ClientResult]:

### Generic Pattern:

python

Apply to client.py

RequestResponder[ReceiveRequestType, SendResultType]

### Tác dụng:

- Wrapper cho requests cần response

- Quản lý request lifecycle (start → process → respond)

- Type safety: Đảm bảo đúng type input/output

### Ví dụ:

python

Apply to client.py

# Server gửi request → Client phải respond

# VD: Server hỏi client "Sampling completion for this text?"

responder = RequestResponder[ServerRequest, ClientResult](...)

with responder:

    result = await client.process_request()

    await responder.respond(result)  # Must respond!

## 📢 2. types.ServerNotification:

### Định nghĩa:

python

Apply to client.py

# Từ code: 

class ServerNotification(RootModel[

    CancelledNotification

    | ProgressNotification  

    | LoggingMessageNotification      # ← Logging từ server

    | ResourceUpdatedNotification

    | ResourceListChangedNotification

    | ToolListChangedNotification

    | PromptListChangedNotification

]):

### Tác dụng:

- One-way messages từ server → client

- Không cần response (khác với RequestResponder)

- Thông báo events: logs, progress, resource changes

### Ví dụ:

python

Apply to client.py

# Server gửi log → Client chỉ nhận, không cần trả lời

notification = LoggingMessageNotification(

    method="notifications/message",

    params=LoggingMessageNotificationParams(

        level="info", 

        data="Processing file..."

    )

)

## 🌐 3. async with streamablehttp_client(...):

python

Apply to client.py

async with streamablehttp_client(f"http://localhost:{port}/mcp") as (

    read_stream,      # ← Nhận messages từ server  

    write_stream,     # ← Gửi messages tới server

    session_callback, # ← Function để lấy session ID

):

### Context Manager Pattern:

- Setup: Tạo HTTP connection tới MCP server

- Cleanup: Tự động đóng connection khi exit

- Return: 3 objects để quản lý communication

### Chi tiết:

- read_stream: MemoryObjectReceiveStream - queue nhận messages

- write_stream: MemoryObjectSendStream - queue gửi messages

- session_callback: Callable[[], str | None] - lấy session ID

### Cách hoạt động:

python

Apply to client.py

# 1. Connect to http://localhost:8000/mcp

# 2. Setup bidirectional streams

# 3. Return streams để ClientSession sử dụng

# 4. Auto cleanup khi exit context

## 📋 4. types.LoggingMessageNotificationParams:

python

Apply to client.py

class LoggingMessageNotificationParams:

    level: str        # "info", "error", "warning", "debug"  

    data: str         # Actual log message

    logger: str | None = None   # Logger name (optional)

    meta: dict | None = None    # Metadata (optional)

### Ví dụ:

python

Apply to client.py

params = LoggingMessageNotificationParams(

    level="info",

    data="Processing file_1.txt (1/3)...",

    logger=None,

    meta=None

)

## 🎯 5. _call_ trong LoggingCollector:

python

Apply to client.py

class LoggingCollector:

    async def __call__(self, params: LoggingMessageNotificationParams) -> None:

        # ← __call__ makes this object "callable" like a function

        self.log_messages.append(params)

        logger.info("MCP Log: %s - %s", params.level, params.data)

### Khi nào được gọi:

1. Server gửi log: await ctx.info("Processing...")

2. Transport layer: Nhận LoggingMessageNotification

3. ClientSession: Detect đây là logging notification

4. Callback: Gọi logging_collector(params) → Trigger __call__

### Flow diagram:

text

Apply to client.py

Server: await ctx.info("Hello")

   ↓

HTTP: LoggingMessageNotification 

   ↓  

ClientSession: Phát hiện logging notification

   ↓

logging_callback(params) 

   ↓

LoggingCollector.__call__(params)  ← Đây!

   ↓

Lưu vào self.log_messages + In ra console

## 🎯 Tóm lại luồng hoạt động:

1. Connect: streamablehttp_client tạo streams

2. Setup: ClientSession với logging_collector callback

3. Tool execution: Server chạy tool, gửi logs qua ctx.info()

4. Receive: Client nhận ServerNotification (logging)

5. Callback: Trigger logging_collector.__call__()

6. Process: LoggingCollector lưu + in logs

7. RequestResponder: Nếu có requests cần response → xử lý riêng

Đây là architecture hoàn chỉnh cho MCP client-server communication! ✨

server

```python
# server.py
from fastapi import FastAPI
from fastapi.responses import StreamingResponse, HTMLResponse
from mcp.server.fastmcp import FastMCP, Context
from mcp.types import (
    TextContent
)
import asyncio
import uvicorn
import os

# Create an MCP server
mcp = FastMCP("Streamable DEMO")

app = FastAPI()

@app.get("/", response_class=HTMLResponse)
async def root():
    html_path = os.path.join(os.path.dirname(__file__), "welcome.html")
    with open(html_path, "r", encoding="utf-8") as f:
        html_content = f.read()
    return HTMLResponse(content=html_content)

async def event_stream(message: str):
    for i in range(1, 4):
        yield f"Processing file {i}/3...\n"
        await asyncio.sleep(1)
    yield f"Here's the file content: {message}\n"

@app.get("/stream")
async def stream(message: str = "hello"):
    return StreamingResponse(event_stream(message), media_type="text/plain")

@mcp.tool(description="A tool that simulates file processing and sends progress notifications")
async def process_files(message: str, ctx: Context) -> TextContent:
    files = [f"file_{i}.txt" for i in range(1, 4)]
    for idx, file in enumerate(files, 1):
        await ctx.info(f"Processing {file} ({idx}/{len(files)})...")
        await asyncio.sleep(1)  
    await ctx.info("All files processed!")
    return TextContent(type="text", text=f"Processed files: {', '.join(files)} | Message: {message}")

if __name__ == "__main__":
    import sys
    if "mcp" in sys.argv:
        # Configure MCP server with streamable-http transport
        print("Starting MCP server with streamable-http transport...")
        # MCP server will create its own FastAPI app with the /mcp endpoint
        mcp.run(transport="streamable-http")
    else:
        # Start FastAPI app for classic HTTP streaming
        print("Starting FastAPI server for classic HTTP streaming...")
        uvicorn.run("server:app", host="127.0.0.1", port=8000, reload=True)
```

welcome.html

```html
<!DOCTYPE html>
<html>
    <head>
        <title>HTTP Streaming Demo</title>
        <style>
            body {
                font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                line-height: 1.6;
                color: #333;
                max-width: 800px;
                margin: 0 auto;
                padding: 20px;
                background-color: #f5f5f5;
            }
            .container {
                background-color: white;
                padding: 30px;
                border-radius: 8px;
                box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
            }
            h1 {
                color: #2c3e50;
                border-bottom: 2px solid #3498db;
                padding-bottom: 10px;
            }
            .description {
                background-color: #f8f9fa;
                padding: 15px;
                border-left: 4px solid #3498db;
                margin-bottom: 20px;
            }
            .endpoints {
                display: flex;
                flex-wrap: wrap;
                gap: 20px;
                justify-content: space-between;
            }
            .endpoint-card {
                background-color: #fff;
                border: 1px solid #ddd;
                border-radius: 6px;
                padding: 20px;
                width: calc(50% - 30px);
                box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
                transition: transform 0.3s, box-shadow 0.3s;
            }
            .endpoint-card:hover {
                transform: translateY(-5px);
                box-shadow: 0 6px 12px rgba(0, 0, 0, 0.1);
            }
            .endpoint-card h2 {
                margin-top: 0;
                color: #3498db;
            }
            .btn {
                display: inline-block;
                background-color: #3498db;
                color: white;
                padding: 10px 15px;
                text-decoration: none;
                border-radius: 4px;
                font-weight: bold;
                margin-top: 15px;
                transition: background-color 0.3s;
            }
            .btn:hover {
                background-color: #2980b9;
            }
            .code {
                background-color: #f8f9fa;
                padding: 8px;
                border-radius: 4px;
                font-family: monospace;
                margin: 10px 0;
                overflow-x: auto;
            }
            footer {
                margin-top: 30px;
                text-align: center;
                font-size: 0.9em;
                color: #7f8c8d;
            }
            @media (max-width: 768px) {
                .endpoint-card {
                    width: 100%;
                }
                .endpoints {
                    flex-direction: column;
                }
            }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>HTTP Streaming Demo</h1>
            
            <div class="description">
                <p>This demo showcases two different approaches to HTTP streaming:</p>
                <ul>
                    <li>Classic HTTP Streaming: Using chunked transfer encoding</li>
                    <li>MCP Streaming: Using structured notifications with JSON-RPC</li>
                </ul>
            </div>
            
            <div class="endpoints">
                <div class="endpoint-card">
                    <h2>Classic HTTP Streaming</h2>
                    <p>Simple chunked transfer encoding to send data in chunks with plain text format.</p>
                    <p>Perfect for simple streaming needs like progress updates.</p>
                    <div class="code">GET /stream</div>
                    <a href="/stream" class="btn">Try it now</a>
                </div>
                
                <div class="endpoint-card">
                    <h2>MCP Streaming</h2>
                    <p>Structured notification system with rich metadata and JSON-RPC protocol.</p>
                    <p>Ideal for complex applications requiring detailed progress information.</p>
                    <div class="code">Connect to /mcp with an MCP client</div>
                    <div class="code">python client.py mcp</div>
                </div>
            </div>
            
            <footer>
                <p>Model Context Protocol (MCP) for Beginners &copy; 2025</p>
            </footer>
        </div>
    </body>
</html>
```



log 
```log
2025-07-20 17:10:07 - mcp_client - INFO - Running MCP client...
2025-07-20 17:10:07 - mcp_client - INFO - Starting client...
2025-07-20 17:10:07 - mcp_client - INFO - Session ID before init: None
2025-07-20 17:10:08 - httpx - INFO - HTTP Request: POST http://localhost:8000/mcp "HTTP/1.1 200 OK"
2025-07-20 17:10:08 - mcp.client.streamable_http - INFO - Received session ID: 417bf9e28045449f975dcd54b42617de
2025-07-20 17:10:08 - mcp.client.streamable_http - INFO - Negotiated protocol version: 2025-06-18
2025-07-20 17:10:08 - mcp_client - INFO - Session ID after init: 417bf9e28045449f975dcd54b42617de
2025-07-20 17:10:08 - mcp_client - INFO - Session initialized, ready to call tools.
2025-07-20 17:10:08 - httpx - INFO - HTTP Request: GET http://localhost:8000/mcp "HTTP/1.1 200 OK"
2025-07-20 17:10:08 - httpx - INFO - HTTP Request: POST http://localhost:8000/mcp "HTTP/1.1 202 Accepted"
2025-07-20 17:10:08 - httpx - INFO - HTTP Request: POST http://localhost:8000/mcp "HTTP/1.1 200 OK"
2025-07-20 17:10:08 - mcp_client - INFO - MCP Log: info - Processing file_1.txt (1/3)...
2025-07-20 17:10:08 - mcp_client - INFO - Received message: root=LoggingMessageNotification(method='notifications/message', params=LoggingMessageNotificationParams(meta=None, level='info', logger=None, data='Processing file_1.txt (1/3)...'), jsonrpc='2.0')
2025-07-20 17:10:08 - mcp_client - INFO - NOTIFICATION: root=LoggingMessageNotification(method='notifications/message', params=LoggingMessageNotificationParams(meta=None, level='info', logger=None, data='Processing file_1.txt (1/3)...'), jsonrpc='2.0')
2025-07-20 17:10:09 - mcp_client - INFO - MCP Log: info - Processing file_2.txt (2/3)...
2025-07-20 17:10:09 - mcp_client - INFO - Received message: root=LoggingMessageNotification(method='notifications/message', params=LoggingMessageNotificationParams(meta=None, level='info', logger=None, data='Processing file_2.txt (2/3)...'), jsonrpc='2.0')
2025-07-20 17:10:09 - mcp_client - INFO - NOTIFICATION: root=LoggingMessageNotification(method='notifications/message', params=LoggingMessageNotificationParams(meta=None, level='info', logger=None, data='Processing file_2.txt (2/3)...'), jsonrpc='2.0')
2025-07-20 17:10:10 - mcp_client - INFO - MCP Log: info - Processing file_3.txt (3/3)...
2025-07-20 17:10:10 - mcp_client - INFO - Received message: root=LoggingMessageNotification(method='notifications/message', params=LoggingMessageNotificationParams(meta=None, level='info', logger=None, data='Processing file_3.txt (3/3)...'), jsonrpc='2.0')
2025-07-20 17:10:10 - mcp_client - INFO - NOTIFICATION: root=LoggingMessageNotification(method='notifications/message', params=LoggingMessageNotificationParams(meta=None, level='info', logger=None, data='Processing file_3.txt (3/3)...'), jsonrpc='2.0')
2025-07-20 17:10:11 - mcp_client - INFO - MCP Log: info - All files processed!
2025-07-20 17:10:11 - mcp_client - INFO - Received message: root=LoggingMessageNotification(method='notifications/message', params=LoggingMessageNotificationParams(meta=None, level='info', logger=None, data='All files processed!'), jsonrpc='2.0')
2025-07-20 17:10:11 - mcp_client - INFO - NOTIFICATION: root=LoggingMessageNotification(method='notifications/message', params=LoggingMessageNotificationParams(meta=None, level='info', logger=None, data='All files processed!'), jsonrpc='2.0')  
2025-07-20 17:10:11 - httpx - INFO - HTTP Request: POST http://localhost:8000/mcp "HTTP/1.1 200 OK"
2025-07-20 17:10:11 - mcp_client - INFO - Tool result: meta=None content=[TextContent(type='text', text='Processed files: file_1.txt, file_2.txt, file_3.txt | Message: hello from client', annotations=None, meta=None)] structuredContent={'result': 'Processed files: file_1.txt, file_2.txt, file_3.txt | Message: hello from client'} isError=False
2025-07-20 17:10:11 - mcp_client - INFO - Collected log messages:
2025-07-20 17:10:11 - mcp_client - INFO - Log: meta=None level='info' logger=None data='Processing file_1.txt (1/3)...'   
2025-07-20 17:10:11 - mcp_client - INFO - Log: meta=None level='info' logger=None data='Processing file_2.txt (2/3)...'   
2025-07-20 17:10:11 - mcp_client - INFO - Log: meta=None level='info' logger=None data='Processing file_3.txt (3/3)...'   
2025-07-20 17:10:11 - mcp_client - INFO - Log: meta=None level='info' logger=None data='All files processed!'
2025-07-20 17:10:12 - httpx - INFO - HTTP Request: DELETE http://localhost:8000/mcp "HTTP/1.1 200 OK"
```