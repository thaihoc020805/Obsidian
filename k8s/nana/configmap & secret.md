ConfigMap:
it's basically your external configuration to your application, so configmap would usually contain configuration data like URLs of a database or some other services that you use. And in k8s you just connect it to the pod so that pod actually gets the data that configmap contains. And now if you change the name of the service the endpoint of the service you just adjust the config map and that's it you don't have to build a new image and have to go through this whole cycle 

  
secret
now part of the external configuration can also  be database username and password right which may also change in the application  deployment process but putting a password or other credentials in a config map in a plain text format would be insecure even though it's an external configuration so for this purpose kubernetes has another component called secret
so secret is just like config map but the difference is that it's used to store secret data credentials for example and it's stored not in a plain  text format of course but in base 64 encoded format so secret would contain things like credentials and of course I mean database user you could also put in config map but what's important is the  passwords certificates things that you don't want other people to have access

and just like config map you just connect it to your pod so that pod can actually see those data and read from the secret, you can actually use the data from config map or Secret inside of your application pod  using for example environmental variables or even as a properties file

![[Pasted image 20250804111515.png]]



nếu các configmap change và muốn pod cập nhật các change config thì 
```bash
kubectl rollout restart deployment <ten-deployment>
```

**ConfigMap** và **Secret** là hai kiểu đối tượng đặc biệt trong Kubernetes dùng để quản lý cấu hình ứng dụng dưới dạng key-value, giúp tách biệt phần cấu hình với mã nguồn và việc triển khai container. Tuy nhiên, có một số điểm khác biệt quan trọng giữa chúng:

## 1. ConfigMap là gì?

- Là đối tượng dùng để lưu trữ các thông tin cấu hình không nhạy cảm, như file cấu hình, biến môi trường, tham số ứng dụng.
    
- Dữ liệu lưu trong ConfigMap là dạng _plain text_ (chuỗi văn bản thông thường)[](https://viblo.asia/p/tao-va-su-dung-configmaps-trong-kubernetes-07LKXk8P5V4)[](https://www.vntechies.dev/courses/k8s-spring-boot/quan-ly-cau-hinh-ung-dung-voi-configmap-va-secret)[](https://aithietke.com/lam-viec-voi-configmap-va-secret-tren-kubernetes/).
    
- Các ứng dụng/pod có thể sử dụng dữ liệu trong ConfigMap thông qua biến môi trường, mount file, hoặc arguments dòng lệnh.
    
- Không có lớp bảo mật/mã hóa đặc biệt — bất kỳ ai có quyền truy cập cluster đều đọc được dữ liệu này.
    

## 2. Secret là gì?

- Là đối tượng lưu trữ dữ liệu _nhạy cảm_: password, token, khóa bí mật, chứng chỉ,...
    
- Dữ liệu trong Secret được _mã hóa/camouflaged_ dưới dạng base64 khi lưu, và có thể được bảo vệ bằng các cơ chế mã hóa “at rest” nếu cluster cấu hình[](https://viblo.asia/p/kubernetes-series-bai-8-configmap-and-secret-truyen-cau-hinh-vao-container-RQqKL6rrl7z)[](https://www.vntechies.dev/courses/k8s-spring-boot/quan-ly-cau-hinh-ung-dung-voi-configmap-va-secret)[](https://viblo.asia/p/k8s-basic-kubernetes-secrets-zOQJwYrdVMP)[](https://locker.io/vi/blog/kubernetes-architecture)[](https://technicaldeepdive.hashnode.dev/bai-8-quan-ly-thong-tin-nhay-cam-voi-secrets-trong-kubernetes).
    
- Pod truy xuất dữ liệu Secret cũng theo cách tương tự ConfigMap (biến môi trường hoặc mount file), nhưng Kubernetes tăng cường bảo mật: ví dụ, chỉ gửi Secret tới node cần dùng, lưu trên node dưới dạng memory (tránh lưu ổ cứng),...
    
- Có thể kiểm soát truy cập (access) bằng RBAC, tách biệt rõ ràng với ConfigMap.
    


**Kết luận ngắn gọn:**

- Cấu hình không nhạy cảm → dùng ConfigMap cho tiện lợi, dễ xem và chia sẻ.
    
- Dữ liệu nhạy cảm (password, API key, token…) → PHẢI dùng Secret để đảm bảo an toàn và tuân thủ best-practice bảo mật trên Kubernetes[](https://viblo.asia/p/kubernetes-series-bai-8-configmap-and-secret-truyen-cau-hinh-vao-container-RQqKL6rrl7z)[](https://www.vntechies.dev/courses/k8s-spring-boot/quan-ly-cau-hinh-ung-dung-voi-configmap-va-secret)[](https://locker.io/vi/blog/kubernetes-architecture)[](https://technicaldeepdive.hashnode.dev/bai-8-quan-ly-thong-tin-nhay-cam-voi-secrets-trong-kubernetes).

