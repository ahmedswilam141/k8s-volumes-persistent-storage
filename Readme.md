# 📦 Kubernetes Volumes & Persistent Storage

<div align="center">

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.29+-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Minikube](https://img.shields.io/badge/Minikube-Compatible-F5A800?style=for-the-badge&logo=kubernetes&logoColor=white)
![CKA Level](https://img.shields.io/badge/Level-CKA-FF6B6B?style=for-the-badge&logo=linux-foundation&logoColor=white)
![Status](https://img.shields.io/badge/Lab-Completed%20✓-4CAF50?style=for-the-badge)

**A hands-on lab covering Kubernetes storage primitives — from ephemeral volumes to fully persistent storage with PV/PVC lifecycle management.**

[Overview](#-overview) • [Prerequisites](#-prerequisites) • [Lab Structure](#-lab-structure) • [Quick Start](#-quick-start) • [Tasks](#-tasks) • [Screenshots](#-screenshots)

</div>

---

## 📖 Overview

This repository contains the full solution for the **Kubernetes Volumes & Persistent Storage Lab (Session 7)** — a CKA-level lab that explores Kubernetes storage from the ground up. By completing this lab, you will understand:

- How **ephemeral volumes** (`emptyDir`) enable inter-container communication within a Pod
- How **hostPath** volumes expose node-level filesystem data to containers
- How to provision and manage **PersistentVolumes (PV)** and **PersistentVolumeClaims (PVC)**
- How to build a **multi-volume Pod** combining all three volume types simultaneously

| Property | Details |
|----------|---------|
| ⏱ Duration | 50 minutes |
| 📊 Total Points | 100 pts |
| 📋 Tasks | 6 tasks across 3 parts |
| 🎯 Level | CKA (Certified Kubernetes Administrator) |
| 🛠 Environment | Minikube |

---

## ✅ Prerequisites

Make sure the following tools are installed and running before starting:

```bash
# Verify kubectl
kubectl version --client

# Verify Minikube is running
minikube status

# Start Minikube if not running
minikube start
```

| Tool | Version | Purpose |
|------|---------|---------|
| `kubectl` | v1.29+ | Kubernetes CLI |
| `minikube` | v1.32+ | Local cluster |
| `curl` | any | HTTP testing inside pods |

---

## 📁 Lab Structure

```
k8s-volumes-persistent-storage/
│
├── 01-emptydir-volume.yaml        # Task 1 — emptyDir shared between containers
├── 02-hostpath-volume.yaml        # Task 2 — hostPath nginx file server
├── 03-persistentvolume.yaml       # Task 3 — PersistentVolume (1Gi, Retain)
├── 04-persistentvolumeclaim.yaml  # Task 4 — PersistentVolumeClaim (500Mi)
├── 05-pod-with-pvc.yaml           # Task 5 — Pod consuming the PVC
├── 06-deployment-with-pvc.yaml    # Task 6 prereq — creates app-pvc
├── 07-multi-volume-pod.yaml       # Task 6 — Pod with 3 simultaneous volumes
└── README.md
```

---

## 🚀 Quick Start

Clone the repo and apply all manifests in order:

```bash
git clone https://github.com/<your-username>/k8s-volumes-persistent-storage.git
cd k8s-volumes-persistent-storage

# Run all tasks in sequence
kubectl apply -f 01-emptydir-volume.yaml
kubectl apply -f 02-hostpath-volume.yaml
kubectl apply -f 03-persistentvolume.yaml
kubectl apply -f 04-persistentvolumeclaim.yaml
kubectl apply -f 05-pod-with-pvc.yaml
kubectl apply -f 06-deployment-with-pvc.yaml
kubectl create configmap app-config --from-literal=KEY=value
kubectl apply -f 07-multi-volume-pod.yaml
```

---

## 📋 Tasks

### Part 1 — emptyDir & hostPath `(30 pts)`

---

#### ⭐ Task 1 — emptyDir: Share Data Between Containers

> Two containers inside one Pod share a common `emptyDir` volume. The `writer` container appends a timestamp every 5 seconds; the `reader` container reads the file every 10 seconds.

**Key concept:** `emptyDir` is created when a Pod is assigned to a node and exists as long as the Pod is running. All containers in the Pod share the same directory. **Data is lost when the Pod is deleted.**

```bash
# Apply the manifest
kubectl apply -f 01-emptydir-volume.yaml

# Wait for pod to be Running
kubectl get pod emptydir-demo -w

# Verify reader sees writer's output
kubectl logs emptydir-demo -c reader

# 📸 Screenshot 1 — exec into writer and confirm the file
kubectl exec emptydir-demo -c writer -- cat /shared/log.txt

# Prove data is ephemeral — delete and recreate
kubectl delete pod emptydir-demo && kubectl apply -f 01-emptydir-volume.yaml
```

<details>
<summary>📄 View YAML</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: writer
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - while true; do
        echo "$(date) - Hello from writer" >> /shared/log.txt;
        sleep 5;
      done
    volumeMounts:
    - name: shared-data
      mountPath: /shared

  - name: reader
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - while true; do
        cat /shared/log.txt 2>/dev/null || echo "File not yet created";
        sleep 10;
      done
    volumeMounts:
    - name: shared-data
      mountPath: /shared

  volumes:
  - name: shared-data
    emptyDir: {}
```

</details>

---

#### ⭐⭐ Task 2 — hostPath: Serve Files from Node

> Mount a directory from the **Minikube node's filesystem** into an nginx container. Changes to the file on the node are reflected instantly inside the pod — without a restart.

**Key concept:** `hostPath` mounts a file or directory from the host node into the Pod. Useful for development and node-level log access, but **not recommended for production** as it couples pods to specific nodes.

```bash
# Create the HTML file on the Minikube node
minikube ssh -- mkdir -p /tmp/webdata
minikube ssh -- 'echo "<h1>From the Node!</h1>" > /tmp/webdata/index.html'

# Apply the manifest
kubectl apply -f 02-hostpath-volume.yaml

# 📸 Screenshot 2 — curl from inside the pod
kubectl exec hostpath-demo -- curl localhost

# Live update — no pod restart needed!
minikube ssh -- 'echo "<h1>Updated!</h1>" > /tmp/webdata/index.html'
kubectl exec hostpath-demo -- curl localhost
```

<details>
<summary>📄 View YAML</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
    volumeMounts:
    - name: webdata
      mountPath: /usr/share/nginx/html

  volumes:
  - name: webdata
    hostPath:
      path: /tmp/webdata
      type: DirectoryOrCreate
```

</details>

---

### Part 2 — PersistentVolume & PVC `(50 pts)`

---

#### ⭐ Task 3 — Create a PersistentVolume

> Provision a cluster-level storage resource with 1Gi capacity and a `Retain` reclaim policy.

**Key concept:** A `PersistentVolume (PV)` is a piece of storage provisioned by an administrator. It exists independently of any Pod and has its own lifecycle. The `Retain` reclaim policy means data is **preserved** even after the PVC is deleted.

```bash
kubectl apply -f 03-persistentvolume.yaml
kubectl get pv my-pv
```

<details>
<summary>📄 View YAML</summary>

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/my-pv-data
    type: DirectoryOrCreate
```

</details>

---

#### ⭐⭐ Task 4 — Create a PVC and Bind to the PV

> Request 500Mi of storage — Kubernetes will automatically bind this claim to the available `my-pv`.

**Key concept:** A `PersistentVolumeClaim (PVC)` is a **request for storage** made by a user. Kubernetes matches it to an available PV that satisfies the access mode and capacity requirements.

```bash
kubectl apply -f 04-persistentvolumeclaim.yaml

# 📸 Screenshot 3 — both should show Bound status
kubectl get pv,pvc
```

<details>
<summary>📄 View YAML</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

</details>

---

#### ⭐⭐ Task 5 — Use PVC in a Pod

> Mount the PVC into an nginx pod and verify that **data survives pod deletion and recreation**.

**Key concept:** This is the key difference between `emptyDir` and PVC-backed storage. A PVC outlives the Pod — data written to the mounted path persists across Pod restarts and deletions.

```bash
kubectl apply -f 05-pod-with-pvc.yaml

# Write data to the persistent volume
kubectl exec pvc-demo -- sh -c 'echo persistent > /usr/share/nginx/html/test.txt'

# Delete and recreate — data must survive
kubectl delete pod pvc-demo && kubectl apply -f 05-pod-with-pvc.yaml
kubectl get pod pvc-demo -w

# 📸 Screenshot 4 — file still exists after pod deletion
kubectl exec pvc-demo -- cat /usr/share/nginx/html/test.txt
# Expected output: persistent
```

<details>
<summary>📄 View YAML</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: persistent-storage
      mountPath: /usr/share/nginx/html

  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

</details>

---

### Part 3 — Multi-Volume Pod `(20 pts)`

---

#### ⭐⭐⭐ Task 6 — Pod with Three Simultaneous Volumes

> The most advanced task: a single Pod mounting three different volume types at the same time.

**Key concept:** Kubernetes Pods can mount multiple volumes of different types simultaneously. This is common in real applications — for example, a RAM-backed cache, a persistent data directory, and an injected configuration file.

| Volume | Type | Mount Path | Purpose |
|--------|------|------------|---------|
| `cache-vol` | `emptyDir (Memory)` | `/tmp/cache` | RAM-backed high-speed cache |
| `data-vol` | PVC (`app-pvc`) | `/data` | Persistent application data |
| `config-vol` | ConfigMap (`app-config`) | `/etc/config` | Injected configuration files |

```bash
# Step 1 — Create prerequisites
kubectl apply -f 06-deployment-with-pvc.yaml       # creates app-pvc
kubectl create configmap app-config --from-literal=KEY=value

# Step 2 — Apply the multi-volume pod
kubectl apply -f 07-multi-volume-pod.yaml
kubectl get pod multi-volume-demo -w

# 📸 Screenshot 5 — verify all 3 mounts
kubectl exec multi-volume-demo -- df -h
kubectl exec multi-volume-demo -- ls /etc/config
```

**Expected output from `df -h`:**
- `/tmp/cache` → `tmpfs` (Memory-backed) ✅
- `/data` → block device (PVC-backed) ✅
- `/etc/config` → filesystem mount with `KEY` file ✅

<details>
<summary>📄 View YAML</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-volume-demo
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: cache-vol
      mountPath: /tmp/cache
    - name: data-vol
      mountPath: /data
    - name: config-vol
      mountPath: /etc/config

  volumes:
  - name: cache-vol
    emptyDir:
      medium: Memory

  - name: data-vol
    persistentVolumeClaim:
      claimName: app-pvc

  - name: config-vol
    configMap:
      name: app-config
```

</details>

---

## 📸 Screenshots

All 5 required screenshots captured and verified:

| # | Screenshot | What it Proves |
|---|-----------|----------------|
| 1 | `kubectl exec emptydir-demo -c writer -- cat /shared/log.txt` | Writer container logging timestamps; shared volume works |
| 2 | `kubectl exec hostpath-demo -- curl localhost` | nginx serving `<h1>From the Node!</h1>` from hostPath |
| 3 | `kubectl get pv,pvc` | Both `my-pv` and `my-pvc` show **Bound** status |
| 4 | `kubectl exec pvc-demo -- cat /usr/share/nginx/html/test.txt` | `persistent` output after pod delete+recreate |
| 5 | `kubectl exec multi-volume-demo -- df -h` + `ls /etc/config` | All 3 mounts active: tmpfs, PVC block, ConfigMap KEY |

---

## 🧠 Key Concepts Summary

| Volume Type | Lifecycle | Use Case | Data Persists? |
|-------------|-----------|----------|----------------|
| `emptyDir` | Pod lifetime | Shared scratch space, caching between containers | ❌ Lost on Pod delete |
| `emptyDir (Memory)` | Pod lifetime | High-speed in-memory cache | ❌ Lost on Pod delete |
| `hostPath` | Node lifetime | Dev environments, node log access | ⚠️ Node-dependent |
| `PersistentVolume` | Cluster lifetime | Databases, stateful apps | ✅ Survives Pod delete |
| `ConfigMap` volume | Config-driven | Inject config files into containers | N/A — read-only config |

---

## 🧹 Cleanup

Remove all resources created during the lab:

```bash
# Delete pods
kubectl delete pod emptydir-demo hostpath-demo pvc-demo multi-volume-demo

# Delete PVC and PV
kubectl delete pvc my-pvc app-pvc
kubectl delete pv my-pv

# Delete deployment and configmap
kubectl delete deployment app-deployment
kubectl delete configmap app-config

# Clean up node data
minikube ssh -- rm -rf /tmp/webdata /tmp/my-pv-data
```

---

## 📚 References

- [Kubernetes Volumes Documentation](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Configure a Pod to Use a PersistentVolume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)
- [CKA Exam Curriculum](https://github.com/cncf/curriculum)

---

<div align="center">

Made with ☸️ for the CKA journey

</div>