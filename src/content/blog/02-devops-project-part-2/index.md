---
title: "Architecting a Production-Ready Infrastructure [Part 2]"
description: "Infrastructure - Terraform"
date: "Apr 09 2026"
---

For infrastructure provisioning, I have selected **Terraform** integrated with **Terraform Cloud**. This configuration allows me to centralize **remote state**, guarantee integrity through **state locking**, and perform **remote executions**. Additionally, it enables a native **CI/CD pipeline** that automates cluster updates whenever changes are detected in the repository's configuration manifests.

First, we must have an account in the following services: 
- GitHub
- AWS
- Terraform Cloud

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

After checking that AWS creds are correctly configured lets test Terraform Cloud, we will need to push our code to git since a workspace cannot be connected to an empty git repo.

We now need to set up Terraform Cloud, so lets start by creating an organization, then a workspace and finally a project. When creating the workspace make sure to choose version control workflow. 

Yo must set the same env variables as before but inside your new created workspace, so head over to your workspace > Overview

To make it work make sure to use the correct env variables to give Terraform credentials for our AWS account. 