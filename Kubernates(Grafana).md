# Kubernetes Notes: Day 12: Grafana and Kubernetes

## 1. What is Grafana?

**Grafana** is an open-source **dashboard and visualisation tool**. It does **not collect or store metrics**. It connects to **data sources** and shows their data as graphs, gauges, tables and alerts.

| Prometheus (Day 11) | Grafana |
|---|---|
| Collects and stores metrics | **Shows** metrics from many sources |
| Basic UI | Rich, customisable dashboards |
| The data collector | The TV screen |

**Why use Grafana with Kubernetes?** Raw Prometheus numbers are hard to read. Grafana turns them into clear dashboards: node CPU, Pod memory, restarts, request rates, errors.

---

## 2. How Grafana Fits in Kubernetes

```
 Kubernetes cluster
+--------------------------------------------------------------+
|                                                              |
|  Nodes / Pods ----> node-exporter, kube-state-metrics        |
|                              |  (scrape)                     |
|                              v                               |
|                       +-------------+                        |
|                       | Prometheus  |   (data source)        |
|                       +------+------+                        |
|                              | PromQL queries                |
|                              v                               |
|   Browser <----------- +-----------+                         |
|   (dashboards)         |  Grafana  |  runs as a Pod          |
|                        +-----------+                         |
+--------------------------------------------------------------+
```

Grafana itself is just **another application running in the cluster**: a Deployment, a Service (to open it) and a PVC (to keep dashboards).

---

## 3. Grafana Key Concepts

| Concept | Meaning |
|---|---|
| **Data source** | Where data comes from (Prometheus, Loki for logs, Elasticsearch, CloudWatch, Azure Monitor) |
| **Dashboard** | A page of panels |
| **Panel** | One graph, gauge, table or stat, built from a **PromQL query** |
| **Variables** | Dropdowns on a dashboard (for example, pick a namespace or Pod) |
| **Alert rules** | Notify (email, Slack, PagerDuty) when a condition is true |
| **Folders / Users / Teams** | Organise dashboards and control who can see or edit them |

**Port:** Grafana listens on **3000**.

---

## 4. Deploy Grafana on Kubernetes

> If you installed **kube-prometheus** on Day 11, Grafana is **already running** in the `monitoring` namespace. This section shows how to deploy it on its own, which also helps you learn Deployments, Services, ConfigMaps and Secrets together.

### Step 1: Namespace and admin password Secret

```bash
kubectl create namespace monitoring

kubectl create secret generic grafana-admin -n monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='ChangeMe123'
```

### Step 2: Data source ConfigMap (connects Grafana to Prometheus)

`grafana-datasource.yml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  prometheus.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        access: proxy
        url: http://prometheus.monitoring.svc:9090   # Service name of your Prometheus
        isDefault: true
```

The URL is the **Prometheus Service DNS name** (Day 4). If you used kube-prometheus, the Service is `prometheus-k8s`, so use `http://prometheus-k8s.monitoring.svc:9090`.

### Step 3: Storage for dashboards (PVC)

`grafana-pvc.yml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Without this, dashboards are **lost when the Pod restarts**.

### Step 4: Deployment

`grafana-deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:latest      # pin a version in real use
          ports:
            - containerPort: 3000
          env:
            - name: GF_SECURITY_ADMIN_USER
              valueFrom:
                secretKeyRef:
                  name: grafana-admin
                  key: admin-user
            - name: GF_SECURITY_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-admin
                  key: admin-password
          volumeMounts:
            - name: storage
              mountPath: /var/lib/grafana
            - name: datasources
              mountPath: /etc/grafana/provisioning/datasources
      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: grafana-pvc
        - name: datasources
          configMap:
            name: grafana-datasources
```

| Part | Purpose |
|---|---|
| `env` from Secret | Admin login (Day 10) |
| ConfigMap mounted at `/etc/grafana/provisioning/datasources` | Grafana loads the Prometheus data source automatically at startup |
| PVC mounted at `/var/lib/grafana` | Keeps dashboards and settings |

### Step 5: Service (to open Grafana)

`grafana-service.yml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  type: NodePort                 # LoadBalancer in the cloud
  selector:
    app: grafana                 # must match the Pod label
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 32000
```

### Step 6: Apply and open

```bash
kubectl apply -f grafana-datasource.yml
kubectl apply -f grafana-pvc.yml
kubectl apply -f grafana-deployment.yml
kubectl apply -f grafana-service.yml

kubectl get pods -n monitoring
kubectl get svc -n monitoring

# Minikube
minikube service grafana -n monitoring --url

# or without a Service
kubectl port-forward -n monitoring svc/grafana 3000:3000     # http://localhost:3000
```

Log in with `admin` and the password from the Secret.

---

## 5. Using Grafana

### 5.1 Check the Data Source

**Connections > Data sources > Prometheus > Save & test.** It should say the data source is working. If it fails, check the Prometheus Service name, namespace and port.

### 5.2 Import a Ready-Made Dashboard

1. Go to **Dashboards > New > Import**.
2. Enter a dashboard ID from the Grafana dashboard library (for example **1860**, "Node Exporter Full") or upload a JSON file.
3. Select **Prometheus** as the data source and click **Import**.

This needs the matching metrics (node-exporter for node dashboards, kube-state-metrics for Kubernetes object dashboards). Search the library for "Kubernetes" to find cluster, namespace and Pod dashboards.

### 5.3 Create a Simple Panel

1. **New dashboard > Add visualization**, select **Prometheus**.
2. Enter a query, for example:

```promql
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)
```

3. Choose a visualisation (Time series, Gauge, Stat), give it a title, and **Save**.

### 5.4 Useful Dashboards to Build

| Dashboard | Query idea |
|---|---|
| Node CPU | `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` |
| Pod memory | `container_memory_usage_bytes{namespace="default"}` |
| Pod restarts | `increase(kube_pod_container_status_restarts_total[1h])` |
| Available replicas | `kube_deployment_status_replicas_available` |

### 5.5 Alerts

**Alerting > Alert rules > New rule**: choose a query, a condition (for example, CPU above 90% for 5 minutes), and a **contact point** (email, Slack, PagerDuty).

---

## 6. Grafana vs Prometheus Alerting

| | Prometheus + Alertmanager | Grafana alerting |
|---|---|---|
| Rules written in | YAML files | Grafana UI |
| Best for | Cluster and infrastructure alerts, kept in Git | Quick alerts, multiple data sources |
| Notifications | Alertmanager | Grafana contact points |

Both work. Many teams keep core alerts in Prometheus and use Grafana mainly for dashboards.

---

## 7. Best Practices

- Use a **PVC**, or store dashboards as code, so they survive restarts.
- Keep the **admin password in a Secret**, and change the default `admin/admin`.
- **Pin the image version**, not `latest`.
- Provision data sources and dashboards with **ConfigMaps** so the setup is repeatable.
- Expose Grafana through an **Ingress with TLS** (Day 7) in production, not NodePort.
- Use **RBAC and teams** in Grafana so people see only what they need.

---

## 8. Troubleshooting

| Problem | Check |
|---|---|
| Grafana Pod `Pending` | PVC not bound (`kubectl get pvc -n monitoring`) |
| Data source test fails | Prometheus Service name, namespace and port in the URL |
| Dashboard shows "No data" | Is the exporter running? Is the target `UP` in Prometheus (Status > Targets)? |
| Cannot open the page | Service selector must match the Pod label, and use `minikube service` or `port-forward` |
| Lost dashboards after restart | No PVC attached |

```bash
kubectl logs -n monitoring deploy/grafana
kubectl describe pod -n monitoring -l app=grafana
kubectl get endpoints grafana -n monitoring
```

---

## Quick Revision

- **Grafana** = dashboards. It reads data from sources like Prometheus but does not store metrics.
- In Kubernetes it runs as a **Deployment + Service + PVC**, with config in a **ConfigMap** and the password in a **Secret**.
- Connect it to Prometheus using the **Prometheus Service DNS name** (`http://<svc>.<namespace>.svc:9090`).
- Mount the data source ConfigMap at `/etc/grafana/provisioning/datasources`, and the PVC at `/var/lib/grafana`.
- Open it with `minikube service` or `kubectl port-forward` (port **3000**).
- **Import dashboards** by ID (for example 1860), or build panels with **PromQL**.
- Prometheus **collects and stores**, Grafana **displays**, Alertmanager or Grafana **alerts**.
