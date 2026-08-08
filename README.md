````markdown
# AKS Deployment with ArgoCD and Helm

This repository demonstrates a GitOps-based deployment workflow using **Azure Kubernetes Service (AKS)**, **Helm**, **ArgoCD**, and **Azure DevOps**.

Instead of deploying directly from Azure DevOps to AKS, Azure DevOps is responsible only for Continuous Integration (CI), while ArgoCD handles Continuous Deployment (CD).

---

# Architecture

```text
Developer
    │
    ▼
Application Repository
    │
    ▼
Azure DevOps Pipeline (CI)
    │
    ├── Run Tests
    ├── Build Docker Image
    ├── Push Image to Azure Container Registry (ACR)
    └── Update Helm values.yaml (Image Tag)
            │
            ▼
      GitOps Repository
            │
            ▼
          ArgoCD
            │
            ▼
           AKS
```

---

# Repository Structure

```text
aks-deployment-with-argocd/

├── main-chart/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│
├── email-chart/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│
├── support-chart/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│
├── argocd/
│   ├── main-app.yaml
│   ├── email-app.yaml
│   └── support-app.yaml
│
└── azure-pipelines.yml
```

---

# Prerequisites

- Azure Subscription
- Azure Kubernetes Service (AKS)
- Azure Container Registry (ACR)
- Azure DevOps Project
- Helm 3.x
- ArgoCD installed on AKS
- GitHub Repository (GitOps)

---

# CI/CD Workflow

## Step 1 - Developer Pushes Code

The developer pushes application code to the source repository.

---

## Step 2 - Azure DevOps Pipeline

The pipeline performs the following tasks:

- Run Tests
- Build Docker Image
- Push Docker Image to ACR
- Update the Helm chart image tag
- Commit and Push changes to the GitOps repository

Azure DevOps **does not deploy to AKS**.

---

## Step 3 - ArgoCD

ArgoCD continuously monitors the GitOps repository.

Whenever a Helm chart changes, ArgoCD automatically:

- Detects the Git commit
- Pulls the latest Helm chart
- Deploys the updated release
- Keeps the cluster synchronized with Git

---

# Deployment Flow

```text
Developer
      │
      ▼
Git Push
      │
      ▼
Azure DevOps
      │
      ├── Run Tests
      ├── Build Docker Image
      ├── Push Image to ACR
      └── Update Helm values.yaml
                     │
                     ▼
              Git Commit
                     │
                     ▼
               GitOps Repository
                     │
                     ▼
                  ArgoCD
                     │
                     ▼
             Deploy to AKS
```

---

# Helm Charts

This repository contains three independent Helm charts.

| Chart | Purpose |
|--------|----------|
| main-chart | Main Application |
| email-chart | Email Service |
| support-chart | Support Service |

Each chart can be deployed independently by ArgoCD.

---

# ArgoCD Applications

Create one ArgoCD Application for each Helm chart.

```
main
email
support
```

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: main
spec:
  project: default

  source:
    repoURL: https://github.com/<username>/<repo>.git
    targetRevision: main
    path: main-chart

  destination:
    server: https://kubernetes.default.svc
    namespace: dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

# Azure DevOps Responsibilities

- Source Code Checkout
- Run Tests
- Build Docker Images
- Push Images to ACR
- Update Helm values.yaml
- Push Changes to Git

Azure DevOps **does not connect to AKS**.

---

# ArgoCD Responsibilities

- Watch Git Repository
- Detect Changes
- Pull Helm Charts
- Deploy to AKS
- Self Heal
- Prune Deleted Resources

---

# Benefits

- GitOps-based deployment
- No direct AKS access from Azure DevOps
- Automatic deployment
- Automatic rollback using Git history
- Declarative infrastructure
- Easy multi-environment management
- Fully automated Continuous Deployment

---

# Technologies Used

- Azure Kubernetes Service (AKS)
- Azure DevOps
- Azure Container Registry (ACR)
- Helm
- ArgoCD
- Kubernetes
- GitHub

---

# Multi-Environment Support

The solution can easily be extended for multiple environments.

```
values-dev.yaml
values-qa.yaml
values-prod.yaml
```

Each ArgoCD Application can reference its own values file.

---

# Author

AKKC
````
