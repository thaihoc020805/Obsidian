Luồng hoạt động khi bạn trigger một DAG trong Apache Airflow qua giao diện web đúng như mô tả, cụ thể:

1. Bạn nhấn nút Trigger trên giao diện web → Yêu cầu được gửi tới Airflow Webserver.
    
2. Webserver tạo một bản ghi mới trong bảng `dag_run` của database với trạng thái `queued`.
    
3. Scheduler liên tục quét bảng `dag_run` trong cơ sở dữ liệu → khi thấy dag_run mới, nó sẽ đọc file DAG tương ứng.
    
4. Scheduler xác định các task đầu tiên chưa có phụ thuộc cần chờ trong DAG.
    
5. Scheduler tạo bản ghi `task_instance` trong database với trạng thái `queued` cho các task đó.
    
6. Scheduler đưa task instance vào hàng đợi (queue) để phân phối cho worker.
    
7. Worker nhận task từ hàng đợi khi có slot rảnh.
    
8. Worker cập nhật trạng thái task instance trong database từ `queued` thành `running`.
    
9. Worker bắt đầu thực thi task và ghi log.
    
10. Khi task hoàn thành hoặc thất bại, worker cập nhật trạng thái cuối cùng (`success`, `failed`,...) vào database.