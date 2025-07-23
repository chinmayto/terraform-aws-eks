# Creating an Amazon EKS Kubernetes Cluster with Terraform Modules

Kubernetes (K8s) has become the de facto standard for orchestrating containerized applications. It provides powerful primitives for deploying, scaling, and managing containerized workloads, making it a top choice for modern DevOps teams and cloud-native development.

In this blog series, we’ll explore how to set up a production-ready Kubernetes environment on AWS using Amazon Elastic Kubernetes Service (EKS) and Terraform, starting with the foundational infrastructure.

### Why EKS?
Amazon EKS is a fully managed Kubernetes service that makes it easy to run Kubernetes on AWS without needing to install or operate your own control plane or nodes. EKS handles high availability, scalability, and patching of the Kubernetes control plane, so you can focus on running your applications instead of managing infrastructure.

Benefits of using EKS:
- Managed control plane: No need to run your own etcd or master nodes.
- Native AWS integration: IAM, VPC, CloudWatch, ALB, and more.
- Secure by default: Runs in a dedicated, isolated VPC.
- Scalable and production-ready.

### Terraform AWS Modules by Anton Babenko
Rather than writing complex Terraform configurations from scratch, we leverage terraform-aws-modules maintained by the community and led by Anton Babenko. These modules follow AWS best practices, are actively maintained, and support production-grade deployments with minimal boilerplate.

In our setup:
- The VPC module creates a network with public and private subnets.
- The EKS module provisions the EKS control plane and worker nodes in private subnets.

### Architecture Overview
Here’s how the setup works at a high level:

- VPC is created with 3 Availability Zones for high availability.
- Each AZ contains both a public and a private subnet.
- EKS worker nodes (EC2 instances) are launched in private subnets for better security.
- A NAT Gateway is provisioned in a public subnet to allow worker nodes in private subnets to pull images and updates from the internet (e.g., from ECR, Docker Hub).
- EKS control plane (managed by AWS) communicates with the worker nodes securely within the VPC.

This setup ensures that your nodes are not directly exposed to the internet while still having outbound internet access via the NAT gateway.

![alt text](/images/architecture.png)

### Step 1: Create the VPC
We start by provisioning a robust VPC with 3 private and 3 public subnets across different AZs:

```terraform
####################################################################################
### VPC Module Configuration
####################################################################################
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = var.vpc_name
  cidr = var.vpc_cidr_block

  azs = slice(data.aws_availability_zones.available_zones.names, 0, 3)


  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  tags = {
    Name        = var.vpc_name
    Environment = var.environment
    Terraform   = "true"
  }
}
```

### Step 2: Create the EKS Cluster
We use the terraform-aws-eks module to spin up the cluster. This will provision:
- A managed EKS control plane
- A node group with autoscaling enabled
- Nodes inside private subnets with internet access via NAT Gateway
```terraform
####################################################################################
###  EKS Cluster Module Configuration
####################################################################################
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.eks_cluster_name
  cluster_version = "1.32"

  cluster_endpoint_public_access = true
  enable_cluster_creator_admin_permissions = true

  eks_managed_node_groups = {
    example = {
      instance_types = ["t3.medium"]
      min_size       = 1
      max_size       = 5
      desired_size   = 2
    }
  }

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  tags = {
    Name        = var.eks_cluster_name
    Environment = var.environment
    Terraform   = "true"
  }

}
```

Terraform apply to create EKS Cluster:
```bash
Apply complete! Resources: 60 added, 0 changed, 0 destroyed.

Outputs:

cluster_endpoint = "https://EA6F63CF5CF44B594EA9533013CF21C4.gr7.us-east-1.eks.amazonaws.com"
cluster_name = "CT-EKS-Cluster"
```

![alt text](/images/eks_1.png)

![alt text](/images/eks_2.png)

### Step 3: Update Your Kubeconfig
Once the cluster is created, run the following command to configure your local kubectl to connect with EKS:
```bash
aws eks update-kubeconfig --name CT-EKS-Cluster --region us-east-1
```

### Step 4: Deploy a Sample Node.js App
Let’s deploy a simple Node.js application from a public ECR repository using a Kubernetes deployment and service YAML file.
```yml
---
apiVersion: v1
kind: Namespace
metadata:
  name: simple-nodejs-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: simple-nodejs-app
  name: deployment-nodejs-app
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: nodejs-app
  replicas: 5
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nodejs-app
    spec:
      containers:
      - image: public.ecr.aws/n4o6g6h8/simple-nodejs-app:latest
        imagePullPolicy: Always
        name: nodejs-app
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          failureThreshold: 3
---
apiVersion: v1
kind: Service
metadata:
  namespace: simple-nodejs-app
  name: service-nodejs-app
spec:
  type: ClusterIP
  ports:
    - port: 80
      name: http
      targetPort: 8080
  selector:
    app.kubernetes.io/name: nodejs-app
```
Apply this manifest:
```bash
$ kubectl apply -f simple-nodejs-app.yaml
namespace/simple-nodejs-app created
deployment.apps/deployment-nodejs-app created
service/service-nodejs-app created
```

Check deployment status:
```bash
$ kubectl get pods -n simple-nodejs-app
NAME                                     READY   STATUS    RESTARTS   AGE
deployment-nodejs-app-55555bc798-9p2h2   1/1     Running   0          29s
deployment-nodejs-app-55555bc798-b68hn   1/1     Running   0          29s
deployment-nodejs-app-55555bc798-grx5w   1/1     Running   0          29s
deployment-nodejs-app-55555bc798-mdzqp   1/1     Running   0          29s
deployment-nodejs-app-55555bc798-xgfz5   1/1     Running   0          29s

$ kubectl get svc -n simple-nodejs-app
NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service-nodejs-app   ClusterIP   172.20.83.36   <none>        80/TCP    54s
```

### Step 5: Access the App via Port Forwarding
Since our pods are in private subnets, you can access them using:
```bash
$ kubectl port-forward -n simple-nodejs-app deployment/deployment-nodejs-app 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```
Now open your browser at http://localhost:8080

![alt text](/images/webpage.png)

### Cleanup
Once you're done experimenting, clean up resources to avoid charges:
Delete the app:
```bash
kubectl delete -f simple-nodejs-app.yaml
```
Delete the EKS cluster:
```bash
terraform destory -auto-approve
```

### Conclusion
In this post, we successfully set up a production-grade Kubernetes cluster on AWS using EKS and Terraform AWS modules. We deployed a sample Node.js application using a public ECR image, managed its lifecycle with kubectl, and accessed it securely via port forwarding.

This is just the beginning. Kubernetes is a vast topic, and future posts will cover advanced concepts.

### References
- Github Repo: https://github.com/chinmayto/terraform-aws-eks
- Terraform EKS modules: https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/18.14.0
