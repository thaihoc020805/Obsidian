**API for engine-v2**

- sửa tab menu thành "FDP Tools"
    
- **job_list**
    
    - list danh sách wf
        
        - lấy danh sách wf và join thêm các thông tin về trạng thái và lịch sử runs của workflow theo dag_id trong airflow
            
    - có trả về theo trang (truyền page size và page index và total number wf)
        
    - search workflow theo name và tags (regex)
        
    - action run workflow (trigger run)
        
    - xóa workflow
        
- **job_create**
    
    - view detail workflow
        
    - create/update workflow (theo wf_id)
        
    - danh sách job type
        
    - danh sách compute
        
    - danh sách trigger rule
        
    - danh sách image