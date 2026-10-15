# Kubernetes Notes: Day 10: ConfigMap, Secrets and Volume Mounts

## 1. The Story

**Raj** is a developer who built a shop app and packed it into a Docker image. Inside the code he hard-coded everything:

```
DB_HOST = "dev-db.company.com"
APP_MODE = "development"
DB_PASSWORD = "MyPass123"
```

Then the problems started:

- To move the app from **dev** to **prod**, he had to **change the code and rebuild the image** just for a new database address.
- The **password was inside the image**, so anyone who could pull the image could read it.
- Every small setting change meant a **new image and a new deployment**.

His teammate **Meera** (a Kubernetes engineer) told him:

> "Keep the app and its settings separate. Put normal settings on a **notice board** (**ConfigMap**) and sensitive ones in a **locked safe** (**Secret**). The Pod will read from both when it starts."

```
 BEFORE                                    AFTER
+------------------------+          +-------------------+   +-------------------+
| Image                  |          | Image             |   | ConfigMap         |
|  code                  |          |  code only        |   |  (notice board)   |
|  DB_HOST   (hard-coded)|   ==>    +---------+---------+   |  DB_HOST, MODE    |
|  PASSWORD  (hard-coded)|                    |             +---------+---------+
+------------------------+                    |             +---------+---------+
 change setting = rebuild                     |             | Secret            |
                                              |             |  (locked safe)    |
                                              |             |  DB_PASSWORD      |
                                              |             +---------+---------+
                                              v                       |
                                         +------------- POD -----------+--+
                                         | app reads settings at start    |
                                         +--------------------------------+
```

**Now:** the same image runs in dev, test and prod. Only the ConfigMap and Secret change. No rebuild needed.

---

## 2. What is a ConfigMap?

A **ConfigMap** stores **non-sensitive configuration** as **key-value pairs** (or whole files), separate from the container image.

**Examples:** database host, app mode, log level, feature flags, `nginx.conf`, `application.properties`.

- Stored in plain text
- Max size **1 MiB**
- Same namespace as the Pod using it

## 3. What is a Secret?

A **Secret** stores **sensitive data** such as passwords, API keys, tokens and TLS certificates.

- Values are **base64-encoded** (not encrypted by default!)
- Can be restricted with **RBAC** (Day 8), and encrypted at rest in etcd if you enable it
- Max size **1 MiB**
- Common types: `Opaque` (generic), `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson` (registry login)

> **Important:** base64 is **encoding, not security**. Anyone who can read the Secret can decode it. Protect Secrets with RBAC, enable encryption at rest, and never commit them to Git.

---

## 4. Difference Between ConfigMap and Secret

| | **ConfigMap** | **Secret** |
|---|---|---|
| **Stores** | Non-sensitive config | Sensitive data (passwords, keys, tokens) |
| **Story analogy** | Notice board | Locked safe |
| **Storage format** | Plain text | base64-encoded |
| **Encryption at rest** | No | Optional (needs to be enabled) |
| **Access control** | Normal RBAC | Stricter RBAC recommended |
| **In `kubectl describe`** | Values shown | Values **hidden** |
| **Kept in memory (tmpfs) when mounted** | No (written to node disk) | Yes (not written to disk on the node) |
| **Size limit** | 1 MiB | 1 MiB |
| **Examples** | `DB_HOST`, `LOG_LEVEL` | `DB_PASSWORD`, `API_KEY` |

---

## 5. Ways to Use Them in a Pod

| Method | How | Updates automatically? |
|---|---|---|
| **Environment variable** | One key as an env var (`env`), or all keys (`envFrom`) | **No**, Pod restart needed |
| **Volume mount** | Keys appear as **files** in a folder inside the container | **Yes**, after a short delay (not with `subPath`) |

### What is a Volume Mount?

A **volume** is a folder that Kubernetes attaches to the Pod. **`volumeMounts`** says **where inside the container** that folder appears.

When a ConfigMap or Secret is mounted as a volume, **each key becomes a file** and its **value becomes the file content**.

```
ConfigMap: app-config                      Container filesystem
+---------------------------+              +------------------------------+
| DB_HOST  = prod-db        |  mounted at  | /etc/config/                 |
| APP_MODE = production     |  =========>  |    DB_HOST   (file: prod-db) |
+---------------------------+              |    APP_MODE  (file: production)
                                           +------------------------------+
```

Two parts are needed in the Pod spec:

| Part | Where | Purpose |
|---|---|---|
| `volumes` | Under `spec` | **Declares** the volume and its source (ConfigMap or Secret) |
| `volumeMounts` | Under the container | **Mounts** it at a path inside the container |

The `name` in `volumeMounts` must match the `name` in `volumes`.

---

## 6. Commands and YAML

### 6.1 Create a ConfigMap

**Command:**

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=DB_HOST=prod-db.company.com \
  --from-literal=APP_MODE=production

# From a file
kubectl create configmap nginx-config --from-file=nginx.conf
```

**YAML** (`configmap.yml`):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: prod-db.company.com
  APP_MODE: production
```

### 6.2 Create a Secret

**Command:**

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD='MyPass123'
```

**YAML** (`secret.yml`), option 1: `stringData` (plain text, Kubernetes encodes it for you):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USER: admin
  DB_PASSWORD: MyPass123
```

Option 2: `data` (you must base64-encode the values yourself):

```bash
echo -n 'admin' | base64          # YWRtaW4=
echo -n 'MyPass123' | base64      # TXlQYXNzMTIz
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_USER: YWRtaW4=
  DB_PASSWORD: TXlQYXNzMTIz
```

> Use `echo -n` (no newline), otherwise a hidden newline gets encoded. **Do not commit Secret YAML with real values to Git.**

### 6.3 Use Both in a Deployment

`deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shop
  template:
    metadata:
      labels:
        app: shop
    spec:
      containers:
        - name: shop
          image: nginx:1.25

          # Method 1: environment variables
          env:
            - name: DB_HOST                  # env var name inside the container
              valueFrom:
                configMapKeyRef:
                  name: app-config           # ConfigMap name
                  key: DB_HOST               # key in the ConfigMap
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret            # Secret name
                  key: DB_PASSWORD           # key in the Secret

          # Method 1b: load ALL keys as env vars
          envFrom:
            - configMapRef:
                name: app-config

          # Method 2: volume mounts (keys become files)
          volumeMounts:
            - name: config-volume            # must match volumes[].name
              mountPath: /etc/config         # folder inside the container
            - name: secret-volume
              mountPath: /etc/secret
              readOnly: true

      volumes:
        - name: config-volume
          configMap:
            name: app-config                 # source: the ConfigMap
        - name: secret-volume
          secret:
            secretName: db-secret            # source: the Secret
```

| Field | Meaning |
|---|---|
| `env[].name` | Name of the env var inside the container |
| `configMapKeyRef` / `secretKeyRef` | Take **one key** from a ConfigMap / Secret |
| `envFrom` | Load **all keys** as env vars |
| `volumes[].configMap.name` / `secret.secretName` | Source object for the volume |
| `volumeMounts[].name` | Must match the `volumes` entry name |
| `volumeMounts[].mountPath` | Folder inside the container where files appear |
| `readOnly: true` | Container cannot modify the files (good for Secrets) |

**Names that must match:** the ConfigMap/Secret **name** and **key** in the Deployment must exactly match the ones you created. If they do not, the Pod will fail with `CreateContainerConfigError`.

### 6.4 Apply and Verify

```bash
kubectl apply -f configmap.yml
kubectl apply -f secret.yml
kubectl apply -f deployment.yml
kubectl get pods
```

**Check the env vars and files inside a Pod:**

```bash
kubectl exec -it <pod-name> -- env | grep -E "DB_|APP_"
kubectl exec -it <pod-name> -- ls /etc/config
kubectl exec -it <pod-name> -- cat /etc/config/DB_HOST
kubectl exec -it <pod-name> -- cat /etc/secret/DB_PASSWORD
```

### 6.5 View, Decode, Update, Delete

```bash
# View
kubectl get configmap
kubectl get secret
kubectl describe configmap app-config          # shows values
kubectl describe secret db-secret              # hides values, shows sizes
kubectl get configmap app-config -o yaml

# Decode a Secret value
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d

# Update (edit, then re-apply)
kubectl edit configmap app-config
kubectl apply -f configmap.yml

# Env vars only change after a restart
kubectl rollout restart deployment shop-app

# Delete
kubectl delete configmap app-config
kubectl delete secret db-secret
```

### 6.6 Env Vars vs Volume Mount: Which to Use?

| Use | When |
|---|---|
| **Env vars** | Simple key-value settings the app reads at startup |
| **Volume mount** | Whole config files (`nginx.conf`), or when you want updates without restarting the Pod |
| **Secrets as volume** | Preferred for sensitive data, since env vars can leak into logs and crash dumps |

---

## Best Practices

- Never hard-code config or passwords in images or Git.
- Put **sensitive data only in Secrets**, everything else in ConfigMaps.
- Limit who can read Secrets with **RBAC**, and enable **encryption at rest**.
- Mount Secrets as **read-only** volumes where possible.
- In production, consider an external secret manager (**AWS Secrets Manager, Azure Key Vault, HashiCorp Vault**) with the External Secrets Operator (a CRD and controller, as in Day 9).
- Use one ConfigMap per app or environment to stay organised.

---

## Quick Revision

- **ConfigMap** = non-sensitive settings (notice board). **Secret** = sensitive data (locked safe, base64 not encryption).
- They **separate config from the image**, so the same image runs in dev, test and prod.
- Use them as **env vars** (no auto-update) or **volume mounts** (each key becomes a file, auto-updates).
- Volume mount needs **`volumes`** (the source) and **`volumeMounts`** (the path), with matching `name`.
- Create: `kubectl create configmap` and `kubectl create secret generic`. Decode: `base64 -d`.
- Names and keys in the Deployment must match the ConfigMap and Secret exactly.
