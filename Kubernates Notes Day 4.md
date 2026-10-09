# Kubernetes Notes: Day 4 (Short): Services

## 1. Why Do We Need Services?

- Pod IPs are **temporary**. When a Pod dies and is replaced, the new Pod gets a **new IP**.
- A Deployment has **many Pods**, each with a different IP.
- Pod IPs are **not reachable from outside** the cluster.

**A Service gives a group of Pods one stable IP and DNS name**, and load balances traffic across them.

```
Client --> Service (fixed IP: 10.96.45.12)
                |--> Pod 1 (10.244.0.3)
                |--> Pod 2 (10.244.0.4)
                |--> Pod 3 (10.244.0.5)
```

## 2. Advantages

| Advantage | How |
|---|---|
| **Self-healing (stable access)** | Service tracks Pods by label. Replaced or scaled Pods are added automatically; unhealthy Pods are removed. The Deployment recreates Pods, the Service hides the change from clients. |
| **Load balancing** | Traffic is spread across all healthy Pods (done by kube-proxy). |
| **Exposing to the world** | NodePort or LoadBalancer types make the app reachable from outside. |
| **Service discovery** | Other apps reach it by name (e.g. `nginx-service`), not IP. |

## 3. How a Service Tracks Pods (Labels)

The Service uses a **selector** to find Pods with matching **labels**. Kubernetes keeps a list of matching Ready Pod IPs (**Endpoints**) and updates it automatically.

```
Service selector: app=nginx  --matches-->  Pods labelled app=nginx
```

```bash
kubectl get endpoints nginx-service    # shows the Pod IPs behind the Service
```

## 4. Service Discovery Mechanism

1. Client Pod asks **CoreDNS** for `nginx-service`.
2. CoreDNS returns the Service's **ClusterIP**.
3. Client connects to the ClusterIP.
4. **kube-proxy** forwards the request to one healthy Pod.

```
Client Pod --(1) DNS lookup--> CoreDNS
Client Pod <--(2) ClusterIP--- CoreDNS
Client Pod --(3) request-----> ClusterIP --(4) kube-proxy--> one of the Pods
```

DNS name format: `<service>.<namespace>.svc.cluster.local` (inside the same namespace, `nginx-service` is enough).

## 5. Service Types

| Type | Access | Use |
|---|---|---|
| **ClusterIP** (default) | Inside cluster only | Internal communication (frontend to backend, app to database) |
| **NodePort** | `<NodeIP>:<30000-32767>` | Dev, testing, demos |
| **LoadBalancer** | Public IP from cloud load balancer | Production internet-facing apps |

Each type includes the previous one: LoadBalancer > NodePort > ClusterIP.

**Port fields:** `port` = Service port, `targetPort` = container port, `nodePort` = port on each node.

### ClusterIP

**Why:** keeps internal services private and secure, with a stable address.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP        # default, can be omitted
  selector:
    app: nginx           # must match Pod labels
  ports:
    - port: 80           # Service port
      targetPort: 80     # container port
```

### NodePort

**Why:** simplest way to reach the app from outside, works on any cluster (no cloud needed).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30007    # optional, range 30000-32767
```

Access: `http://<NodeIP>:30007`

```
Browser --> NodeIP:30007 --> Service:80 --> Pod:80
```

### LoadBalancer

**Why:** the cloud provider creates a load balancer with one public IP/DNS and standard ports (80/443). Standard for production on AWS, Azure and GCP.

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

```
Internet --> Cloud Load Balancer --> Node (NodePort) --> Service --> Pods
```

## 6. Quick Comparison

| | ClusterIP | NodePort | LoadBalancer |
|---|---|---|---|
| Reachable from | Inside cluster | Outside via node IP | Outside via public IP |
| Needs cloud | No | No | Yes |
| Cost | Free | Free | Cloud LB charges |
| Best for | Internal apps | Dev/test | Production |

## 7. Commands

```bash
kubectl apply -f service.yml
kubectl get svc
kubectl describe svc nginx-service
kubectl get endpoints nginx-service

# Minikube
minikube service nginx-nodeport --url     # reach a NodePort service
minikube tunnel                           # gives LoadBalancer an external IP

kubectl delete svc nginx-service
```

## Key Takeaways

- Service = **stable IP + DNS name + load balancing** for changing Pods.
- It finds Pods using **label selectors**, and updates automatically when Pods change.
- **ClusterIP** = internal, **NodePort** = external via node port, **LoadBalancer** = external via cloud load balancer.
