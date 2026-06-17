

![[Pasted image 20250731223948.png]]

With docker you were able to run a single instance of an application using the docker
run command. Which is great! Running an application has never been so easy before.
With ==kubernetes==, using the kubernetes CLI known as kubectl, ==you can run a 1000==
==instance of the same application with a single command==. ==Kubernetes can scale it==
==up to 2000 with another command.==
Kubernetes can even be configured to do these ==automatically== so that ==instances and==
==the infrastructure itself== can ==scale up and down based on user load.==
Kubernetes can ==upgrade== these ==2000 instances of application in a rolling fashion (cập nhật từ từ )== ==one at a time, with a single command==.If something goes wrong, it can help you
roll back these images with a single command.
Kubernetes can help you ==test new features== of your application by ==only upgrading==
==a percentage of these instances through A/B testing methods.==


![[Pasted image 20250731224813.png]]

With Kubernetes you are able to ==define== the ==expected state of your application==. For
example you are ==able to define== that ==your application consists of 4 different services==.
The ==webserver must have 3 instances running==, the ==payment service to have 2==. There
should be a ==redis service with 3 instances running== and a ==database service to which==
==these services connect to==. And you are able to ==define== these ==in code and Kubenetes==
==will ensure that the state you have defined for your application is maintained at all==
==times.==



![[Pasted image 20250731225525.png]]

Let's us understand the ==basic components== in a ==Kubernetes Cluster== first. Let us start
with Nodes. ==A node is a machine – physical or virtual – on which kubernetes is==
==installed==. ==A node is a worker machine== and this is ==where containers will be launched by==
==kubernetes.==
But ==what if== the ==node== on which our application is ==running fails==? Well, ==obviously our==
==application goes down. So you need to have more than one nodes.== 



![[Pasted image 20250731230320.png]]

==A cluster is a set of nodes grouped together==. This way ==even if one node fails you==
==have your application still accessible from the other nodes==. Moreover having multiple
nodes helps in ==sharing load== as well.
Now we have a cluster, but ==who is responsible for managing the cluster?== Where is the
==information about the members of the cluster stored==? ==How== are the ==nodes==
==monitored?== When a node fails ==how== do you ==move the workload of the failed node to==
==another worker node?== That’s where the ==ControlPlane== comes in. Also ==previously known==
==as the master node.==



![[Pasted image 20250731233345.png]]
The ==controlplane is another node with Kubernetes components installed in it.== The
controlplane watches over the nodes in the cluster and is responsible for the actual
==orchestration of containers== ==on the worker nodes.==



![[Pasted image 20250731233553.png]]

When you ==install Kubernetes== on a System, you are ==actually installing the following==
==components.== 
- ==An API Server.== 
- ==An ETCD service.== 
- ==Controllers and Schedulers.==

==The API server acts as the front-end for kubernetes==. ==The users, management devices,==
==Command line interfaces all talk to the API server to interact with the kubernetes==
==cluster.==

Next is the ==ETCD key store==. ETCD is a ==distributed reliable key-value store== used by
kubernetes to ==store all data used to manage the cluster==. This is ==where information==
==about the nodes in the cluster, the application running on the cluster are stored.==

The controllers are the ==brain== behind ==orchestration==. They are ==responsible for noticing==
==and responding when nodes, containers or endpoints goes down==. The controllers
==makes decisions to bring up new containers in such cases.==

The scheduler is r==esponsible for distributing work or containers across multiple==


On the ==worker nodes== you have the ==kubelet== which is the ==agent== that runs on each node
in the cluster. The agent is ==responsible for making sure that the containers are==
==running on the nodes as expected.==

You also have the ==kube-proxy== that is ==responsible for maintaining rules on the nodes==.

On the worker nodes you also have ==container runtime== that is ==responsible for running==
==containers==. And one example of that is ==Docker==. Now, it ==used to be Docker== for a long
time ==in the past== – because ==Kubernetes was originally built to orchestrate Docker==
==containers specifically==. However over a period of time it ==evolved== to ==support other==
==container runtimes==. So it ==no longer supports Docker directly==, but supports the
==runtime component of Docker which today is managed by ContainerD==. 
There is aseparate video that talks about the whole history of Kubernetes and Docker and how they started out together and what changed. 
So going forward we are going to refer to container runtime in kubernetes as
containerD.
And that's the high level architecture of a kubernetes cluster. And next we will look
into the kubernetes CLI.



