# AWS EKS AutoScaler for Node & pod
**EKS master Node AutoScaling is fully managed by EKS.** 

There are two Scaling in EKS: 
  1. Node AutoScaling (Cluster AutoScaler & karpenter)
  2. POD AutoScaling (Hpa)

 1. Node AutoScaling: 
    - It is "Cluster AutoScaler" means increases worker node, For this EKS use aws ASG service(Auto scaling group).
    - There are two Autoscaling are available for Node that is "Cluster AutoScaler" & "karpenter".
    - for Autoscaling metrics is important, EKS uses aws "Cloud Watch" for matrics the node.
      Matrics means the program keep on monitoring the CPU,RAM of Nodes. 

2. POD AutoScaling:
   - For Pod AutoScaling Hpa(Horizontal Pod AutoScaling) is used.
   - Kubernetes give metrics-server for pod autoscaling, we need to download that metrics server which helps us for AutoScaling. (In new Version v1.30 metrics-server is pre-configured)
   
## AutoScaler of Node:
Steps: 
1.  Create cluster with enabling asg access.
2.  Download Cluster Autoscaler YAML file.
3.  Edit Cluster Autoscaler YAML file.
4.  Apply Cluster Autoscaler YAML file. (Start Pod)

### Step-1: [This command is for create cluster with enable asg (--asg-access) (AutoScaler group)] 

       eksctl create cluster  --name pscluster2  --region ap-south-1  --version 1.30  --nodegroup-name psnodegp --instance-types t2.micro --nodes 3  --nodes-min 3  --nodes-max 6  --node-volume-size 8  --node-volume-type gp3  --ssh-access   --enable-ssm --instance-name psworkernode --asg-access  --managed

![image](https://github.com/user-attachments/assets/f58bac64-f176-41fa-a615-163758d6362a)

### Step-2: [Download asg file (This command download GitHub file named as "cluster-autoscaler-autodiscover.yaml")]
#### **What is cluster-autoscaler-autodiscover.yml in AWS EKS?**
The cluster-autoscaler-autodiscover.yml file is a Kubernetes manifest that deploys the Cluster Autoscaler in an EKS cluster. 
It is responsible for automatically scaling worker nodes (EC2 instances) based on pod demand.

#### **Why is cluster-autoscaler-autodiscover.yml Important?**
Even though AWS Auto Scaling Groups (ASG) handle scaling at the EC2 instance level, ASG alone does not understand Kubernetes pod scheduling. 
Cluster Autoscaler (CA) is required to ensure the number of worker nodes dynamically adjusts based on pending pods.

#### **Key Features of Cluster Autoscaler:**
- Monitors Pending Pods: If pods cannot be scheduled due to insufficient node capacity, CA requests ASG to scale out.
- Node Scale-In: If nodes are underutilized (empty), CA scales them down safely.
- Integrates with ASG: Uses AWS tags to discover and manage ASG dynamically.
- Works with K8s Scheduler: Ensures Kubernetes schedules pods efficiently.

           curl -O https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

### Step-3: [Open and edit YAML file "cluster-autoscaler-autodiscover.yaml"]
The YAML file sets up permissions and deploys the Cluster Autoscaler, which automatically scales our Kubernetes cluster based on workload demands.

     notepad cluster-autoscaler-autodiscover.yaml

Check right version of image as compare to kubernetes version while creating cluster:

![image](https://github.com/user-attachments/assets/7ae1c097-845d-49f1-bf24-3a5ae79afad7)

Set Auto scale down time:

![image](https://github.com/user-attachments/assets/9996df76-19a1-4ca7-a30e-529712833a0e)

In this file we replace "<YOUR CLUSTER NAME >"  > "pscluster2" my cluster name

### Step-4: [Apply file command(Create auto scaler pod)]

    kubectl apply -f cluster-autoscaler-autodiscover.yaml

After apply check cluster-autoscaler run:

     kubectl get pods -n kube-system
          
![image](https://github.com/user-attachments/assets/0c5bc16f-3902-47b1-8370-99ab1bce7de1)

Check lastest version or on which version our kubernetes work and use that image for autoscaler pod: (https://github.com/kubernetes/autoscaler/releases)

In my case i use version -->> **--version 1.30**  so i use image version ":v1.30.1"

command for deployment.apps/cluster-autoscaler image updated: (Optional commad/not needed)

      kubectl set image deployment cluster-autoscaler -n kube-system cluster-autoscaler=registry.k8s.io/autoscaling/cluster:v1.30.1
           
we can check nodes details info using command: 

      kubectl describe nodes
    
we can check specific nodes details info using command:

      kubectl describe nodes <nodeName>
      
## AutoScaling of POD:
The manual Horizontal scaling of pods we can done using command:

    kubectl scale deployment myps1  --replicas=5

Steps:  (Steps for Autoscaling of pods)
 1.  Install/Download Metrics server & check that is running.
 2.  Start Hps for deployment.

Note:
- There is no program running which give metrics of pods, without metrics server we can't use AutoScaling because of this Kubernetes provide "Metrics server".
- Metrics server keep on monitoring on all pods(CPU, RAM), If load come then auto horizontal scaling done by using Hpa.
- Metrics server is one kind of program running as pod.
- Hpa: Horizontal pod AutoScaling

### Step-1: [Install metrics server from YAML file]
Search on browser 'kubernetes metric server' [Metrics-server-pod-link](https://github.com/kubernetes-sigs/metrics-server)

    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

![image](https://github.com/user-attachments/assets/d56b865e-1477-478a-b6d3-0e29cd5cabd4)

Check running metrics server pod command:

    kubectl get pods -n kube-system

![image](https://github.com/user-attachments/assets/c1382a20-1d0a-4892-bbc1-b00cc5957a01)

    kubectl get deployment -n kube-system

![image](https://github.com/user-attachments/assets/93d02a37-a9bf-42b7-b501-1c7ffe9c93c1)

### Step-2: [Start Hpa for deployment]
- We can launch a pod with Hpa, it means we set that pod/deployment as Hpa which autoscale according to load.
- Hpa is just progarm which attach to Pod/deployment.
- for thid i use load generator app which provided by Kubernetes for checking AutoScaling working
[Load-generator-app-link](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough)

To demonstrate a HorizontalPodAutoscaler, you will first start a Deployment that runs a container using the hpa-example image, 
and expose it as a Service using the following manifest:

    kubectl apply -f https://k8s.io/examples/application/php-apache.yaml

![image](https://github.com/user-attachments/assets/0d9d8cf9-d507-4f89-896d-4482ae98bc40)

Create the HorizontalPodAutoscaler:

    kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

Here we use autoscale keyword which start hpa and set **cpu limit=50** and minimum and maximum replicas/scale limit.
 
![image](https://github.com/user-attachments/assets/05146fb9-9477-4dd4-8ecb-676ae52f108e)

Check Hpa start cmd:

    kubectl get hpa

![image](https://github.com/user-attachments/assets/67bd0c12-6f6a-4fd0-944a-028ac863d786)

Now Increase load (This command need fresh terminal or clear terminal):

    kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

![image](https://github.com/user-attachments/assets/956d05d4-0f3e-4d5d-a742-7dd239557392)

We can check the the our load increases by watch command:

    kubectl get hpa php-apache --watch

![image](https://github.com/user-attachments/assets/eaaadd03-1b40-4dcc-bdde-0e45c59027ae)

## See that Autoscaling of Pods and Nodes work:
Pods: 
  - The metrics server keep on monitoring on pod as soon as load increases on pod/deployment, It send information to hpa (Horizontal pod autoscaler) and based 
 upon that information Hpa keeps on launching & terminating pods.

![image](https://github.com/user-attachments/assets/5cf7457d-ce86-4e6f-bf2c-4e775da9154c)

![Screenshot 2024-08-10 115601](https://github.com/user-attachments/assets/64da63f6-a50b-45e7-b1f2-03117ecb49d2)

**Nodes:**
  **"Cluster Autoscaler"(YMAl file) keep on monitoring and send information to "asg". It capture information & that information send to asg
   and based upon information it keeps in launching and terminating Nodes(Instances).**

![image](https://github.com/user-attachments/assets/a348f046-68f9-4ad4-a809-265c947125e7)
   
![Screenshot 2024-08-10 115803](https://github.com/user-attachments/assets/5372de42-5a75-4274-9158-193586f728ab)

**Note:**
- As soon as we stop load generator then 1st pods scale-in to 1 and then Nodes Scale-in to 3 because min=3

![image](https://github.com/user-attachments/assets/e15b10e3-748e-4b53-94ab-80ef5ecde78a)
