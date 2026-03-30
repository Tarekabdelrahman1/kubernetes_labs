
# Kubernetes Administration Lab: Kubeadm & K3s Setup

This repository contains the step-by-step guide and configurations for provisioning a multi-node Kubernetes cluster on Red Hat-based systems using `kubeadm` (v1.34.6), deploying a highly available Nginx application, and setting up a lightweight `k3s` cluster.

## 🎯 Lab Objectives
1. **Core Task (10 Points):** Install Kubernetes v1.34.6 using `kubeadm` on 2 VMs (1 Control Plane, 1 Worker) and configure the Flannel CNI.
2. **Deployment Task (10 Points):** Deploy a 3-replica Nginx application to the cluster.
3. **Bonus Task (5 Points):** Re-do the lab using `k3s` to provision a Server and Agent node.

## ⚙️ Prerequisites
* 2 Virtual Machines running a Red Hat-based OS (RHEL, Rocky, AlmaLinux, CentOS).
* Sudo privileges on both machines.
* Minimum 2GB RAM and 2 CPUs per VM.

---

## Part 1: Provisioning the Kubeadm Cluster

### 1. System Preparation (Run on BOTH Nodes)
Kubernetes requires specific system configurations to function correctly, particularly on security-hardened systems like Red Hat.

```bash
# Disable swap (Kubelet fails if swap is enabled)
sudo swapoff -a

# Set SELinux to Permissive (Allows containers to access the host filesystem)
sudo setenforce 0
```

# Load required kernel modules for networking
```text
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

# Enable IP Forwarding for Pod network routing
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```
## 2. Install Container Runtime (containerd) (Run on BOTH Nodes)
```Bash
# Add Docker repo and install containerd
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo [https://download.docker.com/linux/centos/docker-ce.repo](https://download.docker.com/linux/centos/docker-ce.repo)
sudo dnf install -y containerd.io
```

# Configure containerd to use systemd as the cgroup driver (Critical for Red Hat)
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
# Start and enable the service
sudo systemctl enable --now containerd

<img width="1462" height="438" alt="image" src="https://github.com/user-attachments/assets/cd4bb717-b76d-4380-a6eb-50ac50dd7554" />

3. Install Kubernetes Components (Run on BOTH Nodes)
Bash
# Add the official Kubernetes repository
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=[https://pkgs.k8s.io/core:/stable:/v1.34/rpm/](https://pkgs.k8s.io/core:/stable:/v1.34/rpm/)
enabled=1
gpgcheck=1
gpgkey=[https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key](https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key)
EOF
```

# Install kubeadm, kubelet, and kubectl
```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```
## 4. Configure Firewall Rules
**On the Control Plane Node:**

```Bash
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=8472/udp # Flannel VXLAN
sudo firewall-cmd --reload
```
**On the Worker Node:**

```Bash
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp # NodePorts
sudo firewall-cmd --permanent --add-port=8472/udp # Flannel VXLAN
sudo firewall-cmd --reload
```

## 5. Initialize the Control Plane (Run on MASTER Node ONLY)
# Initialize the cluster specifying the Pod network CIDR for Flannel
sudo kubeadm init --config ~/kubeadm.yaml --upload-certs
<img width="1476" height="641" alt="image" src="https://github.com/user-attachments/assets/24dc4b42-b593-473e-ab7f-41c9bd41be86" />

<img width="793" height="165" alt="image" src="https://github.com/user-attachments/assets/2b7698af-7f14-4735-9aec-526531180559" />



# Configure kubectl for the current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install the Flannel Network Plugin (CNI)
```
kubectl apply -f [https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml](https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml)
```
<img width="1522" height="338" alt="image" src="https://github.com/user-attachments/assets/9aebe928-3565-4c9a-b2a0-7a697d8ad204" />




## 6. Join the Worker Node (Run on WORKER Node ONLY)
Note: Use the exact join command outputted at the end of the kubeadm init process. or use **kubeadm token create --print-join-command**
<img width="1470" height="131" alt="image" src="https://github.com/user-attachments/assets/b1e8c17d-080f-4505-b3d8-4e03c9f0f9a9" />
**After working node Joined the master node**
**worker node**
<img width="815" height="411" alt="image" src="https://github.com/user-attachments/assets/eb259f74-c51a-4d92-9292-2878e99d8db5" />
**master node**
<img width="851" height="193" alt="image" src="https://github.com/user-attachments/assets/491c207f-ae99-4b13-a975-8c6484eba30a" />


# Part 2: Deploying the Application
Objective: Deploy a 3-replica Nginx application to ensure the cluster can schedule and manage workloads.

## 1. Create the Deployment Manifest
Create a file named nginx-deployment.yaml with the following content:

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
## 2. Apply the Configuration
```Bash
kubectl apply -f nginx-deployment.yaml
```
<img width="840" height="176" alt="image" src="https://github.com/user-attachments/assets/a918b7c8-859f-4729-b67b-1a87d2e29dbc" />

### Verify the deployment created 3/3 ready replicas
```Bash
kubectl get deployments
```
### Verify all 3 Pods are "Running" and scheduled on the Worker Node
```bash
kubectl get pods -o wide
```
<img width="1553" height="303" alt="image" src="https://github.com/user-attachments/assets/39d897f3-1a3b-46de-86c6-2f5123a73dbb" />


# Part3 : Lightweight K3s Cluster Setup 



## Step 1: Initialize the Server (Control Plane)

**Target Machine:** VM 1 (Server IP: `10.0.0.15`)

1. Install the K3s server. This single command downloads the binary, starts the service, and configures the internal network:
   ```bash
   curl -sfL [https://get.k3s.io](https://get.k3s.io) | sh -

<img width="1391" height="565" alt="image" src="https://github.com/user-attachments/assets/5ff6f7c0-3a0a-47f4-9a17-b616ca43f25d" />
<img width="1185" height="562" alt="image" src="https://github.com/user-attachments/assets/691f9a0a-3265-44e0-8fbe-d95c470a8892" />

2.Extract the auto-generated cluster token. The agent node will need this password to authenticate and join the cluster:
```
Bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
<img width="1378" height="148" alt="image" src="https://github.com/user-attachments/assets/e6b4f246-22b8-4125-9226-57a33a148cf1" />

## Step 2: Join the Agent (Worker Node)
Target Machine: VM 2 (Agent)
```bash
curl -sfL [https://get.k3s.io](https://get.k3s.io) | K3S_URL=[https://10.0.0.16:6443](https://10.0.0.16:6443) K3S_TOKEN=K109242cd017c8b6656d0dc75655b647ba654218c6f5dac6c5e09a2b036e8846fd0::server:fc6f4f0bd5c9075998b332759128b60d sh -
```
<img width="747" height="182" alt="image" src="https://github.com/user-attachments/assets/6a1be73f-d718-4b6c-8053-0324f2d98d8e" />





