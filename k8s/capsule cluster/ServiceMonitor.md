`ServiceMonitor` là một **Custom Resource của Prometheus Operator**, dùng để nói cho Prometheus biết:

> “Hãy đi scrape metrics từ Service này.”

Ví dụ app của bạn expose metrics ở:

```text
Pod
  └── :8080/metrics
```

Bạn có Kubernetes `Service` trỏ vào Pod:

```text
Service: capsule-controller-manager
  └── port: 8080
```

Rồi tạo `ServiceMonitor`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: capsule
spec:
  selector:
    matchLabels:
      app: capsule

  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

Luồng sẽ là:

```text
ServiceMonitor
     │
     │ selector
     ▼
Kubernetes Service
     │
     ▼
Pod
     │
     │ GET /metrics
     ▼
Prometheus
```

Điểm dễ nhầm là **ServiceMonitor không tự scrape gì cả**.

Nó chỉ là một object cấu hình:

```text
ServiceMonitor
     ↓ được đọc bởi
Prometheus Operator
     ↓ cấu hình
Prometheus
     ↓ scrape
/metrics
```

Và `ServiceMonitor` cũng **không phải resource có sẵn của Kubernetes**. Nó tồn tại vì Prometheus Operator cài CRD:

```bash
kubectl get crd | grep servicemonitor
```

thường sẽ thấy:

```text
servicemonitors.monitoring.coreos.com
```

Nên câu trước:

```text
Capsule muốn tạo ServiceMonitor
        ↓
Prometheus Operator phải có trước
```

thực chất chính xác hơn là:

```text
Prometheus Operator được cài
        ↓
tạo ServiceMonitor CRD
        ↓
Kubernetes mới hiểu kind: ServiceMonitor
        ↓
Capsule có thể tạo ServiceMonitor
```

**ServiceMonitor ≈ “chỉ dẫn cho Prometheus scrape Service nào, port nào, path nào, bao lâu scrape một lần”.**