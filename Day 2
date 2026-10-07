# Kubernetes Notes: Day 2

## Table of Contents

1. [Kubernetes Services and Distributions (EKS, AKS, GKE, OpenShift, etc.)](#1-kubernetes-services-and-distributions)
2. [KOps](#2-kops-kubernetes-operations)
3. [Pods in Detail (the Wrapper)](#3-pods-in-detail)
4. [What is Minikube?](#4-what-is-minikube)
5. [Installing Minikube and kubectl](#5-installing-minikube-and-kubectl)
6. [Nginx Pod YAML vs Docker Command](#6-nginx-pod-yaml-vs-docker-command)
7. [Hands-on Commands Explained](#7-hands-on-commands-explained)
8. [Quick Cheat Sheet](#8-quick-cheat-sheet)

---

## 1. Kubernetes Services and Distributions

Running Kubernetes yourself (installing the control plane, patching, upgrading, backing up etcd, securing it) is hard. There are several ways to get a cluster, ranging from **fully self-managed** to **fully managed**.

> Note: This section is about the **products and platforms** that provide Kubernetes. The Kubernetes **Service object** (ClusterIP, NodePort, LoadBalancer) is a separate topic covered later.

### 1.1 Ways to Run Kubernetes

| Approach | Who manages the control plane? | Examples |
|---|---|---|
| **Local / learning** | You (single machine) | Minikube, kind, k3d, Docker Desktop |
| **Self-managed (DIY)** | You | `kubeadm`, KOps, Kubespray |
| **Managed cloud service** | Cloud provider | EKS, AKS, GKE |
| **Enterprise platform** | You or vendor, with extra tooling | Red Hat OpenShift, Rancher, VMware Tanzu |

### 1.2 Managed Services from Cloud Providers

| Service | Provider | Key points |
|---|---|---|
| **EKS** (Elastic Kubernetes Service) | AWS | Managed control plane (API server, etcd) across multiple AZs. Worker nodes via managed node groups, self-managed EC2, or **Fargate** (serverless pods). Integrates with IAM, VPC, ELB, EBS/EFS, CloudWatch. |
| **AKS** (Azure Kubernetes Service) | Microsoft Azure | Control plane is managed (no charge for the free tier control plane). Integrates with Microsoft Entra ID (Azure AD), Azure Monitor, Azure Container Registry. |
| **GKE** (Google Kubernetes Engine) | Google Cloud | The most mature managed offering (Google created Kubernetes). Offers **Autopilot** mode where Google manages nodes too. Strong auto-scaling and auto-upgrade features. |

**What "managed" means:** the provider runs and patches the **control plane** (kube-apiserver, etcd, scheduler, controller-manager) and keeps it highly available. You are still responsible for your **worker nodes (in most modes), your workloads, and application security**.

**Benefits:** no control plane operations, built-in high availability, easy upgrades, deep integration with cloud networking, storage and identity.

**Trade-offs:** cost for the control plane (on EKS), some vendor lock-in through cloud-specific integrations, less control over control plane configuration.

### 1.3 OpenShift

**Red Hat OpenShift** is an **enterprise Kubernetes platform**: Kubernetes plus a lot of opinionated, built-in tooling.

What it adds on top of vanilla Kubernetes:

- **Developer tooling**: built-in CI/CD (OpenShift Pipelines, based on Tekton), Source-to-Image (S2I) builds, web console for developers and admins
- **Security by default**: stricter defaults, Security Context Constraints (SCCs), containers do not run as root by default
- **Built-in image registry** and image streams
- **Routes** (OpenShift's original ingress mechanism) for exposing apps
- **Operators and OperatorHub** for installing and managing software
- **Integrated monitoring and logging**
- **Enterprise support** from Red Hat

**Deployment options:**

| Offering | Description |
|---|---|
| **OpenShift Container Platform (OCP)** | Self-managed, runs on-prem or in any cloud |
| **ROSA** | Red Hat OpenShift Service on AWS (managed jointly by Red Hat and AWS) |
| **ARO** | Azure Red Hat OpenShift (managed jointly by Red Hat and Microsoft) |
| **OpenShift Dedicated** | Red Hat-managed service on public cloud |
| **OKD** | The free, community upstream version of OpenShift |
| **OpenShift Local (formerly CRC)** | Single-node local cluster for development |

### 1.4 Other Platforms and Tools

| Tool | What it is |
|---|---|
| **Rancher** | Multi-cluster management platform, lets you manage many clusters (EKS, AKS, on-prem) from one UI |
| **VMware Tanzu** | Kubernetes platform for VMware environments |
| **kubeadm** | Official tool to bootstrap a cluster on machines you provide; a building block for DIY clusters |
| **Kubespray** | Ansible-based installer for production clusters on-prem or cloud |
| **KOps** | CLI to create and manage production clusters on AWS (see next section) |
| **k3s** | Lightweight Kubernetes distribution for edge and IoT, by Rancher/SUSE |
| **kind** | "Kubernetes in Docker", runs cluster nodes as containers (great for CI and testing) |

### 1.5 Choosing an Option

| Situation | Typical choice |
|---|---|
| Learning on a laptop | Minikube or kind |
| Production on AWS with minimal ops | EKS |
| Production on Azure / GCP | AKS / GKE |
| Regulated enterprise, hybrid cloud, strong built-in security | OpenShift |
| Need full control and are on AWS | KOps (or kubeadm / Kubespray) |

---

## 2. KOps (Kubernetes Operations)

**KOps** is a command-line tool that helps you **create, upgrade, and delete production-grade, highly available Kubernetes clusters**. Think of it as "`kubectl` for clusters": `kubectl` manages things *inside* a cluster, while `kops` manages the **cluster itself**.

- **Best supported on AWS** (general availability). Support for other platforms such as Google Cloud, DigitalOcean and others exists at varying maturity levels.
- It provisions the **cloud infrastructure** for you: VPC, subnets, security groups, EC2 instances, Auto Scaling Groups, load balancers, IAM roles, and DNS records.
- You manage the **control plane yourself**, unlike EKS, so you get more control but also more responsibility.

### 2.1 What KOps Does

- Creates highly available (multi-master / multi-AZ) clusters
- Automates **rolling updates** and Kubernetes version upgrades
- Supports **Terraform output** (`--target=terraform`) so you can manage infrastructure as code
- Manages **instance groups** (groups of nodes with the same configuration)
- Supports several networking options (Calico, Cilium, Flannel, etc.)
- Stores cluster state in an **S3 bucket** (the "state store")

### 2.2 Prerequisites on AWS

1. An AWS account with an IAM user/role that has the required permissions (EC2, S3, IAM, VPC, Route 53, Auto Scaling, ELB)
2. **AWS CLI** installed and configured (`aws configure`)
3. **kubectl** installed
4. **kops** installed
5. An **S3 bucket** to store cluster state (versioning recommended)
6. DNS: a Route 53 hosted zone, **or** a "gossip-based" cluster name ending in `.k8s.local` (no DNS needed, handy for learning)

### 2.3 Typical Workflow

```bash
# 1. Create an S3 bucket for the state store
aws s3 mb s3://my-kops-state-store --region eu-west-1

# 2. Tell kops where the state is stored
export KOPS_STATE_STORE=s3://my-kops-state-store

# 3. Define the cluster (this only creates the configuration, nothing is built yet)
kops create cluster \
  --name=demo.k8s.local \
  --zones=eu-west-1a \
  --node-count=2 \
  --node-size=t3.medium \
  --control-plane-size=t3.medium

# 4. Review (optional)
kops edit cluster demo.k8s.local

# 5. Actually build the cluster in AWS
kops update cluster --name demo.k8s.local --yes --admin

# 6. Wait until it is ready
kops validate cluster --wait 10m

# 7. Use it
kubectl get nodes

# 8. Delete it when done (avoid AWS charges!)
kops delete cluster --name demo.k8s.local --yes
```

### 2.4 KOps vs EKS

| Aspect | KOps | EKS |
|---|---|---|
| Control plane | You manage (runs on EC2 instances) | AWS manages |
| Control | Very high | Moderate |
| Operational effort | Higher | Lower |
| Cost | Pay for control plane EC2 instances | Fixed control plane fee plus nodes |
| Upgrades | You run `kops rolling-update` | Managed upgrade process |
| Cloud support | AWS mainly, others vary | AWS only |

> Many teams today choose **EKS** for production on AWS. KOps is still valuable for learning how clusters are built and for cases that need full control.

---

## 3. Pods in Detail

### 3.1 What is a Pod?

A **Pod** is the **smallest deployable unit in Kubernetes**. Kubernetes does **not run containers directly**; it runs **Pods**, and a Pod **wraps one or more containers**.

```
+-------------------------- POD (one IP address) --------------------------+
|                                                                          |
|   +---------------+   +---------------+                                  |
|   |  Container 1  |   |  Container 2  |   (optional sidecar etc.)        |
|   |   (nginx)     |   |  (log shipper)|                                  |
|   +---------------+   +---------------+                                  |
|                                                                          |
|   Shared: network namespace (IP, ports) | volumes | IPC | hostname       |
+--------------------------------------------------------------------------+
```

> **Wrapper idea:** just as a container wraps an application and its dependencies, a **Pod wraps containers** and adds Kubernetes-specific configuration: shared networking, shared storage, restart policy, resource requests/limits, health probes and so on.

### 3.2 Why Does Kubernetes Use Pods Instead of Plain Containers?

- Some applications need **tightly coupled helper processes** (log shipper, proxy, config reloader) that must run together, share `localhost`, and live and die together.
- A Pod gives them a **single unit of scheduling, scaling and lifecycle**.
- It adds a **uniform abstraction**: Kubernetes can work with different container runtimes (containerd, CRI-O) while the Pod spec stays the same.

### 3.3 What Do Containers in a Pod Share?

| Resource | Sharing behaviour |
|---|---|
| **Network** | Same IP address and port space; containers talk to each other over `localhost` |
| **Storage** | Can mount the same **volumes** |
| **IPC / process namespaces** | Can share IPC (and optionally the process namespace) |
| **Lifecycle** | Scheduled together on the **same node**, started and stopped as a unit |

Each container still has its **own filesystem and image**, and its own CPU and memory limits.

> Behind the scenes, a tiny **"pause" container** holds the Pod's network namespace so other containers can join it.

### 3.4 Key Characteristics

- **One IP per Pod**, not per container. Pods get a cluster-internal IP.
- **Ephemeral**: Pods are disposable. If a Pod dies it is **replaced by a new one with a new IP**, not repaired.
- **Not self-healing on their own**: a bare Pod that you create manually is *not* recreated if its node fails. That is why we use higher-level controllers (**Deployment, ReplicaSet, StatefulSet, DaemonSet, Job**) in real environments.
- **One main container per Pod** is the most common pattern.

### 3.5 Multi-Container Pod Patterns

| Pattern | Purpose | Example |
|---|---|---|
| **Sidecar** | Extends or supports the main container | Log shipper, service mesh proxy (Envoy) |
| **Init container** | Runs to completion **before** the main containers start | Wait for DB, run migrations, download config |
| **Ambassador** | Proxies network connections for the main container | Local proxy to an external database |
| **Adapter** | Standardises or transforms output | Convert app metrics to Prometheus format |

### 3.6 Pod Lifecycle (Phases)

| Phase | Meaning |
|---|---|
| **Pending** | Accepted by the cluster, but containers are not running yet (waiting for scheduling or image pull) |
| **Running** | Bound to a node, and at least one container is running or starting |
| **Succeeded** | All containers terminated successfully and will not restart (typical for Jobs) |
| **Failed** | All containers terminated and at least one failed |
| **Unknown** | The Pod state cannot be determined (usually a node communication problem) |

Common container-level states and errors you will see in `kubectl get pods`: `ContainerCreating`, `CrashLoopBackOff`, `ImagePullBackOff` / `ErrImagePull`, `Completed`, `Terminating`.

### 3.7 Anatomy of a Pod Manifest

Every Kubernetes object YAML has four top-level fields:

```yaml
apiVersion: v1          # API version of the object
kind: Pod               # Type of object
metadata:               # Name, labels, namespace, annotations
  name: nginx
spec:                   # Desired state: containers, volumes, etc.
  containers:
    - name: nginx
      image: nginx:latest
```

---

## 4. What is Minikube?

**Minikube** is a tool that runs a **single-node Kubernetes cluster on your local machine** (laptop or desktop), designed for **learning, development and testing**.

- Control plane and worker components run together on **one node**
- Runs inside a **VM or a container** on your machine, depending on the driver
- Supports **drivers**: Docker, Hyperkit, VirtualBox, VMware, Hyper-V, KVM, Podman
- Ships with useful **add-ons** (dashboard, ingress, metrics-server) that you enable with `minikube addons enable <name>`
- Not meant for production use

| Feature | Minikube | Production cluster |
|---|---|---|
| Nodes | 1 (multi-node possible but limited) | Many |
| High availability | No | Yes |
| Purpose | Learn, develop, test | Run real workloads |
| Setup time | Minutes | Hours to days |

**Minimum requirements:** 2 CPUs, 2 GB free memory, 20 GB free disk space, internet connection, and a container or VM manager (e.g. Docker).

---

## 5. Installing Minikube and kubectl

You need two tools:

- **minikube**: creates and manages the local cluster
- **kubectl**: the command-line client used to talk to *any* Kubernetes cluster

### 5.1 Prerequisite: a Driver (Docker recommended)

Install **Docker Desktop** (Windows/macOS) or **Docker Engine** (Linux) and make sure it is running:

```bash
docker --version
```

### 5.2 Install on Linux

```bash
# Install minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
minikube version
kubectl version --client
```

If using Docker on Linux, add your user to the docker group so you do not need `sudo`:

```bash
sudo usermod -aG docker $USER && newgrp docker
```

### 5.3 Install on macOS

```bash
brew install minikube
brew install kubectl

minikube version
kubectl version --client
```

### 5.4 Install on Windows

Using **winget** (PowerShell):

```powershell
winget install Kubernetes.minikube
winget install Kubernetes.kubectl

minikube version
kubectl version --client
```

Or using **Chocolatey**:

```powershell
choco install minikube kubernetes-cli
```

### 5.5 Start the Cluster

```bash
minikube start --driver=docker
```

Verify:

```bash
minikube status
kubectl get nodes
```

> Tip: To make Docker the default driver permanently, run `minikube config set driver docker`.

---

## 6. Nginx Pod YAML vs Docker Command

### 6.1 The Docker Way

```bash
docker run -d --name nginx -p 80:80 nginx:latest
```

| Part | Meaning |
|---|---|
| `docker run` | Create and start a container |
| `-d` | Detached (run in background) |
| `--name nginx` | Name of the container |
| `-p 80:80` | Publish host port 80 to container port 80 |
| `nginx:latest` | Image to use |

### 6.2 The Kubernetes Way: `pod.yml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

Create it with:

```bash
kubectl create -f pod.yml
```

### 6.3 Side-by-Side Comparison

| Concept | Docker command | Kubernetes `pod.yml` |
|---|---|---|
| Resource type | (implicit: a container) | `kind: Pod` |
| Name | `--name nginx` | `metadata.name: nginx` |
| Image | `nginx:latest` | `spec.containers[].image: nginx:latest` |
| Container name | same as `--name` | `spec.containers[].name: nginx` |
| Port | `-p 80:80` | `ports.containerPort: 80` |
| Labels | `--label app=nginx` | `metadata.labels` |
| Style | **Imperative** (do this now) | **Declarative** (this is the desired state) |
| Reusable / version controlled | Not by itself | Yes, YAML is stored in Git |

> **Important difference:** `containerPort: 80` in a Pod is mostly **informational**. It documents which port the container listens on but does **not** publish it outside the cluster the way `-p 80:80` does. To expose a Pod you use a **Service** (or `kubectl port-forward` for quick testing).

### 6.4 `kubectl create` vs `kubectl apply`

| Command | Behaviour |
|---|---|
| `kubectl create -f pod.yml` | Creates the object. **Fails** if it already exists. |
| `kubectl apply -f pod.yml` | Creates **or updates** the object. Preferred for declarative workflows. |

---

## 7. Hands-on Commands Explained

### 7.1 `minikube start`

```bash
minikube start
# or specify a driver
minikube start --driver=docker
```

**What it does:**

1. Checks the system and chosen driver
2. Downloads the Kubernetes base image/ISO (first run only)
3. Creates a node (a container or VM)
4. Installs and starts the Kubernetes control plane components and the kubelet
5. Configures **kubectl** (updates `~/.kube/config`) to point at the new cluster

**Sample output:**

```
😄  minikube v1.xx on Ubuntu 22.04
✨  Using the docker driver based on user configuration
📌  Using Docker driver with root privileges
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=4000MB) ...
🐳  Preparing Kubernetes v1.xx on Docker ...
🔎  Verifying Kubernetes components...
🏄  Done! kubectl is now configured to use "minikube" cluster
```

Useful related commands:

| Command | Purpose |
|---|---|
| `minikube status` | Show cluster component status |
| `minikube stop` | Stop the cluster (keeps state) |
| `minikube delete` | Delete the cluster completely |
| `minikube dashboard` | Open the Kubernetes web dashboard |
| `minikube ip` | Show the node IP address |

### 7.2 `kubectl get nodes`

```bash
kubectl get nodes
```

**What it does:** lists the **nodes** (machines) in the cluster and their status.

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   2m    v1.xx.x
```

| Column | Meaning |
|---|---|
| **NAME** | Node name |
| **STATUS** | `Ready` means the node is healthy and can run pods |
| **ROLES** | `control-plane` here because Minikube is a single node doing both jobs |
| **AGE** | Time since the node joined |
| **VERSION** | Kubernetes (kubelet) version |

If the status is `NotReady`, the cluster is not healthy, so check `minikube status`.

### 7.3 `kubectl create -f pod.yml`

```bash
kubectl create -f pod.yml
```

```
pod/nginx created
```

**What happens behind the scenes:**

1. `kubectl` reads the YAML and sends it to the **API server**
2. The API server validates it and stores it in **etcd**
3. The **scheduler** picks a node (here, `minikube`)
4. The **kubelet** on that node tells the container runtime to **pull the nginx image** and start the container
5. The Pod moves from `Pending` to `ContainerCreating` to `Running`

### 7.4 `kubectl get pods`

```bash
kubectl get pods
```

```
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          30s
```

| Column | Meaning |
|---|---|
| **NAME** | Pod name |
| **READY** | `ready containers / total containers` (1/1 means the one container is ready) |
| **STATUS** | Phase or container state (`Running`, `Pending`, `CrashLoopBackOff`, etc.) |
| **RESTARTS** | How many times containers in the Pod have restarted |
| **AGE** | Time since creation |

Add `-w` (`kubectl get pods -w`) to **watch** status changes live.

### 7.5 `kubectl get pods -o wide`

```bash
kubectl get pods -o wide
```

```
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          1m    10.244.0.3   minikube   <none>           <none>
```

**What it adds:** extra columns, most importantly the **Pod IP** and the **node** it is running on.

| Extra column | Meaning |
|---|---|
| **IP** | The Pod's cluster-internal IP address (note it for the `curl` step) |
| **NODE** | Which node the Pod was scheduled on |
| **NOMINATED NODE** | Used during preemption scheduling (usually `<none>`) |
| **READINESS GATES** | Extra conditions for readiness (usually `<none>`) |

> Your Pod IP will differ from the example. Use the one shown on your machine.

### 7.6 `minikube ssh`

```bash
minikube ssh
```

**What it does:** opens a shell **inside the Minikube node** (the VM or container that is acting as your Kubernetes node).

```
docker@minikube:~$
```

**Why it is needed:** the Pod IP (e.g. `10.244.0.3`) lives on the **cluster's internal network**. From inside the node you can reach it directly, which is exactly how you test the nginx Pod.

Type `exit` to leave the node shell.

### 7.7 `curl <Pod IP>`

Run this **inside** the node (after `minikube ssh`):

```bash
curl 10.244.0.3
```

**Expected output (nginx welcome page):**

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and working.</p>
...
</html>
```

This confirms the Pod is running and serving traffic on port 80.

**Why does it only work from inside the node?**
Pod IPs are **cluster-internal** and are not routable from your laptop (especially with the Docker driver on macOS and Windows). To reach the Pod from your own machine, use one of:

```bash
# Option 1: port-forward (quick testing)
kubectl port-forward pod/nginx 8080:80
# then open http://localhost:8080 in your browser

# Option 2: expose via a Service (covered in a later day)
kubectl expose pod nginx --type=NodePort --port=80
minikube service nginx --url
```

### 7.8 `kubectl describe pod nginx`

```bash
kubectl describe pod nginx
```

**What it does:** shows **detailed information** about the Pod, including its configuration, current state, and an **Events** log. It is the **first command to run when troubleshooting**.

**Sample output (trimmed):**

```
Name:             nginx
Namespace:        default
Node:             minikube/192.168.49.2
Start Time:       Tue, 07 Oct 2026 10:15:00 +0100
Labels:           app=nginx
Status:           Running
IP:               10.244.0.3
Containers:
  nginx:
    Container ID:   docker://abc123...
    Image:          nginx:latest
    Port:           80/TCP
    State:          Running
      Started:      Tue, 07 Oct 2026 10:15:12 +0100
    Ready:          True
    Restart Count:  0
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  1m    default-scheduler  Successfully assigned default/nginx to minikube
  Normal  Pulling    1m    kubelet            Pulling image "nginx:latest"
  Normal  Pulled     50s   kubelet            Successfully pulled image "nginx:latest"
  Normal  Created    50s   kubelet            Created container nginx
  Normal  Started    50s   kubelet            Started container nginx
```

| Section | What to look for |
|---|---|
| **Node / IP** | Where it runs and its IP |
| **Containers → State** | `Running`, `Waiting` (with reason), or `Terminated` (with exit code) |
| **Conditions** | Scheduling and readiness status |
| **Events** | Step-by-step history; errors such as `Failed to pull image`, `FailedScheduling`, `BackOff` appear here |

Related troubleshooting commands:

```bash
kubectl logs nginx            # container logs
kubectl logs nginx -f         # follow logs
kubectl exec -it nginx -- /bin/bash   # shell inside the container
```

### 7.9 `kubectl delete pod nginx`

```bash
kubectl delete pod nginx
```

```
pod "nginx" deleted
```

**What it does:** deletes the Pod. Kubernetes sends `SIGTERM` to the containers, waits for the **grace period** (30 seconds by default), then sends `SIGKILL` if needed.

Other ways to delete:

```bash
kubectl delete -f pod.yml              # delete using the manifest file
kubectl delete pod nginx --force --grace-period=0   # force immediately (use with care)
```

> **Key point:** because this was a **standalone Pod** (not managed by a Deployment or ReplicaSet), once deleted it **does not come back**. If it had been managed by a Deployment, Kubernetes would automatically create a replacement. This demonstrates why bare Pods are not used in production.

### 7.10 The Complete Practice Flow

```bash
# 1. Start the cluster
minikube start --driver=docker

# 2. Verify the node
kubectl get nodes

# 3. Create pod.yml (see section 6.2), then create the Pod
kubectl create -f pod.yml

# 4. Check the Pod
kubectl get pods
kubectl get pods -o wide        # note the IP

# 5. Test the Pod from inside the node
minikube ssh
curl <POD-IP>
exit

# 6. Inspect the Pod
kubectl describe pod nginx

# 7. Clean up
kubectl delete pod nginx
kubectl get pods                # confirm it is gone

# 8. (Optional) stop or delete the cluster
minikube stop
minikube delete
```

---

## 8. Quick Cheat Sheet

| Command | Purpose |
|---|---|
| `minikube start` | Start a local Kubernetes cluster |
| `minikube status` | Check cluster status |
| `minikube ssh` | Shell into the Minikube node |
| `minikube stop` / `minikube delete` | Stop / remove the cluster |
| `kubectl get nodes` | List cluster nodes |
| `kubectl create -f pod.yml` | Create a Pod from a manifest |
| `kubectl apply -f pod.yml` | Create or update from a manifest |
| `kubectl get pods` | List Pods |
| `kubectl get pods -o wide` | List Pods with IP and node |
| `kubectl describe pod <name>` | Detailed Pod information and events |
| `kubectl logs <name>` | View container logs |
| `kubectl exec -it <name> -- bash` | Shell inside a container |
| `kubectl port-forward pod/<name> 8080:80` | Access a Pod from localhost |
| `kubectl delete pod <name>` | Delete a Pod |
| `kubectl delete -f pod.yml` | Delete resources defined in a file |

### Key Takeaways

- **EKS / AKS / GKE** = managed control plane. **OpenShift** = enterprise Kubernetes platform. **KOps** = DIY cluster creation on AWS.
- A **Pod wraps one or more containers** and gives them a shared IP, storage and lifecycle.
- Pods are **ephemeral**; use Deployments in real environments.
- **Minikube** = single-node local cluster for learning; **kubectl** = the client for any cluster.
- Docker is **imperative** (`docker run`); Kubernetes is **declarative** (YAML describing desired state).
