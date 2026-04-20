---
title: "Architecting a Production-Ready Infrastructure [Part 3]"
description: "App Deployments - Terraform & Helm provider"
date: "Apr 20 2026"
---

## Table of Contents
1. [New project organization](#1-new-project-organization)
2. [Observability apps](#2-observability-apps)
    - 2.1. [Prometheus](#21-prometheus)
    - 2.2. [Grafana](#22-grafana)

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

### 2.1. Prometheus



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

### 2.2. Grafana