Bilkul bhai. Agar tum **Azure Application Gateway Ingress Controller (AGIC)** use karna chahte ho to main production-style steps de raha hoon.

> **Note:** Microsoft ab **Application Gateway for Containers (ALB Controller)** ko recommend karta hai Gateway API aur Ingress ke liye, lekin **AGIC** abhi bhi bahut saare production AKS clusters me use hota hai.

---

# Architecture

```text
                Internet
                    │
             Public IP
                    │
      Azure Application Gateway
                    │
          AGIC (Running in AKS)
                    │
        Watches Kubernetes Ingress
                    │
        ┌───────────┴────────────┐
        │                        │
   Service-A                Service-B
        │                        │
      Pods                    Pods
```

---

# Prerequisites

* Azure Subscription
* AKS Cluster
* Azure CLI
* kubectl
* Helm
* Contributor Permission

Check versions

```bash
az version
kubectl version
helm version
```

---

# Step 1 Create Resource Group

```bash
az group create \
--name aks-rg \
--location centralindia
```

---

# Step 2 Create VNet

Application Gateway aur AKS same VNet me hone chahiye.

```bash
az network vnet create \
-g aks-rg \
-n aks-vnet \
--address-prefix 10.0.0.0/8 \
--subnet-name aks-subnet \
--subnet-prefix 10.240.0.0/16
```

Application Gateway ke liye alag subnet

```bash
az network vnet subnet create \
-g aks-rg \
--vnet-name aks-vnet \
-n appgw-subnet \
--address-prefix 10.1.0.0/24
```

---

# Step 3 Create Public IP

```bash
az network public-ip create \
-g aks-rg \
-n appgw-pip \
--sku Standard
```

---

# Step 4 Create Application Gateway

```bash
az network application-gateway create \
-g aks-rg \
-n appgw \
--location centralindia \
--capacity 2 \
--sku Standard_v2 \
--public-ip-address appgw-pip \
--vnet-name aks-vnet \
--subnet appgw-subnet
```

---

# Step 5 Create AKS

AKS ko same VNet me deploy karo.

```bash
az aks create \
-g aks-rg \
-n myaks \
--network-plugin azure \
--vnet-subnet-id \
$(az network vnet subnet show \
-g aks-rg \
--vnet-name aks-vnet \
-n aks-subnet \
--query id -o tsv) \
--generate-ssh-keys
```

Credentials

```bash
az aks get-credentials \
-g aks-rg \
-n myaks
```

Check

```bash
kubectl get nodes
```

---

# Step 6 Enable Managed Identity

```bash
az aks show \
-g aks-rg \
-n myaks \
--query identity
```

---

# Step 7 Assign Contributor Role

Application Gateway ko modify karne ke liye AGIC ko permission chahiye.

AKS Managed Identity

```bash
AKS_IDENTITY=$(az aks show \
-g aks-rg \
-n myaks \
--query identity.principalId \
-o tsv)
```

Application Gateway ID

```bash
APPGW_ID=$(az network application-gateway show \
-g aks-rg \
-n appgw \
--query id \
-o tsv)
```

Assign Role

```bash
az role assignment create \
--assignee $AKS_IDENTITY \
--scope $APPGW_ID \
--role Contributor
```

---

# Step 8 Install AGIC

Add Helm Repo

```bash
helm repo add ingress-azure https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/

helm repo update
```

Application Gateway Resource ID

```bash
APPGW_RESOURCE_ID=$(az network application-gateway show \
-g aks-rg \
-n appgw \
--query id \
-o tsv)
```

Install

```bash
helm install ingress-azure ingress-azure/ingress-azure \
--set appgw.applicationGatewayID=$APPGW_RESOURCE_ID \
--set armAuth.type=managedIdentity \
--set armAuth.identityClientID=$(az aks show \
-g aks-rg \
-n myaks \
--query identityProfile.kubeletidentity.clientId \
-o tsv)
```

Check

```bash
kubectl get pods -n default
```

or

```bash
kubectl get deployment
```

---

# Step 9 Install Sample App

Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
```

Apply

```bash
kubectl apply -f deployment.yaml
```

---

# Step 10 Create Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

```bash
kubectl apply -f service.yaml
```

---

# Step 11 Create Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

Apply

```bash
kubectl apply -f ingress.yaml
```

---

# Step 12 Verify

Ingress

```bash
kubectl get ingress
```

Pods

```bash
kubectl get pods
```

Service

```bash
kubectl get svc
```

Application Gateway

```bash
az network application-gateway show \
-g aks-rg \
-n appgw
```

---

# Step 13 Access Application

Get Public IP

```bash
az network public-ip show \
-g aks-rg \
-n appgw-pip \
--query ipAddress \
-o tsv
```

Open

```
http://<ApplicationGatewayPublicIP>
```

NGINX page open ho jayegi.

---

# AGIC Flow

```text
Ingress YAML
      │
kubectl apply
      │
      ▼
Kubernetes API
      │
      ▼
AGIC Controller
      │
Reads Ingress
      │
Creates
  • Listener
  • Backend Pool
  • HTTP Settings
  • Health Probe
  • Routing Rule
      │
      ▼
Azure Application Gateway
      │
      ▼
Service
      │
      ▼
Pods
```

---

# Useful Commands

```bash
kubectl get ingress

kubectl describe ingress nginx-ingress

kubectl logs deployment/ingress-azure

kubectl get svc

kubectl get endpoints

kubectl get pods -o wide
```

---

# Production Best Practices

* Use **Application Gateway Standard_v2** or **WAF_v2** SKU.
* Enable **Managed Identity** instead of service principals.
* Use a **dedicated subnet** for Application Gateway (mandatory).
* Use **ClusterIP** services behind AGIC.
* Configure **TLS** with certificates stored in **Azure Key Vault**.
* Enable **HTTP to HTTPS redirection**.
* Use **health probes** and proper readiness/liveness probes in pods.
* Monitor AGIC and Application Gateway using **Azure Monitor** and **Log Analytics**.
* If exposing multiple applications, use **host-based** or **path-based** routing in separate Ingress resources.

## Important

Agar tum **AKS 2026** me naya deployment kar rahe ho, to interview aur learning dono ke liye **dono** technologies samajhna useful hai:

* **AGIC (Application Gateway Ingress Controller)** — traditional Ingress controller for Azure Application Gateway.
* **Application Gateway for Containers (ALB Controller)** — Microsoft's newer solution supporting **Gateway API** and modern Kubernetes networking.

Aaj kal naye production deployments me ALB Controller ko zyada preference mil rahi hai, jabki AGIC existing environments me bahut common hai.

Agar chaho, main **AGIC ka complete production setup** (AKS + VNet + Application Gateway + Helm + SSL + Multiple Microservices + Path-based Routing + Terraform + Azure DevOps pipeline) end-to-end bana sakta hoon, jaisa real company projects me use hota hai.
