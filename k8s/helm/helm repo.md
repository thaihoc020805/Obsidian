Lệnh **`helm repo list`** dùng để liệt kê tất cả các **Helm chart repositories** mà bạn đã add vào local Helm client.

---

### 📌 Cách dùng

`helm repo list`

---

### 📌 Ví dụ kết quả

`NAME       	URL bitnami    	https://charts.bitnami.com/bitnami ingress-nginx	https://kubernetes.github.io/ingress-nginx prometheus-community	https://prometheus-community.github.io/helm-charts`

- **NAME**: tên repo (do bạn đặt khi add).
    
- **URL**: đường dẫn đến Helm chart repository (nơi chứa các chart packages).
    

---

### 📌 Liên quan

- **Thêm repo**:
    
    `helm repo add bitnami https://charts.bitnami.com/bitnami`
    
- **Cập nhật repo (fetch index.yaml mới nhất)**:
    
    `helm repo update`
    
- **Xoá repo**:
    
    `helm repo remove bitnami`
    



`helm search hub` dùng để **tìm chart trên Helm Hub (Artifact Hub)** 🌐 – tức là tìm trên **kho chart công cộng** chứ không phải chỉ trong local repo của bạn.

---

### 📌 Cách dùng

`helm search hub <keyword>`

- `<keyword>`: từ khóa muốn tìm, ví dụ `nginx`, `prometheus`, `mysql`…
    

---

### 📌 Ví dụ

`helm search hub nginx`

Kết quả có thể ra:

`URL                                                 CHART VERSION  APP VERSION  DESCRIPTION https://artifacthub.io/packages/helm/bitnami/nginx  13.2.16        1.25.1       Chart for the NGINX server https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx ...`

- **URL**: link trực tiếp đến chart trên Artifact Hub.
    
- **CHART VERSION**: version của chart.
    
- **APP VERSION**: version của ứng dụng thật (nginx, prometheus…).
    
- **DESCRIPTION**: mô tả chart.
    

---

### 📌 Khác biệt với `helm search repo`

- `helm search hub`: tìm **chart trên Artifact Hub** (tất cả public charts).
    
- `helm search repo`: tìm **chart trong repo local** (chỉ những repo bạn đã `helm repo add`).
    

---

👉 Tóm gọn:

- `helm search hub` = tìm chart trên internet (Artifact Hub).
    
- `helm search repo` = tìm chart trong repo bạn đã add vào máy.
    

---


**Artifact Hub** là một **trang web / kho tập trung** (do CNCF duy trì) để tìm, khám phá và chia sẻ các package cho hệ sinh thái cloud-native.

---

## 📌 Artifact Hub là gì?

- Nó giống như **npm registry** cho Node.js hay **PyPI** cho Python.
    
- Nhưng thay vì code library, Artifact Hub chứa các **package cloud-native**:
    
    - **Helm Charts** (cài app vào Kubernetes).
        
    - **OLM Operators** (Kubernetes Operators).
        
    - **Falco rules** (luật bảo mật).
        
    - **OPA policies** (chính sách kiểm soát).
        
    - **Tinkerbell actions** và nhiều thứ khác.
        

👉 Bạn có thể coi Artifact Hub là “chợ ứng dụng” cho Kubernetes & CNCF.

---

## 📌 Dùng Artifact Hub để làm gì?

1. **Tìm Helm chart chính thức** (ví dụ nginx, prometheus, grafana, mysql…).
    
    - Tránh phải đi tìm repo rải rác trên GitHub.
        
    - Ví dụ: https://artifacthub.io/packages/helm/bitnami/nginx
        
2. **Quản lý repo Helm**: khi bạn chạy
    
    `helm search hub nginx`
    
    Helm sẽ tìm trực tiếp trên Artifact Hub.
    
3. **Xem review, maintainers, security reports** của chart.
    

---

## 📌 Ví dụ thực tế

- Bạn muốn deploy **Prometheus** vào cluster:
    
    - Vào Artifact Hub, tìm “prometheus”.
        
    - Nó trả ra nhiều chart: `prometheus-community`, `bitnami/prometheus`…
        
    - Copy repo URL và add vào Helm:
        
        `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts helm install my-prometheus prometheus-community/prometheus`
        

---