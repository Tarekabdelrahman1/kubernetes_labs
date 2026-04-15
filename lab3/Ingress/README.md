# 🌐 Kubernetes Lab - Ingress & Path-Based Routing

> A comprehensive lab demonstrating the creation of multiple deployments, internal ClusterIP services, and configuring a Traefik Ingress resource for path-based routing.

---

## 🎯 Lab Objectives
- Create dedicated namespaces (`iti-46` and `world`).
- Deploy two regional web applications (`africa` and `europe`).
- Expose the applications internally using ClusterIP services on port 8888.
- Configure local DNS resolution via `/etc/hosts`.
- Implement path-based routing using a Kubernetes Ingress resource.

---

## 🛠️ Step 1: Namespaces & Deployments

First, set up the namespaces and deploy the two web applications.

### 1. Create the Namespaces
```bash
kubectl create namespace iti-46
kubectl create namespace world
```
### 2. Create the Deployments (world-deployments.yaml)
Create the deployment file:
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: africa
  namespace: world
spec:
  replicas: 2
  selector:
    matchLabels:
      app: africa-app
  template:
    metadata:
      labels:
        app: africa-app
    spec:
      containers:
        - name: africa-web
          image: husseingalal/africa:latest
          ports:
            - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: europe
  namespace: world
spec:
  replicas: 2
  selector:
    matchLabels:
      app: europe-app
  template:
    metadata:
      labels:
        app: europe-app
    spec:
      containers:
        - name: europe-web
          image: husseingalal/europe:latest
          ports:
            - containerPort: 80
```
Apply the deployments:

```Bash
kubectl apply -f world-deployments.yaml
```
<img width="962" height="352" alt="image" src="https://github.com/user-attachments/assets/769cc381-f79d-4a23-a042-f507a03d07a7" />

## 🧩 Step 2: Internal Service Exposure
Expose the deployments internally using ClusterIP services. The services must match the deployment names.

```Bash
kubectl expose deployment africa --port=8888 --target-port=80 -n world
kubectl expose deployment europe --port=8888 --target-port=80 -n world
```
<img width="1215" height="241" alt="image" src="https://github.com/user-attachments/assets/c1c7d2cf-54a5-4bde-8a02-120a41e6fa43" />

## 🚦 Step 3: DNS & Ingress Setup
1. Configure Local DNS
Map the cluster's internal Node IP to the requested domain name.

```Bash
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "$NODE_IP  world.universe.mine" | sudo tee -a /etc/hosts
```
2. Create the Ingress Resource (world-ingress.yaml)
Note: We define the paths exactly as /europe and /africa (without trailing slashes) to ensure Nginx serves the correct files without throwing a 404.
```YAML
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: world
  namespace: world
spec:
  rules:
    - host: world.universe.mine
      http:
        paths:
          - path: /europe
            pathType: Prefix
            backend:
              service:
                name: europe
                port:
                  number: 8888
          - path: /africa
            pathType: Prefix
            backend:
              service:
                name: africa
                port:
                  number: 8888
```
Apply the Ingress:

```Bash
kubectl apply -f world-ingress.yaml
```
<img width="1582" height="622" alt="image" src="https://github.com/user-attachments/assets/d979f741-a598-4b80-a2da-50151c11602f" />

✅ Step 4: Verification
Test the routing to ensure traffic is correctly directed to the respective pods:

Bash
curl [http://world.universe.mine/europe](http://world.universe.mine/europe)
# Expected Output: welcome to europe

curl [http://world.universe.mine/africa](http://world.universe.mine/africa)
# Expected Output: welcome to africa
<img width="915" height="352" alt="image" src="https://github.com/user-attachments/assets/a1f4d3b8-77b5-45d3-b291-abf237f741a4" />

🔧 Troubleshooting Notes
During this lab, you may encounter a few common cluster networking hurdles:

502 Bad Gateway: If the Ingress controller cannot communicate with the Pods, it throws a 502 error. In a multi-node environment (like K3s), this is often caused by the Linux firewall blocking internal VXLAN overlay network traffic.

Fix: Disable the firewall on all nodes: **sudo systemctl stop firewalld && sudo systemctl disable firewalld.**

404 Not Found: If you reach the pod but receive a 404, check your Ingress path syntax. Including a trailing slash (/africa/) may cause the backend web server (Nginx) to look for a directory that doesn't exist, rather than serving the file mapped to that specific path.
