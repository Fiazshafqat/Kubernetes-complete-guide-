# Kubernetes Complete Guide

A comprehensive reference covering Kubernetes concepts, architecture, workloads, networking, storage, security, and operations — from beginner to production.

---

## Table of Contents

1. [What is Kubernetes?](#1-what-is-kubernetes)
2. [Docker vs Kubernetes](#2-docker-vs-kubernetes)
3. [Problems Docker has that Kubernetes Solves](#3-problems-docker-has-that-kubernetes-solves)
4. [How Kubernetes Works](#4-how-kubernetes-works)
5. [Kubernetes Architecture](#5-kubernetes-architecture)
6. [Control Plane Components](#6-control-plane-components)
7. [Data Plane Components](#7-data-plane-components)
8. [Container Network Interface (CNI)](#8-container-network-interface-cni)
9. [kubectl and kubeconfig](#9-kubectl-and-kubeconfig)
10. [Pods](#10-pods)
11. [Workloads — Deployment, ReplicaSet, StatefulSet, DaemonSet](#11-workloads)
12. [Stateless vs Stateful Applications](#12-stateless-vs-stateful-applications)
13. [Deployment vs StatefulSet](#13-deployment-vs-statefulset)
14. [Jobs and CronJobs](#14-jobs-and-cronjobs)
15. [Services](#15-services)
16. [Ingress](#16-ingress)
17. [ConfigMaps and Secrets](#17-configmaps-and-secrets)
18. [Persistent Storage](#18-persistent-storage)
19. [CSI Drivers](#19-csi-drivers)
20. [Auto-scaling and Resource Management](#20-auto-scaling-and-resource-management)
21. [Health Probes](#21-health-probes)
22. [Role-Based Access Control (RBAC)](#22-role-based-access-control-rbac)
23. [Taints and Tolerations](#23-taints-and-tolerations)
24. [Node Affinity and Anti-Affinity](#24-node-affinity-and-anti-affinity)
25. [Pod Affinity and Anti-Affinity](#25-pod-affinity-and-anti-affinity)
26. [Multi-Container Patterns](#26-multi-container-patterns)
27. [Init Containers](#27-init-containers)
28. [Troubleshooting and Common Errors](#28-troubleshooting-and-common-errors)
29. [Deployment Strategies](#29-deployment-strategies)
30. [Most Commonly Used kubectl Commands](#30-most-commonly-used-kubectl-commands)
31. [Helm — Package Management](#31-helm--package-management)
32. [Cluster Setup and Management](#32-cluster-setup-and-management)

---

## 1. What is Kubernetes?

Kubernetes (also written as **K8s**) is an open-source container orchestration platform originally developed by Google and donated to the Cloud Native Computing Foundation (CNCF) in 2014.

It automates the deployment, scaling, and management of containerized applications across a cluster of machines.

**Key capabilities:**
- Automated deployment and rollback
- Self-healing (restarts failed containers automatically)
- Horizontal scaling (scale up/down based on load)
- Service discovery and load balancing
- Storage orchestration
- Secret and configuration management
- Batch execution and scheduling

---

## 2. Docker vs Kubernetes

| Feature | Docker | Kubernetes |
|---------|--------|------------|
| Purpose | Build and run containers | Orchestrate containers at scale |
| Scope | Single host | Multi-node cluster |
| Scaling | Manual (`docker run` multiple times) | Automatic (HPA, replicas) |
| Self-healing | No | Yes (restarts failed pods) |
| Load balancing | Basic | Advanced (Services, Ingress) |
| Storage | Volumes on one host | Distributed persistent storage |
| Networking | Bridge, host, overlay | CNI plugins (Calico, Flannel) |
| Rolling updates | No built-in support | Native rolling updates |
| Multi-host | Docker Swarm (limited) | Native multi-node |

> Docker is the engine that builds and runs containers. Kubernetes is the platform that manages those containers across many machines.

---

## 3. Problems Docker has that Kubernetes Solves

### Problem 1 — Single host limitation
Docker runs containers on one machine. If that machine fails, all containers go down.

**Kubernetes fix:** Runs containers across multiple nodes. If one node fails, pods are rescheduled to healthy nodes automatically.

### Problem 2 — No auto-healing
If a Docker container crashes, it stays crashed unless you manually restart it.

**Kubernetes fix:** Deployment controllers continuously monitor pod health and restart failed containers automatically.

### Problem 3 — No auto-scaling
Docker cannot automatically scale containers based on CPU or memory usage.

**Kubernetes fix:** Horizontal Pod Autoscaler (HPA) automatically adds or removes pods based on metrics.

### Problem 4 — No rolling updates
Deploying a new version in Docker means stopping the old container and starting a new one — causing downtime.

**Kubernetes fix:** Rolling updates replace pods gradually with zero downtime.

### Problem 5 — No load balancing across hosts
Docker provides basic load balancing within one host only.

**Kubernetes fix:** Services and Ingress distribute traffic across all pods on all nodes.

### Problem 6 — No built-in storage management
Persistent storage in Docker is manual and host-specific.

**Kubernetes fix:** PersistentVolumes, PersistentVolumeClaims, and StorageClasses abstract storage from the infrastructure.

---

## 4. How Kubernetes Works

When you deploy an application to Kubernetes, here is what happens:

```
1. You write a YAML manifest describing your desired state
2. kubectl sends the manifest to the API Server
3. API Server stores the desired state in etcd
4. Scheduler assigns pods to nodes based on resources
5. Kubelet on each node starts the containers via the container runtime
6. Controller Manager watches the state and reconciles any drift
7. kube-proxy sets up networking rules so pods can communicate
```

The core concept is the **reconciliation loop** — Kubernetes continuously compares the *desired state* (what you declared) with the *actual state* (what is running) and takes action to make them match.

---

## 5. Kubernetes Architecture

Kubernetes is made up of two planes:

```
┌─────────────────────────────────────────────┐
│              Control Plane (Master)          │
│  API Server | etcd | Scheduler | Controller  │
└─────────────────┬───────────────────────────┘
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│  Node 1  │ │  Node 2  │ │  Node 3  │
│ Kubelet  │ │ Kubelet  │ │ Kubelet  │
│ KubeProxy│ │ KubeProxy│ │ KubeProxy│
│ Runtime  │ │ Runtime  │ │ Runtime  │
│ [Pods]   │ │ [Pods]   │ │ [Pods]   │
└──────────┘ └──────────┘ └──────────┘
         Data Plane (Worker Nodes)
```

---

## 6. Control Plane Components

The control plane is the brain of Kubernetes. It makes global decisions about the cluster.

### API Server

The **central gateway** for all communication in Kubernetes. Every command you run with `kubectl` goes through the API Server.

- Exposes the Kubernetes REST API
- Validates and processes API requests
- The only component that reads from and writes to etcd
- All other components communicate through the API Server

```bash
# Example: kubectl get pods sends a GET request to:
# https://<api-server>/api/v1/namespaces/default/pods
```

### Scheduler

Watches for newly created pods that have no node assigned, and assigns them to a suitable node.

**Scheduling decisions are based on:**
- Resource availability (CPU, memory)
- Node selectors and affinity rules
- Taints and tolerations
- Pod anti-affinity rules
- Resource requests and limits

### Controller Manager

Runs a collection of controllers in a single process. Each controller watches the state of the cluster and makes changes to move the current state toward the desired state.

**Key controllers:**
- **Node Controller** — monitors node health, marks nodes as unavailable
- **Replication Controller** — ensures the correct number of pod replicas
- **Endpoints Controller** — populates the Endpoints object (links Services to Pods)
- **Service Account Controller** — creates default service accounts for namespaces

### etcd

A distributed key-value store that stores the entire state of the Kubernetes cluster.

- Only the API Server communicates directly with etcd
- All cluster data is stored here: pods, secrets, configmaps, service accounts
- Highly available when deployed with multiple replicas
- Should be backed up regularly in production

```bash
# Backup etcd snapshot
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

---

## 7. Data Plane Components

The data plane (worker nodes) runs the actual workloads.

### Kubelet

An agent that runs on every worker node. It ensures that containers described in PodSpecs are running and healthy.

- Registers the node with the API Server
- Receives pod specs from the API Server
- Starts and stops containers via the container runtime
- Reports node and pod status back to the API Server
- Runs health checks (liveness, readiness, startup probes)

### kube-proxy

A network proxy that runs on every node and maintains network rules.

- Implements the Kubernetes Service concept
- Routes traffic to the correct pods using iptables or IPVS rules
- Handles load balancing across pod replicas
- Updates rules when pods are added or removed

### Container Runtime

The software that actually runs containers on a node.

**Common container runtimes:**
| Runtime | Description |
|---------|-------------|
| `containerd` | Default in most modern Kubernetes clusters |
| `CRI-O` | Lightweight runtime designed for Kubernetes |
| `Docker Engine` | Deprecated in Kubernetes 1.24+ as a direct runtime |

> Kubernetes communicates with the container runtime via the **Container Runtime Interface (CRI)**.

---

## 8. Container Network Interface (CNI)

CNI is a standard specification for configuring network interfaces in Linux containers. Kubernetes uses CNI plugins to provide pod networking.

**What CNI provides:**
- Assigns IP addresses to pods
- Enables pod-to-pod communication across nodes
- Sets up network routing and policies

**Popular CNI plugins:**

| Plugin | Features |
|--------|----------|
| Calico | Network policies, BGP routing, high performance |
| Flannel | Simple overlay network, easy to set up |
| Weave | Encrypted networking, easy multi-host setup |
| Cilium | eBPF-based, advanced observability and security |
| AWS VPC CNI | Native VPC networking on AWS EKS |

```bash
# Check which CNI is installed
ls /etc/cni/net.d/
kubectl get pods -n kube-system | grep -E "calico|flannel|weave|cilium"
```

---

## 9. kubectl and kubeconfig

### kubectl

`kubectl` is the command-line tool for interacting with Kubernetes clusters. It communicates with the API Server on your behalf.

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

### kubeconfig

A YAML file (default location: `~/.kube/config`) that stores cluster connection information — server address, credentials, and context.

```yaml
# ~/.kube/config structure
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://192.168.49.2:8443
    certificate-authority: /path/to/ca.crt
  name: minikube
contexts:
- context:
    cluster: minikube
    user: minikube
    namespace: expense-app
  name: minikube
current-context: minikube
users:
- name: minikube
  user:
    client-certificate: /path/to/client.crt
    client-key: /path/to/client.key
```

```bash
# Useful kubeconfig commands
kubectl config get-contexts                    # List all contexts
kubectl config current-context                 # Show active context
kubectl config use-context <name>              # Switch context
kubectl config set-context --current \
  --namespace=expense-app                      # Set default namespace
export KUBECONFIG=~/.kube/config1:~/.kube/config2   # Merge multiple configs
```

---

## 10. Pods

A **Pod** is the smallest deployable unit in Kubernetes. It wraps one or more containers that share the same network namespace and storage.

### Pod vs Container

| | Container | Pod |
|--|-----------|-----|
| What it is | A running process (Docker/containerd) | A Kubernetes wrapper around containers |
| IP address | No — gets it from the pod | Yes — one IP per pod |
| Managed by | Container runtime | Kubernetes |
| Scaling | Not directly scalable | Scaled via Deployments |
| Lifecycle | Managed by runtime | Managed by Kubernetes |

```yaml
# Simple pod example
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: expense-app
spec:
  containers:
  - name: app
    image: nginx:latest
    ports:
    - containerPort: 80
```

> In practice, you rarely create pods directly. You use Deployments, which manage pods for you.

---

## 11. Workloads

### Deployment

The most common workload. Manages stateless application pods with rolling updates and rollback.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:latest
```

### ReplicaSet

Ensures a specified number of pod replicas are running at all times. Deployments manage ReplicaSets automatically — you rarely create ReplicaSets directly.

```bash
# A Deployment creates and manages a ReplicaSet
kubectl get replicaset -n <namespace>
```

### StatefulSet

For stateful applications that need stable network identity and persistent storage (databases, message queues).

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:               # Creates a PVC per pod automatically
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### DaemonSet

Ensures that one pod runs on every node (or a subset of nodes). Used for logging agents, monitoring agents, and network plugins.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      containers:
      - name: log-agent
        image: fluent/fluent-bit:latest
```

**Use cases for DaemonSet:**
- Log collection agents (Fluentd, Fluent Bit)
- Monitoring agents (Prometheus Node Exporter)
- Network plugins (Calico, Flannel)
- Storage plugins

---

## 12. Stateless vs Stateful Applications

### Stateless Applications

Do not store any data locally. Every request is independent. Can be restarted, replicated, or rescheduled anywhere without data loss.

**Examples:** Web servers, REST APIs, frontend apps

**Managed by:** Deployment

### Stateful Applications

Store data that must persist across restarts. Each instance may have a unique identity and data.

**Examples:** MySQL, PostgreSQL, MongoDB, Kafka, Zookeeper

**Managed by:** StatefulSet

| | Stateless | Stateful |
|--|-----------|----------|
| Data persistence | Not needed | Required |
| Pod identity | Random names (pod-abc123) | Stable names (mysql-0, mysql-1) |
| Scaling | Easy — add/remove pods freely | Ordered — scaled one at a time |
| Storage | No PVC needed | PVC per pod via volumeClaimTemplates |
| Kubernetes resource | Deployment | StatefulSet |

---

## 13. Deployment vs StatefulSet

| | Deployment | StatefulSet |
|--|-----------|-------------|
| Pod naming | Random suffix (app-74d9b-abc) | Ordered index (app-0, app-1, app-2) |
| Pod startup | All start simultaneously | Starts in order (0 first, then 1, then 2) |
| Pod shutdown | Simultaneous | Reverse order (2 first, then 1, then 0) |
| Storage | Shared or none | Unique PVC per pod |
| Network identity | Changes on restart | Stable DNS name that persists |
| Headless service | Not needed | Required for stable DNS |
| Use case | Stateless apps | Databases, queues |

---

## 14. Jobs and CronJobs

### Job

Runs a pod to completion for a one-time task. The pod is deleted after successful completion.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      containers:
      - name: migration
        image: my-app:latest
        command: ["python", "manage.py", "migrate"]
      restartPolicy: Never     # Never or OnFailure for jobs
  backoffLimit: 3              # Retry up to 3 times on failure
```

```bash
kubectl get jobs
kubectl logs job/db-migration
```

### CronJob

Runs a Job on a scheduled basis (like Linux cron).

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 2 * * *"        # Every day at 2:00 AM (cron syntax)
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:latest
            command: ["sh", "-c", "mysqldump -u root -p$PASS mydb > backup.sql"]
          restartPolicy: OnFailure
  successfulJobsHistoryLimit: 3   # Keep last 3 successful job records
  failedJobsHistoryLimit: 1       # Keep last 1 failed job record
```

**Cron schedule syntax:**
```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, Sun=0)
│ │ │ │ │
* * * * *

"0 2 * * *"    = Every day at 02:00
"*/5 * * * *"  = Every 5 minutes
"0 0 * * 0"    = Every Sunday at midnight
```

---

## 15. Services

A **Service** gives a stable network endpoint (IP + DNS name) to a set of pods. Since pod IPs change on restart, Services provide a consistent way to reach them.

### ClusterIP (default)

Only accessible within the cluster. Used for internal communication between services.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
```

### NodePort

Exposes the service on a static port (30000–32767) on every node's IP. Accessible from outside the cluster.

```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080       # External access: <node-ip>:30080
```

### LoadBalancer

Provisions an external cloud load balancer (AWS ELB, GCP Load Balancer, Azure LB). Each service gets its own public IP.

```yaml
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
```

### Headless Service

No cluster IP assigned (`clusterIP: None`). Used for StatefulSets to give each pod a stable DNS name.

```yaml
spec:
  clusterIP: None          # Headless — no virtual IP
  selector:
    app: mysql
  ports:
  - port: 3306
```

DNS resolves directly to pod IPs:
```
mysql-0.mysql.namespace.svc.cluster.local
mysql-1.mysql.namespace.svc.cluster.local
```

### Problems with LoadBalancer Service Type

**Problem 1 — No advanced load balancing features**
- No path-based routing (`/api` → service A, `/web` → service B)
- No host-based routing (`api.example.com` → service A)
- No sticky sessions, no SSL termination, no rate limiting
- No WAF or security features

**Problem 2 — Cost of public IP addresses**
- Each `LoadBalancer` service gets its own cloud load balancer with a unique public IP
- Cloud providers charge per load balancer (AWS: ~$18/month per NLB/ALB)
- 10 services = 10 load balancers = 10× the cost

**Both problems are solved by Ingress.**

---

## 16. Ingress

**Ingress** is a Kubernetes resource that manages external HTTP/HTTPS traffic routing into cluster services. It acts as an intelligent reverse proxy and load balancer.

```
Internet → Ingress Controller (1 LoadBalancer) → Routes → Services → Pods
```

### Why Ingress?

- **One load balancer** for all services (saves cost — 1 IP instead of N)
- **Path-based routing** — `/api` → backend service, `/` → frontend service
- **Host-based routing** — `api.example.com` → API service, `app.example.com` → web service
- **SSL/TLS termination** — HTTPS handled at ingress, HTTP inside cluster
- **Advanced features** — rate limiting, authentication, redirects, rewrites

### Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: expense-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  tls:
  - hosts:
    - app.example.com
    secretName: tls-secret
```

### Ingress Controller

An Ingress resource alone does nothing — you need an **Ingress Controller** to implement it.

| Controller | Provider | Notes |
|------------|----------|-------|
| NGINX Ingress | Community / NGINX Inc | Most widely used |
| Traefik | Traefik Labs | Auto service discovery |
| AWS ALB Ingress | AWS | Native ALB integration on EKS |
| GCE Ingress | Google | Native on GKE |
| Kong | Kong Inc | API gateway features |
| Istio Gateway | Istio | Service mesh integration |

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Verify
kubectl get pods -n ingress-nginx
```

---

## 17. ConfigMaps and Secrets

### ConfigMap

Stores non-sensitive configuration data as key-value pairs.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: expense-app
data:
  DB_HOST: mysql
  DB_PORT: "3306"
  DB_NAME: expenses_tracker
  SPRING_DATASOURCE_URL: "jdbc:mysql://mysql:3306/expenses_tracker"
```

### Secret

Stores sensitive data (passwords, tokens, keys). Values are base64-encoded.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: expense-app
type: Opaque
data:
  DB_PASSWORD: VGVzdEAxMjM=       # base64 of "Test@123"
  DB_USER: YXBwdXNlcg==           # base64 of "appuser"
```

```bash
# Encode a value
echo -n "Test@123" | base64      # VGVzdEAxMjM=

# Decode a value
echo "VGVzdEAxMjM=" | base64 -d  # Test@123
```

### Using ConfigMap and Secret in a Pod

```yaml
spec:
  containers:
  - name: app
    image: my-app:latest
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DB_PASSWORD
```

### ConfigMap vs Secret

| | ConfigMap | Secret |
|--|-----------|--------|
| Data type | Non-sensitive config | Sensitive credentials |
| Storage | Plain text in etcd | Base64-encoded in etcd |
| Use for | URLs, ports, feature flags | Passwords, tokens, keys |
| Best practice | Safe to commit YAML | Never commit to Git |

---

## 18. Persistent Storage

### Why Persistent Storage?

By default, pod storage is ephemeral — data is lost when the pod restarts. For stateful workloads (databases), you need storage that outlives the pod.

### Storage Components

#### StorageClass

Defines **how** storage is provisioned — what type of disk, which provider, what reclaim policy.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs-sc
provisioner: ebs.csi.aws.com      # CSI driver provisioner
parameters:
  type: gp3                        # SSD type
  encrypted: "true"
reclaimPolicy: Retain              # Keep data after PVC deleted
allowVolumeExpansion: true
```

#### PersistentVolume (PV)

The actual storage resource in the cluster.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  reclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /mnt/data/mysql          # Local path (development only)
```

#### PersistentVolumeClaim (PVC)

A request for storage by a pod.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: expense-app
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
```

#### Volume Mount in Pod

```yaml
spec:
  containers:
  - name: mysql
    volumeMounts:
    - name: mysql-data
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-data
    persistentVolumeClaim:
      claimName: mysql-pvc
```

### Access Modes

| Mode | Short | Meaning | Use Case |
|------|-------|---------|----------|
| ReadWriteOnce | RWO | One node read/write | Databases |
| ReadWriteMany | RWX | Many nodes read/write | Shared file storage |
| ReadOnlyMany | ROX | Many nodes read only | Config/static files |

### Reclaim Policies

| Policy | Behaviour | Use For |
|--------|-----------|---------|
| `Retain` | Keep data after PVC deleted | Production databases |
| `Delete` | Delete data after PVC deleted | Dev/test environments |

### Storage Provisioning

**Manual provisioning:** Admin creates PV manually. PVC binds to it.

**Dynamic provisioning:** StorageClass automatically creates the PV when a PVC is created. This is the preferred method in production cloud environments.

```
PVC created → StorageClass provisioner triggered → PV auto-created → PVC bound → Pod uses storage
```

---

## 19. CSI Drivers

**CSI = Container Storage Interface**

A standard API that allows Kubernetes to communicate with any storage provider without modifying the Kubernetes codebase. Storage vendors write their own CSI driver that implements the standard interface.

### Why CSI?

Before CSI, storage drivers were built directly into Kubernetes source code (in-tree). Any bug fix or new feature required a Kubernetes release. CSI moved all storage code out of Kubernetes (out-of-tree).

### How CSI Works

```
PVC → Kubernetes CSI Interface → CSI Driver → Cloud Disk API → Physical Disk
```

CSI drivers run as pods inside the cluster and implement three operations:
- `CreateVolume` — provision a new disk from the cloud API
- `AttachVolume` — attach the disk to the node
- `MountVolume` — mount into the pod filesystem

### Common CSI Drivers

| Provider | CSI Driver | StorageClass Provisioner |
|----------|------------|--------------------------|
| AWS EBS | `aws-ebs-csi-driver` | `ebs.csi.aws.com` |
| GCP Persistent Disk | `gcp-pd-csi-driver` | `pd.csi.storage.gke.io` |
| Azure Disk | `azuredisk-csi-driver` | `disk.csi.azure.com` |
| Minikube | Built-in | `k8s.io/minikube-hostpath` |
| NetApp Trident | `trident-csi` | `csi.trident.netapp.io` |

```bash
# Check CSI drivers installed
kubectl get csidrivers

# Check CSI pods
kubectl get pods -n kube-system | grep csi
```

---

## 20. Auto-scaling and Resource Management

### Resource Requests and Limits

```yaml
resources:
  requests:
    memory: "256Mi"     # Minimum guaranteed (used for scheduling)
    cpu: "250m"         # 0.25 CPU core guaranteed
  limits:
    memory: "512Mi"     # Maximum — OOMKilled if exceeded
    cpu: "500m"         # Maximum — throttled if exceeded
```

`250m` = 250 millicores = 0.25 CPU core

### ResourceQuota

Limits total resource usage within a namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: expense-app-quota
  namespace: expense-app
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
    pods: "20"
```

### LimitRange

Sets default resource requests and limits for pods in a namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: expense-app
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 500m
    defaultRequest:
      memory: 256Mi
      cpu: 250m
    type: Container
```

### Horizontal Pod Autoscaler (HPA)

Automatically scales the number of pod replicas based on observed CPU/memory usage or custom metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: expense-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # Scale up when CPU > 70%
```

```bash
kubectl get hpa -n expense-app
kubectl describe hpa backend-hpa -n expense-app
```

### Vertical Pod Autoscaler (VPA)

Automatically adjusts the CPU and memory requests/limits of individual containers based on historical usage.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: backend-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-deployment
  updatePolicy:
    updateMode: "Auto"            # Auto, Recreate, or Off (recommendation only)
```

| | HPA | VPA |
|--|-----|-----|
| What it scales | Number of replicas | CPU/memory of each pod |
| Best for | Stateless apps with variable load | Apps needing right-sized resources |
| Works with | CPU, memory, custom metrics | Historical resource usage |
| Disruption | No restart needed | Pod restart required to apply |

---

## 21. Health Probes

Kubernetes uses probes to check if a container is healthy and ready to serve traffic.

### Liveness Probe

Checks if the container is alive. If it fails, Kubernetes **restarts** the container.

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 60    # Wait 60s before first check
  periodSeconds: 15          # Check every 15s
  timeoutSeconds: 5          # Fail if no response in 5s
  failureThreshold: 3        # Restart after 3 consecutive failures
```

### Readiness Probe

Checks if the container is ready to receive traffic. If it fails, Kubernetes **removes the pod from the Service endpoints** (stops sending traffic) but does NOT restart it.

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3
```

### Startup Probe

Checks if the application has started. Useful for slow-starting applications. Disables liveness and readiness probes until it succeeds.

```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  failureThreshold: 30       # Allow up to 30×10 = 300s to start
  periodSeconds: 10
```

### Probe Types

| Type | How it checks |
|------|---------------|
| `httpGet` | HTTP GET request — success if status 200-399 |
| `tcpSocket` | TCP connection — success if port is open |
| `exec` | Runs a command inside container — success if exit code is 0 |

### When to Use Each

| | Liveness | Readiness | Startup |
|--|----------|-----------|---------|
| Failure action | Restart pod | Remove from service | Disable other probes |
| Use for | Detect deadlocks/crashes | Prevent traffic during startup or DB reconnect | Slow-starting applications |

---

## 22. Role-Based Access Control (RBAC)

RBAC controls who can do what in a Kubernetes cluster.

### Core Concepts

**Role** — defines permissions within a namespace
**ClusterRole** — defines permissions across the entire cluster
**RoleBinding** — grants a Role to a user/group/service account within a namespace
**ClusterRoleBinding** — grants a ClusterRole cluster-wide
**ServiceAccount** — identity for pods (like a user account for applications)

### Role Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: expense-app
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

### RoleBinding Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: expense-app
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: expense-app
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccount Example

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: expense-app
```

```yaml
# Use ServiceAccount in a pod
spec:
  serviceAccountName: my-service-account
  containers:
  - name: app
    image: my-app:latest
```

### ClusterRole and ClusterRoleBinding

```yaml
# ClusterRole — cluster-wide read access to all pods
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-pod-reader-binding
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Check permissions
kubectl auth can-i get pods -n expense-app --as jane
kubectl auth can-i delete deployments -n expense-app --as jane
```

---

## 23. Taints and Tolerations

### Taints

Applied to **nodes** to repel pods from being scheduled on them unless the pod has a matching toleration.

```bash
# Add a taint to a node
kubectl taint nodes node1 key=value:NoSchedule

# Remove a taint
kubectl taint nodes node1 key=value:NoSchedule-
```

**Taint effects:**
| Effect | Behaviour |
|--------|-----------|
| `NoSchedule` | New pods will not be scheduled on this node |
| `PreferNoSchedule` | Kubernetes tries to avoid scheduling but may if needed |
| `NoExecute` | Evicts existing pods AND prevents new scheduling |

### Tolerations

Applied to **pods** to allow them to be scheduled on tainted nodes.

```yaml
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

**Common use case:** Reserve GPU nodes for GPU workloads only.

```bash
# Taint the GPU node
kubectl taint nodes gpu-node gpu=true:NoSchedule

# Only pods with this toleration will run on gpu-node
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

---

## 24. Node Affinity and Anti-Affinity

Node affinity allows pods to be **attracted to** or **repelled from** nodes based on node labels.

### Node Affinity

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:    # Hard rule
        nodeSelectorTerms:
        - matchExpressions:
          - key: disk-type
            operator: In
            values:
            - ssd
      preferredDuringSchedulingIgnoredDuringExecution:   # Soft rule
      - weight: 1
        preference:
          matchExpressions:
          - key: region
            operator: In
            values:
            - us-east-1
```

### Node Anti-Affinity

Use `NotIn` operator to repel pods from certain nodes:

```yaml
matchExpressions:
- key: environment
  operator: NotIn
  values:
  - production
```

### Node Affinity vs Taints/Tolerations

| | Taints/Tolerations | Node Affinity |
|--|-------------------|---------------|
| Defined on | Node (taint) + Pod (toleration) | Pod only |
| Direction | Node repels, pod tolerates | Pod is attracted to node |
| Use case | Dedicated nodes, special hardware | Topology, region, disk type |

---

## 25. Pod Affinity and Anti-Affinity

Controls how pods are scheduled **relative to other pods**.

### Pod Affinity

Schedule pods **together** on the same node or zone.

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache          # Schedule near cache pods
        topologyKey: kubernetes.io/hostname
```

**Use case:** Co-locate a web server with its cache (Redis) to reduce network latency.

### Pod Anti-Affinity

Spread pods **away from** each other for high availability.

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: my-app         # Don't schedule near other my-app pods
        topologyKey: kubernetes.io/hostname
```

**Use case:** Ensure replicas of the same app run on different nodes — if one node fails, other replicas survive.

---

## 26. Multi-Container Patterns

Multiple containers can run inside a single pod and share network and storage.

### Sidecar

A helper container that extends or enhances the main container.

```yaml
containers:
- name: app                         # Main application
  image: my-app:latest
- name: log-shipper                 # Sidecar — ships logs to Elasticsearch
  image: fluent/fluent-bit:latest
  volumeMounts:
  - name: log-volume
    mountPath: /var/log
```

**Use cases:** Log shippers, metric collectors, certificate refreshers, config watchers

### Adapter

Transforms or normalises the output of the main container for external consumers.

```yaml
containers:
- name: app
  image: my-app:latest
- name: metrics-adapter             # Converts app metrics to Prometheus format
  image: prometheus-adapter:latest
```

**Use cases:** Metric format conversion, protocol translation, data transformation

### Ambassador

Acts as a proxy between the main container and the outside world.

```yaml
containers:
- name: app
  image: my-app:latest
- name: db-proxy                    # Ambassador — handles DB connection pooling
  image: envoy:latest
```

**Use cases:** Database connection pooling, service discovery proxy, rate limiting proxy

---

## 27. Init Containers

Init containers run **before** the main application container starts. They must complete successfully before the main container begins.

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:latest
    command: ['sh', '-c', 'until nc -z mysql 3306; do echo waiting for mysql; sleep 2; done']
  - name: run-migrations
    image: my-app:latest
    command: ['python', 'manage.py', 'migrate']
  containers:
  - name: app
    image: my-app:latest
```

**Use cases:**
- Wait for a dependency (database, API) to be ready
- Run database migrations before the app starts
- Download configuration or secrets from external sources
- Set up file permissions

**Init containers vs regular containers:**
- Run to completion before main containers start
- Run sequentially (one at a time)
- Can use different images with different tools
- If an init container fails, Kubernetes restarts it (and the pod) until it succeeds

---

## 28. Troubleshooting and Common Errors

### Common Errors and Fixes

#### ImagePullBackOff / ErrImagePull

The container image cannot be pulled from the registry.

```bash
# Check the error details
kubectl describe pod <pod-name> -n <namespace>

# Common causes and fixes:
# 1. Wrong image name or tag
#    Fix: Check image name and tag in deployment yaml

# 2. Private registry — missing credentials
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<username> \
  --docker-password=<password>

# 3. Minikube — image not loaded into Minikube
minikube image load my-app:latest

# 4. imagePullPolicy: Never — image not present locally
eval $(minikube docker-env) && docker build -t my-app:latest .
```

#### CrashLoopBackOff

The container keeps crashing and restarting.

```bash
# Check logs of the crashed container
kubectl logs <pod-name> -n <namespace>

# Check logs of the PREVIOUS crashed instance
kubectl logs <pod-name> -n <namespace> --previous

# Describe pod for events
kubectl describe pod <pod-name> -n <namespace>

# Common causes:
# - Application error on startup (check logs)
# - Missing environment variables or config
# - Wrong command in ENTRYPOINT/CMD
# - Liveness probe failing too early (increase initialDelaySeconds)
```

#### OOMKilled

Container exceeded its memory limit and was killed.

```bash
kubectl describe pod <pod-name> | grep -A 5 "OOMKilled"

# Fix: Increase memory limit
resources:
  limits:
    memory: "1Gi"    # Increase from previous value
```

#### Pending

Pod is created but not scheduled to any node.

```bash
kubectl describe pod <pod-name>
# Look for: "Events" section at the bottom

# Common causes:
# 1. Insufficient resources on nodes
kubectl describe nodes | grep -A 5 "Allocated resources"

# 2. PVC not bound
kubectl get pvc -n <namespace>

# 3. Node selector / affinity not matching any node
kubectl get nodes --show-labels
```

#### CreateContainerConfigError

Usually missing ConfigMap or Secret.

```bash
kubectl describe pod <pod-name>
# Look for: "configmap not found" or "secret not found"

# Fix: Create the missing resource
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
```

### General Debugging Commands

```bash
# Full pod details with events
kubectl describe pod <pod-name> -n <namespace>

# Live log streaming
kubectl logs <pod-name> -n <namespace> -f

# Shell into running pod
kubectl exec -it <pod-name> -n <namespace> -- bash

# Check resource usage
kubectl top pods -n <namespace>
kubectl top nodes

# Check all events in namespace
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

---

## 29. Deployment Strategies

### Rolling Update (default)

Gradually replaces old pods with new ones. Zero downtime. Kubernetes controls the pace with `maxSurge` and `maxUnavailable`.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Max pods above desired count during update
      maxUnavailable: 0     # Max pods unavailable during update (0 = zero downtime)
```

**Use case:** Standard application updates where backward compatibility is maintained.

### Recreate

Stops all old pods first, then starts new ones. Causes downtime but ensures no two versions run simultaneously.

```yaml
spec:
  strategy:
    type: Recreate
```

**Use case:** Applications that cannot run two versions at the same time (breaking schema changes).

### Blue-Green Deployment

Two identical environments — Blue (current) and Green (new). Switch traffic instantly by updating the Service selector.

```bash
# Blue deployment (current — v1)
kubectl apply -f blue-deployment.yaml     # label: version: blue

# Green deployment (new — v2)
kubectl apply -f green-deployment.yaml    # label: version: green

# Switch traffic to green (instant cutover)
kubectl patch service my-service \
  -p '{"spec":{"selector":{"version":"green"}}}'

# Rollback instantly if needed
kubectl patch service my-service \
  -p '{"spec":{"selector":{"version":"blue"}}}'
```

**Use case:** Zero-downtime deployments with instant rollback capability.

### Canary Deployment

Route a small percentage of traffic to the new version before full rollout.

```yaml
# Stable deployment — 9 replicas (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: my-app
      track: stable
---
# Canary deployment — 1 replica (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      track: canary
```

```yaml
# Service routes to both (by matching only "app: my-app")
spec:
  selector:
    app: my-app            # Matches both stable and canary pods
```

**Use case:** Test new version with a small subset of real users before full rollout.

### Deployment Strategy Comparison

| Strategy | Downtime | Rollback Speed | Resource Cost | Use Case |
|----------|----------|---------------|---------------|----------|
| Rolling Update | None | Moderate | Low | Standard updates |
| Recreate | Yes | Fast | Low | Breaking changes |
| Blue-Green | None | Instant | High (2× resources) | Zero-downtime with instant rollback |
| Canary | None | Fast | Low-Medium | Risk-controlled rollout |

---

## 30. Most Commonly Used kubectl Commands

```bash
# ─────────────────────────────────────────
# VIEWING RESOURCES
# ─────────────────────────────────────────
kubectl get pods -n <ns>                           # List all pods in namespace
kubectl get all -n <ns>                            # List every resource in namespace
kubectl get nodes                                  # List cluster nodes and status
kubectl get deployments -n <ns>                    # List all deployments
kubectl get services -n <ns>                       # List all services
kubectl get pv                                     # List persistent volumes
kubectl get pvc -n <ns>                            # List persistent volume claims
kubectl get configmap -n <ns>                      # List config maps
kubectl get secret -n <ns>                         # List secrets (values hidden)
kubectl get namespaces                             # List all namespaces in cluster
kubectl get events -n <ns>                         # Show recent cluster events
kubectl get storageclass                           # List available storage classes

# ─────────────────────────────────────────
# CREATING & UPDATING
# ─────────────────────────────────────────
kubectl apply -f <file>.yaml                       # Apply or update a manifest file
kubectl apply -f k8s/                              # Apply all files in a folder
kubectl apply --dry-run=client -f <file>           # Validate manifest without deploying
kubectl create namespace <name>                    # Create a new namespace
kubectl replace --force -f <file>                  # Force delete and recreate resource
kubectl scale deploy/<name> --replicas=3           # Scale deployment to N replicas

# ─────────────────────────────────────────
# DELETING
# ─────────────────────────────────────────
kubectl delete pod <name> -n <ns>                  # Delete a specific pod by name
kubectl delete -f <file>.yaml                      # Delete resource defined in file
kubectl delete all --all -n <ns>                   # Delete everything in namespace
kubectl delete namespace <name>                    # Delete namespace and all contents
kubectl delete pod -l app=<name> -n <ns>           # Delete pods matching a label

# ─────────────────────────────────────────
# DEBUGGING & INSPECTING
# ─────────────────────────────────────────
kubectl describe pod <name> -n <ns>                # Full pod details, events, errors
kubectl logs <pod> -n <ns>                         # View pod logs
kubectl logs <pod> -n <ns> -f                      # Stream logs in real time
kubectl logs <pod> --previous -n <ns>              # Logs from previously crashed container
kubectl exec -it <pod> -n <ns> -- bash             # Open shell inside running pod
kubectl describe node <node>                       # Node capacity, conditions, events
kubectl top pods -n <ns>                           # Live pod CPU and memory usage
kubectl top nodes                                  # Live node CPU and memory usage

# ─────────────────────────────────────────
# ROLLOUTS & RESTARTS
# ─────────────────────────────────────────
kubectl rollout restart deploy/<name> -n <ns>      # Rolling restart with zero downtime
kubectl rollout status deploy/<name> -n <ns>       # Watch rollout progress live
kubectl rollout history deploy/<name> -n <ns>      # Show all revision history
kubectl rollout undo deploy/<name> -n <ns>         # Rollback to previous revision
kubectl rollout undo deploy/<name> --to-revision=2 # Rollback to specific revision number
```

---

## 31. Helm — Package Management

**Helm** is the package manager for Kubernetes. It bundles all Kubernetes manifests into a reusable, versioned package called a **Chart**.

### Why Helm?

| Without Helm | With Helm |
|-------------|-----------|
| `kubectl apply` each file manually | Single `helm install` command |
| Edit every file to change image tag | Change one value in `values.yaml` |
| No rollback mechanism | `helm rollback` in one command |
| Hard to manage dev/prod differences | `--values dev.yaml` or `prod.yaml` |
| Manual cleanup | `helm uninstall` removes everything |

### Chart Structure

```
my-chart/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # Default configuration values
└── templates/          # Kubernetes manifest templates
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    └── secret.yaml
```

### Chart.yaml

```yaml
apiVersion: v2
name: expense-app
description: Expense Tracker Spring Boot + MySQL Application
type: application
version: 1.0.0
appVersion: "1.0.0"
```

### values.yaml

```yaml
app:
  image:
    repository: expense-app
    tag: "latest"
    pullPolicy: Never
  service:
    type: NodePort
    port: 8080
    nodePort: 30080
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"

mysql:
  image:
    repository: mysql
    tag: "8.0"
  storage:
    size: 5Gi
```

### Template Example

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  namespace: {{ .Values.namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: expense-app
  template:
    spec:
      containers:
      - name: expense-app
        image: {{ .Values.app.image.repository }}:{{ .Values.app.image.tag }}
        imagePullPolicy: {{ .Values.app.image.pullPolicy }}
        resources:
          requests:
            memory: {{ .Values.app.resources.requests.memory | quote }}
            cpu: {{ .Values.app.resources.requests.cpu | quote }}
          limits:
            memory: {{ .Values.app.resources.limits.memory | quote }}
            cpu: {{ .Values.app.resources.limits.cpu | quote }}
```

### Helm Commands

```bash
# Install a new release
helm install expense-app ./expense-app --namespace expense-app --create-namespace

# Recommended — install or upgrade
helm upgrade --install expense-app ./expense-app --namespace expense-app --create-namespace

# List all releases
helm list -n expense-app

# Upgrade an existing release
helm upgrade expense-app ./expense-app -n expense-app

# Check release status
helm status expense-app -n expense-app

# View revision history
helm history expense-app -n expense-app

# Rollback to previous revision
helm rollback expense-app 1 -n expense-app

# Uninstall a release
helm uninstall expense-app -n expense-app

# Lint — check chart for errors
helm lint ./expense-app

# Dry run — render templates without deploying
helm install expense-app ./expense-app --dry-run

# Render templates without connecting to cluster
helm template expense-app ./expense-app

# Package chart into .tgz archive
helm package ./expense-app
```

---

## 32. Cluster Setup and Management

### Minikube (Local Development)

Minikube runs a single-node Kubernetes cluster on your local machine using a Docker container or VM.

```bash
# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64 && sudo mv minikube-linux-amd64 /usr/local/bin/minikube

# Start Minikube with Docker driver
minikube start --driver=docker

# Start with WSL folder mount for persistent storage
minikube start --driver=docker \
  --mount=true \
  --mount-string="$HOME/mysql-data:/mnt/data/mysql"

# Check status
minikube status

# Load local Docker image into Minikube
minikube image load my-app:latest

# Open service in browser
minikube service my-service -n my-namespace

# SSH into Minikube node
minikube ssh

# Stop Minikube
minikube stop

# Delete Minikube cluster
minikube delete

# Point shell to Minikube Docker daemon
eval $(minikube docker-env)
```

### Kind (Kubernetes in Docker)

Kind runs Kubernetes clusters using Docker containers as nodes. Supports multi-node clusters locally.

```bash
# Install Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# Create single-node cluster
kind create cluster --name my-cluster

# Create multi-node cluster with config
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

# Load image into Kind cluster
kind load docker-image my-app:latest --name my-cluster

# Delete cluster
kind delete cluster --name my-cluster
```

### kubeadm (Production Cluster Bootstrap)

kubeadm automates the process of setting up a production-grade Kubernetes cluster on bare metal or VMs.

```bash
# ── On all nodes ──────────────────────────────
# Install container runtime (containerd)
apt-get install -y containerd
systemctl enable containerd

# Install kubeadm, kubelet, kubectl
apt-get update && apt-get install -y kubeadm kubelet kubectl
apt-mark hold kubeadm kubelet kubectl

# Disable swap (required by Kubernetes)
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# ── On control plane node only ────────────────
# Initialize the cluster
kubeadm init --pod-network-cidr=192.168.0.0/16

# Configure kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# Install CNI plugin (Calico)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# Get join command for worker nodes
kubeadm token create --print-join-command

# ── On each worker node ───────────────────────
kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### Cluster Comparison

| | Minikube | Kind | kubeadm |
|--|----------|------|---------|
| Purpose | Local dev | Local multi-node | Production |
| Nodes | 1 | Multi (in Docker) | Multi (real VMs) |
| Setup time | 2 minutes | 2 minutes | 30+ minutes |
| Persistence | Limited (Docker) | Limited (Docker) | Full |
| Production ready | No | No | Yes |
| Resources needed | Low | Medium | High |

---

## Quick Reference

### Essential File Checklist for a New App

```
k8s/
├── namespace.yaml
├── configmap.yaml
├── secret.yaml
├── mysql-pv.yaml
├── mysql-pvc.yaml
├── mysql-deployment.yaml
├── mysql-service.yaml
├── app-deployment.yaml
└── app-service.yaml
```

### Apply Order

```
Namespace → ConfigMap → Secret → PV → PVC → MySQL Deployment
→ MySQL Service → App Deployment → App Service
```

### Key Concepts Summary

| Concept | What it is |
|---------|------------|
| Pod | Smallest deployable unit, wraps containers |
| Deployment | Manages stateless pods with rolling updates |
| StatefulSet | Manages stateful pods with stable identity and storage |
| DaemonSet | Runs one pod per node |
| Service | Stable network endpoint for pods |
| Ingress | HTTP routing to services, one load balancer for all |
| ConfigMap | Non-sensitive configuration |
| Secret | Sensitive credentials |
| PV | Actual storage resource |
| PVC | Storage request from a pod |
| StorageClass | How storage is provisioned |
| HPA | Auto-scales replica count |
| VPA | Auto-sizes container resources |
| RBAC | Access control — who can do what |
| Helm | Package manager for Kubernetes |

---

*Generated as part of DevOps learning journey — Docker → Kubernetes → Helm*
