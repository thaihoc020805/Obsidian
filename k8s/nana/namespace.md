by default k8s give you 4 namespace
![[Pasted image 20250804171021.png]]

kubernetes-dashboard only with minicube, it's specific to mini cube installation

![[Pasted image 20250804172112.png]]


![[Pasted image 20250804172230.png]]

![[Pasted image 20250804172736.png]]



![[Pasted image 20250804172821.png]]

Khi khởi tạo một cụm Kubernetes mới, mặc định sẽ có 4 namespace được tạo sẵn với các vai trò riêng biệt:

## 1. **default**

- Là namespace mặc định.
    
- Nếu bạn không chỉ rõ namespace khi tạo resource (Pod, Service, Deployment...), resource đó sẽ nằm trong namespace này.
    
- Thích hợp cho thử nghiệm, demo hoặc chạy những workload nhỏ. Không nên dùng cho sản phẩm lớn hoặc môi trường production vì khó quản lý về sau[](https://www.plural.sh/blog/kubernetes-namespaces-guide/)[](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)[](https://www.aquasec.com/cloud-native-academy/kubernetes-101/kubernetes-namespace/)[](https://www.geeksforgeeks.org/devops/kubernetes-namespace/)[](https://www.tigera.io/learn/guides/kubernetes-security/kubernetes-namespace/).
    

## 2. **kube-system**

- Chứa các thành phần hệ thống cốt lõi của Kubernetes như kube-dns, kube-proxy, kube-controller-manager, kube-scheduler...
    
- Tuyệt đối không nên xóa hay chỉnh sửa các resource ở đây nếu không biết rõ mục đích, vì có thể ảnh hưởng tới toàn bộ cụm[](https://www.plural.sh/blog/kubernetes-namespaces-guide/)[](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)[](https://www.aquasec.com/cloud-native-academy/kubernetes-101/kubernetes-namespace/)[](https://www.tigera.io/learn/guides/kubernetes-security/kubernetes-namespace/).
    

## 3. **kube-public**

- Chứa các tài nguyên có thể public, tức là toàn bộ user (kể cả chưa xác thực) đều đọc được.
    
- Thường dùng để chia sẻ thông tin cluster hoặc cấu hình chung (như thông tin truy cập cluster cho client).
    
- Tuyệt đối không lưu dữ liệu nhạy cảm ở đây vì mức độ bảo mật thấp[](https://www.plural.sh/blog/kubernetes-namespaces-guide/)[](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)[](https://www.aquasec.com/cloud-native-academy/kubernetes-101/kubernetes-namespace/)[](https://www.geeksforgeeks.org/devops/kubernetes-namespace/)[](https://www.tigera.io/learn/guides/kubernetes-security/kubernetes-namespace/).
    

## 4. **kube-node-lease**

- Chứa các object “lease” cho từng node. Mỗi node sẽ có một lease object do kubelet quản lý, dụng để gửi heartbeat (kiểm tra node còn "sống" hay không) về cho control plane.
    
- Giúp phát hiện node bị mất kết nối, hỗ trợ cơ chế autoscaling, health monitoring của Kubernetes.
    
- Người dùng thường không thao tác trực tiếp với namespace này, chủ yếu phục vụ hệ thống[](https://www.plural.sh/blog/kubernetes-namespaces-guide/)[](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)[](https://www.geeksforgeeks.org/devops/kubernetes-namespace/)[](https://www.tigera.io/learn/guides/kubernetes-security/kubernetes-namespace/).
    

**Tóm tắt:**

- `default`: nơi chứa resource mặc định, không nên lạm dụng cho production.
    
- `kube-system`: chỉ chứa các thành phần hệ thống, không can thiệp nếu không cần.
    
- `kube-public`: nơi chứa dữ liệu có thể truy cập công khai (không bảo mật).
    
- `kube-node-lease`: giúp quản lý trạng thái node trong cluster, phục vụ cho health check và autoscaling.


để tạo 1 namespace : kubectl create namespace your-namespace

![[Pasted image 20250804173512.png]]


![[Pasted image 20250804173624.png]]



![[Pasted image 20250804173727.png]]



![[Pasted image 20250804174005.png]]


![[Pasted image 20250804174257.png]]


![[Pasted image 20250804174348.png]]

![[Pasted image 20250804174452.png]]


![[Pasted image 20250804174518.png]]


![[Pasted image 20250804175252.png]]

![[Pasted image 20250804175413.png]]


![[Pasted image 20250804175613.png]]



![[Pasted image 20250804175803.png]]


## Cấu trúc DNS trong Kubernetes

Khi bạn muốn truy cập một service ở namespace khác, bạn cần dùng **tên chuẩn DNS** dạng:

`<service-name>.<namespace> hoặc đầy đủ: <service-name>.<namespace>.svc.cluster.local`



