Let's take a look at the Kubectl utility. Kubectl is the command line utility of
Kubernetes. This is the tool or command you would use to ==operate the kubernetes==
==cluster such as to view the status of the cluster, to provision application, to scale up,==
==scale down, delete and many other things.==

![[Pasted image 20250801001405.png]]

![[Pasted image 20250801001534.png]]


To identify the ==version of the kubectl client and the kubernetes server==, run the ==kubectl==
==version== command. This lists the client and server version along with the version of
any other tool installed in the system.


![[Pasted image 20250801001648.png]]

The help option lists basic help information such as the basic commands that can be
run. We will dig into these commands later in this tutorial.

![[Pasted image 20250801001947.png]]

Let's begin with a few simple commands. To ==see a list of nodes in your cluster== run the
==kubectl get nodes== command. ==The output shows you the name of the node, it's status,==
==the roles, how long the machine has been up and the version of Kubernetes running==
==on that system.==

![[Pasted image 20250801002321.png]]
kubectl get nodes ==-o wide==
To ==get== a more verbose output with ==more details such as internal IP, OSImage, kernel==
==version, container runCme etc,== run the same command with the ==–o== wide opCon.



The ==**Kubernetes control plane** URL== can be found by running the command ==`kubectl cluster-info`.==

https://kodekloud.com/topic/labs-familiarize-with-lab-environment/


