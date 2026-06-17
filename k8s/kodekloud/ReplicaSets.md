![[Pasted image 20250802070926.png]]

So what is a replica and why do we need a replicaset? Let’s go back to our ﬁrst scenario where we had a single POD running our application. What if for some reason, <mark style="background: #FFF3A3A6;">our application crashes and the POD fails? Users will no longer be able to access our application.</mark>

![[Pasted image 20250802071138.png]]

To prevent users from losing access to our application, we would <mark style="background: #FFF3A3A6;">like to have more than one instance or POD running at the same time</mark>. That way<mark style="background: #BBFABBA6;"> if one fails we still have our application running on the other one</mark>. And the <mark style="background: #BBFABBA6;">replicaset brings the failed one back to ensure a pre-defined number of replicas are always running</mark>. The replicaset helps us <mark style="background: #FFF3A3A6;">run multiple instances of a single POD</mark> <mark style="background: #FFF3A3A6;">in </mark>the kubernetes <mark style="background: #FFF3A3A6;">cluster</mark> thus providing <mark style="background: #FFF3A3A6;">High Availability</mark>.

So does that mean you can’t use a replicaset if you plan to have a single POD? No! Even <mark style="background: #FFF3A3A6;">if you have a single POD, the replication controller can help by automatically bringing up a new POD when the existing one fails</mark>. Thus the<mark style="background: #BBFABBA6;"> replicaset ensures that the specified number of PODs are running at all times. Even if it’s just 1 or 100.</mark>

Another reason we need replicaset is to <mark style="background: #BBFABBA6;">create multiple PODs to share the load across them.</mark> For example, in this <mark style="background: #FFF3A3A6;">simple scenario</mark> we have <mark style="background: #FFF3A3A6;">a single POD serving a set of users</mark>. When the <mark style="background: #FFF3A3A6;">number of users increase</mark> and I<mark style="background: #FFF3A3A6;">f we were to run out of resources on the first node</mark>,
![[Pasted image 20250802072115.png]]

We could <mark style="background: #BBFABBA6;">deploy additional PODs across other nodes in the cluster</mark>. As you can see, <mark style="background: #BBFABBA6;">the replicaset spans across multiple nodes in the cluster</mark>. It helps us <mark style="background: #FFF3A3A6;">balance the load across multiple pods on different nodes</mark> as well as <mark style="background: #FFF3A3A6;">scale our application when the demand increases. </mark>
So a <mark style="background: #FF5582A6;">Pod has a one-to-one relationship with a node</mark>. <mark style="background: #FF5582A6;">A pod can only run on one node</mark>. You <mark style="background: #FF5582A6;">cannot move a pod from one node to the other</mark>. A<mark style="background: #BBFABBA6;"> replicaset spans across the entire cluster</mark>.

![[Pasted image 20250802072444.png]]

So a Pod has a one-to-one relationship with a node. A pod can only run on one node at a time. You cannot move a pod from one node to the other. <mark style="background: #FF5582A6;">You'll have to kill it and recreate it on another node</mark>. Well, <mark style="background: #FFF3A3A6;">technically the scheduler decides which node a pod gets assigned to</mark> and there are ways for you to control that which is out of scope for this crash course. We discuss those in much detail in our CKA course. For now we will just stick to the basics. So <mark style="background: #FFF3A3A6;">a pod lives on one node.</mark>

A <mark style="background: #FFF3A3A6;">replicaset spans across the entire cluster.</mark> <mark style="background: #FFF3A3A6;">A replicaset can deploy a pod on any node in the cluster.</mark> <mark style="background: #FFF3A3A6;">It monitors the number of pods in the cluster and ensures enough are deployed at all times.</mark>



![[Pasted image 20250802073754.png]]

Let us now look at <mark style="background: #FFF3A3A6;">how we create a replicaset</mark>. As with the previous lecture, we start by <mark style="background: #FFF3A3A6;">creating a replicaset definition file</mark>. We will name it replicaset-definition.yml. As with any kubernetes definition file, we will have <mark style="background: #FFF3A3A6;">4 sections</mark>. The<mark style="background: #FFF3A3A6;"> apiVersion, kind, metadata and spec.</mark> The <mark style="background: #FFF3A3A6;">apiVersion is specific to what we are creating</mark>. In this case<mark style="background: #BBFABBA6;"> replicaset is supported in kubernetes apiVersion apps/v1</mark>.<mark style="background: #FF5582A6;"> If you get this wrong, you are likely to get an error that looks like this</mark>. It would say <mark style="background: #FFF3A3A6;">no match for kind ReplicaSet</mark>, because the<mark style="background: #FFF3A3A6;"> specified kubernetes api version has no support for ReplicaSet</mark>.

<mark style="background: #FFF3A3A6;">The kind as we know is ReplicaSet</mark>. Under <mark style="background: #FFF3A3A6;">metadata</mark>, we<mark style="background: #FFF3A3A6;"> will add a name and we will call it myapp-replicaset</mark>. And we will also add a few <mark style="background: #FFF3A3A6;">labels, app and type and assign values to them</mark>. So far, it has been very similar to how we created a POD in the previous section. The next is the most crucial part of the definition file and that is the specification written as <mark style="background: #FFF3A3A6;">spec</mark>. For any kubernetes definition file, the <mark style="background: #FF5582A6;">spec section defines what’s inside the object we are creating</mark>. In this case we know that the replicaset creates multiple instances of a POD. But what POD? We create a <mark style="background: #BBFABBA6;">template section</mark> under spec to <mark style="background: #BBFABBA6;">provide a POD template</mark> to be used by the replicaset to create replicas. Now, how do we DEFINE the POD template? It’s not that hard because, we have already done that in the previous exercise. 

![[Pasted image 20250802080116.png]]
Remember,We created a pod-definition file in the previous exercise. We could <mark style="background: #FFF3A3A6;">re-use the contents of the same file to populate the template section</mark>. <mark style="background: #FFF3A3A6;">Move all the contents of the pod-definition file into the template section of the replication controller</mark>, <mark style="background: #BBFABBA6;">except for the first two lines – which are apiVersion and kind</mark>. Remember whatever <mark style="background: #FFF3A3A6;">we move must be UNDER the template section</mark>. Meaning, they should be indented to the right and have more spaces before them than the template line itself. Looking at our file, we now have two metadata sections – one is for the ReplicaSet and another for the POD and we have two spec sections – one for each. We have nested two definition files together. The replication controller being the parent and the pod-definition being the child.

Now, there is something still missing. We haven’t mentioned h<mark style="background: #BBFABBA6;">ow many replicas we need</mark> in the replication controller. For that, add another property to the spec called <mark style="background: #BBFABBA6;">replicas</mark> and <mark style="background: #FFF3A3A6;">input the number of replicas you need under it</mark>. Remember that the template and replicas are direct children of the spec section. So they are siblings and must be on the same vertical line : having equal number of spaces before them.

Replica set requires a <mark style="background: #BBFABBA6;">selector definition</mark>. The selector section helps the replicaset <mark style="background: #FFF3A3A6;">identify what pods fall under it (chịu sự quản lý bởi nó)</mark>. But why would you have to specify what PODs fall under it, if you have provided the contents of the pod-definition file itself in the template? It’s BECAUSE, <mark style="background: #BBFABBA6;">replica set can ALSO manage pods that were not created as part of the replicaset creation</mark>. Say for example, <mark style="background: #FFF3A3A6;">there were pods created BEFORE the creation of the ReplicaSet that match the labels specified in the selector, the replica set will also take THOSE pods into consideration when creating the replicas.</mark> I will elaborate this in the next slide. For now know that it has to be written in the form of matchLabels as shown here. <mark style="background: #FFF3A3A6;">The matchLabels selector simply matches the labels specified under it to the labels on the PODs</mark>. The replicaset selector also provides many other options for matching labels that were not available in a replication controller.

Once the file is ready, run the<mark style="background: #FFF3A3A6;"> kubectl create command and input the file using the –f parameter</mark>. The replicaset Is created<mark style="background: #BBFABBA6;">. When the replicaset is created it first creates the PODs using the pod-definition template as many as required, which is 3 in this case</mark>. To <mark style="background: #FFF3A3A6;">view the list of created replicaset run the kubectl get replicaset command and you will see the replicaset listed</mark>. We can <mark style="background: #FFF3A3A6;">also see the desired number of replicas or pods</mark>, the <mark style="background: #FFF3A3A6;">current number of replicas and how many of them are ready</mark>. If you would like to see the pods that were created by the replicaset, run <mark style="background: #FFF3A3A6;">the kubectl get pods command and you will see 3 pods running</mark>. Note that <mark style="background: #BBFABBA6;">all of them are starting with the name of the replicaset which is myapp-replicaset indicating that they are all created automatically by the replicaset.</mark>


![[Pasted image 20250802084312.png]]

So what is the deal with<mark style="background: #FFF3A3A6;"> Labels and Selectors</mark>? <mark style="background: #FFF3A3A6;">Why do we label our PODs and objects in kubernetes</mark>? Let us look at a simple scenario. Say we <mark style="background: #FFF3A3A6;">deployed 3 instances of our frontend web application as 3 PODs</mark>. We would <mark style="background: #FFF3A3A6;">like to create a replication controller or replica set to ensure that we have 3 active PODs at anytime</mark>. And YES that is one of the use cases of replica sets. <mark style="background: #BBFABBA6;">You CAN use it to monitor existing pods,</mark> <mark style="background: #FFF3A3A6;">if you have them already created, as it IS in this example</mark>. <mark style="background: #FFF3A3A6;">In case they were not created, the replica set will create them for you.</mark> The<mark style="background: #FFF3A3A6;"> role of the replicaset is to monitor the pods and if any of them were to fail, deploy new ones</mark>. The <mark style="background: #BBFABBA6;">replica set is in FACT a process that monitors the pods</mark>. Now, <mark style="background: #FFF3A3A6;">how does the replicaset KNOW what pods to monitor</mark>? There could be 100s of other PODs in the cluster running different application. This is were <mark style="background: #FFF3A3A6;">labelling our PODs during creation comes in handy.</mark> We could now provide these <mark style="background: #FFF3A3A6;">labels as a filter for replicaset.</mark> <mark style="background: #FFF3A3A6;">Under the selector section we use the matchLabels filter and provide the same label that we used while creating the pods. This way the replicaset knows which pods to monitor</mark>.

![[Pasted image 20250802084735.png]]
Now let me ask you a question along the same lines. In the replicaset specification section we learned that there are 3 sections: Template, replicas and the selector. We need 3 replicas and we have updated our selector based on our discussion in the previous slide. Say for instance we have the same scenario as in the previous slide where <mark style="background: #FFF3A3A6;">we have 3 existing PODs that were created already and we need to create a replica set to monitor the PODs to ensure there are a minimum of 3 running at all times.</mark> <mark style="background: #BBFABBA6;">When the replication controller is created, it is NOT going to deploy a new instance of POD as 3 of them with matching labels are already created</mark>. <mark style="background: #FF5582A6;">In that case, do we really need to provide a template section in the replica-set specification</mark>, since we are not expecting the replicaset to create a new POD on deployment? <mark style="background: #BBFABBA6;">Yes we do</mark>, <mark style="background: #FF5582A6;">BECAUSE in case one of the PODs were to fail in the future, the replicaset needs to create a new one to maintain the desired number of PODs</mark>. <mark style="background: #BBFABBA6;">And for the replica set to create a new POD, the template definition section IS required.</mark>

![[Pasted image 20250802085011.png]]

Let’s look at how we scale the replicaset. <mark style="background: #FFF3A3A6;">Say we started with 3 replicas and in the future we decide to scale to 6.</mark> <mark style="background: #FFF3A3A6;">How do we update</mark> our replicaset to scale to 6 replicas? Well there are multiple ways to do it. T<mark style="background: #FFF3A3A6;">he first, is to update the number of replicas in the definition file to 6</mark>. Then run the <mark style="background: #BBFABBA6;">kubectl replace command </mark>specifying the same file using the <mark style="background: #BBFABBA6;">–f parameter</mark> and that <mark style="background: #FFF3A3A6;">will update the replicaset to have 6 replicas.</mark>

The second way to do it is to run the<mark style="background: #BBFABBA6;"> kubectl scale command</mark>.<mark style="background: #FFF3A3A6;"> Use the replicas parameter to provide the new number of replicas and specify the same file as input</mark>. <mark style="background: #BBFABBA6;">You may either input the definition file or provide the replicaset name in the TYPE Name format</mark>. However, Remember that<mark style="background: #BBFABBA6;"> using the file name as input will not result in the number of replicas being updated automatically in the file</mark>. In otherwords,<mark style="background: #FF5582A6;"> the number of replicas in the replicaset-definition file will still be 3 even though you scaled your replicaset to have 6 replicas using the kubectl scale command and the file as input.</mark>


<mark style="background: #FFF3A3A6;">There are also options available for automatically scaling the replicaset based onload, but that is an advanced topic and we will discuss it at a later time.</mark>

![[Pasted image 20250802085711.png]]

Let us now review the commands real quick. <mark style="background: #FFF3A3A6;">The kubectl create command, as we know, is used to create a replca set</mark>. You must provide the input file using the<mark style="background: #FFF3A3A6;"> –f parameter</mark>. <mark style="background: #FFF3A3A6;">Use the kubectl get command to see list of replicasets created</mark>. Use the<mark style="background: #FFF3A3A6;"> kubectl delete replicaset command followed by the name of the replica set to delete the replicaset</mark>. And then we have the <mark style="background: #FFF3A3A6;">kubectl replace command to replace or update replicaset and also the kubectl scale command to scale the replicas simply from the command line without having to modify the file.</mark>

<mark style="background: #BBFABBA6;">describe a replicaset : </mark>
controlplane ~ ➜  kubectl describe replicaset new-replica-set
Name:         new-replica-set
Namespace:    default
Selector:     name=busybox-pod
Labels: 
Annotations: 
Replicas:     4 current / 4 desired
Pods Status:  0 Running / 4 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  name=busybox-pod
  Containers:
   busybox-container:
    Image:      busybox777
    Port:    
    Host Port:  
    Command:
      sh
      -c
      echo Hello Kubernetes! && sleep 3600
    Environment: 
    Mounts:   
  Volumes:     
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  95s   replicaset-controller  Created pod: new-replica-set-kddvh
  Normal  SuccessfulCreate  95s   replicaset-controller  Created pod: new-replica-set-jkbv6
  Normal  SuccessfulCreate  95s   replicaset-controller  Created pod: new-replica-set-kpx68
  Normal  SuccessfulCreate  95s   replicaset-controller  Created pod: new-replica-set-gqvld



<mark style="background: #BBFABBA6;">if you want to delete many pods, using : kubectl delete pods -l name=busybox-pod
or things similar that.
</mark>


https://kodekloud.com/topic/labs-pods-with-yaml/