# 🚀 Kubernetes Lab - Cross-Namespace DNS & Services

> A comprehensive lab demonstrating Kubernetes namespace isolation, deployment creation, NodePort service exposure, and cross-namespace DNS resolution.

---

## 🎯 Lab Objectives
- Create a specific namespace (`iti`) to logically isolate our web application.
- Deploy a highly available Nginx application with 2 replicas.
- Expose the application using a NodePort service to act as a load balancer.
- Verify Service discovery and DNS resolution from a completely different namespace (`default`).

---

## 🛠️ Task 1: Namespace & Deployment Configuration

### 1. Namespace Creation
Create a dedicated namespace called `iti` for our web application.
```bash
kubectl create namespace iti
```
### 2. The Web Deployment Manifest
Create a file named web.yaml with the following YAML specification to deploy 2 replicas of an Nginx web server inside the iti namespace:
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: iti
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
```
### 3. Apply the Configuration
Apply the file to the cluster to create the deployment.

```Bash
kubectl apply -f web.yaml
```


## 🧩 Task 2: Service Exposure (NodePort)
To load balance the traffic to our pods, the deployment will be exposed through a Service named web-svc. We will configure it as a NodePort type, listening on port 5000 and forwarding to the container port 80.

### 1. Expose the Deployment
```Bash
kubectl expose deployment web --name=web-svc --namespace=iti --type=NodePort --port=5000 --target-port=80
```
### 2. Verify the Service
Check that the service was created successfully and note its Cluster-IP and ports.
```Bash
kubectl get svc -n iti
```
<img width="1462" height="477" alt="1" src="https://github.com/user-attachments/assets/c090b2a1-2d04-40cd-88d7-037bea7bea14" />

## 📦 Task 3: Cross-Namespace DNS Verification
To demonstrate DNS resolution across namespaces, a temporary test pod will be created in the default namespace. We will access the shell of this pod to test the connection.

Service FQDN: web-svc.iti.svc.cluster.local:5000

### 1. Run the Test Pod
Launch a temporary, interactive pod using a curl image in the default namespace:

```Bash
kubectl run test-pod --image=curlimages/curl -n default -it --rm --restart=Never -- sh
```
2. Verify DNS and Connectivity
Once inside the interactive shell, check the /etc/resolv.conf to understand the DNS search paths, and then execute a curl request against the service domain:

Bash
### Check the DNS configuration
#### Query the SRV record
```
nslookup -type=SRV _web-svc._tcp.web-svc.iti.svc.cluster.local
```
<img width="1567" height="403" alt="1 9" src="https://github.com/user-attachments/assets/bbeea568-993d-4b00-893f-fc57d3aa9a01" />

#### Send a request to the web service across namespaces
```
curl http://web-svc.iti.svc.cluster.local:5000
```
<img width="1545" height="643" alt="2" src="https://github.com/user-attachments/assets/125b1ecb-896c-4925-bd2f-e719e7c834c2" />
<img width="1263" height="667" alt="3" src="https://github.com/user-attachments/assets/73964af8-b3c9-4d5d-8ee2-71aeef92240a" />

