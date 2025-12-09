# Interview_qna_k8s

This repository provides a clear, concise, and interview-ready Q&A guide on Kubernetes fundamentals and architecture. Perfect for CKA, CKAD, or DevOps interviews.

## Q&A for Kubernetes Interview

### 1. Explain Kubernetes Architecture

**Question**: Can you explain the Kubernetes architecture and its core components?

**Answer**:

Kubernetes has two main planes:

#### **Control Plane (Master Nodes)**
Responsible for managing the cluster state and orchestration.

| Component             | Responsibility                                                                                          |
|-----------------------|----------------------------------------------------------------------------------------------------------|
| **kube-apiserver**    | Front-end of the control plane. All communication (kubectl, pods, users) goes through the API server. Handles authentication, authorization, validation, and CRUD operations on objects. |
| **etcd**              | Distributed key-value store — the only stateful component. Stores all cluster data (pods, services, configmaps, secrets, etc.). Highly available and consistent. |
| **kube-scheduler**    | Watches for newly created pods without a node assignment. Decides which node a pod should run on based on resource requirements, node affinity/anti-affinity, taints/tolerations, etc. |
| **kube-controller-manager** | Runs controllers in a single process: <br>• ReplicaSet controller <br>• Deployment controller <br>• StatefulSet controller <br>• DaemonSet controller <br>• Job/CronJob controller etc. Ensures desired state matches actual state. |
| **cloud-controller-manager** (optional) | Integrates with cloud providers (AWS, GCP, Azure) for LoadBalancers, Volumes, Nodes, Routes, etc. |

#### **Data Plane (Worker Nodes)**
Where your actual workloads (containers) run.

| Component             | Responsibility                                                                                          |
|-----------------------|----------------------------------------------------------------------------------------------------------|
| **kubelet**           | Agent running on each node. Communicates with API server. Ensures containers in pods are running and healthy. Interacts with container runtime (containerd, CRI-O, Docker). |
| **kube-proxy**        | Maintains network rules on nodes (iptables or IPVS). Enables service abstraction and load balancing. Watches Services and Endpoints, updates routing rules so traffic reaches correct pods. |
| **Container Runtime** | Software that actually runs containers (e.g., containerd, CRI-O). Must be CRI-compliant. |

### 2. How do various Kubernetes components interact?

**Question**: Walk me through what happens when you run `kubectl create deployment web --image=nginx`

**Answer**:
```text
1. kubectl → kube-apiserver (via HTTPS + authentication)
2. API server validates request → stores Deployment object in etcd
3. Controller Manager (Deployment Controller) detects new Deployment
4. Creates ReplicaSet with desired replicas
5. ReplicaSet Controller detects new ReplicaSet → creates Pods
6. Scheduler watches for unscheduled pods → selects best node → updates Pod spec with nodeName → writes back via API server → stored in etcd
7. kubelet on selected node sees pod scheduled to it → asks container runtime to pull nginx image and start container
8. kubelet reports pod status back to API server (Running)
9. If a pod dies → ReplicaSet controller notices mismatch → creates new pod → cycle repeats
```

### 3. What is the purpose of a Service in Kubernetes?

**Question**: Why do we need Services in Kubernetes?

**Answer**:
A **Service** provides stable networking for a set of pods.

Key purposes:
- **Service Discovery**: Pods get a stable DNS name (e.g., `my-svc.my-namespace.svc.cluster.local`)
- **Load Balancing**: Traffic is automatically distributed across healthy pods (round-robin by default)
- **Decoupling**: Consumers talk to the Service, not individual pod IPs
- **Pod IP Impermanence**: Pod IPs change on restart — Service IP and DNS name remain constant

Without a Service, you would have to hardcode pod IPs — which is fragile and anti-pattern.

### 4. Why is hardcoding Pod IPs a bad practice?

**Question**: A developer hardcodes Pod B’s IP in Pod A. Why is this a bad idea?

**Answer**:
Kubernetes pods are **ephemeral**:
- Pod IP changes on every restart/recreation
- Pods can be rescheduled to different nodes
- During scaling or rolling updates, old pods are terminated → new IPs assigned

**Result**: Hardcoded IPs break communication immediately after any change.

**Correct way**: Use a **Service** with labels/selectors:
```yaml
# Pod B
labels:
  app: backend

# Service selects pods with label app=backend
selector:
  app: backend
```
Now Pod A talks to `backend-service` DNS name — always works, even if pods change.

### 5. What are the different types of Services in Kubernetes?

**Question**: Explain the different Service types.

**Answer**:

| Type            | Use Case                                    | Accessibility                         | Notes                                      |
|-----------------|---------------------------------------------|----------------------------------------|--------------------------------------------|
| **ClusterIP**   | Default type                                | Only inside cluster                    | Gets internal cluster DNS & IP             |
| **NodePort**    | Expose app on each node’s IP + static port  | From outside via `<NodeIP>:<NodePort>` | Port range 30000–32767                     |
| **LoadBalancer**| Expose publicly via cloud provider LB      | Public internet                        | Creates cloud LB (AWS ELB, GCP, etc.)      |
| **ExternalName**| Map service to external DNS name            | N/A (returns CNAME)                    | No proxying or load balancing              |
| **Headless**    | `clusterIP: None`                           | Direct pod IPs returned in DNS         | Used by StatefulSets, custom discovery     |

### 6. What are Labels and Selectors?

**Question**: Explain Labels and Selectors in Kubernetes.

**Answer**:
- **Labels**: Key-value pairs attached to objects (pods, services, etc.) for identification and grouping.
  ```yaml
  labels:
    app: frontend
    env: production
    tier: web
  ```

- **Selectors**: Used by controllers/services to select pods based on labels.
  ```yaml
  selector:
    app: frontend
    tier: web
  ```

Used by:
- Deployments/ReplicaSets → to manage pods
- Services → to route traffic to correct pods
- NetworkPolicies, HPA, etc.

**Best Practice**: Always label your pods meaningfully!

### 7. NodePort vs LoadBalancer Service — which would you recommend and why?

**Question**: When would you choose NodePort vs LoadBalancer?

**Answer**:

| Criteria               | NodePort                            | LoadBalancer (Recommended)               |
|------------------------|-------------------------------------|-------------------------------------------|
| Exposure               | External via `<NodeIP>:<NodePort>`  | Public IP / DNS name from cloud provider  |
| Cost                   | Free                                | Costs money (ELB/ALB/NLB)                 |
| High Availability      | Manual (need external LB)           | Automatic across AZs                      |
| TLS Termination        | Not built-in                        | Supported (via ALB)                       |
| Cloud Integration      | Works everywhere                    | Requires cloud controller manager         |
| Production Readiness   | Only for testing/dev                | Yes — production standard                 |

**Recommendation**:  
Use **LoadBalancer** in production (with cloud provider).  
Use **NodePort** only in bare-metal, Minikube, or quick testing scenarios.

For bare-metal/on-prem clusters → use **Ingress** with MetalLB or external LB.

## Conclusion

Understanding Kubernetes architecture, component interaction, Services, Labels/Selectors, and Service types is fundamental for any Kubernetes certification (CKA/CKAD) or production role. These answers are structured exactly how interviewers expect — clear, accurate, and with real-world reasoning.
