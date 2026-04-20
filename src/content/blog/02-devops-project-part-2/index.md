---
title: "Architecting a Production-Ready Infrastructure [Part 2]"
description: "Infrastructure - Terraform"
date: "Apr 09 2026"
---

## Table of Contents
1. [Connecting Terraform to AWS](#1-connecting-terraform-to-aws)
2. [Connecting Terraform Cloud to AWS](#2-connecting-terraform-cloud-to-aws)
3. [Setting up the EKS architecture](#3-setting-up-the-eks-architecture)
- 3.1. [Cloud configuration in Terraform](#31-cloud-configuration-in-terraform)
- 3.2. [EKS Terraform configuration](#32-eks-terraform-configuration)

---

## 1. Connecting Terraform to AWS

For infrastructure provisioning, I have selected **Terraform** integrated with **Terraform Cloud**. This configuration allows me to centralize **remote state**, guarantee integrity through **state locking**, and perform **remote executions**. Additionally, it enables a native **CI/CD pipeline** that automates cluster updates whenever changes are detected in the repository's configuration manifests.

First, we must have an account in the following services: 
- GitHub
- AWS
- Terraform Cloud

And you must install the following clis:
- [AWS cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Terraform cli](https://developer.hashicorp.com/terraform/install)

Once thats donde we can start writing code.

I will create a repo on Github to store all my Terraform configuration files, then, after making sure everything works correctly, we will connect the repo to Terraform Cloud.

Since we are working with AWS we sill be using the Terraform AWS [provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs). Let's test the provider by deploying a simple S3 bucket from our local machine. 

We will have to grant access to Terraform from out AWS console so lets start by creating a User inside the IAM service and asigning administrator policies (it will be simpler this way). Create an access key for that user and set the following env variables:

```
% export AWS_ACCESS_KEY_ID="anaccesskey"
% export AWS_SECRET_ACCESS_KEY="asecretkey"
% export AWS_REGION="us-west-2"
```

**main.tf** looks like this:
```
# Configure the AWS Provider
provider "aws" {}

resource "aws_s3_bucket" "example" {
  bucket = "ergasap-test-bucket"

  tags = {
    Name        = "My bucket"
    Environment = "Dev"
  }
}
```

## 2. Connecting Terraform Cloud to AWS

After checking that AWS creds are correctly configured lets test Terraform Cloud, we will need to push our code to git since a workspace cannot be connected to an empty git repo.

We now need to set up Terraform Cloud, so lets start by creating an organization, then a workspace and finally a project. When creating the workspace make sure to choose version control workflow. 

Yo must set the same env variables as before but inside your new created workspace, so head over to your workspace, go to Variables and create them. 

Finally, lets make a plan, go to your workspace > Runs and "Start Plan". Your plan should look like this: ![Execution Plan](/02-devops-project-part-2/execution-plan.png).  

Run the plan and check in AWS console if everything was deployed succesfully. 

Finally, you can destroy everything by going to your workspace > Settings > Destruction and Deletion > Queue destroy plan.

Since we've connected our repo to this workspace, everytime a new commit occurs a run will be triggered.

We've made our first infra deployment using Terraform Cloud in AWS. 

## 3. Setting up the EKS architecture

### 3.1. Cloud configuration in Terraform

It's now time to set up the Kubernetes cluster architecture defined in Part 1 of the blog. 

Firstly, in our **main.tf** we will add this block of code to make remote execution from local applies, this will speed up the development phase since we don't need to be pushing our code everytime we want to test our code.

```
terraform {
  cloud {
    organization = "your-organization"
    workspaces {
      name = "your-workspace"
    }
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```
Then type **terraform login** in the console and make sure to create a new api token, paste the token in the console and type **terraform init**. Make a plan to make sure everything is connected correctly.

### 3.2. EKS Terraform configuration

For an EKS production ready environment using Terraform the best practice is using the predefined aws provider modules. For this project we will be using [**vpc**](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest) and [**eks**](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest) AWS Terraform modules. 

The **main.tf** file will look like this:

```
// main.tf

# Configure the AWS Provider
provider "aws" {}

# Obtener AZ disponible
data "aws_availability_zones" "available" {}

# VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "ergasap-vpc"
  cidr = "10.0.0.0/16"

  azs             = [data.aws_availability_zones.available.names[0], data.aws_availability_zones.available.names[1]]
  private_subnets = ["10.0.1.0/24", "10.0.11.0/24"]
  public_subnets  = ["10.0.2.0/24", "10.0.12.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  tags = {
    Environment = "dev"
  }
}

# EKS Cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.8"

  cluster_name    = "ergasap-eks"
  cluster_version = "1.29"

  subnet_ids = module.vpc.private_subnets
  vpc_id     = module.vpc.vpc_id

  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    default = {
      ami_type       = "AL2_x86_64"
      instance_types = ["t3.small", "t3a.small"]
      min_size       = 1
      max_size       = 2
      desired_size   = 1
      capacity_type = "SPOT"
    }
  }

  tags = {
    Environment = "dev"
  }
}
```

Terraform will deploy a VPC with 2 AZs, both AZs having a public and private subnet. A node group with one node on the private subnet. A NAT Gateway for the workers to have internet connection through an Internet Gateway.

The architecture should look something like this:

![AWS Architecture](/02-devops-project-part-2/aws-architecture.png)


**BEFORE THE APPLY** we must set our credentials to access correctly the EKS cluster. For this we must update out eks module with this new block of code: 

```
// main.tf

...

# EKS Cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.8"

  cluster_name    = "ergasap-eks"
  cluster_version = "1.29"

  subnet_ids = module.vpc.private_subnets
  vpc_id     = module.vpc.vpc_id

  cluster_endpoint_public_access = true
  enable_cluster_creator_admin_permissions = true
  authentication_mode = "API_AND_CONFIG_MAP"

  access_entries = {
    ergasap_macbook = {
      principal_arn     = "YOUR_ARN" 
      policy_associations = {
        admin = {
          policy_arn = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
          access_scope = {
            type = "cluster"
          }
        }
      }
    }
  }

  eks_managed_node_groups = {
    default = {
      ami_type       = "AL2_x86_64"
      instance_types = ["t3.small", "t3a.small"]
      min_size       = 1
      max_size       = 2
      desired_size   = 1
      capacity_type = "SPOT"
    }
  }

  tags = {
    Environment = "dev"
  }
}
```

A new "access_entries" must be created and filled with your corresponding ARN. To retrieve you ARN chain you must use this command:

```
aws eks update-kubeconfig --region eu-central-1 --name ergasap-eks
aws sts get-caller-identity
```

You can now apply your newly infra configuration with terraform. After the run has finished, you access your cluster with "kubectl"

![kubectl-get-all](/02-devops-project-part-2/kubectl-get-all.png)

Finally, we have deployed a Kubernetes cluster using EKS AWS service via Terraform Cloud using a VCS workflow!