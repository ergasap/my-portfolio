---
title: "Architecting a Production-Ready Infrastructure [Part 3]"
description: "App Deployments - Terraform & Helm provider"
date: "Apr 20 2026"
---

## Table of Contents
1. [New project organization](#1-new-project-organization)
2. [Observability apps](#2-observability-apps)
    - 2.1. [Installing Prometheus and Grafana](#21-prometheus-and-grafana)
    - 2.2. [Accessing the instances](#22-accessing-the-instances)
3. [NINGX webpage](#3-nginx-webpage)
    - 3.1. [Deploying the project locally](#31-deploying-the-project-locally)
    - 3.2. [Making changes to the web and triggering pipeline](#32-making-changes-to-the-web-and-triggering-pipeline)
    - 3.3. [Visualizing changes](#33-visualizing-changes)

---

## 1. New project organization

In this section we will be talking about deploying apps in our Kubernetes cluster.

To deploy either Grafana, Prometheus or WordPress instances using Terraform, we will modify our Terraform project. Now our Terraform project will be divided in 2 parts/files:

- **eks.tf** : will contain all the infrastructure configuration we already have (vpc module + eks module).
- **apps.tf** : will contain all the configuration for deploying all the k8s apps on our eks cluster (prometheus + grafana + wordpress).

So the folder structure will look something like this:

![New project folder structure](/03-devops-project-part-3/new-project-organization.png)

## 2. Observability apps

Observability tools like Prometheus and Grafana are essential because they let you measure, visualize, and understand what is happening inside your systems in real time.

They help detect failures early, identify performance bottlenecks, and reduce troubleshooting time by turning raw metrics into actionable insights.

In cloud and Kubernetes environments, they are key to maintaining reliability, improving availability, and making informed operational decisions.

### 2.1. Installing Prometheus and Grafana

To install these 2 observability tools we will update our terraform script with this new piece of code. Here we are installing both instances using Helm.

**NOTE**: Is important to know that it is posible that our eks instance node group is too small to handle new apps, so it is important to scale our cluster up.

```
# --- Monitoring: Prometheus + Grafana en EKS ---

terraform {
  required_providers {
    helm = {
      source = "hashicorp/helm"
    }
  }
}

variable "grafana_admin_password" {
  description = "Contraseña admin de Grafana"
  type        = string
  sensitive   = true
}

data "aws_eks_cluster" "this" {
  name       = module.eks.cluster_name
  depends_on = [module.eks]
}

data "aws_eks_cluster_auth" "this" {
  name       = module.eks.cluster_name
  depends_on = [module.eks]
}

provider "helm" {
  kubernetes {
    host                   = data.aws_eks_cluster.this.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.this.certificate_authority[0].data)
    token                  = data.aws_eks_cluster_auth.this.token
  }
}

resource "helm_release" "kube_prometheus_stack" {
  name             = "kube-prometheus-stack"
  repository       = "https://prometheus-community.github.io/helm-charts"
  chart            = "kube-prometheus-stack"
  namespace        = "monitoring"
  create_namespace = true

  depends_on = [module.eks]

  values = [
    yamlencode({
      grafana = {
        enabled       = true
        adminPassword = var.grafana_admin_password
        service = {
          type = "ClusterIP"
        }
      }

      prometheus = {
        enabled = true
        service = {
          type = "ClusterIP"
        }
        prometheusSpec = {
          retention = "10d"
        }
      }

      alertmanager = {
        enabled = false
      }
    })
  ]
}
```

After deploying we should have our pods looking like this:

![prometheus-grafana](/03-devops-project-part-3/prometheus-grafana.png)

### 2.2. Accessing the instances

To quickly access our newly created application we will use kubectl port forwarding option:

```
#Port-forward to Prometheus
kubectl port-forward svc/kube-prometheus-stack-prometheus -n monitoring 9090:9090

# Port-forward to Grafana
kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 3000:80
```

Then open browser and go to:

- Prometheus: [http://localhost:9090](http://localhost:9090)
- Grafana: [http://localhost:3000](http://localhost:3000)

## 3. NGINX webpage

To simulate a professional production environment, this project utilizes a NGINX instance integrated with a robust CI/CD pipeline. Every code commit triggers a GitHub Actions workflow that automates the image build and upload workflow to Docker Hub.

### 3.1. Deploying the project locally

To simulate a professional production environment, this project first utilizes a local Kind (Kubernetes in Docker) cluster for thorough validation before deployment. This approach minimizes costs by testing the NINGX CI/CD integration locally before pushing to AWS. Every code commit triggers a GitHub Actions workflow that automates the image build and upload to Docker Hub. 

To deploy the Nginx instance we could be using either Helm or plain Deployment manifests. For this experiment we will be using the second one.

The nginx deployment and service manifest should look something like this:

**nginx-deployment-service.yaml**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-nginx
  namespace: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mi-nginx
  template:
    metadata:
      labels:
        app: mi-nginx
    spec:
      containers:
      - name: nginx
        image: <YOUR_DOCKERHUB_USERNAME>/mi-nginx-custom:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: mi-nginx
  namespace: nginx
spec:
  selector:
    app: mi-nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

As seen the Deployment uses ***mi-nginx-custom*** image.

### 3.2. Making changes to the web and triggering pipeline

When a change is made to the nginx frontend website a commit must be done to ensure good git practices. A Github Action pipeline has been created to build a new docker image on every new commit of the website.

**build-push.yaml**
```
name: Build and Push Nginx Image

on:
  push:
    branches:
      - main
    paths:
      - 'mi-web/**'
      - 'Dockerfile'
      - '.github/workflows/**'

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ secrets.DOCKERHUB_USERNAME }}/mi-nginx-custom
          tags: |
            type=sha,prefix=,suffix=,format=short
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64
```
**DOCKERHUB_USERNAME** and **DOCKERHUB_TOKEN** env variables should be created inside the git repo (Settings > Secrets and variables > Actions > Repository Secrets).

When done we can test the pipeline by making any new changes to the website and commiting the changes to the repo triggering the pipeline. 

### 3.3. Visualizing changes

So we already have our nginx deployment running on our local cluster and a github pipeline that creates a new docker image, but how do we force the changes live on our pod? The Deployment describe on section 3.1 shows that *imagePullPolicy* is set to *Always* so eventually by deleting the running pod K8s will force the new created pod to download the new docker image that will reflect our changes. 

I tried to automate this process and not make it manually but the solutions I found where not exactly what I wanted. I studied the following solutions:
- Kell
- ArgoCD Image Updater
- Github Local Runners

The last one was the one I really liked the most. Setting up a runner on K8s that reads the Github Repo on new changes and deletes the pod, but as my cluster is locally the setting was really complex for what I wanted to make.

