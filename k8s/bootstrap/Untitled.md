Kubernetes architecture

![[Pasted image 20260623132514.png]]
![[Pasted image 20260623142928.png]]

![[Pasted image 20260623143258.png]]

Just masternode (use for stagging k8s or startup company)

![[Pasted image 20260623132632.png]]


etcd : key value db lưu thông tin liên quan ứng dụng đang chạy trên wk node, status app

admin request 
apiserver nhận rq , sau đó hỏi scheduler , scheduler sẽ xem thông tin trong etcd, tính toán con worker node nào phù hợp nhất sau đó sẽ communicate với api server là pod đó nên triển khai trên wknode a, sau đó api server sẽ gửi rq triển khai cái pod đó cho kubelet ở wknode a 
trong wk node có kubelet agent nhận yêu cầu từ api server 
kubelet là internal manager của node

sau đó kubelet sẽ đi deploy các cái pod này trên wk node đến containerd ,  ...

thông thường triển khai HA api server và etcd
làm sao để quản lý là ht luôn có pod available, đang sống

controller manager,  sẽ là agent kiểm soát baseline cluster ổn định, monitor replicaset,.... cm sẽ làm việc với api server 

bên trong controlplan có 1 thành phần nữa nói chuyện với cloud provider
là cloud control manager 

container runtime , 

kubelet ngoài nhận rq, còn gửi report trạng thái các dịch vụ trong node cho api server

để các service nói chuyện vs nhau thì có hệ thống quản lí việc nói chuyện giữa các service, các container pod ở đây riêng , tên là kube proxy