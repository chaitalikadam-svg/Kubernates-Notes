# Kubernetes Notes: Day 6: Deploy NodePort and LoadBalancer Services, Kubeshark Basics

## Part A: Deploy a NodePort Service (Step by Step)

**Goal:** run nginx with 3 replicas and reach it from outside the cluster using `<NodeIP>:30007`.

```
Browser --> NodeIP:30007 --> Service:80 --> Pod:80 (x3)
```

### Step 1: Start the cluster

```bash
minikube start --driver=docker
kubectl get nodes
```

### Step 2: Create the Deployment

`deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
          image: nginx:1.25
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment.yml
kubectl get pods -o wide
```

### Step 3: Create the NodePort Service

`service-nodeport.yml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx          # must match Pod labels
  ports:
    - port: 80          # Service port
      targetPort: 80    # container port
      nodePort: 30007   # node port (30000-32767)
```

```bash
kubectl apply -f service-nodeport.yml
kubectl get svc nginx-nodeport
```

```
NAME             TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-nodeport   NodePort   10.96.78.101   <none>        80:30007/TCP   8s
```

### Important: What Must Match Between deployment.yml and service.yml

The Service finds Pods **only by label**. If the label in the Service does not exactly match the label on the Pods, the Service will have **no Pods** and nothing will work.

```
deployment.yml                              service-nodeport.yml
---------------                             ---------------------
spec:                                       spec:
  selector:                                   selector:
    matchLabels:                                app: nginx      <-- MUST be the same
      app: nginx          <-- same ----+      ports:
  template:                            |        - port: 80
    metadata:                          |          targetPort: 80   <-- = containerPort
      labels:                          |
        app: nginx        <-- same ----+      (Service looks for Pods with app=nginx)
    spec:
      containers:
        - name: nginx
          ports:
            - containerPort: 80   <-- targetPort must be this
```

| In deployment.yml | In service.yml | Rule |
|---|---|---|
| `template.metadata.labels` (`app: nginx`) | `spec.selector` (`app: nginx`) | **Must be identical** (key and value, case-sensitive) |
| `spec.selector.matchLabels` | n/a | Must match `template.metadata.labels` in the same file |
| `containerPort: 80` | `targetPort: 80` | Must be the same port |
| n/a | `metadata.name` (`nginx-nodeport`) | Free to choose, this is the Service name used for DNS and `kubectl` commands |

**Common mistakes:**

- The Service selector is `app: nginx-app` but the Pods are labelled `app: nginx`. Result: no endpoints.
- Labels on the **Deployment's own `metadata.labels`** or the Service's own `metadata.labels` are **not** used for matching. Only the **Pod template labels** matter.
- `targetPort: 8080` while the container listens on `80`. Result: connection refused.

**To use a different label name (for example `app: myweb`)**, change it in all three places:

```yaml
# deployment.yml
spec:
  selector:
    matchLabels:
      app: myweb          # 1
  template:
    metadata:
      labels:
        app: myweb        # 2

# service.yml
spec:
  selector:
    app: myweb            # 3
```

**How to check:**

```bash
kubectl get pods --show-labels         # labels on the Pods
kubectl describe svc nginx-nodeport    # shows the Selector it is using
kubectl get endpoints nginx-nodeport   # must list Pod IPs, not <none>
```

The same rule applies to the LoadBalancer Service in Part B.

### Step 4: Verify the Service found the Pods

```bash
kubectl get endpoints nginx-nodeport     # should list 3 Pod IPs
```

### Step 5: Access the application

```bash
minikube service nginx-nodeport --url    # prints a URL, open it in the browser
```

Alternative (from inside the node):

```bash
minikube ip
minikube ssh
curl localhost:30007
```

> With the Docker driver on macOS/Windows, the node IP is not reachable directly, so use `minikube service`. Keep that terminal open.

### Step 6: Clean up

```bash
kubectl delete svc nginx-nodeport
```

---

## Part B: Deploy a LoadBalancer Service (Step by Step)

**Goal:** expose the same nginx Deployment through a load balancer with an external IP.

```
Internet --> Load Balancer (External IP) --> NodePort --> Service --> Pods
```

### Step 1: Make sure the Deployment is running

```bash
kubectl get deployment nginx-deployment
```

### Step 2: Create the LoadBalancer Service

`service-loadbalancer.yml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f service-loadbalancer.yml
kubectl get svc nginx-lb
```

### Step 3: Get the external IP

**On a cloud cluster (EKS / AKS / GKE):** the cloud creates a load balancer. `EXTERNAL-IP` shows `<pending>` for a minute, then a public IP or DNS name.

```bash
kubectl get svc nginx-lb -w       # -w watches until the IP appears
```

**On Minikube:** there is no cloud, so `EXTERNAL-IP` stays `<pending>`. Run a tunnel in a **separate terminal** and keep it open:

```bash
minikube tunnel
```

Then:

```bash
kubectl get svc nginx-lb
```

```
NAME       TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-lb   LoadBalancer   10.96.12.55    127.0.0.1     80:31456/TCP   2m
```

### Step 4: Access the application

Open `http://<EXTERNAL-IP>` in the browser (on Minikube: `http://127.0.0.1`).

### Step 5: Clean up

```bash
kubectl delete svc nginx-lb
kubectl delete deployment nginx-deployment
```

> **Cost warning:** on a real cloud, each LoadBalancer Service creates a paid cloud load balancer. Delete it when you are done.

### NodePort vs LoadBalancer (Quick Recap)

| | NodePort | LoadBalancer |
|---|---|---|
| Access | `<NodeIP>:30007` | `<External IP>:80` |
| Needs cloud | No | Yes (Minikube uses `minikube tunnel`) |
| Port | 30000-32767 | Any (80, 443) |
| Use | Dev, test | Production |

### Common Problems

| Problem | Fix |
|---|---|
| `ENDPOINTS` is `<none>` | Service selector does not match Pod labels |
| Cannot open NodePort URL | Use `minikube service <name> --url` |
| `EXTERNAL-IP` stuck on `<pending>` | Run `minikube tunnel` (Minikube), or check cloud permissions |

---

## Part C: Kubeshark Basics

### What is Kubeshark?

**Kubeshark** is an open-source **network observability / traffic analyzer for Kubernetes**. It is often described as **"Wireshark / tcpdump for Kubernetes"**.

It captures the traffic going in, out and between Pods, nodes and services, and shows it in a **real-time web dashboard** with Kubernetes context (which Pod, Service, Namespace).

### Why use it?

| Use case | Example |
|---|---|
| **Debugging** | See the exact request and response between two services |
| **Troubleshooting** | Find failing API calls, slow responses, retries |
| **Learning** | Watch how a Service load balances across Pods |
| **Security** | Spot suspicious or unexpected connections |

You can see **HTTP, gRPC, DNS, Redis, Kafka** and other protocols, with request and response details.

### Components

```
+-------------------- Kubernetes Cluster --------------------+
|                                                            |
|   Node 1                 Node 2                            |
|  +----------+           +----------+                       |
|  | Worker   |           | Worker   |   captures traffic    |
|  +----+-----+           +----+-----+   on each node        |
|       |                      |                             |
|       +----------+-----------+                             |
|                  v                                         |
|             +---------+      +----------+                  |
|             |   Hub   | ---> |  Front   | --> Dashboard    |
|             +---------+      | (web UI) |    (browser)     |
|          collects and        +----------+                  |
|          processes data                                   |
+------------------------------------------------------------+
```

| Component | Role |
|---|---|
| **Worker** | Runs on every node, captures network traffic |
| **Hub** | Collects and processes the captured data |
| **Front** | Web dashboard you open in the browser |

### Install Kubeshark (using kubectl)

Kubeshark publishes a ready-made manifest with every release, so only `kubectl` is needed.

```bash
# 1. Pick a release tag from https://github.com/kubeshark/kubeshark/tags (example below)
export TAG=v52.3.92

# 2. Install
kubectl apply -f https://raw.githubusercontent.com/kubeshark/kubeshark/refs/tags/$TAG/manifests/complete.yaml

# 3. Check that the Pods are running
kubectl get pods

# 4. Open the dashboard (keep this terminal open)
kubectl port-forward service/kubeshark-front 8899:80
```

Open `http://localhost:8899` in the browser.

Remove it when finished:

```bash
kubectl delete -f https://raw.githubusercontent.com/kubeshark/kubeshark/refs/tags/$TAG/manifests/complete.yaml
```

> Use the latest tag from the releases page instead of the example tag.

### Try It with Our nginx Service

1. Deploy nginx and a Service (Part A or B).
2. Install Kubeshark and open the dashboard.
3. Generate traffic:

```bash
kubectl run test --rm -it --image=busybox:1.36 --restart=Never -- \
  sh -c 'for i in 1 2 3 4 5; do wget -qO- http://nginx-nodeport > /dev/null; done'
```

4. In the dashboard you will see the HTTP `GET /` requests from the `test` Pod to the nginx Pods, with the source, destination, status code and response. You can see requests being distributed across different nginx Pods, which shows the Service load balancing.

### Things to Know

- Capturing traffic needs **elevated privileges** on nodes, so use it on **dev/test clusters** first.
- Captured traffic can contain **sensitive data**, so be careful in production.
- It adds some **CPU and memory overhead**, which can be noticeable on a small Minikube. Give Minikube more resources if needed (`minikube start --cpus=4 --memory=6g`).
- It is a **debugging and observability tool**, not a replacement for Prometheus and Grafana (metrics) or logging tools.

---

## Quick Revision

- **NodePort:** `<NodeIP>:30007`, test with `minikube service <name> --url`.
- **LoadBalancer:** external IP from the cloud, on Minikube run `minikube tunnel`.
- Service `selector` must match the Pod template `labels`, and `targetPort` must match `containerPort`.
- Always check `kubectl get endpoints <svc>` to confirm the Service found its Pods.
- **Kubeshark** = Wireshark for Kubernetes. Components: **Worker** (captures), **Hub** (processes), **Front** (dashboard).
- Install with `kubectl apply -f <release manifest>`, then `kubectl port-forward service/kubeshark-front 8899:80` and open `localhost:8899`.
