## GitOps Tools: ArgoCD

ArgoCD is a GitOps tool designed to keep Kubernetes clusters synchronized with Git repositories.

Key Components of ArgoCD:

VCS: Connects to Git repositories and retrieves the desired state.

Application Controller: Runs inside the Kubernetes cluster and compares the current state of deployed resources to the desired state defined in Git.

API Service: Provides authentication and UI/CLI access to interact with ArgoCD.

Redis: Caches data to improve performance and handle state restoration after crashes.

Dex: Provides SSO (Single Sign-On) capabilities for authentication.

## ArgoCD Installation
```bash
You can install ArgoCD using three methods:

YAML Manifests

Helm

Operator
```
## Commands to Install ArgoCD (via YAML):
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
## Access ArgoCD:
Change the ArgoCD service type to NodePort for external access:

```bash
kubectl edit svc argocd-server -n argocd # Change from ClusterIP to NodePort
```
Once the service is exposed, access ArgoCD UI using the external IP and port.

## CI/CD Pipeline Setup in Azure DevOps
```bash
We will set up a CI pipeline for each of the three microservices in the Voting Application:
Vote: Python-based service
Result: Node.js-based service
Worker: .NET-based service
Additionally, we will set up an Azure Container Registry (ACR) to store the Docker images.
```

## Steps to Set Up CI Pipelines
Clone the Voting App Repository:
```bash
git clone https://github.com/dockersamples/example-voting-app
```

## Set Up Azure DevOps Project:
```bash
Create a new project in Azure DevOps and import the above repository.
Set the main branch as the default branch.
Create Azure Container Registry (ACR):
Go to Azure Portal > Container Registry > Create a new registry with the name saikrishna.
Create Azure Pipelines for Each Microservice: We will set up a separate pipeline for each service (Vote, Result, Worker).
```

Example Azure Pipeline for Worker Service
```bash
trigger:
  paths:
    include:
      - worker/*

resources:
  - repo: self

variables:
  dockerRegistryServiceConnection: 'ff16b353-46e3-406b-a6f1-832d047f6c95'
  imageRepository: 'workerapp'
  containerRegistry: 'saikrishna.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/worker/Dockerfile'
  tag: '$(Build.BuildId)'

pool:
  name: "azureagent"

stages:
  - stage: Build
    displayName: Build 
    jobs:
      - job: Build
        displayName: Build
        steps:
          - task: Docker@2
            displayName: Build
            inputs:
              containerRegistry: '$(dockerRegistryServiceConnection)'
              repository: '$(imageRepository)'
              command: 'build'
              Dockerfile: 'worker/Dockerfile'
              tags: '$(tag)'

  - stage: push
    displayName: Push
    jobs:
      - job: push
        displayName: Push
        steps:
          - task: Docker@2
            displayName: Push
            inputs:
              containerRegistry: '$(dockerRegistryServiceConnection)'
              repository: '$(imageRepository)'
              command: 'push'
              tags: '$(tag)'
```
## Kubernetes Cluster Setup
Create AKS Cluster:

```bash
az aks create --resource-group <Resource-Group-Name> --name <AKS-Cluster-Name> --node-count 1 --enable-addons monitoring --generate-ssh-keys
Login to AKS Cluster:

```bash
az aks get-credentials --resource-group <Resource-Group-Name> --name <AKS-Cluster-Name>
```
Install ArgoCD in the Kubernetes cluster:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Access ArgoCD UI: Change the ArgoCD service type to NodePort and access the UI via the Node IP and Port.

## GitOps Workflow with ArgoCD
```bash
Once the CI pipeline builds and pushes the Docker images to ACR, GitOps will monitor the Azure Git repository and sync the changes with the Kubernetes cluster.

Push Changes to Git Repository:
Update the Docker image version in the Kubernetes manifest files under k8s-specifications directory.
ArgoCD Sync:
ArgoCD detects the change in the Git repository, retrieves the updated YAML file, and deploys the changes to the Kubernetes cluster.
Script for Auto-Update of Images in Kubernetes Manifests
Create a shell script to update Kubernetes image versions based on the ACR repository.
```
```bash
#!/bin/bash

set -x

# Set the repository URL
REPO_URL="https://<ACCESS-TOKEN>@dev.azure.com/<AZURE-DEVOPS-ORG-NAME>/voting-app/_git/voting-app"

# Clone the git repository into the /tmp directory
git clone "$REPO_URL" /tmp/temp_repo

# Navigate into the cloned repository directory
cd /tmp/temp_repo

# Make changes to the Kubernetes manifest file(s)
# For example, let's say you want to change the image tag in a deployment.yaml file
sed -i "s|image:.*|image: <ACR-REGISTRY-NAME>/$2:$3|g" k8s-specifications/$1-deployment.yaml

# Add the modified files
git add .

# Commit the changes
git commit -m "Update Kubernetes manifest"

# Push the changes back to the repository
git push

# Cleanup: remove the temporary directory
rm -rf /tmp/temp_repo
```

## The above script will:

Clone the Git repository.
Modify the Kubernetes manifests to update the image version.
Commit and push the changes back to the repository, triggering ArgoCD to deploy the new image.
Continuous Reconciliation and Drift Fixing
GitOps ensures that the state in Kubernetes always matches the state defined in Git, and ArgoCD will continuously reconcile any drift. Any unauthorized changes to the cluster will be reverted automatically, keeping the system secure and predictable.
