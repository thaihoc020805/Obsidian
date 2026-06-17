sequenceDiagram
    participant AF as Airflow Operator
    participant K8s as K8s API
    participant ZP as Zeppelin Pod
    participant ZD as Zeppelin Daemon<br/>(inside Pod)
    participant SI as Spark Interpreter<br/>(inside Pod)
    participant EX as Executor Pods<br/>(K8s)

    AF->>K8s: 1. Create Pod (Zeppelin image + notebook)
    K8s->>ZP: Pod starts
    ZP->>ZD: 2. Start Zeppelin daemon (~30s)
    ZP->>ZD: 3. Import .ipynb (convert to .zpln)
    ZP->>ZD: 4. POST /api/notebook/job/{id} (run all)
    ZD->>SI: 5. Start Spark interpreter
    SI->>K8s: 6. spark-submit to k8s://
    K8s->>EX: 7. Create executor pods
    
    Note over ZD,EX: Notebook cells execute sequentially<br/>Python → Scala → SQL → R<br/>(shared SparkSession)
    
    EX-->>SI: Results
    SI-->>ZD: Paragraph results
    ZD-->>ZP: All paragraphs done
    ZP-->>AF: Exit code 0/1
    AF->>K8s: Cleanup (Pod terminates, executors cleanup)