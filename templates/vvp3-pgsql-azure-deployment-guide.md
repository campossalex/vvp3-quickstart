# Ververica Platform 3 (VVP 3) - Azure Deployment Guide

This guide details how to deploy Ververica Platform 3 on Microsoft Azure using Azure Kubernetes Service (AKS), Azure Database for PostgreSQL Flexible Server, Azure Blob Storage with Workload Identity, Microsoft Entra ID (OIDC) authentication, and Ingress exposure.

---

## Architecture Overview

- **Kubernetes Cluster**: Azure Kubernetes Service (AKS) with Workload Identity and OIDC Issuer enabled.
- **Database Backend**: Azure Database for PostgreSQL (Flexible Server, PostgreSQL 15+).
- **Object Storage**: Azure Blob Storage (Container for checkpoints, savepoints, and artifacts) accessed via Azure Workload Identity (federated credentials).
- **Authentication**: Microsoft Entra ID via OpenID Connect (OIDC).
- **Networking**: Ingress Controller (e.g., NGINX) with TLS terminating at the VVP `api-gateway` service.

---

## 1. Azure Infrastructure Setup (Azure CLI)

### 1.1 Variables Configuration

```bash
RESOURCE_GROUP="rg-vvp3"
LOCATION="westeurope"
CLUSTER_NAME="aks-vvp3"
DB_SERVER_NAME="psql-vvp3-$RANDOM"
DB_ADMIN_USER="vvpadmin"
DB_ADMIN_PASS="<STRONG_PASSWORD>"
STORAGE_ACCOUNT="stvvp$RANDOM"
CONTAINER_NAME="vvp-data"
IDENTITY_NAME="id-vvp3"
```

### 1.2 Resource Group

```bash
az group create --name $RESOURCE_GROUP --location $LOCATION
```

### 1.3 Azure Kubernetes Service (AKS)

Deploy AKS with OIDC Issuer and Workload Identity enabled:

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --enable-managed-identity \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --node-count 3 \
  --node-vm-size Standard_D8s_v5 \
  --os-disk-size 128 \
  --generate-ssh-keys

az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
```

### 1.4 Azure Database for PostgreSQL Flexible Server

VVP 3.1+ requires PostgreSQL 15+. VVP automatically initializes required sub-databases (e.g. `vvp-appmanager`, `vvp-meta`, `accesscontrol`) during startup (`createIfMissing: true`), so ensure the configured user has `CREATEDB` privileges.

```bash
az postgres flexible-server create \
  --resource-group $RESOURCE_GROUP \
  --name $DB_SERVER_NAME \
  --location $LOCATION \
  --admin-user $DB_ADMIN_USER \
  --admin-password $DB_ADMIN_PASS \
  --sku-name Standard_D4ds_v5 \
  --tier GeneralPurpose \
  --version 16 \
  --storage-size 128

# Allow connection from Azure services / AKS subnet
az postgres flexible-server firewall-rule create \
  --resource-group $RESOURCE_GROUP \
  --name $DB_SERVER_NAME \
  --rule-name AllowAllAzureIPs \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

### 1.5 Azure Storage Account

```bash
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hierarchical-namespace false

az storage container create \
  --account-name $STORAGE_ACCOUNT \
  --name $CONTAINER_NAME \
  --auth-mode login
```

---

## 2. Managed Identity & Azure Workload Identity

Azure Workload Identity allows VVP pods and Flink pods to securely authenticate against Azure Blob Storage without storing static account keys or connection strings.

### 2.1 Create Managed Identity & Assign Roles

```bash
az identity create --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP

USER_ASSIGNED_CLIENT_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query clientId -o tsv)
STORAGE_ID=$(az storage account show --name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP --query id -o tsv)

# Assign Storage Blob Data Contributor to the managed identity
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee $USER_ASSIGNED_CLIENT_ID \
  --scope $STORAGE_ID
```

### 2.2 Configure Federated Credentials

Federate the identity with ServiceAccounts in `vvp-system` (VVP platform services) and `vvp-deploy` (Flink application pods):

```bash
AKS_OIDC_ISSUER="$(az aks show -n $CLUSTER_NAME -g $RESOURCE_GROUP --query "oidcIssuerProfile.issuerUrl" -otsv)"

# For VVP system components
az identity federated-credential create \
  --name "vvp-system-fedcred" \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --issuer "$AKS_OIDC_ISSUER" \
  --subject "system:serviceaccount:vvp-system:vvp-workload-sa" \
  --audience "api://AzureADTokenExchange"

# For Flink Deployment pods
az identity federated-credential create \
  --name "vvp-deploy-fedcred" \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --issuer "$AKS_OIDC_ISSUER" \
  --subject "system:serviceaccount:vvp-deploy:vvp-workload-sa" \
  --audience "api://AzureADTokenExchange"
```

### 2.3 Create Namespaces and Kubernetes ServiceAccounts

```yaml
# workload-serviceaccounts.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: vvp-system
---
apiVersion: v1
kind: Namespace
metadata:
  name: vvp-deploy
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vvp-workload-sa
  namespace: vvp-system
  annotations:
    azure.workload.identity/client-id: "<USER_ASSIGNED_CLIENT_ID>"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vvp-workload-sa
  namespace: vvp-deploy
  annotations:
    azure.workload.identity/client-id: "<USER_ASSIGNED_CLIENT_ID>"
```

Apply with:
```bash
sed -i "s/<USER_ASSIGNED_CLIENT_ID>/${USER_ASSIGNED_CLIENT_ID}/g" workload-serviceaccounts.yaml
kubectl apply -f workload-serviceaccounts.yaml
```

---

## 3. Microsoft Entra ID (OIDC) Registration

1. Navigate to **Microsoft Entra ID** > **App registrations** > **New registration**:
   - **Name**: `Ververica Platform 3`
   - **Supported account types**: `Accounts in this organizational directory only` (Single tenant)
   - **Redirect URI**: Web platform -> `<redacted URL><YOUR_VVP_FQDN>/authentication/callback`
2. Under **Certificates & secrets**, generate a new **Client secret**. Note down the value.
3. Under **Token configuration** *(optional, for VVP 3.1.1+ group-based RBAC)*:
   - Click **Add groups claim** -> Select **Security groups**.
4. Retrieve the following values:
   - `Application (client) ID` (`<ENTRA_CLIENT_ID>`)
   - `Directory (tenant) ID` (`<TENANT_ID>`)
   - `Client secret value` (`<ENTRA_CLIENT_SECRET>`)

---

## 4. VVP 3 Helm Configuration

### 4.1 Create Database Secret

```bash
kubectl -n vvp-system create secret generic postgres-secret \
  --from-literal=postgres-password="${DB_ADMIN_PASS}"
```

### 4.2 Create Registry Secret

If using Ververica's private container registry:
```bash
kubectl -n vvp-system create secret docker-registry ververica-registry \
  --docker-server=registry.ververica.cloud \
  --docker-username="<REGISTRY_USERNAME>" \
  --docker-password="<REGISTRY_PASSWORD>"
```

### 4.3 `values-vvp-azure.yaml`

```yaml
global:
  imagePullSecretName: ververica-registry

  # Azure Database for PostgreSQL Flexible Server
  database:
    provider: postgresql
    host: "<DB_SERVER_NAME>.postgres.database.azure.com"
    port: 5432
    name: "postgres"
    user: "vvpadmin"
    passwordSecret: postgres-secret
    ssl:
      mode: require

  # Azure Blob Storage Configuration
  blobStorage:
    baseUri: "wasbs://vvp-data@<STORAGE_ACCOUNT>.blob.core.windows.net/"

  # Microsoft Entra ID (OIDC) Authentication
  authentication:
    callbackUrl: "<redacted URL><YOUR_VVP_FQDN>/authentication/callback"
    oidc:
      enabled: true
      clientId: "<ENTRA_CLIENT_ID>"
      clientSecret: "<ENTRA_CLIENT_SECRET>"
      discoveryUri: "<redacted URL><TENANT_ID>/v2.0/.well-known/openid-configuration"
      groupsClaim: "groups"

  # Initial Admin access bootstrapping
  initialAccessFileContent:
    initialAccess:
      - userId: "alex.campos@yourdomain.com"
        workspaceId: defaultworkspace
        role: ADMIN
      - userId: "alex.campos@yourdomain.com"
        workspaceId: defaultworkspace
        namespaceId: default
        role: ADMIN

# Azure Workload Identity configuration for VVP system pods
vvp:
  podAnnotations:
    azure.workload.identity/use: "true"
  serviceAccount:
    name: vvp-workload-sa
```

### 4.4 Deploy via Helm

```bash
helm upgrade --install ververica-platform \
  oci://registry.ververica.cloud/platform-charts/ververica-platform \
  --version 3.1.3 \
  --namespace vvp-system \
  --values values-vvp-azure.yaml
```

---

## 5. Exposing the VVP 3 Web Console

All UI and API traffic in VVP 3 is served by the `api-gateway` service (port 8080) in the `vvp-system` namespace.

### Option A: Ingress with TLS (Production Standard)

Requires `ingress-nginx` and `cert-manager`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vvp-ingress
  namespace: vvp-system
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "1024m"
spec:
  tls:
    - hosts:
        - vvp.yourdomain.com
      secretName: vvp-tls-cert
  rules:
    - host: vvp.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-gateway
                port:
                  number: 8080
```

### Option B: Azure Public Load Balancer (Quick Validation)

Patch the `api-gateway` service directly to provision an Azure Public IP:

```bash
kubectl -n vvp-system patch svc api-gateway -p '{"spec": {"type": "LoadBalancer"}}'
kubectl -n vvp-system get svc api-gateway -w
```

*Note: If testing over HTTP with the public IP, update the Entra ID Redirect URI to `<redacted URL><EXTERNAL_IP>:8080/authentication/callback`.*

### Option C: Local Port Forwarding

```bash
kubectl port-forward svc/api-gateway 8080:8080 -n vvp-system
```
Access at `<redacted URL>`.

