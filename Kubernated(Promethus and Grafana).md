# Kubernetes Notes: Day 11: Monitoring with Prometheus and Grafana

## 1. Why Monitor Kubernetes?

Pods come and go, nodes fail, and traffic changes. Without monitoring you only find problems when users complain.

| Monitoring helps you | Example |
|---|---|
| **Detect problems early** | Pod restarting repeatedly, node out of memory |
| **Alert the team** | Send an email or PagerDuty alert when CPU is above 90% |
| **Understand performance** | Which service is slow, how much CPU/memory is used |
| **Plan capacity** | Do we need more nodes? |
| **Debug incidents** | Look back at what happened at 2 AM |

**What to monitor:**

| Level | Examples |
|---|---|
| **Nodes** | CPU, memory, disk, network |
| **Cluster / control plane** | API server, etcd, scheduler health |
| **Pods and containers** | CPU, memory, restarts, status |
| **Kubernetes objects** | Deployments (desired vs available replicas), Pod phases |
| **Applications** | Request rate, errors, response time |

---

## 2. Prometheus and Grafana in One Line Each

| Tool | Job | Analogy |
|---|---|---|
| **Prometheus** | **Collects and stores** metrics, and **raises alerts** | The **data collector** and storage |
| **Grafana** | **Shows** the metrics as dashboards and graphs | The **TV screen** that displays the data |

```
Targets (Pods, Nodes) --metrics--> Prometheus (stores) --queries--> Grafana (dashboards)
                                        |
                                        +--alerts--> Alertmanager --> Email / PagerDuty / Slack
```

---

## 3. What is Prometheus?

**Prometheus** is an open-source **monitoring and alerting system**, and a **CNCF graduated project**. It is the standard monitoring tool for Kubernetes.

**Key features:**

- Stores data as **time series** (a metric value plus a timestamp, with labels)
- **Pull model**: Prometheus **scrapes** (fetches) metrics from targets over HTTP, usually from `/metrics`
- Powerful query language: **PromQL**
- Built-in **service discovery** for Kubernetes, so new Pods are found automatically
- **Alerting** through Alertmanager

A metric looks like this:

```
http_requests_total{method="GET", status="200", pod="shop-abc"}  1027
```

`http_requests_total` is the metric name, the `{...}` are **labels** (same idea as Kubernetes labels), and `1027` is the value.

**Metric types:**

| Type | Meaning | Example |
|---|---|---|
| **Counter** | Only goes up | Total requests |
| **Gauge** | Goes up and down | Memory in use, temperature |
| **Histogram** | Counts values in buckets | Request duration |

---

## 4. Prometheus Architecture

Based on the diagram you shared.

![Prometheus Architecture](Prom_arch.png)

### 4.1 Components Explained

| Component | Role |
|---|---|
| **Prometheus targets (Jobs / exporters)** | The things being monitored. Apps expose a `/metrics` endpoint, and **exporters** do this for things that cannot (e.g. node-exporter for machines) |
| **Service discovery** | Automatically **finds targets**. In Kubernetes it asks the API server for Pods, Services and Nodes. `file_sd` reads targets from a file |
| **Retrieval** | The part of the Prometheus server that **pulls (scrapes) metrics** from the discovered targets |
| **TSDB** (Time Series Database) | **Stores** the metrics as time series |
| **HDD / SSD** | The **disk on the node** where the TSDB saves data |
| **HTTP server** | Answers **PromQL queries** from the UI, Grafana and API clients |
| **Pushgateway** | For **short-lived jobs** (batch jobs) that finish before Prometheus can scrape them. They **push** their metrics to the Pushgateway at exit, and Prometheus pulls from it |
| **Alertmanager** | Receives **alerts** from Prometheus, then **groups, deduplicates, silences and routes** them to **PagerDuty, email, Slack**, etc. |
| **PromQL** | The query language used to read the data |
| **Prometheus web UI** | Built-in page for running queries and viewing targets |
| **Grafana** | Dashboards that query Prometheus using PromQL |
| **API clients** | Any tool that reads data through the HTTP API |

### 4.2 Flow in Simple Steps

1. **Service discovery** finds the targets (for example, all Pods in the cluster).
2. **Retrieval** **pulls metrics** from the targets at a set interval (for example, every 15 to 30 seconds).
3. Short-lived jobs **push** metrics to the **Pushgateway**, and Prometheus pulls from there.
4. Data is stored in the **TSDB** on disk.
5. **Alert rules** are checked, and if a condition is true, Prometheus **pushes the alert to Alertmanager**.
6. Alertmanager **notifies** the team (email, PagerDuty, etc.).
7. **Grafana** and the **web UI** query the data using **PromQL** through the HTTP server.

### 4.3 Pull vs Push

| | Pull (Prometheus default) | Push (Pushgateway) |
|---|---|---|
| Who starts it | Prometheus fetches from targets | Job sends its metrics |
| Use for | Long-running apps and services | **Short-lived jobs** only |
| Benefit | Easy to tell if a target is down | Metrics are not lost when the job exits |

---

## 5. Metric Sources in Kubernetes

| Source | What it provides |
|---|---|
| **node-exporter** | Node (machine) metrics: CPU, memory, disk, network |
| **kube-state-metrics** | State of Kubernetes objects: Deployments, Pod status, replicas |
| **cAdvisor** (built into kubelet) | Container CPU and memory usage |
| **kubelet, API server, etcd** | Control plane and node component metrics |
| **Your apps** | Custom metrics at `/metrics` |
| **metrics-server** | Not Prometheus. A lightweight source for `kubectl top` and the autoscaler (HPA) |


---

## 6. Basic PromQL Examples

```promql
# Is the target up? (1 = up, 0 = down)
up

# Memory used by each container
container_memory_usage_bytes

# CPU usage rate per Pod over the last 5 minutes
rate(container_cpu_usage_seconds_total{namespace="default"}[5m])

# Number of Pods that are Running
count(kube_pod_status_phase{phase="Running"})

# Containers restarting
increase(kube_pod_container_status_restarts_total[1h])
```



## 7. What is Grafana?

**Grafana** is an open-source **visualisation and dashboard tool**. It does **not store metrics**. It connects to **data sources** (Prometheus, Loki, Elasticsearch, etc.) and shows the data as graphs, gauges and tables.

- Add **Prometheus as a data source**, then build panels with PromQL
- Use **ready-made dashboards** from the Grafana dashboard library (for example, Node Exporter or Kubernetes cluster dashboards)
- Set up alerts, and share dashboards with the team

| | Prometheus | Grafana |
|---|---|---|
| Collects data | **Yes** | No |
| Stores data | **Yes** (TSDB) | No (reads from sources) |
| Dashboards | Basic | **Rich, customisable** |
| Alerts | Alert rules and Alertmanager | Also supported |
| Data sources | Prometheus only | Many |

---



## Quick Revision

- **Monitoring** = detect problems early, alert, understand performance, plan capacity.
- **Prometheus** = collects, stores and alerts (pull model, TSDB, PromQL). **Grafana** = dashboards that read from Prometheus.
- **Architecture:** Service discovery finds targets, **Retrieval** pulls metrics, **TSDB** stores them on disk, **HTTP server** answers PromQL, **Alertmanager** sends notifications, **Pushgateway** is for short-lived jobs.
- **Kubernetes sources:** node-exporter (nodes), kube-state-metrics (objects), cAdvisor (containers), your app `/metrics`.
- **Pull** for long-running targets, **push** (Pushgateway) only for short-lived jobs.
- Install the full stack with **kube-prometheus**, then port-forward to Prometheus (9090), Grafana (3000) and Alertmanager (9093).
