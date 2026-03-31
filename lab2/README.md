# 🚀 Kubernetes Administration Lab - ITI Intake 46

> A comprehensive lab demonstrating Kubernetes cluster provisioning, configuration management, custom `kubectl` plugins, and declarative application deployments using a lightweight `k3s` environment.

---

## 🎯 Lab Objectives
- Provision a 2-node cluster (1 Control Plane, 1 Worker) using `k3s`.
- Create a specific namespace (`iti-46`) and configure a new `kubectl` context to default to it.
- Extend Kubernetes functionality by creating a custom plugin (`kubectl hostnames`).
- Deploy a highly available Nginx application with custom environment variables.

---


## 🛠️ Task 1: Kubectl Config 

### 1. Provision the K3s Cluster
**On the Server Node (Control Plane):**
```bash
# Install k3s server
curl -sfL [https://get.k3s.io](https://get.k3s.io) | sh -

# Extract the node token
sudo cat /var/lib/rancher/k3s/server/node-token
```
** On Worker node**
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://10.0.0.16:6443 K3S_TOKEN=K10ead8ca8f7a2f616f05201d1197ad15c5b39f4b971f7067e4721176e30c9f53d8::server:b271efcc3a23d74b76b40f073203c566 sh -
```
### 2. Namespace & Context Configuration
```bash
kubectl create namespace iti-46
```
<img width="768" height="175" alt="image" src="https://github.com/user-attachments/assets/30cc5245-b9ee-454a-9470-1542d594b93b" />
#### Create a new context linking the default user/cluster to the new namespace
```bash
kubectl config set-context iti-context --cluster=default --user=default --namespace=iti-46
```
<img width="1358" height="187" alt="image" src="https://github.com/user-attachments/assets/8564ce47-4e26-416b-87ee-3ab75dc07aa6" />

#### Switch to the new context to make it active
```bash
kubectl config use-context iti-context
```
<img width="708" height="177" alt="image" src="https://github.com/user-attachments/assets/af532f13-d8fd-4601-b5e5-9ccd1f38f3e3" />

## 🧩 Task 2: Kubectl Plugin 
Kubernetes allows extending kubectl commands. We will create a custom plugin to list the hostnames of all nodes in the cluster.

<img width="927" height="495" alt="image" src="https://github.com/user-attachments/assets/5417f736-7e9a-4a49-a2d1-1449d1f1ce18" />

## 📦 Task 3: Creating Deployments (10 Points)
This task requires deploying 3 replicas of an Nginx application with a specific environment variable (FOO=ITI).

### 1. The Deployment Manifest
Create a file named deployment.yaml with the following YAML specification:

```YAML
apiVersion: v1
kind: Deployment
metadata:
  name: nginx-iti-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:alpine
        env:
        - name: FOO
          value: "ITI"
```
### 2. Apply the Configuration
Apply the file to the cluster. (Since we switched contexts earlier, this will automatically deploy into the iti-46 namespace).
```Bash
kubectl apply -f deployment.yaml
```
<img width="666" height="207" alt="image" src="https://github.com/user-attachments/assets/1a3d9567-b021-43a1-8bbd-4f372a42f905" />

```bash
kubectl get pods
```
<img width="1065" height="195" alt="image" src="https://github.com/user-attachments/assets/1ff7dd20-0342-4466-8916-42d84ebf49a8" />


