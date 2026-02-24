# Kubernetes — Basic Questions

---

## 1. What is the Kubernetes architecture? Explain the components of the master node and worker node.

Kubernetes follows a **control plane + data plane** architecture. The control plane (master) makes decisions about the cluster. The worker nodes run the actual application workloads.

```
┌─────────────────── Control Plane (Master) ──────────────────┐
│  API Server  │  etcd  │  Scheduler  │  Controller Manager   │
└──────────────────────────────────────────────────────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
     ┌────────────┐ ┌────────────┐ ┌────────────┐
     │ Worker Node│ │ Worker Node│ │ Worker Node│
     │  kubelet   │ │  kubelet   │ │  kubelet   │
     │  kube-proxy│ │  kube-proxy│ │  kube-proxy│
     │  container │ │  container │ │  container │
     │  runtime   │ │  runtime   │ │  runtime   │
     └────────────┘ └────────────┘ └────────────┘
```

---

**Control Plane Components:**

**1. API Server (`kube-apiserver`)**
The single entry point for all cluster operations. Every `kubectl` command, every controller, every node — all communicate through the API Server. It validates requests, authenticates/authorises them, and stores the result in etcd.

```bash
kubectl apply -f deployment.yaml
# → kubectl sends HTTP request to API Server
# → API Server validates, authenticates, writes to etcd
# → Controllers detect the change and act
```

**2. etcd**
A distributed key-value store that holds the entire cluster state — all resource definitions (Deployments, Services, ConfigMaps, Secrets). If etcd is lost without a backup, the cluster state is gone. This is why etcd backups are critical.

```bash
# etcd stores everything as key-value pairs
/registry/deployments/default/myapp → <deployment spec JSON>
/registry/pods/default/myapp-xyz    → <pod spec JSON>
```

**3. Scheduler (`kube-scheduler`)**
Watches for newly created pods with no assigned node, and selects the best node to run them based on: resource requests (CPU/memory), node affinity/anti-affinity, taints and tolerations, and available capacity.

**4. Controller Manager (`kube-controller-manager`)**
Runs multiple control loops (controllers) that watch the cluster state and reconcile it to the desired state:
- **Deployment controller** — ensures the right number of ReplicaSet pods are running
- **Node controller** — detects when nodes go down, marks pods as failed
- **Endpoints controller** — updates Service endpoints when pods change
- **Job controller** — tracks completion of batch jobs

---

**Worker Node Components:**

**1. kubelet**
An agent running on every worker node. It communicates with the API Server, receives pod specs, and ensures the containers described in those specs are running and healthy. It reports node and pod status back to the API Server.

```bash
# kubelet reads this pod spec from API Server and runs the container
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    resources:
      requests: { cpu: "100m", memory: "128Mi" }
```

**2. kube-proxy**
Maintains network rules on each node to implement Kubernetes Services. It handles routing — when you hit a Service ClusterIP, kube-proxy routes the request to one of the healthy pod IPs behind that Service.

```
Request → Service ClusterIP (172.20.45.10:8080)
              ↓ kube-proxy (iptables/IPVS rules)
         Pod IP (10.0.11.45:8080)  ← actual pod
```

**3. Container Runtime**
The software that actually runs containers. Kubernetes supports any runtime implementing the CRI (Container Runtime Interface):
- **containerd** (most common, used by EKS/GKE/AKS by default)
- **CRI-O** (Red Hat / OpenShift)
- Docker (deprecated as a runtime in K8s 1.24+)

**4. Pod**
The smallest deployable unit in Kubernetes — one or more containers that share a network namespace and storage. All containers in a pod share the same IP address.

---

**Communication flow example — deploying an app:**

```
1. kubectl apply -f deployment.yaml
      ↓
2. API Server validates + stores in etcd
      ↓
3. Deployment Controller detects new Deployment → creates ReplicaSet
      ↓
4. ReplicaSet Controller detects pods needed → creates Pod objects in etcd
      ↓
5. Scheduler assigns each pod to a worker node → writes nodeName to pod spec in etcd
      ↓
6. kubelet on the assigned node sees the pod assigned to it
      ↓
7. kubelet tells containerd to pull the image and start the container
      ↓
8. kubelet reports pod status back to API Server → etcd updated
      ↓
9. kube-proxy updates iptables rules so Service routes to new pod IP
```

---

## 2. What is the difference between a Deployment and a StatefulSet in Kubernetes?

Both manage pods, but they handle identity, storage, and ordering differently. The right choice depends on whether your application is stateless or stateful.

**Deployment — for stateless applications.**

A Deployment manages a set of identical, interchangeable pods. Any pod can be replaced by another — they have no persistent identity, no stable hostname, and no persistent storage attached to a specific pod.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: myapp:1.0
        ports:
        - containerPort: 8080
```

Pods are named: `web-app-7d9f8c-xkj2p`, `web-app-7d9f8c-mn4rt` — random suffixes, no stable identity. Use for: API servers, web frontends, worker processes.

**StatefulSet — for stateful applications.**

A StatefulSet gives each pod a stable, persistent identity: a predictable name (`db-0`, `db-1`, `db-2`), a stable network hostname, and its own PersistentVolumeClaim that survives pod restarts.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 50Gi
```

**When to use each:**

| Requirement | Use |
|---|---|
| API server, web frontend, microservice | Deployment |
| Database (PostgreSQL, MySQL, MongoDB) | StatefulSet |
| Message queue (Kafka, RabbitMQ) | StatefulSet |
| Any app that stores data on local disk | StatefulSet |

---

## 3. What is the difference between liveness and readiness probes?

Both probes let Kubernetes check the health of a running pod, but they serve different purposes and trigger different actions when they fail.

**Readiness Probe — is the pod ready to receive traffic?**

When a readiness probe fails, Kubernetes removes the pod from the Service endpoints. Traffic stops being sent to that pod. The pod keeps running — it is not restarted.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

**Liveness Probe — is the pod alive and should it be restarted?**

When a liveness probe fails, Kubernetes kills the pod and restarts it.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

**Summary:**

| Probe | Failure action | Use case |
|---|---|---|
| Readiness | Remove from Service endpoints | App starting up, temporarily overloaded |
| Liveness | Restart the pod | App deadlocked, stuck in bad state |
| Startup | Disable liveness/readiness during init | Apps with long initialisation |
