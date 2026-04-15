## Kubernetes Lab - Persistent Volumes (hostPath)

> A practical lab demonstrating how to provision persistent storage using `hostPath`, create Persistent Volume Claims (PVC), and mount them inside a multi-replica Deployment.

---

## 🎯 Lab Objectives
- Provision a 1GB Persistent Volume (PV) named `nginx-pv` using the `hostPath` storage type.
- Configure the PV with a `Recycle` reclaim policy to allow multiple uses.
- Create a Persistent Volume Claim (PVC) to request storage from the PV.
- Prepare an `index.html` file on the host node containing a custom string.
- Deploy an Nginx application with 3 replicas pinned to the specific node to successfully mount the `hostPath` volume.

---

## 🛠️ Step 1: Host Node Preparation

Since we are using `hostPath`, the data must physically exist on the node where the pods will be scheduled.

### 1. Create the Storage Directory
Execute this on the specific node (e.g., `linux`) that will host the pods:
```bash
sudo mkdir -p /opt/k8s-lab/nginx-data
```

### 2. Create the Custom Index File
Write your name into the `index.html` file so it can be served by Nginx.
```bash
echo "<h1>Marwan Tarek</h1>" | sudo tee /opt/k8s-lab/nginx-data/index.html
```

---

## 📦 Step 2: Persistent Volume Configuration

### 1. The Persistent Volume (PV) Manifest
Create a file named `nginx-pv.yaml`. This defines the actual storage capacity and path on the node.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  hostPath:
    path: /opt/k8s-lab/nginx-data
```

### 2. Apply the PV
```bash
kubectl apply -f nginx-pv.yaml
```

---

## 🔖 Step 3: Persistent Volume Claim (PVC)

### 1. The Persistent Volume Claim Manifest
Create a file named `nginx-pvc.yaml`. This acts as a "request" for the storage we just provisioned.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

### 2. Apply the PVC
```bash
kubectl apply -f nginx-pvc.yaml
```

Check that the PVC is successfully bound to the PV:
```bash
kubectl get pv,pvc
```

---

## 🚀 Step 4: The Deployment

### 1. The Deployment Manifest
Create a file named `nginx-deploy.yaml`. To ensure all 3 replicas can access the `hostPath` data, the deployment uses `nodeName` to pin the pods to the specific node where we created the directory.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-pv-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-storage
  template:
    metadata:
      labels:
        app: web-storage
    spec:
      nodeName: linux
      containers:
        - name: nginx-server
          image: nginx:alpine
          volumeMounts:
            - name: html-storage
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html-storage
          persistentVolumeClaim:
            claimName: nginx-pvc
```

### 2. Apply the Deployment
```bash
kubectl apply -f nginx-deploy.yaml
```

---

