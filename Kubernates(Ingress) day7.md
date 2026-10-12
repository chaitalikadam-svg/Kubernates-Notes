# Kubernetes Notes: Day 7: Ingress

## 1. What is Ingress?

**Ingress** is a Kubernetes object that defines **rules to route external HTTP/HTTPS traffic to Services** inside the cluster, using the **hostname** and **URL path**.

Think of it as a **smart receptionist at the front door**: one entrance, and it sends each visitor to the right room.

```
                         INTERNET
                            |
                  +---------v----------+
                  |  Ingress Controller|   (one entry point, one IP)
                  |  (reads Ingress    |
                  |   rules)           |
                  +----+----------+----+
        foo.com/first  |          |  foo.com/second
                       v          v
                 +-----------+ +-----------+
                 | http-svc  | | meow-svc  |   (Services, ClusterIP)
                 +-----+-----+ +-----+-----+
                       v             v
                    Pods           Pods
```

### Without vs With Ingress

| Without Ingress | With Ingress |
|---|---|
| One LoadBalancer per Service (many public IPs, high cost) | **One** entry point for many Services |
| Or NodePort with odd ports (30000+) | Standard ports 80/443 |
| No host or path routing | Host-based and path-based routing |
| TLS configured per app | Central TLS/SSL handling |

### Uses of Ingress

- **Host-based and path-based routing** to many Services through one IP
- **TLS/SSL termination** (HTTPS) in one place
- **Cost saving**: one load balancer instead of many
- **Load balancing** across Pods
- Clean URLs on ports 80/443

> Ingress works at **Layer 7 (HTTP/HTTPS)**. For raw TCP/UDP, use a Service of type LoadBalancer/NodePort.

---

## 2. Why Do We Need an Ingress Controller?

An **Ingress** is **only a set of rules** (a YAML object). It does nothing by itself. An **Ingress Controller** is the program that **reads those rules and actually routes the traffic**.

| | Ingress | Ingress Controller |
|---|---|---|
| What | Routing rules (YAML) | A running reverse proxy / load balancer (Pods) |
| Does work? | No | Yes |
| Comes with Kubernetes? | Yes (API object) | **No, you must install one** |

```
Ingress (rules)  --watched by-->  Ingress Controller  --routes-->  Services --> Pods
```

**Popular controllers:** NGINX, Traefik, HAProxy, AWS ALB Controller, GKE Ingress, Istio/Envoy-based.

The controller itself is usually exposed outside the cluster with a **LoadBalancer** (or NodePort) Service. That is the one public entry point.

**Minikube:**

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```


---

## 3. Sample ingress.yaml


### 3.1 Host-Based Rule 
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-auth
spec:
  ingressClassName: nginx        # which controller handles this (recommended)
  rules:
    - host: "*.bar.com"          # wildcard host
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: http-svc
                port:
                  number: 80
```

### 3.2 Path-Based Rule

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-auth
spec:
  ingressClassName: nginx
  rules:
    - host: foo.bar.com
      http:
        paths:
          - path: /first
            pathType: Prefix
            backend:
              service:
                name: http-svc
                port:
                  number: 80
          - path: /second
            pathType: Prefix
            backend:
              service:
                name: meow-svc
                port:
                  number: 80
```


### 3.3 Field Explanation

| Field | Meaning |
|---|---|
| `apiVersion: networking.k8s.io/v1` | API group for Ingress |
| `kind: Ingress` | Object type |
| `metadata.name` | Name of the Ingress |
| `ingressClassName` | Which Ingress Controller should handle it |
| `rules` | List of routing rules |
| `host` | Hostname to match (e.g. `foo.bar.com`, or `*.bar.com`) |
| `path` | URL path to match (`/first`) |
| `pathType: Prefix` | Match any URL that **starts with** the path (`/first`, `/first/abc`). `Exact` matches only the exact path |
| `backend.service.name` | **Service** to send traffic to (must exist in the same namespace) |
| `backend.service.port.number` | Service **port** (the Service's `port`, not the Pod's) |

**Name matching:** `backend.service.name` must equal the Service's `metadata.name`, and the port must be a port the Service exposes. Ingress routes to **Services**, never directly to Pods.

### 3.4 Commands

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress ingress-with-auth

# Test (replace <IP> with the ADDRESS from `kubectl get ingress`)
curl -H "Host: foo.bar.com" http://<IP>/first
curl -H "Host: foo.bar.com" http://<IP>/second
```

---

## 4. Types of Ingress Routing

### 4.1 Path-Based (Fan-out)

**One host, different paths go to different Services.**

```
                       foo.bar.com
                            |
               +------------+------------+
               v                         v
     foo.bar.com/first         foo.bar.com/second
               |                         |
           http-svc                  meow-svc
```

Use when one domain hosts several apps or microservices (`/api`, `/shop`, `/blog`).

### 4.2 Host-Based (Name-Based Virtual Hosting)

**Different hostnames go to different Services.**

```
        Same Ingress Controller IP
                    |
         +----------+-----------+
         v                      v
   app1.bar.com           app2.bar.com
         |                      |
      app1-svc               app2-svc
```

```yaml
spec:
  rules:
    - host: app1.bar.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app1-svc
                port:
                  number: 80
    - host: app2.bar.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app2-svc
                port:
                  number: 80
```

Use when each app has its own domain or subdomain.

| | Path-based | Host-based |
|---|---|---|
| Routes by | URL path | Hostname |
| Example | `foo.com/first` and `foo.com/second` | `app1.foo.com` and `app2.foo.com` |
| DNS | One name | One record per host (or wildcard) |

You can **combine both** (host plus paths).

---

## 5. TLS / SSL

There are three ways to handle HTTPS with an Ingress.

### 5.1 TLS Termination (Edge)

The Ingress Controller **decrypts HTTPS**, then sends **plain HTTP** to the Pods.

```
Client --HTTPS--> [Ingress Controller: decrypt] --HTTP--> Pod
```

- Certificate lives at the Ingress (stored in a Kubernetes **TLS Secret**)
- Simple and most common; the controller can inspect and route by host/path

```bash
kubectl create secret tls foo-tls --cert=tls.crt --key=tls.key
```

```yaml
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - foo.bar.com
      secretName: foo-tls        # the TLS secret
  rules:
    - host: foo.bar.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: http-svc
                port:
                  number: 80
```

### 5.2 SSL Passthrough

The Ingress Controller **does not decrypt**. It forwards the **encrypted traffic** straight to the Pod, which holds the certificate.

```
Client --HTTPS--> [Ingress: forwards encrypted, TCP level] --HTTPS--> Pod (decrypts)
```

- **End-to-end encryption**; the controller never sees the data
- Because it cannot read the request, it **cannot route by path** (only by hostname via SNI), and cannot add headers
- Needs controller support. With NGINX: annotation `nginx.ingress.kubernetes.io/ssl-passthrough: "true"` and the controller started with `--enable-ssl-passthrough`

### 5.3 SSL Bridging (Re-encrypt)

The Ingress Controller **decrypts, inspects, then encrypts again** to the Pod.

```
Client --HTTPS--> [Ingress: decrypt, route, re-encrypt] --HTTPS--> Pod (decrypts again)
```

- Two certificates: one at the Ingress (public), one at the Pod (internal)
- Traffic is encrypted **everywhere**, and the controller can still route by **path and headers**
- With NGINX: annotation `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`

### 5.4 Comparison

| | Termination (Edge) | Passthrough | Bridging (Re-encrypt) |
|---|---|---|---|
| Decrypted at Ingress? | Yes | **No** | Yes, then re-encrypted |
| Ingress to Pod traffic | HTTP (plain) | HTTPS | HTTPS |
| Certificates | Ingress only | Pod only | Ingress and Pod |
| Path/header routing | Yes | **No** | Yes |
| Security | Good | Highest (end-to-end, no inspection) | High (inspection plus encryption) |
| Complexity | Low | Medium | Higher |

---

## 6. OpenShift Routes

**OpenShift Routes** are OpenShift's own way to expose a Service to the outside world. They existed **before** Kubernetes Ingress and are handled by OpenShift's built-in **router (HAProxy)**.

```
Internet --> OpenShift Router (HAProxy) --> Route rules --> Service --> Pods
```

| | Kubernetes Ingress | OpenShift Route |
|---|---|---|
| Controller | You install one | **Built-in** router (HAProxy) |
| Object | `Ingress` | `Route` |
| TLS options | Depends on controller and annotations | **Built into the Route spec** |
| Where | Any Kubernetes | OpenShift only |

OpenShift also **accepts Ingress objects** and automatically creates Routes from them.

**Route TLS types (same three ideas as above):**

| Route type | Same as | Behaviour |
|---|---|---|
| **edge** | TLS termination | Router decrypts, sends HTTP to Pod |
| **passthrough** | SSL passthrough | Router forwards encrypted traffic, Pod decrypts |
| **re-encrypt** | SSL bridging | Router decrypts, then re-encrypts to Pod |

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: http-route
spec:
  host: foo.bar.com
  to:
    kind: Service
    name: http-svc
  port:
    targetPort: 80
  tls:
    termination: edge        # or passthrough / reencrypt
```

```bash
oc expose service http-svc          # quick way to create a Route
oc get routes
```

---

## Quick Revision

- **Ingress** = HTTP/HTTPS routing rules to Services by host and path. **Ingress Controller** = the program that applies them (you must install it).
- One Ingress = one entry point for many Services, saving cost versus many LoadBalancers.
- **Path-based:** `foo.com/first`, `foo.com/second`. **Host-based:** `app1.foo.com`, `app2.foo.com`.
- `backend.service.name` must match the Service name; Ingress routes to Services, not Pods.
- **TLS termination** = decrypt at Ingress. **Passthrough** = Pod decrypts, no path routing. **Bridging (re-encrypt)** = decrypt, then encrypt again.
- **OpenShift Route** = built-in HAProxy router with edge, passthrough and re-encrypt types.
