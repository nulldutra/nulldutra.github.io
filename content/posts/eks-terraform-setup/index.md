+++
title = "Deploying an EKS cluster using Terraform"
date = "2024-05-25"
+++

In this post, I will explain how to deploy an EKS cluster using Terraform and the public 
module of the project terraform-aws-module.

<!--more-->

## Environment variables

The module execute kubectl commands to configure the aws-auth configmap, we need
export environment variables such as AWS region, credentials, etc.

Example:

```
export AWS_ACCESS_KEY_ID=xxxxxx
export AWS_SECRET_ACCESS_KEY=xxxxxx
export AWS_REGION=us-east-1
```

## Configuring the virtual private network (VPC)

We need a virtual private network to deploy the EKS cluster and node group. AWS has
the default network, but, in this case, I choose to create the VPC.

Open a file with name vpc.tf

```
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "eks-example-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.10.0/24", "10.0.11.0/24"]

  enable_nat_gateway = true
  enable_vpn_gateway = false

  tags = {
    Terraform = "true"
    Environment = "dev"
  }
}
```

```
terraform init && terraform apply
```

## Creating the eks.tf

I created the cluster with public access enabled. In a production environment, this is a not
good practice, change the input 'cluster_endpoint_public_access' to false and connect on cluster using
a private network.

The module of the EKS has many options to deploy the EKS and the node groups.
I will deploy one node group with spot EC2, see the complete documentation for other
examples:

* https://github.com/terraform-aws-modules/terraform-aws-eks/tree/master/examples

Open the file eks.tf and add the content

```
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "eks-example"
  cluster_version = "1.29"

  cluster_endpoint_public_access = true

  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent = true
    }
  }

  vpc_id                   = module.vpc.vpc_id
  subnet_ids               = module.vpc.private_subnets
  control_plane_subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    spot-node-group = {
      min_size     = 2
      max_size     = 3
      desired_size = 2

      instance_types = ["t3.large"]
      capacity_type  = "SPOT"
    }
  }

  enable_cluster_creator_admin_permissions = true

  tags = {
    Environment = "dev"
    Terraform   = "true"
  }
}
```

```
terraform init && terraform apply
```

## Accessing the EKS

```
aws eks update-kubeconfig --name eks-example --region us-east-1
```

### Listing all pods

```
kubectl get pods -A
```

```
NAMESPACE     NAME                     READY   STATUS    RESTARTS   AGE
kube-system   aws-node-6kpns           2/2     Running   0          38m
kube-system   aws-node-8tjtq           2/2     Running   0          38m
kube-system   coredns-bf47b49b-rqzlk   1/1     Running   0          38m
kube-system   coredns-bf47b49b-wx77w   1/1     Running   0          38m
kube-system   kube-proxy-5jz72         1/1     Running   0          38m
kube-system   kube-proxy-88x7r         1/1     Running   0          38m
```

### Listing nodes

```
kubectl get nodes
```

```
NAME                        STATUS   ROLES    AGE   VERSION
ip-10-0-1-88.ec2.internal   Ready    <none>   40m   v1.29.3-eks-ae9a62a
ip-10-0-2-22.ec2.internal   Ready    <none>   40m   v1.29.3-eks-ae9a62a
```

