
# ⚙️ Kubernetes Lab - ConfigMap Management

## 🎯 Lab Objectives
- Create a declarative ConfigMap from a YAML file (`birke`).
- Create an imperative ConfigMap via the CLI (`trauerweide`).
- Inject specific configuration keys as Environment Variables into a Pod.
- Mount a full ConfigMap as a filesystem volume inside a container.

---

## 🛠️ Step 1: ConfigMap Provisioning

### 1. Declarative ConfigMap (`birke`)
A configuration file was created at `/opt/cm.yaml` to store park department data.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: birke
data:
  tree: birke
  level: "3"
  department: park
```
**Apply Command:**
```bash
kubectl apply -f /opt/cm.yaml
```
<img width="505" height="172" alt="image" src="https://github.com/user-attachments/assets/a0ba3f22-59ae-449d-b54a-6ba1e667bddc" />


### 2. Imperative ConfigMap (`trauerweide`)
The second ConfigMap was generated directly using the CLI:
```bash
kubectl create configmap trauerweide --from-literal=tree=trauerweide
```
<img width="1083" height="140" alt="image" src="https://github.com/user-attachments/assets/f28c44f0-4a79-4ad0-9b36-1b51070d5498" />

---

## 📦 Step 2: Pod Deployment (`pod1`)

The Pod manifest was configured to consume both ConfigMaps. To ensure high availability and bypass potential networking issues on secondary nodes, the Pod was pinned to the master node.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  nodeName: linux  # Scheduled on master for direct connectivity
  containers:
  - name: pod1
    image: nginx:alpine
    env:
    - name: TREE1
      valueFrom:
        configMapKeyRef:
          name: trauerweide
          key: tree
    volumeMounts:
    - name: config-volume
      mountPath: /etc/birke
  volumes:
  - name: config-volume
    configMap:
      name: birke
```

---

## ✅ Step 3: Verification & Troubleshooting

### 1. Environment Variable Check
Verified that the `TREE1` variable correctly pulls the value from the `trauerweide` ConfigMap.
```bash
kubectl exec pod1 -- env | grep TREE1
```

<img width="881" height="283" alt="image" src="https://github.com/user-attachments/assets/cfdfe7b0-f95b-4c25-bd4d-d83be1916107" />


### 2. Volume Mount Check
Verified that the keys from the `birke` ConfigMap are present as files in the specified directory.
```bash
kubectl exec pod1 -- ls /etc/birke

```
<img width="735" height="231" alt="image" src="https://github.com/user-attachments/assets/d0058b1a-32d5-401b-9df4-223af09514f6" />


### 3. Content Validation
```bash
kubectl exec pod1 -- cat /etc/birke/tree
# Output: birke
```
<img width="697" height="167" alt="image" src="https://github.com/user-attachments/assets/3472af47-2a59-4b74-847b-4e4827698648" />

---

## 🔧 Troubleshooting Summary
- **502/NotReady Errors:** Addressed by disabling `firewalld` on cluster nodes and using `nodeName` to ensure Pods run on the control-plane node.
- **FailedMount Warnings:** Resolved by ensuring the `volumes.configMap.name` in the Pod manifest matches the `metadata.name` of the created ConfigMap exactly.


``
