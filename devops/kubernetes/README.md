# Kubernetes in One Shot — Lecture Notes

Oct 8, 2026 · Shubham Patel

English notes from a Hindi one-shot Kubernetes lecture (TrainWithShubham), covering concepts, hands-on commands and projects.

> **Repo layout:** this `README.md` plus an `images/` folder (study roadmap, architecture diagram and the lecture's whiteboard drawings). Push both together so the images show on GitHub. Every diagram is also drawn in Mermaid, which GitHub renders natively.

## Contents

- [Overview: what to study, in order](#overview-what-to-study-in-order)
- [1. What Kubernetes is](#1-what-kubernetes-is-and-why-it-matters)
- [2. Monolith vs microservices](#2-monolith-vs-microservices)
- [3. Architecture](#3-kubernetes-architecture)
- [4. Ways to create a cluster](#4-ways-to-create-a-cluster)
- [5. YAML basics](#5-yaml-basics-and-manifest-structure)
- [6. Namespaces (start to end), pods and kubectl](#6-namespaces-start-to-end-pods-and-kubectl-essentials)
- [7. Workloads](#7-workloads)
- [8. Storage: PV, PVC, StorageClass](#8-storage-pv-pvc-and-storageclass)
- [9. Networking: Services and Ingress](#9-networking-services-and-ingress)
- [10. ConfigMaps and Secrets](#10-configmaps-secrets-and-the-statefulset-example)
- [11. Scheduling and scaling](#11-scheduling-and-scaling)
- [12. RBAC](#12-rbac--role-based-access-control)
- [13. Dashboard, CRDs, operators](#13-dashboard-crds-and-operators)
- [14. Helm](#14-helm--the-package-manager-for-kubernetes)
- [15. Init containers, sidecars, Istio](#15-init-containers-sidecars-and-service-mesh-istio)
- [16. Projects](#16-projects)
- [17. Cheat sheet](#17-cheat-sheet-and-interview-tips)

---

## Overview: what to study, in order

Study Kubernetes in eight layers, each building on the last; the section numbers point to where each topic is covered below.

![Kubernetes study roadmap](images/k8s-study-roadmap.png)

```mermaid
flowchart TD
    A["1 · Prerequisites (§5)<br/>Linux, Docker, YAML"] --> B["2 · Core concepts (§2–6)<br/>Monolith vs microservices, architecture,<br/>setup, kubectl, pods, namespaces,<br/>labels, selectors, annotations"]
    B --> C["3 · Workloads (§7)<br/>Deployments, ReplicaSets, StatefulSets,<br/>DaemonSets, Jobs, CronJobs"]
    C --> D["4 · Networking (§9)<br/>Cluster networking, Services,<br/>Ingress, network policies"]
    D --> E["5 · Storage & config (§8, §10)<br/>PV, PVC, StorageClasses,<br/>ConfigMaps, Secrets"]
    E --> F["6 · Scaling & scheduling (§11)<br/>HPA, VPA, node affinity, taints/tolerations,<br/>resource quotas, limits, probes"]
    F --> G["7 · Admin & advanced (§12–15)<br/>RBAC, monitoring, Helm, CRDs, operators,<br/>init/sidecar containers, service mesh"]
    G --> H["8 · Projects (§16)<br/>Chat app, Prometheus + Grafana, EKS"]
    style H fill:#e6eefb,stroke:#2f6fd6,stroke-width:2px
```

Finish the prerequisites and core concepts before workloads; storage, scaling and admin topics all assume you can already run a Deployment behind a Service.

- [ ] 1. Prerequisites — Linux, Docker, YAML (§5)
- [ ] 2. Core concepts — monolith vs microservices, architecture, setup on local/AWS EC2, kubectl, pods, namespaces, labels, selectors, annotations (§2–6)
- [ ] 3. Workloads — Deployments, StatefulSets, DaemonSets, ReplicaSets, Jobs, CronJobs (§7)
- [ ] 4. Networking — cluster networking, Services, Ingress, network policies (§9)
- [ ] 5. Storage — PV, PVC, StorageClasses, ConfigMaps, Secrets (§8, §10)
- [ ] 6. Scaling and scheduling — HPA, VPA, node affinity, taints/tolerations, resource quotas, limits, probes (§11)
- [ ] 7. Cluster admin and advanced — RBAC, monitoring, Helm, CRDs, operators, init/sidecar containers, service mesh (§12–15)
- [ ] 8. Projects — chat app, Prometheus + Grafana monitoring, EKS (§16)

---

## 1. What Kubernetes is and why it matters

Kubernetes is a **container orchestration tool**: it decides how many containers run, where they run, how they talk to each other, and what happens when one crashes.

**Origin.** Google's own website kept crashing under load and engineers had to bring systems back by hand. Google built an internal system called **Borg** that could self-heal crashed applications and autoscale them under heavy traffic. That design was later open-sourced as **Kubernetes** (around 2014), now roughly 10 years old.

**Name.** K + 8 letters + s = **K8s**. The logo is a ship's steering wheel: Docker's logo is a whale carrying containers, and Kubernetes is the captain steering that ship.

**CNCF.** Kubernetes is a *graduated* project of the **Cloud Native Computing Foundation**, which funds and supports open-source projects such as Kubernetes, Argo CD and Helm.

**Analogy used in the lecture.** A single Docker container is like a startup — one unit that struggles to scale and sometimes goes down. Kubernetes is like a multinational company (MNC): a headquarters that manages many branch offices where the actual work happens.

**Why learn it**

- Most companies moved from monolithic apps to microservices, and microservices need an orchestrator.
- Kubernetes skills are in very high demand for DevOps roles; even freshers with solid basics get hired.
- Core benefits: **auto-healing**, **auto-scaling**, rolling updates, service discovery and load balancing.
- Certifications such as **CKA** (Certified Kubernetes Administrator) are widely sought.
- Prerequisites: Linux basics and Docker.

## 2. Monolith vs microservices

Microservices split one big app into small independent services; Kubernetes is the tool that runs and coordinates them.

```mermaid
flowchart LR
    subgraph Mono["Monolith — one repo, one deploy"]
        direction TB
        L1[Login] --- S1[Signup] --- C1[Cart] --- P1[Products]
    end
    subgraph Micro["Microservices — one service each"]
        direction TB
        L2[Login service]
        S2[Signup service]
        C2[Cart service ✗ down]
        P2[Products service]
    end
    Mono --> X1["One failure can stop the whole app"]
    Micro --> X2["A failure stays inside one service"]
    style C2 stroke:#d64545,stroke-width:2px
```

**Monolith = one big supermarket (D-Mart).** Shampoo, clothes, groceries — everything under one roof. In software: login, signup, cart and products all live in one repository and deploy as one application (e.g. a monolithic amazon.com).

**Microservices = many small specialised shops.** One shop for shoes, one for vegetables, one for clothes. Each feature becomes its own service.

| Aspect | Monolith | Microservices |
| --- | --- | --- |
| Codebase | Single repository | One repo/service per feature |
| Failure | One broken part can take the whole app down | Only the failing service is affected |
| Updates | Small change = redeploy everything | Update and deploy one service independently |
| Cost | Big servers for everything | Small servers for small services (e.g. auth) |
| Management | Hard to manage as it grows | Easier, but needs an orchestrator → Kubernetes |

## 3. Kubernetes architecture

A cluster is one **master node (control plane)** that manages work plus several **worker nodes** where containers actually run. *Node = server; cluster = more than one node.*

**Analogy:** the master node is the company headquarters where managers sit and assign work; worker nodes are branch offices (Pune, Bangalore, Noida) where the real work is done. No application containers run on the master.

![Kubernetes cluster architecture](images/k8s-architecture.png)

*The lecture's whiteboard version — the control plane (scheduler, etcd, API server, controller manager) talks to worker nodes 1–3; on each worker the kubelet runs the containers and the service proxy lets the user reach them; the CNI network spans the whole cluster:*

![Architecture whiteboard from the lecture](images/architecture-whiteboard.png)

```mermaid
flowchart TB
    K["kubectl<br/>(your commands)"] --> API
    subgraph CP["Control plane (master node)"]
        API["API Server<br/>gateway for all communication"]
        SCH["Scheduler<br/>picks a node for each pod"]
        ETCD["etcd<br/>key-value store of cluster state"]
        CM["Controller Manager<br/>keeps the desired state"]
        API --> SCH
        API --> ETCD
        API --> CM
    end
    subgraph W1["Worker node 1"]
        KL1[kubelet] --- KP1[kube-proxy]
        PD1["Pods (containers via containerd)"]
    end
    subgraph W2["Worker node 2"]
        KL2[kubelet] --- KP2[kube-proxy]
        PD2["Pods (containers via containerd)"]
    end
    API --> KL1
    API --> KL2
    KL1 --> PD1
    KL2 --> PD2
    W1 -.- CNI["CNI network (Calico, Weave Net)"]
    W2 -.- CNI
    style API fill:#e6eefb,stroke:#2f6fd6,stroke-width:2px
```

Every request enters through the API server; the scheduler, etcd and controller manager work behind it, while kubelets on the workers actually run the pods.

### Control plane (master) components

| Component | Role | Office analogy |
| --- | --- | --- |
| **API Server** | Gateway for all communication; every component and kubectl talks through it | Team lead |
| **Scheduler** | Decides which worker node a new pod runs on | HR placing a new hire |
| **etcd** | Key-value store holding the whole cluster's state and data | Company records database |
| **Controller Manager** | Watches nodes, pods and the cluster and keeps them in the desired state | Project manager |

### Worker node components

| Component | Role | Office analogy |
| --- | --- | --- |
| **kubelet** | Agent on each worker; ensures pods are running and reports to the API server | Branch manager checking attendance |
| **kube-proxy (service proxy)** | Gives outside users network access to pods, which are otherwise isolated | The insider who passes messages in and out |
| **Container runtime** (containerd) | Actually runs the containers | — |
| **Pod** | Smallest unit; wraps one or more containers | Employee desk |

### Connecting it together

- **CNI (Container Network Interface)** — the network over which nodes talk, e.g. Calico, Weave Net (like Teams/Slack between offices).
- **kubectl** — the CLI you use to give orders to the cluster via the API server (the CEO/director).

**Flow of a pod creation:**

```mermaid
sequenceDiagram
    participant U as kubectl
    participant A as API Server
    participant S as Scheduler
    participant E as etcd
    participant K as kubelet (worker)
    U->>A: kubectl apply -f pod.yml
    A->>E: store desired state
    A->>S: new pod needs a node
    S-->>A: assign to worker N
    A->>K: run this pod
    K->>K: pull image, start container
    K-->>A: pod Running
    A->>E: store status
```

## 4. Ways to create a cluster

For learning, use **kind** or **minikube** on one machine; use **kubeadm** for multi-server clusters; use **EKS/AKS/GKE** for managed clusters in the cloud.

| Method | What it is | Best for |
| --- | --- | --- |
| **kind** (Kubernetes IN Docker) | Each node runs as a Docker container on one machine | Local practice, multi-node on one box |
| **minikube** | Single-node cluster on a laptop or one VM | Quick local learning |
| **kubeadm** | Install on 2+ real servers; `kubeadm init` makes the master, others `join` | Bare-metal / self-managed clusters |
| **EKS / AKS / GKE** | AWS / Azure / Google managed Kubernetes; the cloud runs the control plane | Production in the cloud |
| Others | Rancher RKE (enterprise), Civo, Vultr | Specialised hosting |

**Lab machine used:** AWS EC2, Ubuntu 24.04, x86 architecture, **t2.medium** minimum (t2.micro is too small), 20–30 GB storage, ports 22/80/443 open. Fix SSH key permission with `chmod 400 key.pem`.

### kind setup

1. Install kind and kubectl using the install script, then `chmod +x` the binaries.
2. Install Docker: `sudo apt-get update && sudo apt-get install docker.io -y`.
3. Fix permission denied: `sudo usermod -aG docker $USER && newgrp docker`.
4. Verify: `docker --version`, `kind --version`, `kubectl version`.
5. Write a cluster config and create the cluster.

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.31.2
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
    image: kindest/node:v1.31.2
  - role: worker
    image: kindest/node:v1.31.2
  - role: worker
    image: kindest/node:v1.31.2
```

```bash
kind create cluster --name tws-cluster --config config.yml
kubectl cluster-info --context kind-tws-cluster
kubectl get nodes        # 1 control-plane + 3 workers
```

### minikube setup

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64 && sudo mv minikube-linux-amd64 /usr/local/bin/minikube
minikube start --driver=docker --vm=true
minikube delete
```

**Switching clusters:** installing a new cluster changes kubectl's default context. Use `kubectl config use-context kind-tws-cluster` or add `--context <name>` to a command.

### kubeadm setup (master + worker)

```mermaid
flowchart LR
    A["Both nodes:<br/>swapoff, kernel modules,<br/>sysctl, containerd,<br/>kubelet + kubeadm + kubectl"] --> B["Master:<br/>kubeadm init<br/>copy admin.conf<br/>apply Calico CNI"]
    B --> C["Master:<br/>kubeadm token create<br/>--print-join-command"]
    C --> D["Worker:<br/>kubeadm reset<br/>sudo kubeadm join …"]
    D --> E["Master:<br/>watch kubectl get nodes<br/>→ worker Ready"]
```

1. Launch 2 EC2 instances (t2.medium) in the same security group; open port **6443** (API server, used for joining).
2. **On both:** `swapoff -a`, load kernel modules (overlay, br_netfilter), set sysctl networking params, install **containerd**, then install kubelet, kubeadm and kubectl.
3. **Master only:** `sudo kubeadm init`, copy `/etc/kubernetes/admin.conf` to `~/.kube/config` and chown it, then install the Calico CNI with `kubectl apply -f <calico manifest>`.
4. Generate a join command: `kubeadm token create --print-join-command`.
5. **Worker only:** `sudo kubeadm reset`, then run the join command with `sudo`.
6. On master: `watch kubectl get nodes` until the worker is Ready.

## 5. YAML basics and manifest structure

Everything in Kubernetes is a YAML **manifest**; applying a manifest creates a Kubernetes object (resource). DevOps does involve code — it is YAML.

YAML has three building blocks:

```yaml
name: Shubham            # key: value
courses:                 # list of values
  - DevOps
  - AWS
  - Python
info:                    # object (nested key-values)
  name: Shubham
  age: 28
```

Every manifest follows the same skeleton:

| Field | Meaning | Example |
| --- | --- | --- |
| `apiVersion` | API group/version of the resource | `v1`, `apps/v1`, `batch/v1` |
| `kind` | Type of resource | Pod, Deployment, Service |
| `metadata` | Name, namespace, labels | `name: nginx-pod` |
| `spec` | Desired state / specification | containers, replicas, ports |

**Tips**

- Never memorise YAML; copy templates from the official docs at kubernetes.io (allowed even in the CKA exam).
- Indentation errors are the most common bug; an editor like VS Code with the Kubernetes extension highlights them.
- `kubectl create` creates once; `kubectl apply` creates **and** updates — prefer `apply`.

## 6. Namespaces (start to end), pods and kubectl essentials

A **namespace** is a named group inside one cluster that isolates related resources — like a WhatsApp group: your friends' group and your family group live in the same app but never mix. A **pod** is the smallest unit and wraps one or more containers.

### 6.1 Why namespaces exist

Running NGINX and MySQL on the same cluster gets confusing fast. Each app has its own chain of resources — **container image → Pod → Deployment → Service → user** — and a namespace puts each chain in its own group so one app's resources never disturb another's.

![Each app's resource chain lives in its own namespace](images/namespaces-nginx-mysql.png)

*Lecture whiteboard: the NGINX chain and the MySQL chain each sit inside their own namespace (the blue dashed-square icon).*

```mermaid
flowchart LR
    subgraph NS1["namespace: nginx"]
        direction LR
        I1["Image: nginx"] --> P1[Pod] --> D1[Deployment] --> S1[Service]
    end
    subgraph NS2["namespace: mysql"]
        direction LR
        I2["Image: mysql"] --> P2[Pod] --> D2["Deployment / StatefulSet"] --> S2[Service]
    end
    S1 --> U1((User))
    S2 --> U2((App / user))
```

What a namespace gives you:

- **Isolation and grouping** — names only need to be unique *within* a namespace (two apps can each have a Service called `web`).
- **Easy clean-up** — `kubectl delete ns <name>` removes every resource inside it.
- **Access control** — RBAC Roles and RoleBindings are namespace-scoped (see §12).
- **Resource limits per team or app** — ResourceQuota and LimitRange apply per namespace (see 6.7).
- **Environments** — e.g. `dev-apache` and `prod-apache` from the same Helm chart (see §14).

### 6.2 Built-in namespaces

```bash
kubectl get ns          # or: kubectl get namespaces
```

| Namespace | What lives there |
| --- | --- |
| `default` | Anything you create without specifying a namespace ("whoever has no one, has God") |
| `kube-system` | Control-plane and system components: etcd, API server, scheduler, controller manager, CoreDNS, kube-proxy, metrics-server |
| `kube-public` | Publicly readable cluster info |
| `kube-node-lease` | Node heartbeat (lease) objects |
| `local-path-storage` | kind's local storage provisioner (kind clusters only) |

```bash
kubectl get pods -n kube-system   # see the control-plane pods
kubectl get pods                  # "No resources found in default namespace" on a fresh cluster
```

### 6.3 Create a namespace

Imperative (quick, from the command line):

```bash
kubectl create ns nginx
```

Declarative (recommended — keep it in Git as a manifest):

```yaml
# namespace.yml
kind: Namespace
apiVersion: v1
metadata:
  name: nginx
```

```bash
kubectl apply -f namespace.yml
```

### 6.4 Put resources into a namespace

Two ways — in the manifest (preferred) or on the command line:

```yaml
metadata:
  name: nginx-pod
  namespace: nginx      # resource lands in the nginx namespace
```

```bash
kubectl run nginx --image=nginx -n nginx     # -n / --namespace on the command line
```

**Common mistake from the lecture:** `kubectl run nginx --image=nginx` without `-n` created the pod in `default`, so `kubectl get pods -n nginx` showed nothing. Delete it and recreate it with `-n nginx`.

### 6.5 Work inside a namespace

```bash
kubectl get pods -n nginx                    # pods in one namespace
kubectl get all -n nginx                     # pods, deployments, replicasets, services…
kubectl get pods -A                          # every namespace (--all-namespaces)
kubectl describe pod nginx-pod -n nginx
kubectl logs nginx-pod -n nginx
kubectl exec -it nginx-pod -n nginx -- bash

# stop typing -n every time: make nginx the default for your current context
kubectl config set-context --current --namespace=nginx
kubectl config view --minify | grep namespace
```

### 6.6 Namespaced vs cluster-scoped resources

Not everything belongs to a namespace.

| Namespaced (need `-n`) | Cluster-scoped (no namespace) |
| --- | --- |
| Pod, Deployment, ReplicaSet, StatefulSet, DaemonSet | Node |
| Job, CronJob | Namespace itself |
| Service, Ingress | PersistentVolume (PV) |
| ConfigMap, Secret | StorageClass |
| PersistentVolumeClaim (PVC) | ClusterRole, ClusterRoleBinding |
| ServiceAccount, Role, RoleBinding | CustomResourceDefinition (CRD) |
| HPA, VPA, ResourceQuota, LimitRange | IngressClass |

```bash
kubectl api-resources --namespaced=true     # list namespaced kinds
kubectl api-resources --namespaced=false    # list cluster-scoped kinds
```

> **Storage note:** the lecture added `namespace:` to the PV manifest. A PV is cluster-scoped, so Kubernetes ignores that field. What must match is the **PVC's** namespace and the **pod's** namespace — the "pvc not found" error came from the PVC being in `default` while the pod was in `nginx`.

### 6.7 Limit what a namespace can use (ResourceQuota and LimitRange)

The syllabus lists **resource quotas**: caps on the total a namespace may consume, so one team can't starve the cluster.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: nginx-quota
  namespace: nginx
spec:
  hard:
    pods: "10"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
---
apiVersion: v1
kind: LimitRange          # default requests/limits for containers that don't set them
metadata:
  name: nginx-limits
  namespace: nginx
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 200m
        memory: 256Mi
```

```bash
kubectl describe resourcequota nginx-quota -n nginx    # used vs hard limits
```

Once a quota sets CPU/memory limits, every new pod in that namespace must declare requests and limits (or get them from a LimitRange), otherwise it is rejected.

### 6.8 Talking across namespaces

Services get a DNS name that includes their namespace:

```
<service>.<namespace>.svc.cluster.local
e.g. http://apache-service.apache.svc.cluster.local
```

Inside the same namespace the short name (`apache-service`) is enough; from another namespace use `apache-service.apache`. Namespaces separate **names**, not the network — pods can still reach each other across namespaces unless a **NetworkPolicy** blocks it.

### 6.9 Namespaces used in this lecture

| Namespace | What ran there |
| --- | --- |
| `nginx` | NGINX pod/deployment/service, Jobs, CronJobs, PV/PVC, Ingress, notes app |
| `mysql` | MySQL StatefulSet, headless Service, ConfigMap, Secret |
| `apache` | Apache deployment, HPA/VPA demos, RBAC Role/ServiceAccount |
| `dev-apache`, `prod-apache` | Helm releases of the same chart |
| `mongo` | MongoDB installed from a Helm chart |
| `chat-app` | Chat app project (frontend, backend, MongoDB) |
| `monitoring` | Prometheus + Grafana stack |
| `kubernetes-dashboard` | Dashboard and its admin-user |
| `ingress-nginx` | NGINX ingress controller |
| `kube-system` | Control plane, metrics-server |

### 6.10 Delete a namespace (end of life)

```bash
kubectl delete ns nginx          # or: kubectl delete -f namespace.yml
kubectl get ns                   # shows Terminating, then it's gone
```

Everything inside — pods, deployments, services, secrets, ingresses — is deleted with it, so it is the fastest way to clean up a whole app. It takes a few seconds because each resource is removed first. Cluster-scoped objects (like a PV) are **not** removed; delete them separately.

**Namespace lifecycle at a glance:**

```mermaid
flowchart LR
    A["Create<br/>kubectl create ns / apply -f namespace.yml"] --> B["Fill<br/>apply manifests with<br/>metadata.namespace or -n"]
    B --> C["Use<br/>get / describe / logs / exec -n<br/>set-context --namespace"]
    C --> D["Govern<br/>ResourceQuota, LimitRange,<br/>Role + RoleBinding"]
    D --> E["Delete<br/>kubectl delete ns<br/>→ Terminating → gone"]
```

### 6.11 Pod

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: nginx-pod
  namespace: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

### 6.12 Essential kubectl commands

```bash
kubectl get ns                                   # list namespaces
kubectl create ns nginx                          # imperative create
kubectl run nginx --image=nginx -n nginx         # quick pod
kubectl apply -f pod.yml                         # declarative create/update
kubectl get pods -n nginx -o wide                # which node each pod is on
kubectl describe pod nginx-pod -n nginx          # events, debugging
kubectl logs <pod> -n nginx                      # container logs
kubectl exec -it nginx-pod -n nginx -- bash      # shell inside the pod
kubectl delete -f pod.yml                        # delete
```

**Pod states to know for interviews:**

```mermaid
stateDiagram-v2
    [*] --> Pending
    Pending --> ContainerCreating
    ContainerCreating --> Running
    ContainerCreating --> ImagePullBackOff: bad image/tag
    Running --> Completed: task finished (Job)
    Running --> CrashLoopBackOff: keeps crashing
    Running --> Terminating: deleted
    Terminating --> [*]
```

`kubectl describe` shows the full story: scheduler assigns the pod to a node → kubelet pulls the image → container starts.

## 7. Workloads

A single pod cannot handle real traffic (think Big Billion Day sale), so workload controllers create and manage **replicas** (identical copies) of a pod.

A **ReplicationController** manages pod replicas. Its modern variants differ in *how* they manage those replicas:

```mermaid
flowchart TB
    RC["Replication controller<br/>(keeps N copies of a pod)"]
    RC --> RS["ReplicaSet<br/>exactly N replicas"]
    RC --> DEP["Deployment<br/>N replicas + rolling updates, rollback"]
    RC --> STS["StatefulSet<br/>numbered pods app-0, app-1 + own storage"]
    RC --> DS["DaemonSet<br/>one pod on every node"]
    J["Job<br/>run once to completion"] --> CJ["CronJob<br/>Job on a schedule"]
    DEP -. manages .-> RS
```

| Workload | What it guarantees | Typical use |
| --- | --- | --- |
| **ReplicaSet** | Exactly N replicas running | Rarely used directly |
| **Deployment** | N replicas + **rolling updates**, rollback, scaling, self-healing | Stateless apps (Django, Flask, Spring Boot, frontends) |
| **StatefulSet** | Replicas with stable, numbered identities (`app-0`, `app-1`…) and per-pod storage | Databases (MySQL, MongoDB) |
| **DaemonSet** | At least one pod on **every** node (like a bhandara: everyone gets food) | Log/metrics agents, node-level tools |
| **Job** | Runs a task to completion, then stops | Backups, patching, one-off scripts |
| **CronJob** | Runs a Job on a schedule | Periodic backups, reports |

### Labels and selectors

A pod gets a **label** (`app: nginx`); the controller's **selector** (`matchLabels: app: nginx`) finds pods with that label. The pod `template` inside a Deployment is just a pod definition with its labels.

### Deployment

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nginx-deployment
  namespace: nginx
spec:
  replicas: 2
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
          image: nginx:latest
          ports:
            - containerPort: 80
```

```bash
kubectl scale deployment nginx-deployment -n nginx --replicas=5
kubectl set image deployment/nginx-deployment -n nginx nginx=nginx:1.27.3   # rolling update
kubectl rollout status deployment/nginx-deployment -n nginx
kubectl rollout undo deployment/nginx-deployment -n nginx
```

**Rolling update:** pods are replaced a few at a time, so some old pods keep serving while new ones start — no downtime. In the demo, a wrong image tag gave `ImagePullBackOff` on new pods while old pods kept running. A ReplicaSet has the same YAML (`kind: ReplicaSet`) but no rolling updates or rollbacks.

```mermaid
flowchart LR
    A["v1 v1 v1 v1"] --> B["v2 v1 v1 v1"] --> C["v2 v2 v1 v1"] --> D["v2 v2 v2 v1"] --> E["v2 v2 v2 v2"]
```

### DaemonSet

Same YAML as a ReplicaSet with `kind: DaemonSet` and **no `replicas` field**; with 3 worker nodes you automatically get 3 pods, one per node.

### Job

```yaml
kind: Job
apiVersion: batch/v1
metadata:
  name: demo-job
  namespace: nginx
spec:
  completions: 1
  parallelism: 1
  template:
    metadata:
      labels:
        app: batch-task
    spec:
      containers:
        - name: batch-container
          image: busybox:latest
          command: ["sh", "-c", "echo Hello Dosto && sleep 10"]
      restartPolicy: Never
```

`busybox` is an image used just to run commands. After it finishes the pod shows **Completed**; re-running requires delete + apply.

### CronJob

Cron schedule fields: `minute hour day-of-month month day-of-week`; `* * * * *` = every minute. Use crontab.guru to build expressions.

```yaml
kind: CronJob
apiVersion: batch/v1
metadata:
  name: minute-backup
  namespace: nginx
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: busybox
              command: ["sh", "-c", "echo Backup started; mkdir -p /backups && cp -r /demo-data /backups; echo Backup completed"]
          restartPolicy: OnFailure
```

The lecture's full version also mounts `hostPath` volumes for `/demo-data` and `/backups`. Common mistakes it hit: `restartPolicy` belongs inside the pod spec, `volumes` sits beside `containers`, and labels go under `metadata`.

## 8. Storage: PV, PVC and StorageClass

Pods are disposable: when a pod is deleted, its data goes with it (the Deployment recreates the pod, but empty). To keep data, it must be **persisted on the host** through volumes.

![Pod data is persisted through a volume to the host](images/storage-pod-volume-pv-pvc-host.png)

*Lecture whiteboard: each pod in the Deployment holds data; the pod mounts a volume (`my-vol` at the app's data path), which is backed by a PVC → PV on the host, carved out of the host machine's 30 GB disk at `/mnt/data`.*

```mermaid
flowchart LR
    H["Host node disk<br/>(e.g. 30 GB)<br/>path /mnt/data"] -- reserves --> PV["PersistentVolume<br/>1Gi carved out<br/>status: Available"]
    PV -- claimed by --> PVC["PersistentVolumeClaim<br/>requests 1Gi<br/>status: Bound"]
    PVC -- mounted into --> POD["Pod<br/>/usr/share/nginx/html<br/>data outlives the pod"]
    SC["StorageClass<br/>(local / cloud / network)"] -.-> PV
    style PVC fill:#e6eefb,stroke:#2f6fd6,stroke-width:2px
```

| Object | What it is | Analogy |
| --- | --- | --- |
| **PersistentVolume (PV)** | A slice of the host's storage reserved for the cluster (e.g. 1 GB of the 30 GB disk) | A bottle placed on the table — available |
| **PersistentVolumeClaim (PVC)** | A request that claims storage from a PV; status becomes **Bound** | Writing your name on the bottle |
| **StorageClass** | The type of storage (local, cloud, network) | Hard disk vs pen drive vs DVD |

![A 1Gi slice of the 30 GB disk becomes an Available PV, which the PVC claims](images/storage-pv-pvc-claim-1gi.png)

*Lecture whiteboard: 1Gi of the host's 30 GB disk at `/mnt/data` is exposed as a PersistentVolume (status **Available**); the PVC **claims** it, after which the PV shows **Bound**.*

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: local-pv
  namespace: nginx
  labels:
    app: local
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: /mnt/data
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: local-pvc
  namespace: nginx
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-storage
```

The pod uses the claim:

```yaml
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          volumeMounts:
            - name: my-volume
              mountPath: /usr/share/nginx/html
      volumes:
        - name: my-volume
          persistentVolumeClaim:
            claimName: local-pvc
```

**Debugging lessons from the lecture**

- Pod stuck `Pending` with "pvc not found" → the PVC was in `default` while the pod was in `nginx`. A PVC must be in the same namespace as the pod that uses it (a PV is cluster-scoped — see §6.6).
- A PV in `Released` state cannot be reused; delete and recreate it so it shows `Available`.
- With kind, nodes are Docker containers, so the host path lives *inside* the worker container: `docker exec -it <worker-container> bash` then `cd /mnt/data`.

## 9. Networking: Services and Ingress

A **Service** exposes a group of pods (selected by label) at a stable address; an **Ingress** routes HTTP paths like `/nginx` and `/` to different Services.

```mermaid
flowchart LR
    U["User / browser"] --> IC["Ingress controller (NGINX)<br/>rules from the Ingress object"]
    IC -- "/nginx" --> S1["nginx-service<br/>ClusterIP :80"]
    IC -- "/" --> S2["notes-app-service<br/>ClusterIP :8000"]
    S1 -- "selector app: nginx" --> D1["nginx-deployment<br/>pod · pod"]
    S2 -- "selector app: notes-app" --> D2["notes-app deployment<br/>pod · pod"]
```

### Service

```yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx-service
  namespace: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80          # port the Service exposes
      targetPort: 80    # port the container listens on
  type: ClusterIP
```

| Service type | Meaning |
| --- | --- |
| **ClusterIP** (default) | Internal IP reachable only inside the cluster |
| **NodePort** | Opens a port (30000–32767) on every node |
| **LoadBalancer** | Provisions a cloud load balancer |
| **ExternalName / external IP** | Points to an outside address or static IP |
| **Headless** (`clusterIP: None`) | No virtual IP; used with StatefulSets |

**In-cluster DNS:** any pod can reach a Service at `http://<service>.<namespace>.svc.cluster.local`.

**Exposing on kind/EC2:** nodes are containers, so forward the port and open it in the EC2 security group.

```bash
sudo -E kubectl port-forward service/nginx-service -n nginx 81:80 --address=0.0.0.0
```

### Mini project: Django notes app

1. `docker build -t notes-app .`, `docker login`, `docker image tag notes-app trainwithshubham/notes-app-k8s:latest`, `docker push …` — Kubernetes pulls images from a registry such as Docker Hub.
2. Write `namespace.yml`, `deployment.yml` (image from Docker Hub, port 8000) and `service.yml` (port/targetPort 8000).
3. Apply them, port-forward 8000, open port 8000 in the security group.

The journey: **image → pod → deployment → service → user**.

![Container image to pod to deployment to service to user](images/app-journey-image-pod-deployment-service.png)

*Lecture whiteboard: the same chain works for any app — NGINX on top, your own app (e.g. the notes app) below; only the container image changes.*

```mermaid
flowchart LR
    A[Dockerfile] --> B[Image on Docker Hub] --> C[Pod] --> D[Deployment] --> E[Service] --> F[Ingress] --> G[User]
```

### Ingress

Ingress needs an **ingress controller** (here the NGINX ingress controller, installed from the kind docs with `kubectl apply -f <ingress-nginx manifest>`; on minikube use `minikube addons enable ingress`). Services must be in the same namespace as the Ingress.

```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: nginx-notes-ingress
  namespace: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
          - path: /nginx
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: notes-app-service
                port:
                  number: 8000
```

The `rewrite-target: /` annotation strips the path prefix so `/nginx` reaches NGINX's root page. Then port-forward the ingress controller's Service: `kubectl port-forward svc/ingress-nginx-controller -n ingress-nginx 8080:80 --address=0.0.0.0`.

## 10. ConfigMaps, Secrets and the StatefulSet example

Keep configuration out of big workload files: plain variables go in a **ConfigMap**, passwords go in a **Secret** (base64-encoded).

```mermaid
flowchart LR
    CMAP["ConfigMap<br/>MYSQL_DATABASE=devops"] -- configMapKeyRef --> ENV
    SEC["Secret<br/>MYSQL_ROOT_PASSWORD (base64)"] -- secretKeyRef --> ENV
    ENV["env vars in container"] --> STS["StatefulSet mysql<br/>mysql-statefulset-0 / -1 / -2"]
    HS["Headless Service<br/>mysql-service (clusterIP: None)"] --- STS
    VCT["volumeClaimTemplates<br/>1Gi per pod"] --- STS
```

**Stateless vs stateful:** web apps (Django, Flask, Spring Boot) are stateless — use Deployments. Databases (MySQL, MongoDB) are stateful — use **StatefulSets**, whose pods keep fixed names (`mysql-statefulset-0`, `-1`, `-2`). Delete pod `-0` and a new pod with the *same name and state* comes back.

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: mysql-config-map
  namespace: mysql
data:
  MYSQL_DATABASE: devops
---
kind: Secret
apiVersion: v1
metadata:
  name: mysql-secret
  namespace: mysql
data:
  MYSQL_ROOT_PASSWORD: cm9vdA==     # echo -n root | base64
```

Referencing them in the StatefulSet container:

```yaml
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: mysql-config-map
                  key: MYSQL_DATABASE
```

The StatefulSet also sets `serviceName: mysql-service` (a **headless** Service with `clusterIP: None`, port 3306) and a `volumeClaimTemplates` block (1Gi, ReadWriteOnce) at the same indentation level as `template`. Each replica is reachable as `mysql-statefulset-0.mysql-service`.

**Note:** base64 is *encoding, not encryption* — `base64 --decode` reveals it instantly. It exists so secrets can be stored as binary data, not to hide them. Watch case: `configMapKeyRef`, `secretKeyRef`.

## 11. Scheduling and scaling

These controls decide how much a pod may consume, whether it is healthy, which node it lands on, and how it grows under load.

### Resource requests and limits

On a t2.medium (2 vCPU, 4 GB RAM) one runaway pod could eat everything. **Requests** = the minimum a pod needs to start; **limits** = the maximum it may use ("Mummy, give me just two rotis").

```yaml
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

### Probes

Probes are internal health-check requests to a pod.

| Probe | Question it answers |
| --- | --- |
| **Liveness** | Is the container still alive? (restart if not) |
| **Readiness** | Is it ready to receive traffic? |
| **Startup** | Has it finished starting up? |

```yaml
          livenessProbe:
            httpGet:
              path: /
              port: 8000
          readinessProbe:
            httpGet:
              path: /
              port: 8000
```

You can add `initialDelaySeconds` and `periodSeconds`. In the demo the readiness probe failed because the app wasn't reachable on the cluster IP; on a cloud cluster it passes.

### Taints and tolerations

A **taint** on a node repels pods (like a relative who taunts you — you stop visiting). A **toleration** on a pod lets it run on a tainted node anyway.

```mermaid
flowchart LR
    P1["Pod without toleration"] -- "✗ blocked" --> N1["Node tainted<br/>prod=true:NoSchedule"]
    P2["Pod with toleration<br/>prod=true:NoSchedule"] -- "✓ allowed" --> N1
    P1 -- "✓ allowed" --> N2["Untainted node"]
    P3["Pod with node affinity<br/>(zone = X)"] -- "attracted to" --> N3["Node in zone X"]
```

```bash
kubectl taint node tws-cluster-worker prod=true:NoSchedule     # add taint
kubectl taint node tws-cluster-worker prod=true:NoSchedule-    # remove (note the -)
```

```yaml
  tolerations:
    - key: prod
      operator: Equal
      value: "true"
      effect: NoSchedule
```

The control-plane node is tainted by default, so app pods never run there. If every worker is tainted, new pods stay `Pending`; pods already running are not evicted by `NoSchedule`.

### Node affinity

The opposite of a taint: it *attracts* a pod to nodes matching labels/expressions (e.g. a specific zone) — "this pod must run here". Also called a node selector.

### HPA and VPA

```mermaid
flowchart TB
    MS["metrics-server<br/>(kubectl top)"] --> HPA & VPA
    subgraph HPA["HPA — scale out"]
        direction LR
        h1[pod] --> h2["pod pod pod pod pod<br/>(more replicas)"]
    end
    subgraph VPA["VPA — scale up"]
        direction LR
        v1["pod 25m CPU"] --> v2["same pod 109m CPU<br/>(bigger resources)"]
    end
```

| | Horizontal Pod Autoscaler | Vertical Pod Autoscaler |
| --- | --- | --- |
| What changes | Number of pod replicas | CPU/memory of the same pod |
| Best for | Stateless apps | Stateful apps (databases) |
| apiVersion | `autoscaling/v2` | `autoscaling.k8s.io/v1` |

Both need the **metrics server**. On minikube: `minikube addons enable metrics-server`. On kind: apply the metrics-server manifest, then `kubectl -n kube-system edit deployment metrics-server` and add `--kubelet-insecure-tls` and `--kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP` to the container args, then `kubectl rollout restart`. Check with `kubectl top node` and `kubectl top pod -n <ns>`.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apache-hpa
  namespace: apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 5
```

**Load test used:** a busybox pod hitting the Service's DNS name in a loop.

```bash
kubectl run -i --tty load-generator --image=busybox -n apache -- /bin/sh
while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done
```

Pods scaled from 1 up to 5 under load and back down when it stopped. For VPA, clone `kubernetes/autoscaler`, run `./hack/vpa-up.sh` in `vertical-pod-autoscaler`, and create a `VerticalPodAutoscaler` with `targetRef` and `updatePolicy.updateMode: Auto` (or `Initial` / `Off`). Under load its CPU recommendation rose from 25m to 109m while the pod count stayed at 1.

**KEDA** (Kubernetes Event-Driven Autoscaling) picks scaling behaviour from events and metrics; it is used less often.

## 12. RBAC — Role-Based Access Control

RBAC controls who may do what to which resources. A **role** lists allowed actions; a **binding** gives that role to a user or ServiceAccount. Without it, any user (even a new intern) could delete production Deployments.

**Household analogy:** "bring vegetables" is a role; you (Shubham) or your sister can do it only once the role is *bound* to you.

```mermaid
flowchart LR
    subgraph NS["Namespace scope"]
        SA["ServiceAccount<br/>apache-user"] -- RoleBinding --> R["Role apache-manager<br/>apiGroups + resources + verbs"]
    end
    subgraph CL["Cluster scope"]
        U["User / ServiceAccount<br/>admin-user"] -- ClusterRoleBinding --> CR["ClusterRole<br/>cluster-admin"]
    end
    R --> RES1["pods, deployments,<br/>services in one namespace"]
    CR --> RES2["resources in every namespace"]
```

| Scope | Who | Permissions | Link |
| --- | --- | --- | --- |
| Namespace | ServiceAccount (a lightweight user) | **Role** | **RoleBinding** |
| Whole cluster | User / ServiceAccount | **ClusterRole** | **ClusterRoleBinding** |

A rule has three parts: **apiGroups** (`""` = core group for pods/services, `apps` = deployments, `batch` = jobs), **resources** (pods, deployments, services) and **verbs** (get, list, watch, create, apply, patch, update, delete).

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: apache-manager
  namespace: apache
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "deployments", "services"]
    verbs: ["get", "list", "apply", "delete"]
---
kind: ServiceAccount
apiVersion: v1
metadata:
  name: apache-user
  namespace: apache
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: apache-manager-rolebinding
  namespace: apache
subjects:
  - kind: ServiceAccount
    name: apache-user
    namespace: apache
roleRef:
  kind: Role
  name: apache-manager
  apiGroup: rbac.authorization.k8s.io
```

**Testing permissions**

```bash
kubectl auth whoami
kubectl auth can-i get pods -n apache
kubectl auth can-i get pods --as=system:serviceaccount:apache:apache-user -n apache
kubectl auth can-i delete pods --as=apache-user -n apache
```

Always re-`apply` the Role after editing it — the lecture's "no" answers came from forgetting this. Removing `delete` from verbs makes `can-i delete` return **no**.

## 13. Dashboard, CRDs and operators

### Kubernetes Dashboard

A web UI showing every workload, service, config and storage object across namespaces.

1. `kubectl apply -f <kubernetes dashboard recommended.yaml>` (creates its namespace, ServiceAccount, Roles, ClusterRole, Deployment, Service).
2. Create an `admin-user` ServiceAccount in `kubernetes-dashboard` and a **ClusterRoleBinding** to the built-in `cluster-admin` ClusterRole.
3. Get a login token: `kubectl -n kubernetes-dashboard create token admin-user`.
4. `kubectl proxy --port=8001 --address=0.0.0.0 --accept-hosts='.*'`, then open `/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`.
5. Sign-in only works over HTTPS or **localhost**, so the lecture ran it on a local machine.

### Custom Resource Definitions (CRDs)

Pods, Services and Deployments are built-in resources. A **CRD** lets you define your own resource type with its own schema — the lecture invented a `DevOpsBatch` resource.

```mermaid
flowchart LR
    CRD["CustomResourceDefinition<br/>group trainwithshubham.com<br/>kind DevOpsBatch + schema"] --> CR1["DevOpsBatch: junoon-batch-9"]
    CRD --> CR2["DevOpsBatch: junoon-batch-10"]
    OP["Operator (Go / Python Kopf)<br/>acts on custom resources"] -.watches.-> CR1 & CR2
```

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: devopsbatches.trainwithshubham.com
spec:
  group: trainwithshubham.com
  names:
    plural: devopsbatches
    singular: devopsbatch
    kind: DevOpsBatch
    shortNames: ["dob"]
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                name: {type: string}
                duration: {type: string}
                mode: {type: string}
                platform: {type: string}
```

A custom resource then uses `apiVersion: trainwithshubham.com/v1`, `kind: DevOpsBatch`, and `kubectl get devopsbatches` lists them.

### Operators and the Kubernetes API

**Operators** are programs (Go, or Python via frameworks like Kopf) that manage custom resources — e.g. the Argo CD and Prometheus operators. CRDs, operators and the raw API are rarely asked in interviews; Helm, service mesh, security and image scanning are.

**Image scanning:** Trivy or **Docker Scout** (built into Docker Desktop) list critical/medium/low vulnerabilities in an image.

## 14. Helm — the package manager for Kubernetes

Helm packages all of an app's manifests into a reusable **chart**, so you set values instead of rewriting Deployment, Service, Ingress and HPA YAML for every app and environment. It is to Kubernetes what `apt` is to Ubuntu.

```mermaid
flowchart LR
    V["values.yaml<br/>replicaCount, image, ports"] --> T["templates/<br/>deployment, service,<br/>ingress, hpa"]
    T --> CH["Chart (helm package)"]
    CH -- "helm install dev-apache" --> DEV["namespace dev-apache"]
    CH -- "helm install prod-apache" --> PROD["namespace prod-apache"]
    PROD -- "helm upgrade / rollback" --> PROD
```

**Install:** download `get_helm.sh`, `chmod 700`, run it; check `helm version`.

**Chart layout** (`helm create apache-helm`):

```
apache-helm/
├── Chart.yaml        # name, version, description
├── values.yaml       # values injected into templates
├── charts/
└── templates/        # deployment.yaml, service.yaml, ingress.yaml, hpa.yaml, serviceaccount.yaml…
```

Templates use placeholders such as `{{ .Values.replicaCount }}` and `{{ .Values.service.port }}`. To add a field (e.g. `targetPort`), add `{{ .Values.service.targetPort }}` in the template and the value in `values.yaml`. In `values.yaml` the lecture set `replicaCount: 2`, `image.repository: httpd`, `tag: "2.4"`, and service `port`/`targetPort: 80`.

### Lifecycle

```bash
helm package apache-helm
helm install dev-apache apache-helm -n dev-apache --create-namespace
helm install prod-apache apache-helm -n prod-apache --create-namespace
helm upgrade prod-apache apache-helm -n prod-apache      # after editing values (e.g. replicas 3) and bumping Chart version
helm rollback prod-apache 1 -n prod-apache
helm uninstall dev-apache -n dev-apache
helm list -n <ns>
```

One chart, many environments: dev, staging and prod become separate releases in separate namespaces.

### Public charts

```bash
helm repo add <name> <url>
helm repo update
helm search repo nginx
helm install mongodb-helm oci://registry-1.docker.io/bitnamicharts/mongodb -n mongo --create-namespace
```

Artifact Hub lists ready charts for NGINX, MongoDB, Argo CD, Prometheus and more. The lecture installed MongoDB in under a minute and opened `mongosh` inside its pod.

## 15. Init containers, sidecars and service mesh (Istio)

### Init vs sidecar containers

Both live in the same pod as the **main container** and help it; they differ in timing.

```mermaid
flowchart LR
    subgraph Init["Pod with init container"]
        direction LR
        I1["init-container<br/>runs first, must finish"] --> M1["main-container<br/>starts after"]
    end
    subgraph Side["Pod with sidecar"]
        direction LR
        M2["main-container<br/>writes /var/log/app.log"] <-- "shared emptyDir volume" --> S2["sidecar<br/>tail -f the log"]
    end
```

| | Init container | Sidecar container |
| --- | --- | --- |
| When it runs | **Before** the main container; must finish first | **Alongside** the main container, the whole time |
| YAML key | `initContainers:` | an extra entry under `containers:` |
| Typical use | Wait for MySQL to be up, create a folder, set up config | Ship/display logs, copy data, proxy traffic |
| Analogy | Prep work before opening the shop | Robin beside Batman |

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: init-test
spec:
  initContainers:
    - name: init-container
      image: busybox:latest
      command: ["sh", "-c", "echo Initialization started; sleep 10; echo Initialization completed"]
  containers:
    - name: main-container
      image: busybox:latest
      command: ["sh", "-c", "echo Main container started"]
```

While the init container runs, `kubectl get pods` shows `Init:0/1`; read each container's logs with `kubectl logs init-test -c init-container`.

Sidecar example: the main container writes `Hello Dosto` to `/var/log/app.log` every 5 seconds; the sidecar runs `tail -f` on the same file. Both mount a shared `emptyDir` volume named `shared-logs` at `/var/log`. Real use: a banking app's init container waits until MySQL is reachable before the app starts.

### Service mesh with Istio

In a microservice app (e.g. Zomato: menu, order, events, delivery, hotel services), services constantly call each other. A **service mesh** manages and visualises that service-to-service traffic, like a traffic policeman at a busy junction. **Istio** is the most popular one.

- **Envoy** — a sidecar proxy injected into every pod; handles ingress/egress and service-to-service traffic.
- **istiod** — Istio's control plane: service discovery, proxy configuration, certificates.
- **Kiali** — dashboard that draws the traffic graph between services.

```mermaid
flowchart LR
    GW["bookinfo gateway"] --> PP["productpage (Python)"]
    PP --> DT["details (Ruby)"]
    PP --> RV["reviews v1/v2/v3 (Java)"]
    RV --> RT["ratings (Node.js)"]
    ISTIOD["istiod control plane"] -.configures Envoy sidecars.-> PP & DT & RV & RT
    KIALI["Kiali dashboard"] -.draws this graph.-> GW
```

```bash
curl -L https://istio.io/downloadIstio | sh -
sudo mv istio-1.24.1/bin/istioctl /usr/local/bin/
istioctl install -f samples/bookinfo/demo-profile-no-gateways.yaml -y
kubectl label namespace default istio-injection=enabled
kubectl apply -f <Gateway API CRDs>
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/gateway-api/bookinfo-gateway.yaml
kubectl port-forward svc/bookinfo-gateway-istio 8080:80
for i in $(seq 1 100); do curl -s -o /dev/null http://localhost:8080/productpage; done
kubectl apply -f samples/addons && istioctl dashboard kiali
```

The **Bookinfo** sample has productpage (Python), details (Ruby), reviews (Java, three versions) and ratings (Node.js). After 100 requests, Kiali's traffic graph showed productpage → details and reviews → ratings — a call path you can't see from the UI alone.

## 16. Projects

The lecture's projects each ran on a different cluster type.

| Project | Stack | Cluster |
| --- | --- | --- |
| Chat app (3-tier) | React + Node.js + MongoDB | minikube (local) |
| Voting app + monitoring | Python, .NET, Node.js, Redis, Postgres; Prometheus + Grafana via Helm | kind on EC2 |
| Wanderlust mega project | 3-tier app; Jenkins CI, Argo CD GitOps, Trivy, OWASP, SonarQube, monitoring | AWS EKS (continued in a separate free video) |

### Project 1: Chat app on minikube

```mermaid
flowchart LR
    U["User → chat-tws.com"] --> ING["Ingress"]
    ING -- "/" --> FE["frontend Service :80<br/>React + NGINX"]
    ING -- "/api" --> BE["backend Service :5001<br/>Node.js"]
    FE -- "proxies /api to host 'backend'" --> BE
    BE -- "MONGODB_URI" --> DB["mongodb Service :27017"]
    DB --- PVC["PVC 5Gi → PV hostPath /data"]
    SEC["Secret: JWT_SECRET"] -.-> BE
```

1. Fork and clone the full-stack chat app; build and push `chatapp-backend` and `chatapp-frontend` images to Docker Hub. MongoDB uses the official `mongo` image.
2. Create namespace `chat-app`.
3. **MongoDB:** PV + PVC (5Gi, `hostPath: /data`), Deployment (port 27017, root username/password env vars, volume from the PVC) and Service `mongodb`.
4. **Backend:** Deployment (port 5001) with env vars `NODE_ENV=production`, `MONGODB_URI=mongodb://mongoadmin:secret@mongodb:27017/dbname?authSource=admin`, `JWT_SECRET` (from a Secret, base64-encoded) and `PORT` (value as a string, "5001"); Service named **`backend`**.
5. **Frontend:** Deployment (port 80) and Service `frontend`.
6. **Ingress:** host `chat-tws.com`, `/` → frontend:80, `/api` → backend:5001; enable with `minikube addons enable ingress`; add `127.0.0.1 chat-tws.com` to `/etc/hosts`.

**Debugging lessons:** the frontend pod hit `CrashLoopBackOff` because its NGINX config proxies `/api` to a host called `backend` — it only started once a Service with exactly that name existed. The backend failed until `MONGODB_URI` pointed at the `mongodb` Service. Read the app's code and config, not just the YAML.

### Project 2: Monitoring with Prometheus and Grafana

**Observability has three pillars:** metrics → *what* is happening (monitoring: Prometheus, Grafana, Datadog), logs → *why* it happened (Loki, Promtail), traces → *how* it happened (Jaeger, OpenTelemetry).

```mermaid
flowchart LR
    subgraph Cluster["Kubernetes cluster"]
        NE["Node Exporter<br/>one per node :9100<br/>CPU, RAM, network"]
        KSM["kube-state-metrics<br/>API server, scheduler,<br/>pods, deployments"]
    end
    NE -- scrape --> PROM["Prometheus<br/>time-series DB, PromQL"]
    KSM -- scrape --> PROM
    PROM -- data source --> GRAF["Grafana<br/>dashboards"]
    GRAF --> OPS["DevOps engineer<br/>spots the hot node → scale"]
```

- **Node Exporter** runs on every node (here 4 pods for 4 nodes) and exports CPU, RAM and network on port 9100.
- **kube-state-metrics** exposes the state of cluster objects and control-plane components (API server, scheduler, controller manager, kubelet, kube-proxy) — no cAdvisor needed.
- **Prometheus** is a time-series database that scrapes all of these and is queried with PromQL.
- **Grafana** visualises Prometheus data as dashboards.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring \
  --set prometheus.service.nodePort=30000 --set grafana.service.nodePort=31000 \
  --set prometheus.service.type=NodePort --set grafana.service.type=NodePort
kubectl port-forward svc/prometheus-stack-kube-prom-prometheus -n monitoring 9090:9090 --address=0.0.0.0
kubectl port-forward svc/prometheus-stack-grafana -n monitoring 3000:80 --address=0.0.0.0
kubectl get secret prometheus-stack-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
```

The Grafana password is not admin/admin — decode it from the Secret (it was `prom-operator`). The Prometheus data source and many dashboards come pre-configured. Build panels by namespace (e.g. `container_network_receive_bytes_total` filtered by namespace) or import community dashboards from grafana.com by ID. The voting app (cats vs dogs) was then deployed in `default`, load was generated, and dashboards showed which worker node took the most load — the signal to scale.

### Project 3: EKS setup (start of the mega project)

```mermaid
flowchart LR
    A["aws configure<br/>IAM user keys, ap-south-1"] --> B["eksctl create cluster<br/>--without-nodegroup<br/>(CloudFormation, 10–20 min)"]
    B --> C["associate IAM OIDC provider"]
    C --> D["eksctl create nodegroup<br/>2 × t2.medium, 20 GB"]
    D --> E["aws eks update-kubeconfig<br/>kubectl get nodes"]
```

1. Install the AWS CLI (`unzip` needed) and run `aws configure` with an IAM user's access key and secret (AdministratorAccess for the demo), region `ap-south-1` (Mumbai).
2. Install `eksctl`.
3. `eksctl create cluster --name=tws-cluster --region=ap-south-1 --version=1.31 --without-nodegroup` (10–20 minutes; built through CloudFormation).
4. `eksctl utils associate-iam-oidc-provider --region ap-south-1 --cluster tws-cluster --approve`.
5. `eksctl create nodegroup --cluster=tws-cluster --region=ap-south-1 --name=tws-cluster-ng --node-type=t2.medium --nodes=2 --nodes-min=2 --nodes-max=2 --node-volume-size=20 --ssh-access --ssh-public-key=<key-pair-name>`.
6. `aws eks update-kubeconfig --region ap-south-1 --name tws-cluster`, then `kubectl get nodes` shows the 2 workers. On EKS, AWS manages the control plane, so no master node appears.

The rest of the mega project (Jenkins CI, Argo CD, monitoring on EKS) was moved to a separate free video because YouTube caps uploads at 12 hours.

## 17. Cheat sheet and interview tips

| Task | Command |
| --- | --- |
| Cluster info | `kubectl cluster-info`, `kubectl get nodes` |
| Switch cluster | `kubectl config use-context <ctx>` |
| Everything in a namespace | `kubectl get all -n <ns>` |
| Debug a pod | `kubectl describe pod <pod> -n <ns>` |
| Logs (one container) | `kubectl logs <pod> -c <container> -n <ns>` |
| Shell into a pod | `kubectl exec -it <pod> -n <ns> -- bash` |
| Watch changes | `watch kubectl get pods -n <ns>` |
| Scale | `kubectl scale deploy <name> --replicas=N -n <ns>` |
| Rolling update / undo | `kubectl set image …`, `kubectl rollout undo …` |
| Resource usage | `kubectl top node`, `kubectl top pod -n <ns>` |
| Expose locally | `kubectl port-forward svc/<svc> <host>:<svc-port> --address=0.0.0.0` |
| Permissions | `kubectl auth can-i <verb> <resource> --as=<user> -n <ns>` |
| Delete a namespace and all inside | `kubectl delete ns <ns>` |

**Interview checklist**

- Draw and explain the architecture: API server, scheduler, etcd, controller manager, kubelet, kube-proxy, CNI, kubectl.
- Know where containers run (workers only) and the pod lifecycle states.
- Explain Deployment vs ReplicaSet vs StatefulSet vs DaemonSet, Job vs CronJob, with use cases.
- Service types, Ingress and the ingress controller; PV vs PVC vs StorageClass; ConfigMap vs Secret (base64 is not encryption).
- HPA vs VPA, requests vs limits, probes, taints/tolerations vs node affinity, RBAC objects.
- Helm, service mesh, image scanning and monitoring come up often; CRDs, operators and the raw API rarely do.
- You won't be asked to write YAML from memory — definitions and use cases matter more.