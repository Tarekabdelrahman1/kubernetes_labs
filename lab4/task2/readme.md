Markdown
# 📉 Kubernetes Lab - Downward API & Dynamic Content

> A specialized lab demonstrating how to utilize the Kubernetes Downward API to inject real-time Pod metadata (Name and IP) into a container and serve it via a persistent web volume.

---

## 🎯 Lab Objectives
- Provision a Persistent Volume and Claim to store web content.
- Utilize the **Downward API** via `fieldRef` to expose Pod metadata as environment variables.
- Implement a startup script within the Deployment to dynamically generate an `index.html` file using that metadata.
- Verify that the web server serves the correct, pod-specific information.

---

## 🛠️ Step 1: Persistent Storage Setup

Before deploying the application, we must prepare the storage where the dynamic `index.html` will be written.

### 1. Create the Host Path
```bash
sudo mkdir -p /mnt/downward-data
```
### 2. Create the Storage Resources (downward-storage.yaml)
This manifest creates both the PV and the PVC required for the deployment.

```YAML
apiVersion: v1
kind: PersistentVolume
metadata:
  name: downward-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/downward-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: downward-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
Apply the storage configuration:

```Bash
kubectl apply -f downward-storage.yaml
```
<img width="947" height="241" alt="image" src="https://github.com/user-attachments/assets/fafeb398-306f-4a17-9666-47967ce02543" />

## 📦 Step 2: The Downward API Deployment
1. The Deployment Manifest (downward.yaml)
This deployment uses env variables to capture metadata from the Kubernetes API. The args field executes a shell command that writes these variables into the Nginx web root before starting the service.

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: downward-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        command: ["/bin/sh", "-c"]
        args: ['echo "Pod Name: $POD_NAME | Pod IP: $POD_IP" > /usr/share/nginx/html/index.html && nginx -g "daemon off;"']
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: downward-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: downward-storage
        persistentVolumeClaim:
          claimName: downward-pvc
```
### 2. Apply the Deployment
```Bash
kubectl apply -f downward.yaml
```
<img width="640" height="165" alt="image" src="https://github.com/user-attachments/assets/d6e359fc-d4b7-450b-bf6d-6f224d50408d" />

## ✅ Step 3: Verification
### 1. Retrieve Pod Networking Info
Find the IP address assigned to the running pod:

```Bash
kubectl get pods -o wide | grep downward
```
<img width="1588" height="348" alt="image" src="https://github.com/user-attachments/assets/ba7651c8-a579-493c-a47d-b154e6590a86" />

