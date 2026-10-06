# Kubernetes Notes

## 1. What is Kubernetes?

**Kubernetes (K8s)** is an open-source **container orchestration platform** originally developed by Google (based on their internal system *Borg*) and now maintained by the **Cloud Native Computing Foundation (CNCF)**.

It automates the **deployment, scaling, networking, and management** of containerized applications across a **cluster of machines** (nodes).

> The name comes from the Greek word for "helmsman / pilot". "K8s" = K + 8 letters + s.

**What Kubernetes does:**

- Schedules containers onto the right machines
- Restarts failed containers automatically
- Scales applications up and down
- Rolls out updates with zero downtime and rolls back on failure
- Provides service discovery and load balancing
- Manages configuration and secrets
- Manages persistent storage

---

## 2. Kubernetes vs Docker

Docker and Kubernetes are **not competitors**. They solve different problems and work together.

| Aspect | Docker | Kubernetes |
|---|---|---|
| **Purpose** | Build, package and run containers | Orchestrate and manage containers at scale |
| **Scope** | Single host (one machine) | Cluster of many machines |
| **Unit of work** | Container | Pod (one or more containers) |
| **Scaling** | Manual (`docker run` repeatedly) | Automatic (HPA, VPA, Cluster Autoscaler) |
| **Self-healing** | Limited (restart policies only) | Built in (reschedules pods, replaces nodes) |
| **Load balancing** | Basic, manual setup | Built-in Services, Ingress |
| **Rolling updates / rollback** | Manual scripting | Native (Deployments) |
| **Networking** | Host-level bridge networks | Cluster-wide flat network, DNS-based discovery |
| **Storage** | Volumes on one host | PersistentVolumes, StorageClasses, dynamic provisioning |
| **Config definition** | Dockerfile, `docker-compose.yml` | YAML manifests, Helm charts |

**Analogy:** Docker is the **shipping container**. Kubernetes is the **port authority** that decides where containers go, tracks them, replaces damaged ones, and handles traffic.

**How they work together:**

```
Developer --> Dockerfile --> Docker image --> Registry --> Kubernetes pulls image --> runs as Pod
```

> Note: Modern Kubernetes no longer uses the Docker Engine directly as its runtime (`dockershim` was removed in v1.24). It uses CRI-compatible runtimes such as **containerd** or **CRI-O**. Docker-built images still work perfectly because they follow the OCI standard.

---

## 3. Why Docker Alone Is Not Enough at Production Level

Docker is excellent for building and running containers. The limitations below apply to **running production workloads with standalone Docker and no orchestrator**.

### Reason 1: Single Host

- Docker runs containers on **one machine**.
- If that host goes down (hardware failure, OS crash, maintenance), **every container on it goes down**.
- It is a **single point of failure**, and there is no built-in way to spread workloads across multiple machines.
- Kubernetes spreads pods across many nodes and reschedules them if a node dies.

### Reason 2: No Auto-Healing

- Docker only offers basic restart policies (`--restart=always`).
- It cannot detect an unhealthy application (running but not responding), and it cannot move a container to another host if the host fails.
- Kubernetes uses **liveness, readiness and startup probes**, and its controllers continuously compare desired state with actual state. A failed pod is **killed and recreated** automatically, and pods from a dead node are **rescheduled** to healthy nodes.

### Reason 3: No Auto-Scaling

- With Docker you scale by manually running more containers.
- There is no built-in reaction to traffic spikes, and no automatic scale-down to save cost.
- Kubernetes provides:
  - **HPA** (Horizontal Pod Autoscaler): scales pod count on CPU, memory or custom metrics
  - **VPA** (Vertical Pod Autoscaler): adjusts pod resource requests/limits
  - **Cluster Autoscaler**: adds or removes nodes

### Reason 4: Lacks Enterprise-Level Features

Production environments need capabilities that standalone Docker does not provide:

| Enterprise need | Kubernetes feature |
|---|---|
| Zero-downtime deployments | Rolling updates, rollbacks, canary / blue-green strategies |
| Service discovery and load balancing | Services, Ingress, CoreDNS |
| Secrets and config management | ConfigMaps, Secrets |
| Access control and security | RBAC, Network Policies, Pod Security Standards |
| Resource governance | Namespaces, ResourceQuotas, LimitRanges |
| Persistent storage | PV, PVC, StorageClass |
| Observability | Metrics Server, Prometheus/Grafana integration, logging |
| Extensibility | CRDs, Operators, Helm |
| Multi-cloud / hybrid portability | Runs on AWS (EKS), Azure (AKS), GCP (GKE), on-prem |

### Summary

| Requirement | Docker (standalone) | Kubernetes |
|---|---|---|
| Multi-host | No | Yes |
| Auto-healing | Basic restart only | Yes |
| Auto-scaling | No | Yes (HPA / VPA / CA) |
| Enterprise features | Limited | Extensive |

> **Nuance:** Docker containers *are* used in production everywhere. The point is that running them in production **at scale and with high availability** needs an orchestrator such as Kubernetes (or Docker Swarm, Nomad, ECS).

---

## 4. Kubernetes Architecture

A Kubernetes **cluster** has two parts:

1. **Control Plane** (master): the brain that makes decisions
2. **Worker Nodes**: the machines that run the application workloads

### 4.1 Architecture Diagram
![Architecture Diagram](architecture.png)



---

### 4.2 Control Plane Components

#### 1. kube-apiserver

- The **front door** of the cluster; every request goes through it (`kubectl`, other components, external tools).
- Exposes the **Kubernetes REST API**.
- Responsibilities: authentication, authorization (RBAC), admission control, validating requests, and reading/writing state to etcd.
- It is the **only component that talks directly to etcd**.
- Stateless and horizontally scalable (run multiple replicas behind a load balancer for HA).

#### 2. etcd

- A **distributed, consistent key-value store** that holds the **entire cluster state**: nodes, pods, configs, secrets, and so on.
- Uses the **Raft consensus algorithm**, so it needs a quorum (typically 3 or 5 members in production).
- The source of truth. **Losing etcd without a backup means losing the cluster state**, so regular snapshots are essential.

#### 3. kube-scheduler

- Watches for **newly created pods with no node assigned** and picks the best node.
- Two phases:
  1. **Filtering**: removes nodes that cannot run the pod (insufficient CPU/memory, taints, node selectors, affinity rules).
  2. **Scoring**: ranks the remaining nodes (resource balance, spreading, locality).
- It only **decides** the placement (writes the node name to the pod). The kubelet does the actual running.

#### 4. kube-controller-manager

- Runs a set of **controllers**, each a control loop that watches the cluster state and moves **actual state toward desired state**.
- Examples:
  - **Node controller**: detects and responds when nodes go down
  - **ReplicaSet / Deployment controller**: keeps the correct number of pods running
  - **Job controller**: runs one-off tasks to completion
  - **Endpoints / EndpointSlice controller**: links Services to pods
  - **ServiceAccount & token controller**: creates default accounts and tokens

#### 5. cloud-controller-manager

- Connects the cluster to the **cloud provider's API** (AWS, Azure, GCP).
- Handles cloud-specific tasks: creating **load balancers** for `type: LoadBalancer` Services, managing node lifecycle in the cloud, and configuring routes and storage volumes.
- Not present in on-prem or local clusters (like Minikube) unless configured.

---

### 4.3 Worker Node Components

#### 1. kubelet

- The **agent running on every node**.
- Registers the node with the API server and **watches for pods assigned to its node**.
- Instructs the container runtime to start/stop containers, runs **health probes** (liveness, readiness, startup), and **reports pod and node status** back to the API server.
- It only manages containers created by Kubernetes.

#### 2. kube-proxy

- A **network proxy** on every node that implements the **Service** abstraction.
- Maintains network rules (using **iptables** or **IPVS**) so traffic sent to a Service's virtual IP is forwarded to one of the healthy backing pods, giving **load balancing and service discovery**.

#### 3. Container Runtime

- The software that **actually runs containers**: pulls images, creates and starts containers.
- Must implement the **CRI (Container Runtime Interface)**.
- Common runtimes: **containerd**, **CRI-O** (Docker Engine is no longer used directly since v1.24).

#### 4. Pods (the workload unit)

- The **smallest deployable unit** in Kubernetes: one or more containers that share the **same network namespace (IP address), storage volumes, and lifecycle**.
- Pods are **ephemeral**; when they die they are replaced, not repaired, so higher-level objects (Deployments, StatefulSets) manage them.

