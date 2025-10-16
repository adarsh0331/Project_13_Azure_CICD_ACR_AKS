
# Flask App Deployment with Azure DevOps, ACR, and AKS

This repository contains a complete setup for deploying a Flask application using Azure DevOps CI/CD pipelines, Azure Container Registry (ACR), and Azure Kubernetes Service (AKS). The process automates building a Docker image, pushing it to ACR, and deploying it to AKS using Kubernetes manifests.

---

## 🎯 Overview

- **Azure Repo**: Hosts the source code and configuration files.
- **Azure DevOps**: Manages the CI/CD pipeline to build and deploy the application.
- **ACR**: Stores the Docker images generated during the build process.
- **AKS**: Runs the containerized Flask application in a Kubernetes cluster.
- **Kubernetes Manifests**: Define the deployment and service configurations for AKS.

---

## 📂 Folder Structure

```
/ (repo root)
  ├── app/                   # Your Flask app source code (e.g., app.py, requirements.txt)
  ├── Dockerfile            # Docker configuration for building the image
  ├── azure-pipelines.yml   # CI/CD pipeline configuration
  └── k8s/
      ├── deployment.yaml  # Kubernetes deployment manifest
      └── service.yaml     # Kubernetes service manifest
```

---

## 🚀 Prerequisites

- An Azure account with an active subscription.
- Azure resources (Resource Group, ACR, and AKS) created manually (see [Manual Creation Steps](#-manual-creation-steps-azure-portal)).
- Azure DevOps project set up with a repository.
- Basic knowledge of Docker, Kubernetes, and Azure services.
- Git installed locally to push code to Azure Repos.

---

## 🧱 Manual Creation Steps (Azure Portal)

If you haven’t created the Azure resources yet, follow these steps:

### 🌐 Step 1 — Create a Resource Group
1. Sign in to the [Azure Portal](https://portal.azure.com).
2. Search **Resource groups** → click **Create**.
3. Fill details:
   - **Subscription**: Your Azure subscription.
   - **Resource group name**: `myCICD-rg` (customizable).
   - **Region**: Choose a nearby region (e.g., East US or Central India).
4. Click **Review + Create → Create**.

### 📦 Step 2 — Create Azure Container Registry (ACR)
1. Search **Container Registries** → click **Create**.
2. Fill details:
   - **Subscription**: Same as above.
   - **Resource group**: `myCICD-rg`.
   - **Registry name**: Unique name (e.g., `adarshacr123`).
   - **Location**: Same as RG.
   - **SKU**: Standard.
3. Click **Review + Create → Create**.
- **Verify**: After deployment, note the ACR URL (e.g., `adarshacr123.azurecr.io`).

### ☸️ Step 3 — Create Azure Kubernetes Service (AKS)
1. Search **Kubernetes services** → click **Create**.
2. **Basics** tab:
   - **Subscription**: Same as before.
   - **Resource Group**: `myCICD-rg`.
   - **Cluster Name**: `myakscluster`.
   - **Region**: Same as ACR.
   - **Node Size**: `Standard_B2s` (for testing).
   - **Node count**: 1 or 2.
3. **Authentication** tab:
   - Use **System-assigned managed identity**.
4. Click **Review + Create → Create**.
- **Wait**: 5–10 minutes for provisioning.

### 🔗 Step 4 — Connect AKS with ACR
1. Go to **AKS cluster → Settings → Integrations → Container registries**.
2. Click **Attach ACR → Select your ACR (e.g., adarshacr123)** → **Apply**.
- **Alternative CLI**:
  ```bash
  az aks update -n myakscluster -g myCICD-rg --attach-acr adarshacr123
  ```

### 🧠 Step 5 — Verify Access
- Use Azure Cloud Shell:
  ```bash
  az aks get-credentials -n myakscluster -g myCICD-rg
  kubectl get nodes
  ```
- If nodes appear, your cluster is ready.

## 📋 Step-by-Step Guide

### 🧱 Step 1 — Kubernetes Manifests

#### `k8s/deployment.yaml`

This file defines the Kubernetes deployment for your Flask application, including the number of replicas and the container image.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app-deployment
  labels:
    app: flask-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask-container
          image: <IMAGE_PLACEHOLDER>
          ports:
            - containerPort: 5000
          env:
            - name: FLASK_ENV
              value: "production"
```

- **Note**: The `<IMAGE_PLACEHOLDER>` will be replaced by the pipeline with the actual image URL from ACR (e.g., `youracrname.azurecr.io/flask-app:buildid`).

#### `k8s/service.yaml`

This file exposes your application using a LoadBalancer service, providing a public IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  type: LoadBalancer
  selector:
    app: flask-app
  ports:
    - port: 80
      targetPort: 5000
```

- **Note**: The `targetPort: 5000` matches the Flask app's default port, while `port: 80` is the external access point.

---

### 🧩 Step 2 — Azure Pipeline Configuration (`azure-pipelines.yml`)

The `azure-pipelines.yml` file automates the build and deployment process. Customize the variables (`ACR_NAME`, `AKS_RG`, `AKS_CLUSTER`) with your resource names.

```yaml
trigger:
  branches:
    include:
      - main

variables:
  ACR_NAME: yourACRname          # e.g., adarshacr123
  IMAGE_NAME: flask-app
  AKS_RG: yourResourceGroup      # e.g., myCICD-rg
  AKS_CLUSTER: yourAKScluster    # e.g., myakscluster

stages:
# ------------------------
# 1️⃣ Build and Push Image
# ------------------------
- stage: Build
  displayName: Build and Push Docker Image
  jobs:
    - job: Build
      pool:
        vmImage: 'ubuntu-latest'
      steps:
        - checkout: self

        - task: AzureCLI@2
          displayName: 'Login to Azure and ACR'
          inputs:
            azureSubscription: 'your-service-connection-name'
            scriptType: bash
            scriptLocation: inlineScript
            inlineScript: |
              echo "Logging into ACR..."
              az acr login -n $(ACR_NAME)

        - script: |
            IMAGE_TAG=$(Build.BuildId)
            echo "Building Docker image..."
            docker build -t $(ACR_NAME).azurecr.io/$(IMAGE_NAME):$IMAGE_TAG .
            echo "Pushing image to ACR..."
            docker push $(ACR_NAME).azurecr.io/$(IMAGE_NAME):$IMAGE_TAG
            echo "##vso[task.setvariable variable=imageTag]$IMAGE_TAG"
          displayName: 'Build and Push Image'

        - publish: $(System.DefaultWorkingDirectory)/k8s
          artifact: k8s-manifests
          displayName: 'Publish K8s Manifests'

# ------------------------
# 2️⃣ Deploy to AKS
# ------------------------
- stage: Deploy
  displayName: Deploy to AKS
  dependsOn: Build
  jobs:
    - deployment: DeployToAKS
      displayName: Deploy to AKS Cluster
      environment: 'aks/$(AKS_CLUSTER)'
      pool:
        vmImage: 'ubuntu-latest'
      strategy:
        runOnce:
          deploy:
            steps:
              - task: DownloadPipelineArtifact@2
                inputs:
                  artifact: k8s-manifests
                  path: $(Pipeline.Workspace)/manifests

              - task: AzureCLI@2
                displayName: 'Deploy to AKS using kubectl'
                inputs:
                  azureSubscription: 'your-service-connection-name'
                  scriptType: bash
                  scriptLocation: inlineScript
                  inlineScript: |
                    echo "Getting AKS credentials..."
                    az aks get-credentials --resource-group $(AKS_RG) --name $(AKS_CLUSTER) --admin
                    echo "Deploying application..."
                    IMAGE_TAG=$(imageTag)
                    sed -i "s|<IMAGE_PLACEHOLDER>|$(ACR_NAME).azurecr.io/$(IMAGE_NAME):$IMAGE_TAG|g" $(Pipeline.Workspace)/manifests/deployment.yaml
                    kubectl apply -f $(Pipeline.Workspace)/manifests/
                    echo "Deployment completed successfully!"
```

- **Variables**: Update `ACR_NAME`, `AKS_RG`, `AKS_CLUSTER`, and `azureSubscription` with your values.
- **Service Connection**: Replace `your-service-connection-name` with the name of your Azure DevOps service connection.

---

### 🧠 Step 3 — Explanation of the Pipeline

| Stage                | What Happens                                               | Tools Used         |
| -------------------- | ---------------------------------------------------------- | ------------------ |
| **Build**            | Pulls code, builds Docker image, and pushes to ACR         | Docker, Azure CLI  |
| **Artifact Publish** | Saves Kubernetes manifests for the deploy stage            | Azure DevOps       |
| **Deploy**           | Downloads manifests, updates image tag, and deploys to AKS | Azure CLI, kubectl |

---

### 🔑 Step 4 — Setup in Azure DevOps

1. **Create a Service Connection**
   - Go to your DevOps project → **Project Settings → Service Connections → New connection**.
   - Select **Azure Resource Manager → Service principal (automatic)**.
   - Name it (e.g., `azure-connection`) and save.
   - Use this name in `azure-pipelines.yml` (`azureSubscription: 'azure-connection'`).

2. **Push Code to Azure Repos**
   - Push your Flask app, `Dockerfile`, `azure-pipelines.yml`, and `k8s/` folder to the `main` branch.

3. **Run the Pipeline**
   - Go to **Pipelines → Create Pipeline → Azure Repos Git → Existing YAML file**.
   - Select `azure-pipelines.yml` → **Run**.

4. **Monitor Logs**
   - Check the **Build stage** logs for Docker image build and push to ACR.
   - Check the **Deploy stage** logs for AKS deployment.

5. **Verify Deployment**
   - Run the following commands in Azure Cloud Shell or locally (after `az aks get-credentials`):
     ```bash
     kubectl get pods
     kubectl get svc
     ```
   - Wait for the `EXTERNAL-IP` in the service output.
   - Open the IP in your browser to access your Flask app.

---

### 🧾 Step 5 — Example Dockerfile

If you don’t have a `Dockerfile`, use this basic version for a Flask app:

```dockerfile
# Base image
FROM python:3.9-slim

# Working directory
WORKDIR /app

# Copy app files
COPY . .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Expose Flask port
EXPOSE 5000

# Run app
CMD ["python", "app.py"]
```

- **Note**: Ensure `requirements.txt` includes your Flask dependencies (e.g., `flask==2.0.1`).

---

## 🌐 Accessing Your App

After deployment, retrieve the `EXTERNAL-IP` from `kubectl get svc` and open it in a browser. If configured correctly, your Flask app should be live!

---

## 🤝 Contributing

Feel free to submit issues or pull requests to improve this setup. Ensure all changes align with the CI/CD pipeline.

---

## 📜 License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
```

This `README.md` provides a comprehensive guide, including setup instructions, code snippets, and explanations. It’s designed to be user-friendly while covering all necessary details for deployment. Let me know if you'd like to adjust or expand any section!
