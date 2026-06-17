db can't be replicated via Deployment

![[Pasted image 20250804112250.png]]


![[Pasted image 20250804112458.png]]

![[Pasted image 20250804112509.png]]

 stateful said just like deployment would take care of replicating the pods and scaling them up or scaling them down but making sure the database reads and writes are synchronized so that no database inconsistencies are offered

but statefulSet not easy, so common practice to host db outside of the k8s cluster