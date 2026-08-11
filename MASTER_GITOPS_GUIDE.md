# Master GitOps & Kubernetes Study Guide

Welcome to your ultimate cheat sheet! This document summarizes everything we built during the Argo Deep Dive study, explains the core concepts, and provides the commands you need to manage your cluster like a Senior DevOps Engineer.

---

## 1. The Overall Architecture

We built a true **Enterprise CI/CD Pipeline** using the "Push-to-GitOps" pattern:

1. **You** push Python code to `deepdive-app`.
2. **GitHub** sends a Webhook via `smee.io` to your local Jenkins server.
3. **Jenkins (CI)** tests the code, packages it into a Docker image, and pushes it to DockerHub.
4. **Jenkins** then acts like a robot developer: it automatically edits the YAML files in `deepdive-gitops` to use the new Docker image, and pushes the commit.
5. **Argo CD (CD)** is running inside your Kubernetes cluster. It sees the new commit in `deepdive-gitops`, pulls the new YAML, and instructs Kubernetes to deploy it!

---

## 2. Core Concepts & Terminology

### Argo CD vs Kubernetes
* **Kubernetes** is the factory floor manager. It knows how to run Docker containers, restart them if they crash, and route network traffic to them.
* **Argo CD** is the factory architect. It reads the blueprints (YAML files) from GitHub and hands them to the factory manager (Kubernetes) to build.

### The Apartment Analogy
* **Node (The Hardware):** A Node is a physical server (or virtual machine). In our setup, Minikube is simulating 3 physical servers (`agro-cluster`, `m02`, `m03`).
* **Namespace (The Software Walls):** A Namespace is an invisible, locked room inside the cluster. Even if a Staging Pod and a Production Pod are running on the exact same physical Node, they cannot see or talk to each other because they are in different Namespaces.
* **Pod (The Tenant):** A Pod is a tiny wrapper around your Docker container. It is the smallest thing Kubernetes can run.
* **Service (The Doorway):** Pods are constantly dying and getting new IP addresses. A Service is a permanent static doorway that routes traffic to whichever Pods happen to be alive at the moment.

---

## 3. Cold Start Guide

If you reboot your laptop, run these commands to wake the environment back up:

```bash
# 1. Start the Virtual Servers
minikube start -p agro-cluster

# 2. Start the Webhook Bridge (Leave this terminal open!)
smee --url https://smee.io/gcwj7BrACLnJsMX --target http://127.0.0.1:8080/github-webhook/

# 3. Open Argo CD UI (in a new terminal)
minikube kubectl -p agro-cluster -- port-forward svc/argocd-server -n argocd 8081:443
# (Then visit https://localhost:8081 in Chrome)

# 4. View your Live Apps
minikube service fastapi-app -p agro-cluster -n staging
minikube service fastapi-app -p agro-cluster -n production
```

---

## 4. Kubernetes Cheat Sheet & Troubleshooting

Prefix all of these commands with: `minikube kubectl -p agro-cluster -- `

### 🔎 Reconnaissance (Looking Around)
* `get nodes` : Shows your physical servers. Are they `Ready` or `NotReady`?
* `get pods -A` : Shows every single Pod running across the entire cluster.
* `get pods -n staging -o wide` : Shows the Pods in the `staging` namespace, and crucially, **which physical Node they are running on** and their internal IP addresses.
* `get svc -n staging` : Shows the Services (Doorways) in the staging namespace.

### 🚑 Troubleshooting (Why is my app crashing?)
If you see a Pod with a Status of `Error`, `CrashLoopBackOff`, or `0/1 Ready`, use these two commands to find out why:

**1. The Logs (What did the app say before it died?)**
```bash
# Look at the application's console output
kubectl logs <pod-name> -n <namespace>

# Look at the logs of the PREVIOUS version of the pod that crashed!
kubectl logs <pod-name> -n <namespace> --previous
```

**2. The Describe Command (What did Kubernetes see?)**
```bash
# Gives you a massive diagnostic report on the Pod, including the exact Events (e.g., "Connection Refused", "OOMKilled", "Liveness Probe Failed").
kubectl describe pod <pod-name> -n <namespace>
```

### 💥 Chaos & Editing
* `delete pod <pod-name> -n <namespace>` : Instantly kills a Pod. Kubernetes will immediately spin up a brand new one to replace it!
* `delete pods --all -n <namespace>` : The "turn it off and on again" command. Kills every pod in the namespace to force a clean restart.
* `edit deployment <deployment-name> -n <namespace>` : Opens a terminal text editor (vim) to edit the live configuration. (Warning: Argo CD will usually instantly overwrite your manual edits because it enforces what is in GitHub!)
* `port-forward svc/<service-name> -n <namespace> 8080:80` : Creates a temporary tunnel from your laptop's localhost directly into the Kubernetes Service.

---

## 5. The Golden Standard: Secret Management (Sealed Secrets)

This was the core reason you started this project: **How do enterprise companies safely store environment variables and database passwords?**

### The Old Way (.env files)
In traditional development, you create a `.env` file on your laptop and server. This is extremely dangerous at an enterprise scale because:
1. If a server dies, the password is lost.
2. You can't track who changed the password or when.
3. You can't automate deployments if a human has to SSH into the server to paste a `.env` file.

### The GitOps Problem
GitOps says: *"If it isn't in Git, it doesn't exist!"* 
But Kubernetes `Secret` files are only base64-encoded, meaning if you put them in GitHub, hackers will scrape your repository and steal your passwords in 3 seconds.

### The Solution: Sealed Secrets
To solve this, we used **Sealed Secrets** (The industry standard):
1. We installed a "Controller" (the unlocking robot) into the `kube-system` namespace. Upon booting up, it generated a massive 4096-bit private RSA encryption key that it never shares with anyone.
2. We used the `kubeseal` tool on your laptop to encrypt your plain-text passwords into a `SealedSecret` file. 
3. Because it is safely encrypted, we pushed it directly to GitHub.
4. **Argo CD** pulled the `SealedSecret` from GitHub and threw it into the cluster.
5. The **Controller** caught it, used its private key to decrypt it, and created a standard, hidden Kubernetes `Secret`.

### How your FastAPI App actually gets the password:
When your FastAPI app boots up, Kubernetes looks at the `api-deployment.yaml`. Inside that file, we told Kubernetes: *"Take the database URL from the decrypted secret, and inject it straight into the FastAPI container's memory as an Environment Variable!"*

The application never knows it came from a Git repository or a Sealed Secret. It just boots up, checks `os.getenv("DATABASE_URL")`, and the password is magically there!
