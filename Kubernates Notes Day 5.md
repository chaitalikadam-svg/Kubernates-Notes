# Kubernetes Notes: Day 5: Interview Questions

---

## Q1. What is the difference between Docker and Kubernetes?

**Answer:** Docker is a tool to **build and run containers** on a single host. Kubernetes is a **container orchestration platform** that manages containers across a **cluster** of machines.

| | Docker | Kubernetes |
|---|---|---|
| Purpose | Build, package, run containers | Orchestrate containers at scale |
| Scope | Single host | Cluster of nodes |
| Self-healing | Basic restart only | Yes (recreates failed Pods) |
| Scaling | Manual | Automatic (HPA) |
| Load balancing | Manual | Built in (Services) |
| Updates | Manual | Rolling updates and rollback |

**One-liner:** Docker creates and runs the containers, Kubernetes manages them in production. They work together, they are not competitors.

---

## Q2. What are the main components of Kubernetes architecture?

**Answer:** A cluster has a **Control Plane** (the brain) and **Data Plane** (run the apps).

```
        CONTROL PLANE                           WORKER NODE
+--------------------------------+      +---------------------------+
| kube-apiserver  (front door)   |<---->| kubelet   (node agent)    |
| etcd            (cluster DB)   |      | kube-proxy (networking)   |
| kube-scheduler  (picks node)   |      | container runtime         |
| controller-manager (keeps      |      |   +-------+  +-------+    |
|   desired state)               |      |   | Pod   |  | Pod   |    |
| cloud-controller-manager       |      |   +-------+  +-------+    |
+--------------------------------+      +---------------------------+
```

| Component | Role |
|---|---|
| **kube-apiserver** | Entry point for all requests; the only component that talks to etcd |
| **etcd** | Key-value store holding all cluster state |
| **kube-scheduler** | Decides which node a new Pod runs on |
| **controller-manager** | Runs control loops to match actual state to desired state |
| **cloud-controller-manager** | Talks to the cloud provider (load balancers, nodes) |
| **kubelet** | Agent on each node that runs and monitors Pods |
| **kube-proxy** | Handles Service networking and load balancing on each node |
| **Container runtime** | Pulls images and runs containers (containerd, CRI-O) |

---

## Q3. What is the difference between Docker Swarm and Kubernetes?

**Answer:** Both are container orchestrators. **Docker Swarm** is simpler and built into Docker. **Kubernetes** is more powerful, has a much larger ecosystem, and is the industry standard.

| | Docker Swarm | Kubernetes |
|---|---|---|
| Setup | Very easy | More complex |
| Learning curve | Low | Steep |
| Scaling | Manual or scripted | Auto-scaling (HPA, VPA, Cluster Autoscaler) |
| Self-healing | Basic | Advanced (probes, controllers) |
| Rolling updates / rollback | Basic | Advanced, with rollback |
| Load balancing | Built-in, simple | Services and Ingress |
| Storage | Limited | PV, PVC, StorageClass |
| Ecosystem | Small | Huge (Helm, Operators, CNCF tools) |
| Managed cloud services | Few | EKS, AKS, GKE |
| Best for | Small, simple setups | Large, production-grade systems |

---

## Q4. What is the difference between a Docker container and a Kubernetes Pod?

**Answer:** A container runs one application. A **Pod wraps one or more containers** and is the smallest unit Kubernetes deploys.

```
+----------------- POD (one IP) -----------------+
|  +-------------+      +-------------+          |
|  | Container 1 |      | Container 2 |          |
|  | (app)       |      | (sidecar)   |          |
|  +-------------+      +-------------+          |
|  Shared: network (localhost), volumes          |
+------------------------------------------------+
```

| | Container | Pod |
|---|---|---|
| What it is | Running instance of an image | Wrapper around one or more containers |
| IP address | Docker assigns per container | **One IP per Pod**, shared by its containers |
| Managed by | Docker / container runtime | Kubernetes |
| Shared resources | Isolated | Containers share network and volumes |
| Lifecycle | Independent | Containers start, stop and schedule together |
| Self-healing | No | Only through a Deployment/ReplicaSet |

**Note:** Pods are temporary. In production, use a **Deployment** to manage them, not bare Pods.

---

## Q5. What is a Namespace in Kubernetes?

**Answer:** A **namespace** is a way to **divide one cluster into virtual sub-clusters**, so teams, projects or environments (dev, test, prod) can be isolated within the same cluster.

```
                 ONE CLUSTER
+--------------+  +--------------+  +--------------+
|  Namespace   |  |  Namespace   |  |  Namespace   |
|     dev      |  |     test     |  |     prod     |
|  Pods, Svc   |  |  Pods, Svc   |  |  Pods, Svc   |
+--------------+  +--------------+  +--------------+
```

**Default namespaces:**

| Namespace | Purpose |
|---|---|
| `default` | Where objects go if no namespace is given |
| `kube-system` | Kubernetes system components (CoreDNS, kube-proxy, etc.) |
| `kube-public` | Publicly readable data |
| `kube-node-lease` | Node heartbeat (lease) objects |

**Benefits:** isolation, resource quotas per team, access control (RBAC), and the same names can be reused in different namespaces.

```bash
kubectl get namespaces
kubectl create namespace dev
kubectl get pods -n dev
kubectl get pods -A                 # all namespaces
```

**Note:** Namespaces separate resources logically, not strictly on the network. Network isolation needs **NetworkPolicies**.

---

## Q6. What is the role of kube-proxy?

**Answer:** kube-proxy runs on **every node** and implements the **Service** networking. It maintains rules (iptables or IPVS) that forward traffic sent to a Service's ClusterIP to one of the healthy backing Pods, which gives **load balancing and service discovery**.

```
Client --> Service IP (10.96.45.12:80)
                |   kube-proxy rules on the node
                +--> Pod 1
                +--> Pod 2
                +--> Pod 3
```

- Watches the API server for Service and Endpoint changes
- Updates the node's network rules automatically when Pods change
- Does **not** handle traffic itself in modern modes; it only programs the rules

---

## Q7. What are the different types of Services in Kubernetes?

| Type | Access | Use |
|---|---|---|
| **ClusterIP** (default) | Inside the cluster only | Internal communication |
| **NodePort** | `<NodeIP>:30000-32767` | Dev, testing |
| **LoadBalancer** | Public IP via cloud load balancer | Production, internet-facing |

```
LoadBalancer  --includes-->  NodePort  --includes-->  ClusterIP
```

---

## Q8. What is the role of kubelet?

**Answer:** The kubelet is the **agent running on every worker node**. It makes sure the containers described in the Pod specs are **running and healthy** on that node.

**Responsibilities:**

- Registers the node with the API server
- Watches the API server for Pods assigned to its node
- Tells the container runtime to start and stop containers
- Runs **liveness and readiness probes** and restarts failed containers
- Reports Pod and node status back to the API server

```
API Server --"run this Pod"--> kubelet --> container runtime --> containers
API Server <--"status report"-- kubelet
```

The kubelet only manages containers created by Kubernetes.

---

## Q9. What are your day-to-day activities on Kubernetes?

| Area | Activities |
|---|---|
| **Deployments** | Write and apply YAML, perform rolling updates, rollbacks, scale replicas |
| **Monitoring** | Check Pod, node and cluster health (`kubectl get`, `top`), use Prometheus and Grafana |
| **Troubleshooting** | Debug `CrashLoopBackOff`, `ImagePullBackOff`, `Pending` Pods using `describe` and `logs` |
| **Networking** | Manage Services, Ingress and DNS issues |
| **Config and secrets** | Manage ConfigMaps and Secrets |
| **Security** | RBAC, namespaces, network policies, image scanning |
| **CI/CD** | Maintain pipelines that build images and deploy to the cluster (Jenkins, GitHub Actions, ArgoCD) |
| **Capacity** | Set resource requests and limits, autoscaling, node management |
| **Maintenance** | Cluster upgrades, node drain and cordon, etcd backups |
| **Cost and reliability** | Right-size workloads, set up alerts, incident response |


## Quick Revision

- **Docker** = build and run containers. **Kubernetes** = orchestrate them across a cluster.
- **Control plane:** apiserver, etcd, scheduler, controller-manager. **Node:** kubelet, kube-proxy, runtime.
- **Swarm** = simple. **Kubernetes** = powerful, industry standard.
- **Pod** wraps containers and shares one IP.
- **Namespace** = virtual cluster for isolation.
- **kube-proxy** = Service routing rules. **kubelet** = runs and monitors Pods on the node.
- **Services:** ClusterIP, NodePort, LoadBalancer, ExternalName, Headless.
