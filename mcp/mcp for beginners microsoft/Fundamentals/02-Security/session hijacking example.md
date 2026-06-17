## **Session Hijacking là gì?**

**Session** = **Phiên làm việc** giữa client và server **Session ID** = **Thẻ tạm thời** để nhận diện bạn

### **Ví dụ như đi xem phim:**

```
🎬 Cinema (Server) cho bạn thẻ số: "SESSION-123"
🎫 Thẻ này chứng minh bạn đã mua vé
👤 Bạn dùng thẻ để vào rạp, mua bỏng ngô, etc.

😈 Kẻ xấu steal thẻ số "SESSION-123"
🎭 Kẻ xấu giả làm bạn, dùng thẻ để làm gì đó xấu
```

## **Session Hijacking trong MCP:**

### **Normal flow:**

```
👤 User login → MCP Server
🔄 MCP Server: "Đây là session ID của bạn: abc123"
👤 User: "Tôi muốn access Google Drive" (kèm session: abc123)
🔄 MCP Server: "OK, abc123 là John, cho phép access"
```

### **Attack flow:**

```
😈 Attacker steal session ID: "abc123"
😈 Attacker → MCP Server: "Access Google Drive" (kèm session: abc123)
🔄 MCP Server: "OK, abc123 là John, cho phép access"
❌ Server nghĩ attacker chính là John!
```

## **3 Types of Session Hijacking Risks:**

### **1. Session Hijack Prompt Injection**

**Scenario:** MCP Server cluster với shared session state

```
🖥️ MCP Server A (handling chat)
🖥️ MCP Server B (handling file operations)
💾 Shared Session Database

👤 John login → Session "xyz789" trên Server A
😈 Attacker có session "xyz789"
😈 Attacker → Server B: "Execute command: rm -rf /" (với session xyz789)
🔄 Server B: "xyz789 là John, execute command!"
💥 Disaster!
```

### **2. Session Hijack Impersonation**

**Ví dụ cụ thể:**

```
👤 John's session: "session_456"
📧 John: "Send email to marketing team"
🔄 MCP Server: "OK, session_456 confirmed"

😈 Attacker steal "session_456"
😈 Attacker: "Send email: 'All employees fired!'" (session_456)
🔄 MCP Server: "OK, session_456 = John, sending email"
📧 Email sent as John!
```

### **3. Compromised Resumable Streams**

**Scenario:** File upload bị interrupt

```
👤 John upload file "report.pdf" → Session "upload_789"
📊 Upload 50% → Connection dropped
😈 Attacker inject malicious content vào stream
👤 John reconnect: "Resume upload session upload_789"
🔄 Server: "OK, resuming..." 
💀 Attacker's malicious content mixed với John's file
```

## **Mitigating Controls với ví dụ:**

### **❌ Vulnerable session management:**

python

```python
# NGUY HIỂM - Session làm authentication
sessions = {"session123": "john_doe"}

@app.post("/api/files")
async def get_files(session_id: str):
    user = sessions.get(session_id)  # Chỉ dựa vào session
    if user:
        return get_user_files(user)  # No additional verification
    raise HTTPException(401, "Not authenticated")
```

### **✅ Secure session management:**

#### **1. Authorization verification:**

python

```python
@app.post("/api/files") 
async def get_files(session_id: str, auth_token: str):
    # Verify BOTH session AND token
    user = verify_session(session_id)
    token_user = verify_jwt_token(auth_token)
    
    if user != token_user:
        raise HTTPException(401, "Session/token mismatch")
    
    return get_user_files(user)
```

#### **2. Secure session IDs:**

python

```python
import secrets
import hashlib

# ❌ Predictable session ID
def bad_generate_session():
    return f"session_{int(time.time())}"  # Có thể đoán được

# ✅ Secure session ID  
def secure_generate_session():
    return secrets.token_urlsafe(32)  # Cryptographically secure
```

#### **3. User-specific session binding:**

python

```python
def create_bound_session(user_id: str) -> str:
    session_id = secrets.token_urlsafe(32)
    
    # Bind session to user
    bound_session = f"{user_id}:{session_id}"
    
    # Store in database
    sessions[bound_session] = {
        "user_id": user_id,
        "created_at": time.time(),
        "ip_address": request.client.host,
        "user_agent": request.headers.get("user-agent")
    }
    
    return bound_session

def verify_bound_session(bound_session: str, current_user_id: str) -> bool:
    try:
        user_id, session_id = bound_session.split(":", 1)
        
        # Check user matches
        if user_id != current_user_id:
            return False
            
        # Check session exists
        return bound_session in sessions
        
    except ValueError:
        return False
```

#### **4. Session expiration:**

python

```python
import time

def cleanup_expired_sessions():
    current_time = time.time()
    expired = []
    
    for session_id, data in sessions.items():
        # 30 minutes timeout
        if current_time - data["created_at"] > 1800:
            expired.append(session_id)
    
    for session_id in expired:
        del sessions[session_id]

# Auto-cleanup mỗi 5 phút
import asyncio

async def session_cleanup_task():
    while True:
        cleanup_expired_sessions()
        await asyncio.sleep(300)  # 5 minutes
```

#### **5. Transport security:**

python

```python
from fastapi import FastAPI
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

app = FastAPI()

# Force HTTPS
app.add_middleware(HTTPSRedirectMiddleware)

# Secure cookie settings
@app.post("/login")
async def login(response: Response):
    session_id = secure_generate_session()
    
    # Set secure cookie
    response.set_cookie(
        key="session_id",
        value=session_id,
        httponly=True,    # Prevent XSS
        secure=True,      # HTTPS only
        samesite="strict", # CSRF protection
        max_age=1800      # 30 minutes
    )
    
    return {"message": "Logged in successfully"}
```

## **Real-world attack example:**

### **Starbucks WiFi Attack:**

```
👤 John connects to "Starbucks-WiFi"
🔄 John login MCP Server → Session: "abc123"
😈 Attacker on same WiFi intercepts session ID
😈 Attacker → MCP Server: "Transfer $1000" (session: abc123)
💰 Money transferred as John!
```

### **Prevention:**

```
✅ Use HTTPS (encrypts session ID)
✅ Bind session to IP address
✅ Require 2FA for sensitive operations
✅ Short session timeout
```

## **Tóm tắt:**

```
Session Hijacking = Kẻ xấu steal "chìa khóa tạm thời" của bạn

Prevention:
🔐 Don't rely only on sessions for auth
🎲 Use random, unpredictable session IDs  
👤 Bind sessions to specific users
⏰ Short expiration times
🔒 Always use HTTPS
```

**Real-world analogy:** Giống như hotel key card - nếu bị mất, người khác có thể vào phòng của bạn và làm gì đó xấu!