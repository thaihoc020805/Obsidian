# Giải thích Multiplexing và Full Duplex

## 🔄 Full Duplex của WebSocket

### Full Duplex là gì?

Giống như **cuộc gọi điện thoại** - cả 2 người có thể nói cùng lúc:

javascript

```javascript
// WebSocket - Full Duplex Example
const ws = new WebSocket('ws://chat.example.com');

// Client có thể GỬI bất cứ lúc nào
setInterval(() => {
  ws.send('Client: Tôi đang gõ tin nhắn...');
}, 1000);

// ĐỒNG THỜI client cũng NHẬN được tin nhắn
ws.onmessage = (event) => {
  console.log('Server gửi:', event.data);
};

// Server cũng vậy - có thể gửi và nhận cùng lúc
// Không cần đợi ai "nói xong" mới được nói
```

### So sánh với Half Duplex:

javascript

```javascript
// Half Duplex (như bộ đàm) - chỉ 1 bên nói 1 lúc
// HTTP Request/Response truyền thống:
// 1. Client gửi request (Client nói)
// 2. Đợi...
// 3. Server gửi response (Server nói)
// 4. Xong rồi mới gửi request mới được
```

### Ví dụ thực tế Full Duplex:

javascript

```javascript
// Game bắn súng online
const gameWs = new WebSocket('ws://game.server');

// Player bắn (gửi) ĐỒNG THỜI nhận vị trí enemies
gameWs.onopen = () => {
  // Gửi vị trí và actions của player
  setInterval(() => {
    gameWs.send(JSON.stringify({
      type: 'player_move',
      x: mouse.x,
      y: mouse.y,
      shooting: mouse.clicked
    }));
  }, 16); // 60 FPS

  // ĐỒNG THỜI nhận updates từ server
  gameWs.onmessage = (event) => {
    const gameState = JSON.parse(event.data);
    // Update enemies, bullets, scores...
    updateGame(gameState);
  };
};
```

## 🎛️ Multiplexing của HTTP/2 (Streamable HTTP)

### Multiplexing là gì?

Giống như **đường cao tốc nhiều làn** - nhiều xe (requests) đi cùng lúc:

javascript

```javascript
// HTTP/1.1 - KHÔNG có Multiplexing
// Giống đường 1 làn - xe phải xếp hàng
fetch('/api/user')     // Chờ...
  .then(() => fetch('/api/posts'))     // Chờ...
  .then(() => fetch('/api/images'))    // Chờ...

// HTTP/2 - CÓ Multiplexing  
// Giống đường 6 làn - nhiều xe đi cùng lúc
Promise.all([
  fetch('/api/user'),      // Lane 1 ← Đi cùng lúc
  fetch('/api/posts'),     // Lane 2 ← Đi cùng lúc
  fetch('/api/images'),    // Lane 3 ← Đi cùng lúc
  fetch('/api/comments'),  // Lane 4 ← Đi cùng lúc
  fetch('/api/likes'),     // Lane 5 ← Đi cùng lúc
  fetch('/api/shares')     // Lane 6 ← Đi cùng lúc
]); // Tất cả chạy song song trên 1 connection!
```

### Ví dụ trực quan:

```
HTTP/1.1 (No Multiplexing):
Connection 1: [====Request1====] → [====Response1====] → [====Request2====] → ...
Connection 2: [====Request3====] → [====Response3====] → [====Request4====] → ...
Connection 3: [====Request5====] → [====Response5====] → [====Request6====] → ...
(Tối đa 6 connections/domain)

HTTP/2 (With Multiplexing):
Connection 1: [Req1][Req2][Req3][Res1][Req4][Res2][Res3][Req5][Res4][Res5]...
              ↑ Tất cả requests/responses đan xen trên 1 connection
```

### Code example Multiplexing:

javascript

```javascript
// Streamable HTTP với Multiplexing
// 1 connection, nhiều streams cùng lúc

// Stream 1: Chat messages
const chatStream = await fetch('/api/chat/stream', {
  method: 'POST',
  body: createChatStream()
});

// Stream 2: File upload (cùng lúc!)
const uploadStream = await fetch('/api/upload/video', {
  method: 'POST', 
  body: videoFile
});

// Stream 3: Live notifications (cùng lúc!)
const notifStream = await fetch('/api/notifications/stream');

// Cả 3 streams chạy song song trên 1 HTTP/2 connection
// Không block lẫn nhau!
```

## ☁️ Tại sao Streamable HTTP tốt cho Cloud Apps?

### 1. **Load Balancer Friendly**

nginx

```nginx
# HTTP/2 Streams dễ dàng route qua load balancers
upstream backend {
  server backend1.example.com;
  server backend2.example.com;
  server backend3.example.com;
}

# WebSocket khó hơn - cần sticky sessions
# HTTP/2 streams có thể route mỗi request độc lập
```

### 2. **Works với HTTP Infrastructure**

javascript

```javascript
// Streamable HTTP tận dụng được:
// - CDNs (Cloudflare, Akamai)
// - API Gateways (Kong, AWS API Gateway)  
// - Service Mesh (Istio, Linkerd)
// - HTTP Caching
// - HTTP Authentication

// WebSocket bypass hết những thứ này!
```

### 3. **Microservices Compatible**

javascript

```javascript
// Service A có thể forward stream đến Service B
app.post('/api/process', async (req, res) => {
  // Forward stream đến microservice khác
  const mlService = await fetch('http://ml-service/analyze', {
    method: 'POST',
    body: req.body, // Forward stream
    duplex: 'half'
  });
  
  // Pipe response back
  mlService.body.pipeTo(res);
});

// WebSocket khó forward giữa services
```

### 4. **Better Observability**

javascript

```javascript
// HTTP/2 Streams có thể:
// - Log như HTTP requests bình thường
// - Monitor với Prometheus/Grafana
// - Trace với Jaeger/Zipkin
// - Debug với Chrome DevTools

// WebSocket cần tools riêng
```

### 5. **Cloud Native Features**

yaml

```yaml
# Kubernetes Ingress supports HTTP/2 naturally
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP2"
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /stream
            pathType: Prefix
            backend:
              service:
                name: streaming-service
                port:
                  number: 80
```

## 📊 So sánh thực tế

### WebSocket in Cloud:

javascript

```javascript
// Khó khăn:
// 1. Cần sticky sessions
// 2. Khó scale horizontally  
// 3. Firewall/Proxy issues
// 4. Cần maintain connection state

// Ví dụ: Scale WebSocket
const Redis = require('redis');
const pub = Redis.createClient();
const sub = Redis.createClient();

// Phải dùng Redis PubSub để sync giữa servers
wss.on('connection', (ws) => {
  sub.subscribe('messages');
  sub.on('message', (channel, message) => {
    ws.send(message); // Broadcast to connected clients
  });
});
```

### Streamable HTTP in Cloud:

javascript

```javascript
// Dễ dàng:
// 1. Stateless - scale thoải mái
// 2. Load balancer tự động
// 3. Works với mọi HTTP tools
// 4. Cloud providers optimize sẵn

// Ví dụ: Scale HTTP/2 Streaming
app.post('/stream', (req, res) => {
  // Không cần state, không cần Redis
  // Mỗi request độc lập
  processStream(req.body)
    .pipe(res);
});
```

## 🎯 Tóm lại

**Full Duplex (WebSocket)** = Cuộc gọi điện thoại

- Cả 2 bên nói cùng lúc được
- Tốt cho real-time games, trading

**Multiplexing (HTTP/2)** = Đường cao tốc nhiều làn

- Nhiều requests/responses cùng lúc trên 1 connection
- Tốt cho modern web apps

**Cloud-friendly** = Hoạt động tốt với infrastructure

- Load balancers, CDNs, API gateways
- Monitoring, logging, debugging tools
- Horizontal scaling dễ dàng