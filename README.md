# Fullstack DevOps - ArgoCD Image Updater Configuration

This repository contains the ArgoCD Image Updater configuration for automated container image updates from Amazon ECR. It watches ECR repositories for new image tags and automatically updates the deployment manifests.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Repository Structure](#repository-structure)
- [Components](#components)
  - [ArgoCD Application](#argocd-application)
  - [Image Updater Configuration](#image-updater-configuration)
  - [ECR Authentication](#ecr-authentication)
  - [ECR Secret Refresh CronJob](#ecr-secret-refresh-cronjob)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
- [How It Works](#how-it-works)
- [Troubleshooting](#troubleshooting)

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS ECR                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │ backend         │  │ frontend-v1     │  │ frontend-v2     │              │
│  │ master-123      │  │ master-456      │  │ master-789      │              │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘              │
│           │                    │                    │                        │
└───────────┼────────────────────┼────────────────────┼────────────────────────┘
            │                    │                    │
            ▼                    ▼                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     ArgoCD Image Updater                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  • Polls ECR every 2 minutes for new images                         │    │
│  │  • Filters tags matching: ^master-[0-9]+$                           │    │
│  │  • Uses newest-build strategy (latest by build date)                │    │
│  │  • Authenticates via ecr-login.sh script                            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                         │
│                                    ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Updates image tags in ArgoCD Application parameters                │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ArgoCD                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  fullstack-devops Application                                       │    │
│  │  • Source: github.com/reggiegunawan88/fullstack-devops-deployment   │    │
│  │  • Destination: fullstack-devops-k0s namespace                      │    │
│  │  • Sync: Automated with prune and self-heal                         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Kubernetes (fullstack-devops-k0s)                         │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐                    │
│  │ backend       │  │ frontend      │  │ frontend-v2   │                    │
│  │ Deployment    │  │ Deployment    │  │ Deployment    │                    │
│  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘                    │
│          │                  │                  │                             │
│          └──────────────────┼──────────────────┘                             │
│                             ▼                                                │
│                    ┌─────────────────┐                                       │
│                    │ ecr-secret      │◄──── ECR Secret Refresh CronJob       │
│                    │ (imagePullSecret)│     (runs every 6 hours)             │
│                    └─────────────────┘                                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Repository Structure

```
.
├── README.md
├── kustomization.yaml                 # Main kustomization file
├── fullstack-devops-app.yaml          # ArgoCD Application definition
├── image-updater-cr.yaml              # Image Updater custom resource
├── configs/
│   └── argocd-image-updater-config.yaml  # Registry configuration
├── patches/
│   └── deployment-patch.yaml          # Patches for image updater deployment
├── scripts/
│   └── ecr-login-config.yaml          # ECR login script ConfigMap
├── secrets/
│   ├── aws-ecr-secret.yaml            # SealedSecret for AWS credentials
│   └── plain-secret.yaml              # Plain secret template (do not commit)
└── ecr-secret-refresh/
    ├── kustomization.yaml             # Kustomization for CronJob
    ├── rbac.yaml                      # ServiceAccount and RBAC
    └── cronjob.yaml                   # CronJob for ECR secret refresh
```

## Components

### ArgoCD Application

**File:** `fullstack-devops-app.yaml`

Defines the ArgoCD Application that deploys workloads to the cluster:

| Property | Value |
|----------|-------|
| Name | `fullstack-devops` |
| Source Repository | `https://github.com/reggiegunawan88/fullstack-devops-deployment.git` |
| Target Revision | `master` |
| Destination Namespace | `fullstack-devops-k0s` |
| Sync Policy | Automated (prune + self-heal) |

### Image Updater Configuration

**File:** `image-updater-cr.yaml`

Configures which images to watch and how to update them:

| Image Alias | ECR Repository | Update Strategy |
|-------------|----------------|-----------------|
| `backend` | `355446107250.dkr.ecr.us-east-1.amazonaws.com/fullstack-devops/backend` | newest-build |
| `frontend-v1` | `355446107250.dkr.ecr.us-east-1.amazonaws.com/fullstack-devops/frontend-v1` | newest-build |
| `frontend-v2` | `355446107250.dkr.ecr.us-east-1.amazonaws.com/fullstack-devops/frontend-v2` | newest-build |

**Tag Filter:** Only tags matching `^master-[0-9]+$` are considered (e.g., `master-1`, `master-42`, `master-100`).

**Write-back Method:** `argocd` - Updates are written directly to ArgoCD Application parameters.

### ECR Authentication

ECR authentication is handled through two mechanisms:

#### 1. Image Updater ECR Login (for polling ECR)

**Files:**
- `configs/argocd-image-updater-config.yaml` - Registry configuration
- `scripts/ecr-login-config.yaml` - Login script
- `patches/deployment-patch.yaml` - Mounts credentials and script

The Image Updater uses an external script (`ecr-login.sh`) to obtain ECR credentials dynamically:

```bash
#!/bin/sh
echo "AWS:$(aws ecr get-login-password --region us-east-1)"
```

This script is mounted into the Image Updater pod and called whenever ECR authentication is needed.

#### 2. Kubernetes ImagePullSecret (for pulling images)

**Files:** `ecr-secret-refresh/`

Pods need `imagePullSecrets` to pull images from ECR. Since ECR tokens expire after 12 hours, a CronJob refreshes the secret automatically.

### ECR Secret Refresh CronJob

**Location:** `ecr-secret-refresh/`

This component automatically refreshes the ECR `imagePullSecret` used by application pods.

#### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    CronJob: ecr-secret-refresh                   │
│                    Namespace: argocd-image-updater-system        │
│                    Schedule: Every 6 hours (0 */6 * * *)         │
├─────────────────────────────────────────────────────────────────┤
│  1. Read AWS credentials from aws-ecr-credentials secret         │
│  2. Call: aws ecr get-login-password                             │
│  3. Delete existing ecr-secret in fullstack-devops-k0s           │
│  4. Create new docker-registry secret with fresh token           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Secret: ecr-secret                            │
│                    Namespace: fullstack-devops-k0s               │
│                    Type: kubernetes.io/dockerconfigjson          │
│                    TTL: ~12 hours (refreshed every 6 hours)      │
└─────────────────────────────────────────────────────────────────┘
```

#### RBAC Configuration

The CronJob runs in `argocd-image-updater-system` (where AWS credentials exist) but creates secrets in `fullstack-devops-k0s`. This requires cross-namespace RBAC:

| Resource | Namespace | Purpose |
|----------|-----------|---------|
| ServiceAccount `ecr-secret-refresh` | argocd-image-updater-system | Identity for the CronJob |
| Role `ecr-secret-refresh` | fullstack-devops-k0s | Allows secret management |
| RoleBinding `ecr-secret-refresh` | fullstack-devops-k0s | Binds SA to Role |

#### Manual Trigger

To manually refresh the ECR secret (e.g., after an `ErrImagePull` error):

```bash
# Create a one-off job from the CronJob
kubectl create job --from=cronjob/ecr-secret-refresh ecr-secret-refresh-manual \
  -n argocd-image-updater-system

# Watch the job
kubectl logs -f job/ecr-secret-refresh-manual -n argocd-image-updater-system

# Clean up after completion
kubectl delete job ecr-secret-refresh-manual -n argocd-image-updater-system
```

## Prerequisites

1. **ArgoCD** installed in the cluster
2. **ArgoCD Image Updater** installed in `argocd-image-updater-system` namespace
3. **Sealed Secrets Controller** installed (for managing AWS credentials securely)
4. **AWS IAM User/Role** with the following permissions:
   - `ecr:GetAuthorizationToken`
   - `ecr:BatchGetImage`
   - `ecr:GetDownloadUrlForLayer`
   - `ecr:DescribeImages`
   - `ecr:ListImages`

## Deployment

### 1. Deploy Image Updater Configuration

```bash
# Apply the main configuration
kubectl apply -k .
```

### 2. Deploy ECR Secret Refresh CronJob

```bash
# Apply the CronJob
kubectl apply -k ecr-secret-refresh/

# Trigger initial secret creation
kubectl create job --from=cronjob/ecr-secret-refresh ecr-secret-refresh-init \
  -n argocd-image-updater-system
```

### 3. Verify Deployment

```bash
# Check ArgoCD Application
kubectl get application fullstack-devops -n argocd

# Check Image Updater logs
kubectl logs -l app.kubernetes.io/name=argocd-image-updater \
  -n argocd-image-updater-system --tail=50

# Check ECR secret exists
kubectl get secret ecr-secret -n fullstack-devops-k0s

# Check CronJob status
kubectl get cronjob ecr-secret-refresh -n argocd-image-updater-system
```

## How It Works

### Image Update Flow

1. **CI/CD Pipeline** builds and pushes new images to ECR with tags like `master-123`
2. **ArgoCD Image Updater** polls ECR every 2 minutes
3. When a new matching tag is found (newer by build date):
   - Image Updater updates the ArgoCD Application parameters
   - ArgoCD detects the change and syncs
4. **ArgoCD** applies the updated manifests to the cluster
5. **Kubernetes** pulls the new image using `ecr-secret` and updates the pods

### ECR Token Lifecycle

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Timeline (24 hours)                                                         │
├──────────��─────────────────────────────────────────────────────────────────┤
│                                                                             │
│ 00:00  CronJob runs → New token (valid 12h)                                │
│   │                                                                         │
│ 06:00  CronJob runs → New token (valid 12h)  ← 6h safety margin            │
│   │                                                                         │
│ 12:00  CronJob runs → New token (valid 12h)                                │
│   │         │                                                               │
│   │         └── Old token from 00:00 would expire here                     │
│   │                                                                         │
│ 18:00  CronJob runs → New token (valid 12h)                                │
│   │                                                                         │
│ 24:00  Cycle repeats                                                        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

## Troubleshooting

### ErrImagePull / ImagePullBackOff

**Cause:** ECR token has expired.

**Solution:**
```bash
# Manually trigger secret refresh
kubectl create job --from=cronjob/ecr-secret-refresh ecr-fix-$(date +%s) \
  -n argocd-image-updater-system

# Delete failing pods to trigger re-pull
kubectl delete pods -n fullstack-devops-k0s --field-selector=status.phase!=Running
```

### Image Updater Not Detecting New Images

**Check logs:**
```bash
kubectl logs -l app.kubernetes.io/name=argocd-image-updater \
  -n argocd-image-updater-system --tail=100 | grep -i error
```

**Common causes:**
- AWS credentials expired or invalid
- Tag doesn't match filter pattern `^master-[0-9]+$`
- ECR repository doesn't exist

### Verify AWS Credentials

```bash
# Check if secret exists
kubectl get secret aws-ecr-credentials -n argocd-image-updater-system

# Test ECR login (from a pod with AWS CLI)
kubectl run test-ecr --rm -it --image=amazon/aws-cli:latest \
  --env="AWS_ACCESS_KEY_ID=$(kubectl get secret aws-ecr-credentials -n argocd-image-updater-system -o jsonpath='{.data.aws_access_key_id}' | base64 -d)" \
  --env="AWS_SECRET_ACCESS_KEY=$(kubectl get secret aws-ecr-credentials -n argocd-image-updater-system -o jsonpath='{.data.aws_secret_access_key}' | base64 -d)" \
  --env="AWS_REGION=us-east-1" \
  -- ecr get-login-password --region us-east-1
```

### Check CronJob History

```bash
# List recent jobs
kubectl get jobs -n argocd-image-updater-system -l job-name=ecr-secret-refresh

# Check CronJob status
kubectl describe cronjob ecr-secret-refresh -n argocd-image-updater-system
```

## Related Repositories

- **Deployment Repository:** [fullstack-devops-deployment](https://github.com/reggiegunawan88/fullstack-devops-deployment) - Contains the Kubernetes manifests that ArgoCD syncs
- **Application Repository:** Contains the source code and CI/CD pipelines that push images to ECR
