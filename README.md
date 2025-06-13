# Multi-Environment Application Deployment with Kustomize

## Table of Contents

1. [Introduction](#introduction)
   - [What is Kustomize?](#what-is-kustomize)
   - [Key Concepts](#key-concepts)
2. [Project Structure](#project-structure)
3. [Prerequisites](#prerequisites)
4. [Setting Up the Environment](#setting-up-the-environment)
   - [Installing Required Tools](#installing-required-tools)
   - [Provisioning Kubernetes Cluster with eksctl](#provisioning-kubernetes-cluster-with-eksctl)
   - [Cloning the Repository](#cloning-the-repository)
5. [Understanding the Configuration](#understanding-the-configuration)
   - [Base Configuration](#base-configuration)
   - [Environment Overlays](#environment-overlays)
     - [Development Environment](#development-environment)
     - [Staging Environment](#staging-environment)
     - [Production Environment](#production-environment)
   - [Service Account and RBAC](#service-account-and-rbac)
6. [Deployment Guide](#deployment-guide)
   - [Deploying to Development](#deploying-to-development)
   - [Deploying to Staging](#deploying-to-staging)
   - [Deploying to Production](#deploying-to-production)
   - [Verifying Deployments](#verifying-deployments)
7. [CI/CD Pipeline](#cicd-pipeline)
   - [Setting Up GitHub Actions](#setting-up-github-actions)
   - [Pipeline Configuration](#pipeline-configuration)
   - [Testing the Pipeline](#testing-the-pipeline)
8. [Troubleshooting](#troubleshooting)
   - [Common Deployment Issues](#common-deployment-issues)
   - [GitHub Actions Issues](#github-actions-issues)
   - [Kustomize Build Issues](#kustomize-build-issues)
9. [Cleanup](#cleanup)
   - [Removing Deployments](#removing-deployments)
   - [Deleting the Cluster](#deleting-the-cluster)
10. [References and Resources](#references-and-resources)

---

## Introduction

### What is Kustomize?

Kustomize is a Kubernetes-native configuration management tool that allows you to customize application manifests without modifying the original YAML files. It provides a way to manage configurations for different environments (e.g., dev, staging, prod) using overlays and bases.

The application deployed in this project is a simple Node.js web application that displays the day of the week. This app serves as a demonstration of multi-environment deployment using Kubernetes.

![Application Screenshot](screenshots/application-screenshot.png)

### Key Concepts

- **Base**: Contains common resources shared across environments.

- **Overlay**: Customizes the base for specific environments.

- **Transformers**: Modify resources dynamically, such as adding labels or annotations.

  Example:

  ```yaml
  apiVersion: builtin
  kind: LabelTransformer
  metadata:
    name: add-common-labels
  labels:
    app.kubernetes.io/managed-by: kustomize
    project: kustomize-capstone
  fieldSpecs:
    - path: metadata/labels
      create: true
  ```

- **Generators**: Create ConfigMaps and Secrets dynamically from literals or files.

  Example:

  ```yaml
  configMapGenerator:
    - name: webapp-config
      literals:
        - APP_MODE=base
        - FEATURE_FLAG=false
  secretGenerator:
    - name: webapp-secret
      literals:
        - DB_PASSWORD=pleasechangeme
        - API_KEY=pleasechangeme
  ```

![Transformers and Generators Screenshot](screenshots/transformers-generators.png)

![Kustomize Workflow Diagram](screenshots/kustomize-workflow.png)

## Project Structure

The project follows a standard Kustomize structure with base configurations and environment-specific overlays:

```plaintext
<project-root>/
├── base/                # Base Kustomize resources (Deployment, Service, ConfigMap, Secret, transformers)
├── overlays/
│   ├── dev/             # Dev environment overlay
│   ├── staging/         # Staging environment overlay
│   └── prod/            # Production environment overlay
└── .github/workflows/   # CI/CD pipeline definition
```

![Project Structure Screenshot](screenshots/project-structure.png)

![Directory Tree View](screenshots/directory-tree.png)

## Prerequisites

Before starting, ensure you have the following:

- AWS CLI installed and configured
- kubectl installed
- eksctl installed
- Git installed
- Access to a GitHub account (for CI/CD setup)

![Prerequisites Tools](screenshots/prerequisites-tools.png)

## Setting Up the Environment

### Installing Required Tools

1. **Install kubectl**:
   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x kubectl
   sudo mv kubectl /usr/local/bin/
   ```

2. **Install Kustomize**:
   ```bash
   curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
   sudo mv kustomize /usr/local/bin/
   ```

![Tools Installation](screenshots/tools-installation.png)

### Provisioning Kubernetes Cluster with eksctl

1. **Install eksctl**:

   ```bash
   curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_$(uname -m).tar.gz" | tar xz -C /tmp
   sudo mv /tmp/eksctl /usr/local/bin
   ```

2. **Create a Kubernetes cluster**:

   ```bash
   eksctl create cluster --name kustomize-cluster --region us-west-2 --nodegroup-name standard-workers --node-type t3.medium --nodes 3
   ```

3. **Verify the cluster is running**:

   ```bash
   kubectl get nodes
   ```

   You should see your nodes listed with status "Ready".

![Cluster Provisioning Screenshot](screenshots/cluster-provisioning.png)

![EKS Console View](screenshots/eks-console.png)

### Cloning the Repository

1. **Clone the repository**:

   ```bash
   git clone https://github.com/your-repo/kustomize-capstone.git
   cd kustomize-capstone
   ```

![Repository Clone](screenshots/repo-clone.png)

## Understanding the Configuration

### Base Configuration

The `base/` directory contains the common Kubernetes resources shared across all environments. These include:

1. **Deployment (`base/deployment.yaml`)**:
   - Defines the application pods and their specifications
   - Sets up container image, ports, and basic resource requests
   - References the service account for pod identity

2. **Service (`base/service.yaml`)**:
   - Exposes the application to the network
   - Defines service type and port mappings

3. **Service Account (`base/service-account.yaml`)**:
   - Creates a dedicated service account for the application
   - Provides identity for pods and CI/CD operations
   - Enables secure access to Kubernetes API

4. **Role and Role Binding (`base/role-binding.yaml`)**:
   - Defines permissions for the service account
   - Grants access to specific resources (pods, configmaps, secrets)
   - Establishes proper RBAC controls for the application

5. **Labels Transformer (`base/labels-transformer.yaml`)**:
   - Adds common labels to all resources
   - Ensures consistent labeling across environments

6. **ConfigMap and Secret Generators**:
   - Dynamically creates configuration and secrets
   - Defines default environment variables

![Base Deployment Screenshot](screenshots/base-deployment.png)
![Base Service Screenshot](screenshots/base-service.png)
![Base Labels Transformer Screenshot](screenshots/base-labels-transformer.png)
![Base Kustomization File](screenshots/base-kustomization.png)

### Environment Overlays

Each environment has its own overlay directory that customizes the base configuration.

#### Development Environment

The `overlays/dev/` directory customizes the base configuration for the development environment:

1. **Patch (`overlays/dev/deployment-patch.yaml`)**:
   - Sets replicas to 1 (minimal resources for development)
   - Configures environment-specific variables
   - Sets resource limits appropriate for development

2. **Labels Transformer (`overlays/dev/labels-transformer.yaml`)**:
   - Adds `environment: dev` label
   - Adds other development-specific labels

![Dev Patch Screenshot](screenshots/dev-patch.png)
![Dev Labels Transformer Screenshot](screenshots/dev-labels-transformer.png)
![Dev Kustomization File](screenshots/dev-kustomization.png)

#### Staging Environment

The `overlays/staging/` directory customizes the base configuration for the staging environment:

1. **Patch (`overlays/staging/deployment-patch.yaml`)**:
   - Sets replicas to 2 (moderate redundancy for testing)
   - Configures staging-specific environment variables
   - Sets appropriate resource limits for staging

2. **Labels Transformer (`overlays/staging/labels-transformer.yaml`)**:
   - Adds `environment: staging` label
   - Adds other staging-specific labels

![Staging Patch Screenshot](screenshots/staging-patch.png)
![Staging Labels Transformer Screenshot](screenshots/staging-labels-transformer.png)
![Staging Kustomization File](screenshots/staging-kustomization.png)

#### Production Environment

The `overlays/prod/` directory customizes the base configuration for the production environment:

1. **Patch (`overlays/prod/deployment-patch.yaml`)**:
   - Sets replicas to 3 (high availability for production)
   - Configures production-specific environment variables
   - Sets higher resource limits for production workloads

2. **Labels Transformer (`overlays/prod/labels-transformer.yaml`)**:
   - Adds `environment: prod` label
   - Adds other production-specific labels

![Prod Patch Screenshot](screenshots/prod-patch.png)
![Prod Labels Transformer Screenshot](screenshots/prod-labels-transformer.png)
![Prod Kustomization File](screenshots/prod-kustomization.png)

### Service Account and RBAC

A custom service account is created to provide a secure identity for the application pods and the CI/CD pipeline.

1. **Service Account (`base/service-account.yaml`)**:
   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: webapp-service-account
     namespace: default
   ```

2. **Role and Role Binding (`base/role-binding.yaml`)**:
   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: webapp-role
     namespace: default
   rules:
   - apiGroups: [""]
     resources: ["pods", "configmaps", "secrets"]
     verbs: ["get", "list", "watch", "create", "update", "delete"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: webapp-role-binding
     namespace: default
   subjects:
   - kind: ServiceAccount
     name: webapp-service-account
     namespace: default
   roleRef:
     kind: Role
     name: webapp-role
     apiGroup: rbac.authorization.k8s.io
   ```

The role grants permissions to `get`, `list`, `watch`, `create`, `update`, and `delete` operations on `pods`, `configmaps`, and `secrets` resources within the `default` namespace.

![Service Account and Role Binding Screenshot](screenshots/service-account-role-binding.png)
![RBAC Diagram](screenshots/rbac-diagram.png)

## Deployment Guide

### Deploying to Development

1. **Build and apply the development overlay**:
   ```bash
   kustomize build overlays/dev | kubectl apply -f -
   ```

2. **Verify the deployment**:
   ```bash
   kubectl get deployments,pods,svc -l environment=dev
   ```

![Dev Deployment Process](screenshots/dev-deployment.png)
![Dev Resources](screenshots/dev-resources.png)

### Deploying to Staging

1. **Build and apply the staging overlay**:
   ```bash
   kustomize build overlays/staging | kubectl apply -f -
   ```

2. **Verify the deployment**:
   ```bash
   kubectl get deployments,pods,svc -l environment=staging
   ```

![Staging Deployment Process](screenshots/staging-deployment.png)
![Staging Resources](screenshots/staging-resources.png)

### Deploying to Production

1. **Build and apply the production overlay**:
   ```bash
   kustomize build overlays/prod | kubectl apply -f -
   ```

2. **Verify the deployment**:
   ```bash
   kubectl get deployments,pods,svc -l environment=prod
   ```

![Prod Deployment Process](screenshots/prod-deployment.png)
![Prod Resources](screenshots/prod-resources.png)

### Verifying Deployments

1. **Get the LoadBalancer service endpoint**:
   ```bash
   kubectl get svc
   ```
   Look for the service with `TYPE` as `LoadBalancer` and note the `EXTERNAL-IP` or `EXTERNAL-DNS`.

2. **Access the application**:
   Open a web browser and navigate to the `EXTERNAL-IP` or `EXTERNAL-DNS` obtained in the previous step.

3. **Check pod logs**:
   ```bash
   kubectl logs pods/project-alpha-prod-7cfc67b888-ft6hg
   ```

![Deployment Process Screenshot](screenshots/deployment-process.png)
![Application Access](screenshots/application-access.png)
![Pod Logs](screenshots/pod-logs.png)

## CI/CD Pipeline

### Setting Up GitHub Actions

1. **Use the existing service account for CI/CD**:
   
   Instead of creating a new service account, you can use the existing `webapp-service-account` defined in your base configuration:
   
   ```bash
   # Apply the base configuration to ensure the service account exists
   kubectl apply -f base/service-account.yaml
   kubectl apply -f base/role-binding.yaml
   ```

2. **Get the service account token**:
   ```bash
   # For Linux/macOS
   kubectl get secret $(kubectl get serviceaccount webapp-service-account -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
   
   # For Windows PowerShell
   kubectl get secret $(kubectl get serviceaccount webapp-service-account -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
   ```

3. **Get the cluster certificate**:
   ```bash
   # For Linux/macOS/Windows
   kubectl get secret $(kubectl get serviceaccount webapp-service-account -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data['ca\.crt']}"
   ```

   Note: If your existing role doesn't have sufficient permissions for CI/CD operations, you may need to update the role in `base/role-binding.yaml` to include additional resources and verbs.

![GitHub Actions Setup](screenshots/github-actions-setup.png)
![Service Account Creation](screenshots/service-account-creation.png)

### Pipeline Configuration

1. **Create a kubeconfig file using the service account token and certificate**:

   ```bash
   # For Linux/macOS - Create a temporary kubeconfig file in /tmp (best practice)
   cat > /tmp/kubeconfig << EOF
   apiVersion: v1
   kind: Config
   clusters:
   - name: kubernetes
     cluster:
       certificate-authority-data: $(kubectl get secret $(kubectl get serviceaccount webapp-service-account -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data['ca\.crt']}")
       server: $(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
   contexts:
   - name: webapp-service-account@kubernetes
     context:
       cluster: kubernetes
       user: webapp-service-account
   current-context: webapp-service-account@kubernetes
   users:
   - name: webapp-service-account
     user:
       token: $(kubectl get secret $(kubectl get serviceaccount webapp-service-account -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode)
   EOF
   
   # For Windows PowerShell - Create a temporary kubeconfig file in %TEMP% (best practice)
   $kubeConfigContent = @"
   apiVersion: v1
   kind: Config
   clusters:
   - name: kubernetes
     cluster:
       certificate-authority-data: $(kubectl get secret $(kubectl get serviceaccount webapp-service-account -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data['ca\.crt']}")
       server: $(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
   contexts:
   - name: webapp-service-account@kubernetes
     context:
       cluster: kubernetes
       user: webapp-service-account
   current-context: webapp-service-account@kubernetes
   users:
   - name: webapp-service-account
     user:
       token: $(kubectl get secret $(kubectl get serviceaccount webapp-service-account -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) })
   "@
   
   $kubeConfigContent | Out-File "$env:TEMP\kubeconfig" -Encoding utf8
   ```

2. **Base64 encode the kubeconfig file**:

   ```bash
   # For Linux/macOS
   KUBE_CONFIG_DATA=$(cat /tmp/kubeconfig | base64 -w 0)
   
   # For Windows PowerShell
   $KUBE_CONFIG_DATA = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes((Get-Content -Raw "$env:TEMP\kubeconfig")))
   ```

3. **Add GitHub repository secrets**:
   - Go to your GitHub repository
   - Navigate to Settings > Secrets > New repository secret
   - Add a secret named `KUBE_CONFIG_DATA` with the base64-encoded value from the previous step

4. **Create the workflow file**:
   The workflow file is located at `.github/workflows/deploy.yml` and defines the CI/CD pipeline:

   ```yaml
   # .github/workflows/deploy.yml
   name: Deploy to Kubernetes
   
   on:
     push:
       branches: [main]
   
   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v2
         
         - name: Set up Kustomize
           uses: imranismail/setup-kustomize@v1
           
         - name: Set up kubectl
           uses: azure/setup-kubectl@v1
           
         - name: Configure kubectl
           run: |
             echo "${{ secrets.KUBE_CONFIG_DATA }}" | base64 -d > /tmp/kubeconfig
             export KUBECONFIG=/tmp/kubeconfig
             
         - name: Deploy to environment
           run: |
             kustomize build overlays/dev | kubectl apply -f -
   ```

![GitHub Secrets](screenshots/github-secrets.png)
![Workflow File](screenshots/workflow-file.png)

### Testing the Pipeline

1. **Make a change to the repository**:
   - Modify a file in the repository
   - Commit and push the changes

2. **Monitor the GitHub Actions workflow**:
   - Go to the Actions tab in your GitHub repository
   - Watch the workflow execution
   - Check for successful deployment

![CI/CD Workflow Screenshot](screenshots/github-actions-workflow.png)
![Workflow Execution](screenshots/workflow-execution.png)
![Deployment Success](screenshots/deployment-success.png)

## Troubleshooting

### Common Deployment Issues

1. **Pod Startup Failures**:
   - Check pod status: `kubectl get pods -l app=webapp`
   - View detailed pod information: `kubectl describe pod <pod-name>`
   - Check pod logs: `kubectl logs <pod-name>`

2. **Service Connectivity Issues**:
   - Verify service exists: `kubectl get svc -l app=webapp`
   - Check service endpoints: `kubectl get endpoints <service-name>`
   - Test connectivity from within the cluster: `kubectl run -it --rm --restart=Never test-pod --image=busybox -- wget -O- <service-name>:<port>`

3. **Resource Constraints**:
   - Check node resources: `kubectl describe nodes`
   - Verify pod resource requests and limits: `kubectl describe pod <pod-name>`

4. **ConfigMap and Secret Issues**:
   - Verify ConfigMaps exist: `kubectl get configmaps`
   - Check Secret existence: `kubectl get secrets`
   - Ensure they're mounted correctly in the pod: `kubectl describe pod <pod-name>`

![Troubleshooting Pods](screenshots/troubleshooting-pods.png)

### GitHub Actions Issues

1. **Authentication Failures**:
   - Verify the `KUBE_CONFIG_DATA` secret is correctly set
   - Check that the service account token is valid
   - Ensure the service account has appropriate permissions

2. **Workflow Failures**:
   - Check the GitHub Actions logs for specific error messages
   - Verify that all required tools are installed in the workflow
   - Test the commands locally with the same kubeconfig

3. **Kustomize Build Errors in CI/CD**:
   - Validate kustomization files locally: `kustomize build overlays/<env>`
   - Check for syntax errors in YAML files
   - Ensure all referenced files exist in the repository

![GitHub Actions Troubleshooting](screenshots/github-actions-troubleshooting.png)

### Kustomize Build Issues

1. **Resource Not Found Errors**:
   - Ensure all referenced resources exist
   - Check file paths in kustomization.yaml
   - Verify that patches correctly target resources

2. **Patch Application Failures**:
   - Ensure patch selectors match target resources
   - Check for YAML syntax errors
   - Verify that the patch structure matches the resource structure

3. **Transformer Issues**:
   - Validate transformer configuration
   - Check that fieldSpecs correctly target fields
   - Test transformers individually

![Kustomize Build Issues](screenshots/kustomize-build-issues.png)

## Cleanup

### Removing Deployments

1. **Delete individual environment deployments**:
   ```bash
   # Delete development environment
   kustomize build overlays/dev | kubectl delete -f -
   
   # Delete staging environment
   kustomize build overlays/staging | kubectl delete -f -
   
   # Delete production environment
   kustomize build overlays/prod | kubectl delete -f -
   ```

2. **Verify resources are removed**:
   ```bash
   kubectl get all -l app=webapp
   ```

### Deleting the Cluster

1. **Delete the EKS cluster**:
   ```bash
   eksctl delete cluster --name kustomize-cluster --region us-west-2
   ```

2. **Verify cluster deletion**:
   ```bash
   aws eks list-clusters --region us-west-2
   ```

3. **Clean up local files**:
   ```bash
   # For Linux/macOS
   rm -f /tmp/kubeconfig
   
   # For Windows PowerShell
   Remove-Item -Path "$env:TEMP\kubeconfig" -Force -ErrorAction SilentlyContinue
   ```

![Cluster Cleanup](screenshots/cluster-cleanup.png)

## References and Resources

- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [eksctl Documentation](https://eksctl.io/)
- [GitHub Actions for Kubernetes](https://github.com/marketplace/actions/kubectl-tool-installer)
- [Kubernetes RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)