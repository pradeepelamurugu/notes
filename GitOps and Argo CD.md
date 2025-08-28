
# GitOps and Argo CD Notes

## What is GitOps?
- GitOps is the practice of treating Infrastructure as Code (IaC) like application code.
- The IaC code is maintained in a Git repository.
- Git rules enforce good practices such as:
  - No direct pushes to the main branch.
  - Changes must be made via Pull Requests (PRs).
  - PRs trigger CI/CD pipelines to validate configuration changes.

## CD Pipeline Approaches

### 1. Push Deployment
- CI/CD pipeline pushes container deployments directly to the environment.

### 2. Pull Deployment
- Tools like **Flux** or **Argo CD** pull deployment configuration from Git continuously.
- The cluster state is synchronized with the Git repository.

## Benefits of GitOps
- **Rollback**: Easy rollback using Git commands like `git revert`.
- **Security**: Git rules (PR reviews, approvals) ensure safe deployments.
- **Single Source of Truth**: Git repository defines the desired cluster state.

## Argo CD Overview
- Argo CD is a **Continuous Delivery (CD)** tool for Kubernetes.
- It constantly monitors a Git repository for changes and ensures the cluster matches the declared state.

### Without Argo CD
1. Developer commits code → triggers CI pipeline.
2. CI runs tests, builds the image, and pushes it to a container registry (e.g., Docker Hub).
3. CD pipeline updates Kubernetes manifests with the new image tag.
4. `kubectl apply` is used to deploy the application.

### With Argo CD
1. CI process remains the same: build, test, and push image.
2. Argo CD runs **inside the Kubernetes cluster**.
3. Argo CD continuously watches the Git repository for changes.
4. When the image tag is updated in the manifests:
   - Argo CD detects the change.
   - Pulls the latest Kubernetes config from Git.
   - Applies the changes to the cluster automatically.
5. If someone makes manual changes directly in the cluster:
   - Argo CD detects the drift.
   - Reverts to match the Git repository (can be configured).

## Summary
- GitOps enforces best practices by making Git the source of truth for IaC.
- Argo CD simplifies Kubernetes CD by continuously syncing cluster state with Git.
- Rollbacks, security, and consistency are inherent benefits of this approach.
