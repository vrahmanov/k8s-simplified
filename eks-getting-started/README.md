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
