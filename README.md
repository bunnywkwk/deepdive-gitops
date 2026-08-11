# Argo Deep Dive: GitOps Repository (CD)

This repository contains the **Infrastructure as Code (YAML)** and drives the **Continuous Deployment (CD)** pipeline.

## What happens here?
Developers do **not** write application code here. This repository acts as the absolute "Source of Truth" for what should be running in the live Kubernetes cluster. If it is not in this repository, it does not exist in the cluster!

## The Pipeline (Argo CD)
We have a robot named **Argo CD** living inside our Kubernetes cluster. 
1. Argo CD constantly watches this repository for changes.
2. When Jenkins updates the `api-deployment.yaml` with a new image tag, Argo CD detects the change.
3. Argo CD downloads the new YAML and tells Kubernetes to safely upgrade the live servers to match the Git repository.

## Environments & Secrets
- **Staging & Production**: We use separate folders (`environments/staging` and `environments/production`) to deploy the exact same application into isolated Kubernetes Namespaces.
- **Sealed Secrets**: We never put plain-text passwords in these YAML files. We use `kubeseal` to encrypt them. Only the robot inside our specific cluster possesses the private key to unlock them!
