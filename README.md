# EKS Getting Started Guide Configuration

This is the full configuration from https://www.terraform.io/docs/providers/aws/guides/eks-getting-started.html

See that guide for additional information.

NOTE: This full configuration utilizes the [Terraform http provider](https://www.terraform.io/docs/providers/http/index.html) to call out to icanhazip.com to determine your local workstation external IP for easily configuring EC2 Security Group access to the Kubernetes master servers. Feel free to replace this as necessary.




#What You’ll Need

Before you get started, you’ll need a few tools installed. Terraform is a tool to create, change, and improve infrastructure. Helm is a package management tool for Kubernetes. You’ll need to install them both:

terraform – https://www.terraform.io
helm – https://helm.sh

#Step By Step Commands
```
git clone https://github.com/vrahmanov/k8s-simplified.git

cd k8s-simplified/eks-getting-started

terraform init

terraform plan 

terraform apply 
```


#Created Resource
By default, the resources are targeted to be created in us-west-2, so bear that in mind if you go looking for the resources created in your console. This apply step will create many of the resources you need to get up and running initially, including:
```
VPC
IAM roles
Security groups
An internet gateway
Subnets
Autoscaling group
Route table
EKS cluster
Your kubectl configuration
```


#Setting Up kubectl

You will need the configuration output from Terraform in order to use kubectl to interact with your new cluster. Create your kube configuration directory, and output the configuration from Terraform into the config file using the Terraform output command:

```
mkdir ~/.kube/
terraform output kubeconfig>~/.kube/config
```

You’ll need kubectl, a command line tool to run commands against Kubernetes clusters, for the next step. Installation instructions can be found here. Once you’ve got this installed, you’ll want to check to make sure that you’re connected to your cluster by running kubectl version. Your output may vary slightly here:
```
$ kubectl version
```
Now let’s add the ConfigMap to the cluster from Terraform as well. The ConfigMap is a Kubernetes configuration, in this case for granting access to our EKS cluster. This ConfigMap allows our ec2 instances in the cluster to communicate with the EKS master, as well as allowing our user account access to run commands against the cluster. You’ll run the Terraform output command to a file, and the kubectl apply command to apply that file:
```
terraform output config_map_aws_auth > configmap.yml
kubectl apply -f configmap.yml
```



Once this is complete, you should see your nodes from your autoscaling group either starting to join or joined to the cluster. Once the second column reads Ready the node can have deployments pushed to it. Again, your output may vary here:
```
kubectl get nodes -o wide
```

At this point, your EKS cluster is up, the nodes have joined, and they are ready for a deployment!

#Helm
Next, you’ll install Helm. First you need to create a Kubernetes ServiceAccount for tiller, which allows helm to talk to the cluster:


```
cat >tiller-user.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: tiller
```

Now, you apply the ServiceAccount with kubectl, and install helm with the init command:
```
$ kubectl apply -f tiller-user.yaml
$ helm init --service-account tiller
```

You will need a way for our Airflow deployment to communicate with the outside world. For this, you will install nginx-ingress, an ingress controller that uses ConfigMap to store nginx configurations. Nginx is an industry standard software for web and proxy servers. We will use the proxy feature to serve up our Airflow web interface. Install nginx-ingress via the helm chart:
```
helm install \
stable/nginx-ingress \
--name my-nginx \
--set rbac.create=true
```

#airflow
You need to override some values in the Airflow chart to tell it to use the nginx ingress controller. You’ll want to replace airflow-k8s.aledade.com with a hostname of your own:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
EOF
```

Finally, you install Airflow via the helm chart and the values file you just created using the helm install command:
```
helm install \
--namespace "airflow" \
stable/airflow \
--name airflow \
-f values.yaml
```
This may take a few moments before all of the pods are ready, and you can monitor the progress with:
```
$ watch "kubectl get pods -n airflow"

```
Even after the pods are running, I’ve found it takes at least five minutes for everything to completely spin up.

You can find out the internet accessible endpoint by querying the services and looking for the LoadBalancer Ingress


```
kubectl describe services |grep ^LoadBalancer LoadBalancer Ingress: 8a69022e5f102be1072e5fb1087f5fbe-e907efv7e8.us-west-2.elb.amazonaws.com

```

If you visit this URL, you will find the flower interface, a web tool for monitoring and administering celery clusters.

To reach the Airflow administrative interface, you will need to add an entry to /etc/hosts, but first you need to get the IP address of that LoadBalancer Ingress, and add it to your /etc/hosts:
```
cat >values.yaml <<EOF
ingress:
  enabled: true
  web:
    path: "/"
    host: "airflow-k8s.aledade.com"
    tls:
      enabled: true
    annotations:
      kubernetes.io/ingress.class: "nginx"
EOF
```

Afterwards, you can reach the Airflow administrative interface from the URL http://airflow-k8s.aledade.com in your browser. Under a production environment, you would replace airflow-k8s.aledade.com with a FQDN that you can add as an alias in route53 to point to the ELB created by the LoadBalancer Ingress.

#CleanUp
To destroy these resources, delete the helm deployments, and issue a destroy with Terraform
```
$ helm del --purge airflow;
$ helm del --purge my-nginx;
$ terraform destroy
```



==========================================================
Kubernetes 101: Pods, Nodes, Containers, and Clusters

'''Kubernetes is quickly becoming the new standard for deploying and managing software in the cloud. With all the power Kubernetes provides, however, comes a steep learning curve. As a newcomer, trying to parse the official documentation can be overwhelming. There are many different pieces that make up the system, and it can be hard to tell which ones are relevant for your use case. This blog post will provide a simplified view of Kubernetes, but it will attempt to give a high-level overview of the most important components and how they fit together.
First, lets look at how hardware is represented'''


*Hardware*

Nodes
![GitHub Logo](https://miro.medium.com/max/3465/1*uyMd-QxYaOk_APwtuScsOg.png)
A node is the smallest unit of computing hardware in Kubernetes. It is a representation of a single machine in your cluster. In most production systems, a node will likely be either a physical machine in a datacenter, or virtual machine hosted on a cloud provider like Google Cloud Platform. Don’t let conventions limit you, however; in theory, you can make a node out of almost anything.
Thinking of a machine as a “node” allows us to insert a layer of abstraction. Now, instead of worrying about the unique characteristics of any individual machine, we can instead simply view each machine as a set of CPU and RAM resources that can be utilized. In this way, any machine can substitute any other machine in a Kubernetes cluster.


The Cluster
![GitHub Logo](https://miro.medium.com/max/3270/1*KoMzLETQeN-c63x7xzSKPw.png)

Although working with individual nodes can be useful, it’s not the Kubernetes way. In general, you should think about the cluster as a whole, instead of worrying about the state of individual nodes.
In Kubernetes, nodes pool together their resources to form a more powerful machine. When you deploy programs onto the cluster, it intelligently handles distributing work to the individual nodes for you. If any nodes are added or removed, the cluster will shift around work as necessary. It shouldn’t matter to the program, or the programmer, which individual machines are actually running the code.
If this kind of hivemind-like system reminds you of the Borg from Star Trek, you’re not alone; “Borg” is the name for the internal Google project Kubernetes was based on.



Persistent Volumes
![GitHub Logo](https://miro.medium.com/max/3437/1*kF57zE9a5YCzhILHdmuRvQ.png)

Because programs running on your cluster aren’t guaranteed to run on a specific node, data can’t be saved to any arbitrary place in the file system. If a program tries to save data to a file for later, but is then relocated onto a new node, the file will no longer be where the program expects it to be. For this reason, the traditional local storage associated to each node is treated as a temporary cache to hold programs, but any data saved locally can not be expected to persist.

To store data permanently, Kubernetes uses Persistent Volumes. While the CPU and RAM resources of all nodes are effectively pooled and managed by the cluster, persistent file storage is not. Instead, local or cloud drives can be attached to the cluster as a Persistent Volume. This can be thought of as plugging an external hard drive in to the cluster. Persistent Volumes provide a file system that can be mounted to the cluster, without being associated with any particular node.


Software

Containers
![GitHub Logo](https://miro.medium.com/max/5000/1*ILinzzMdnD5oQ6Tu2bfBgQ.png)

Programs running on Kubernetes are packaged as Linux containers. Containers are a widely accepted standard, so there are already many pre-built images that can be deployed on Kubernetes.
Containerization allows you to create self-contained Linux execution environments. Any program and all its dependencies can be bundled up into a single file and then shared on the internet. Anyone can download the container and deploy it on their infrastructure with very little setup required. Creating a container can be done programmatically, allowing powerful CI and CD pipelines to be formed.
Multiple programs can be added into a single container, but you should limit yourself to one process per container if at all possible. It’s better to have many small containers than one large one. If each container has a tight focus, updates are easier to deploy and issues are easier to diagnose.


Pods
![GitHub Logo](https://miro.medium.com/max/6000/1*8OD0MgDNu3Csq0tGpS8Obg.png)

Unlike other systems you may have used in the past, Kubernetes doesn’t run containers directly; instead it wraps one or more containers into a higher-level structure called a pod. Any containers in the same pod will share the same resources and local network. Containers can easily communicate with other containers in the same pod as though they were on the same machine while maintaining a degree of isolation from others.
Pods are used as the unit of replication in Kubernetes. If your application becomes too popular and a single pod instance can’t carry the load, Kubernetes can be configured to deploy new replicas of your pod to the cluster as necessary. Even when not under heavy load, it is standard to have multiple copies of a pod running at any time in a production system to allow load balancing and failure resistance.
Pods can hold multiple containers, but you should limit yourself when possible. Because pods are scaled up and down as a unit, all containers in a pod must scale together, regardless of their individual needs. This leads to wasted resources and an expensive bill. To resolve this, pods should remain as small as possible, typically holding only a main process and its tightly-coupled helper containers (these helper containers are typically referred to as “side-cars”).


Deployments
![test](https://miro.medium.com/max/1200/1*iTAVk3glVD95hb-X3HiCKg.png)



Although pods are the basic unit of computation in Kubernetes, they are not typically directly launched on a cluster. Instead, pods are usually managed by one more layer of abstraction: the deployment.
A deployment’s primary purpose is to declare how many replicas of a pod should be running at a time. When a deployment is added to the cluster, it will automatically spin up the requested number of pods, and then monitor them. If a pod dies, the deployment will automatically re-create it.
Using a deployment, you don’t have to deal with pods manually. You can just declare the desired state of the system, and it will be managed for you automatically.


Ingress
![GitHub Logo](https://miro.medium.com/max/3282/1*tBJ-_g4Mk5OkfzLEHrRsRw.png)

Using the concepts described above, you can create a cluster of nodes, and launch deployments of pods onto the cluster. There is one last problem to solve, however: allowing external traffic to your application.
By default, Kubernetes provides isolation between pods and the outside world. If you want to communicate with a service running in a pod, you have to open up a channel for communication. This is referred to as ingress.
There are multiple ways to add ingress to your cluster. The most common ways are by adding either an Ingress controller, or a LoadBalancer. The exact tradeoffs between these two options are out of scope for this post, but you must be aware that ingress is something you need to handle before you can experiment with Kubernetes.
