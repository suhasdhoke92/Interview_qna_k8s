# KubernetesInterviewQ&A

This repository provides a comprehensive Q&A guide for preparing for Kubernetes-related interview questions. It covers Kubernetes architecture, components interaction, services, labels/selectors, service types, kube-proxy, headless services, cross-namespace access, network policies, deployment strategies, and rollback approaches.

## Table of Contents

 1. Explain Kubernetes Architecture
 2. How do various Kubernetes components interact with each other?
 3. What is the purpose of a Service in Kubernetes?
 4. Why is hardcoding Pod IPs a bad practice?
 5. What are the different types of Services in Kubernetes?
 6. How are Kubernetes Services related to kube-proxy?
 7. What is the disadvantage of using LoadBalancer-type Services?
 8. What is a Headless Service in Kubernetes, and when do you use it?
 9. What are Labels and Selectors in Kubernetes?
 10. NodePort vs LoadBalancer Service — which would you recommend?
 11. Can a Pod access a Service in a different namespace?
 12. How do you restrict access to a database Pod so only one application can access it?
 13. Explain the deployment strategy followed in your organization?
 14. Explain the rollback strategy followed in your organization?
 15. How would you design a solution to avoid rollbacks?
 16. Explain the deployment strategies that you used in the past?
 17. Explain the role of CoreDNS in k8s?
 18. A DevOps engineer tainted a node as "NoSchedule". Can you still schedule a Pod?
 19. Pod is stuck in CrashLoopBackOff?
 20. What is the difference between liveness and readiness probes?
 21. Explain the difference between Ingress and LoadBalancer service type.
 22. Your app works with ClusterIP but fails with Ingress — how do you troubleshoot it?
 23. Why do I need to setup ingress controller after creating ingress?
 24. We have an in-house LB, can we use ingress with our load balancer?
 25. Your deployment has replicas: 3 but only 1 pod is running — what could be wrong?
 26. Your pod mounts a ConfigMap but changes to the ConfigMap are not reflected?
 27. Explain concept of node affinity?
 28. What is the difference between node affinity and node label selector?
 29. What is container runtime in Kubernetes?
 30. What is k8s QoS (Quality of Service)?
 31. K8s request and limits
 32. Can we use k8s master for scheduling the pods?
 33. Explain horizontal vs vertical scaling?
 34. Types of secrets in k8s? And why are we have different types?
 35. challenges that you faced while workingon k8s.
     35.1 Resource shareing - multiple NS, (Dev / Qa / Prod), (leaking memory) - **Resource quota** (limit allocate at NS) for NS 
     35.2 **Resource limit** - (Resource **request** and **limit**) one perticular pod consuming high leaking memory 
     35.3 OOM killed issue with pod. - kil -3 for **thread dump** , jstack - **heap dump** in java lang. shared with developer.
     35.4 pod eviction issue on one of the node
     35.5 upgrade cluster. - First take the backup, Release Notes, control plane-> ETCD, API, Scheduler, worker plane-> cordon
36. how many types of networks exist in Kubernetes? -> Pod Network (Cluster Network), Service Network(ClusterIP Network), Node Network (Underlying Infrastructure) , Optional / Advanced Networks , Ingress / External Networking


## Q&A for Kubernetes Interview

### 1. Explain Kubernetes Architecture

**Question**: Explain the Kubernetes architecture and its core components.

**Answer**:

**Control Plane (Master Node Components)**

| Component             | Responsibility                                                                                           |
|-----------------------|-----------------------------------------------------------------------------------------------------------|
| **kube-apiserver**    | Front-end of the control plane. All requests (from `kubectl`, UI, CI/CD) go through the API server. Handles authentication, authorization, validation, and serves the Kubernetes API. |
| **etcd**              | Highly available key-value store. Acts as the cluster brain — stores all cluster state and configuration (Pods, Services, Deployments, etc.). |
| **kube-scheduler**    | Watches for newly created Pods without a node assignment and decides which node should run the Pod based on resource requirements, node affinity/anti-affinity, taints/tolerations, etc. |
| **kube-controller-manager** | Runs controllers in a single process: Node controller, ReplicaSet controller, Deployment controller, Endpoints controller, etc. Ensures desired state matches actual state. |

**Data Plane (Worker Node Components)**

| Component             | Responsibility                                                                                           |
|-----------------------|-----------------------------------------------------------------------------------------------------------|
| **kubelet**           | Agent on each worker node. Communicates with the API server, ensures containers in Pods are running and healthy. Uses container runtime (containerd, CRI-O, Docker) to start/stop containers. |
| **kube-proxy**        | Maintains network rules on nodes. Watches Services and Endpoints, updates iptables or IPVS rules so traffic to a Service’s ClusterIP is forwarded to the correct backend Pods. |
| **Container Runtime** | Actually runs containers (containerd, CRI-O, Docker). |

### 2. How do various Kubernetes components interact with each other?

**Question**: Walk through the flow when you run `kubectl create deployment nginx --image=nginx`.

**Answer**:
1. `kubectl` sends the request to **kube-apiserver**.
2. **kube-apiserver** authenticates & authorizes the request, validates the YAML, then stores the Deployment object in **etcd**.
3. **controller-manager** (Deployment controller) detects the new Deployment, creates a ReplicaSet.
4. **controller-manager** (ReplicaSet controller) sees the ReplicaSet and creates the desired number of Pods, storing them in **etcd**.
5. **kube-scheduler** watches for unscheduled Pods, evaluates node constraints, and assigns a node → updates Pod spec with `nodeName` in **etcd**.
6. **kubelet** on the assigned node sees the new Pod via the API server watch, asks the container runtime to create the container(s).
7. Once containers are running, **kubelet** updates Pod status in **etcd** via API server.
8. If a Pod dies, the ReplicaSet controller (in controller-manager) detects the mismatch and creates a new Pod → loop continues.

### 3. What is the purpose of a Service in Kubernetes?

**Question**: Why do we need Services in Kubernetes?

**Answer**:
A Service provides stable networking for Pods:

- **Service Discovery**: Pods get a stable DNS name (e.g., `my-svc.my-namespace.svc.cluster.local`) instead of using ephemeral Pod IPs.
- **Load Balancing**: Traffic to the Service’s ClusterIP is automatically load-balanced across all healthy backend Pods (round-robin by default).
- **Decoupling**: Frontend Pods talk to the Service name, not individual Pod IPs. Even if Pods are recreated with new IPs, communication continues without changes.

### 4. Why is hardcoding Pod IPs a bad practice?

**Question**: Convince a developer who hardcodes Pod IPs that it’s a bad idea.

**Answer**:
Pods are **ephemeral** — they can be terminated, rescheduled, scaled, or evicted at any time. When a Pod restarts, it gets a **new IP address**.

- Hardcoding Pod IPs will break the application the moment a Pod restarts.
- You lose automatic load balancing and failover.
- It defeats the entire purpose of Kubernetes’ self-healing and scaling capabilities.

**Correct approach**: Use a **Service** with labels/selectors. The Service provides a stable DNS name and automatically routes traffic to healthy Pods — no code changes required even when Pods come and go.

### 5. What are the different types of Services in Kubernetes?

| Type            | Use Case                                      | External Access? | DNS Name Example                                      |
|-----------------|-----------------------------------------------|------------------|--------------------------------------------------------|
| **ClusterIP**   | Internal communication only (default)        | No               | `my-svc.my-namespace.svc.cluster.local`               |
| **NodePort**    | Expose service on each node’s IP + static port| Yes (via node IP)| `node-ip:3XXXX`                                        |
| **LoadBalancer**| Cloud-provider external load balancer         | Yes              | Cloud LB public IP/DNS                                 |
| **ExternalName**| Map service to external DNS name              | N/A              | Returns CNAME to external name                         |
| **Headless**    | (`clusterIP: None`) Direct access to Pods     | No               | Individual A records: `pod-name.my-svc.my-namespace...`|

### 6. How are Kubernetes Services related to kube-proxy?

**Question**: Explain the relationship between Services, Endpoints, and kube-proxy.

**Answer**:
- A **Service** has selectors that match Pod labels.
- Kubernetes automatically creates an **Endpoints** (or EndpointSlice) object listing the IPs of ready Pods matching the selector.
- **kube-proxy** on every node watches the API server for Service and Endpoints objects.
- kube-proxy programs **iptables** (or IPVS) rules on the node so that:
  - Traffic sent to the Service’s ClusterIP is forwarded to one of the backend Pod IPs.
  - Traffic is load-balanced across healthy Pods (round-robin).
- **Flow**:  
  `Client → Service ClusterIP → kube-proxy → iptables/IPVS rules → Pod IP`

Without kube-proxy, the Service ClusterIP would be unreachable.

### 7. What is the disadvantage of using LoadBalancer-type Services?

**Question**: Why do we prefer Ingress over multiple LoadBalancer services?

**Answer**:
- Each `LoadBalancer` service provisions a **separate cloud load balancer** (e.g., AWS ELB/ALB, GCP GLB).
- If you have 10 services → 10 separate load balancers → **very expensive**.
- Harder to manage TLS certificates, domain routing, path-based routing.
- In on-premises environments without a Cloud Controller Manager, LoadBalancer services stay in `<pending>` forever.

**Solution**: Use a single **Ingress controller** (NGINX, Traefik, ALB Ingress) with one LoadBalancer/Ingress resource that routes traffic to many Services based on host/path.

### 8. What is a Headless Service in Kubernetes, and when do you use it?

**Question**: Explain Headless Service and give a real-world use case.

**Answer**:
A Headless Service is created by setting `clusterIP: None`.

- No ClusterIP is allocated.
- No load balancing by kube-proxy.
- DNS returns **A records** pointing directly to individual Pod IPs.
  - Example: `pod-0.my-svc.my-namespace.svc.cluster.local → 10.244.1.5`

**Use Cases**:
- **StatefulSets** (e.g., MySQL, MongoDB replica sets, Kafka, Zookeeper) — each member needs direct access to others.
- Applications that implement their own client-side load balancing or sharding (e.g., Cassandra).
- Debugging or direct Pod-to-Pod communication without proxying.

**Example**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None      # This makes it headless
  selector:
    app: mysql
  ports:
    - port: 3306
```

### 9. What are Labels and Selectors in Kubernetes?

**Question**: Explain Labels and Selectors.

**Answer**:
- **Labels**: Key-value pairs attached to objects (Pods, Services, Deployments) for identification and grouping.
  ```yaml
  labels:
    app: nginx
    env: production
    tier: frontend
  ```
- **Selectors**: Used by controllers (Deployments, ReplicaSets, Services) to identify which Pods they manage.
  ```yaml
  selector:
    matchLabels:
      app: nginx
      env: production
  ```

**Use Cases**:
- Services select backend Pods.
- Deployments/ReplicaSets manage Pods.
- `kubectl get pods -l app=nginx,env=prod`

### 10. NodePort vs LoadBalancer Service — which would you recommend?

**Question**: When would you choose NodePort over LoadBalancer, or vice versa?

**Answer**:
| Scenario                         | Recommended Service Type | Reason                                                                 |
|----------------------------------|--------------------------|------------------------------------------------------------------------|
| Internal testing / dev cluster   | NodePort                 | No cost, works everywhere (even on-premises)                           |
| Production external access       | LoadBalancer + Ingress   | Cloud-native LB, public IP/DNS, integrates with Ingress for path routing|
| Bare-metal / on-prem             | NodePort or Ingress with MetalLB | LoadBalancer won't provision without cloud provider integration      |
| Cost-sensitive environment       | NodePort or Ingress      | Avoid paying for multiple cloud LBs                                    |

**Best Practice**: Use **Ingress** (with one LoadBalancer) instead of many LoadBalancer services.

### 11. Can a Pod access a Service in a different namespace?

**Question**: Can a Pod in one namespace access a Service in another namespace?

**Answer**:
Yes, Services are accessible across namespaces by default using their fully qualified domain name (FQDN).

- **FQDN Format**: `<service-name>.<namespace>.svc.cluster.local`
- **Example**:
  - Service `db-service` in namespace `database`.
  - Pod in namespace `frontend` can access it via:
    ```
    db-service.database.svc.cluster.local:3306
    ```
- **Short Name (Same Namespace)**: If in the same namespace, just `<service-name>` works.
- **DNS Search Domains**: Kubernetes adds search domains, so sometimes `db-service.database` suffices.

- **Additional Notes**:
  - Network Policies can restrict cross-namespace traffic if applied.
  - Ensure DNS resolution is working (`CoreDNS` pods healthy).

### 12. How do you restrict access to a database Pod so only one application can access it?

**Question**: How would you allow only a specific application Pod to access a database Pod in the same namespace?

**Answer**:
By default, all Pods in a namespace can communicate. To restrict access, use **Network Policies**.

- **Steps**:
  1. Label the database Pod:
     ```yaml
     metadata:
       labels:
         app: db
     ```
  2. Label the application Pod:
     ```yaml
     metadata:
       labels:
         app: myapp
     ```
  3. Create a NetworkPolicy on the database:
     ```yaml
     apiVersion: networking.k8s.io/v1
     kind: NetworkPolicy
     metadata:
       name: db-access-policy
       namespace: my-namespace
     spec:
       podSelector:
         matchLabels:
           app: db
       policyTypes:
         - Ingress
       ingress:
         - from:
             - podSelector:
                 matchLabels:
                   app: myapp
           ports:
             - protocol: TCP
               port: 3306
     ```
- **Effect**: Only Pods with label `app: myapp` can send traffic to the database Pod on port 3306. All other traffic is denied.

- **Additional Notes**:
  - Requires a CNI plugin supporting Network Policies (e.g., Calico, Weave Net).
  - Network Policies are deny-by-default; explicit allow rules are needed.

### 13. Explain the deployment strategy followed in your organization?

**Question**: What deployment strategies do you use in production, and why?

**Answer**:
In production, we avoid direct rollouts due to risk. We use:

1. **Canary Deployments**:
   - Roll out the new version to a small percentage of users (e.g., 10%).
   - Monitor metrics (errors, latency) for a period.
   - Gradually increase traffic: 10% → 20% → 50% → 100%.
   - Implementation:
     - Create separate Services/Ingress for old and new versions.
     - Use Ingress annotations (e.g., NGINX `canary` weight) or Istio traffic splitting to route percentage-based traffic.
     - Example with NGINX Ingress:
       ```yaml
       metadata:
         annotations:
           nginx.ingress.kubernetes.io/canary: "true"
           nginx.ingress.kubernetes.io/canary-weight: "10"
       ```

2. **Blue-Green Deployments**:
   - Deploy the new version (green) alongside the old (blue).
   - Switch traffic instantly via Load Balancer or Ingress routing.
   - Rollback by switching back to blue.

- **Why**:
  - Minimizes impact: Issues affect only a subset of users.
  - Allows quick rollback without downtime.
  - Supports A/B testing and feature flags.

### 14. Explain the rollback strategy followed in your organization?

**Question**: How do you handle rollbacks in production?

**Answer**:
We follow a GitOps approach with ArgoCD or Helm for deployments:

- **Rollback Process**:
  - Revert the Helm chart or manifest version in Git to the previous stable commit.
  - Since ArgoCD auto-sync is enabled, it detects the change and applies the previous version to the cluster.
  - For Helm: `helm rollback <release-name> <revision>`.

- **Prevention**:
  - Use canary deployments to catch issues early with minimal user impact.
  - Monitor with Prometheus/Grafana during rollout.
  - Automated tests (smoke, integration) in staging before production.

- **Goal**: Minimize rollback frequency by validating changes gradually.

### 15. How would you design a solution to avoid rollbacks?

**Question**: What strategies do you use to minimize or avoid rollbacks?

**Answer**:
To avoid rollbacks:

1. **Progressive Delivery**:
   - **Canary Deployments**: Expose new version to 10% of traffic, monitor, then increase.
   - **Feature Flags**: Enable/disable features without redeploying.

2. **Extensive Pre-Production Testing**:
   - CI/CD with unit, integration, and end-to-end tests.
   - Staging environment mirroring production.

3. **Observability**:
   - Real-time monitoring (Prometheus, Grafana) and alerting.
   - Distributed tracing (Jaeger) to catch issues early.

4. **GitOps with Auto-Sync**:
   - Changes are reviewed and merged via PRs.
   - ArgoCD syncs only validated manifests.

- **Result**: Issues are caught before full rollout, reducing the need for rollbacks.


### 16. Explain the deployment strategies that you used in the past?

**Question**: Explain the deployment strategies that you used in the past.

**Answer**:
In past projects, I have used:

1. **Blue-Green Deployments**:
   - Deploy the new version (green) alongside the current version (blue).
   - Switch traffic instantly by updating the Load Balancer or Service routing to point to green.
   - **Advantages**: Zero downtime, instant rollback by switching back to blue.
   - **Rollback**: Simple — repoint traffic to the old version.

2. **Canary Deployments**:
   - Gradually shift traffic to the new version (e.g., 10% → 20% → 50% → 100%).
   - Monitor metrics during each phase.
   - **Implementation**: Use Ingress controllers with canary annotations or tools like Istio/Flagger for traffic splitting.
   - **Advantages**: Limits blast radius; issues affect only a small user subset.

- **Why These**:
  - Blue-Green for low-risk, instant switches.
  - Canary for safer, gradual rollouts with real-user feedback.
  - Combined with monitoring (Prometheus) and automated tests.

### 17. Explain the role of CoreDNS in k8s?

**Question**: Explain the role of CoreDNS in Kubernetes.

**Answer**:
CoreDNS is the default DNS server in Kubernetes clusters, responsible for DNS resolution.

- **Key Functions**:
  - Translates Service names to ClusterIPs (e.g., `my-svc.my-namespace.svc.cluster.local` → `10.96.0.1`).
  - Handles Pod DNS for headless services (A records to Pod IPs).
  - Forwards external DNS queries via upstream servers (e.g., `/etc/resolv.conf`).
  - Supports plugins for custom resolution (e.g., rewrite, cache).

- **Example**:
  - A Pod accesses `payment.default.svc.cluster.local:8080`.
  - CoreDNS resolves `payment.default.svc.cluster.local` to the Service’s ClusterIP.

- **Configuration**:
  - Defined in the `coredns` ConfigMap in `kube-system` namespace.
  - Runs as a Deployment with replicas for high availability.

- **Troubleshooting**:
  - Check CoreDNS Pods: `kubectl get pods -n kube-system -l k8s-app=kube-dns`.
  - Inspect logs for resolution failures.


### 18. A DevOps engineer tainted a node as "NoSchedule". Can you still schedule a Pod?

**Question**: A node is tainted with `NoSchedule`. Can you schedule a Pod on it?

**Answer**:
By default, **no** — the taint repels Pods, and the scheduler will avoid placing new Pods on the node.

- **Exception**: Add a **toleration** to the Pod spec to allow scheduling:
  ```yaml
  spec:
    tolerations:
      - key: "app"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
  ```

- **Commands**:
  - Apply taint:
    ```bash
    kubectl taint nodes node1 app=true:NoSchedule
    ```
  - Pod with toleration will schedule on the node; others will not.

- **Use Cases**:
  - Reserve nodes for specific workloads (e.g., GPU nodes).
  - Drain nodes for maintenance.

### 19. Pod is stuck in CrashLoopBackOff?

**Question**: A Pod is in CrashLoopBackOff state. How would you troubleshoot?

**Answer**:
CrashLoopBackOff means the container exits repeatedly.

- **Steps**:
  1. **Check Logs**:
     ```bash
     kubectl logs <pod-name> --previous  # Logs from last crashed container
     kubectl logs <pod-name> -f         # Follow current logs
     ```
     Look for application errors (e.g., missing config, crash on startup).

  2. **Describe Pod**:
     ```bash
     kubectl describe pod <pod-name>
     ```
     Check events for reasons (e.g., OOMKilled, ImagePullBackOff).

  3. **Liveness Probe Issues**:
     - Misconfigured liveness probe (e.g., wrong path `/healthz` instead of `/health`).
     - Fix probe path, initial delay, or thresholds:
       ```yaml
       livenessProbe:
         httpGet:
           path: /health
           port: 8080
         initialDelaySeconds: 30
         periodSeconds: 10
       ```

- **Common Causes**:
  - Application crash on startup.
  - Wrong command/args in Pod spec.
  - Resource limits causing OOM.

### 20. What is the difference between liveness and readiness probes?

**Question**: Explain the difference between liveness and readiness probes.

**Answer**:

| Probe Type     | Purpose                                                                 | Failure Action                                      |
|----------------|-------------------------------------------------------------------------|-----------------------------------------------------|
| **Liveness**   | Checks if the container is alive and healthy.                           | Container is restarted if probe fails.              |
| **Readiness**  | Checks if the container is ready to serve traffic.                      | Pod is removed from Service endpoints if probe fails (no restart). |

- **Liveness Probe**:
  - Detects deadlocks or crashed apps.
  - Example: HTTP GET `/healthz` returns 200 → alive.
  - Failure → Kubernetes restarts the container.

- **Readiness Probe**:
  - Ensures Pod is ready (e.g., connected to DB, loaded config).
  - Failure → Traffic not sent to Pod (removed from Service).
  - Pod remains running.

- **Configuration Example**:
  ```yaml
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 15
    periodSeconds: 10
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 5
  ```

- **Use Both**: Liveness for restarts, readiness for traffic control.

### 21. Explain the difference between Ingress and LoadBalancer service type.

**Question**: Explain the difference between Ingress and LoadBalancer-type Services.

**Answer**:
Both expose applications externally, but differ in cost, flexibility, and use cases.

- **LoadBalancer Service**:
  - Provisions a dedicated cloud load balancer (e.g., AWS ALB/ELB) for each Service.
  - Unique external IP/DNS per Service.
  - **Disadvantages**:
    - Expensive (one LB per Service → high cost for many Services).
    - Limited routing (no host/path-based rules natively).
    - Works only in clouds with Cloud Controller Manager.

- **Ingress**:
  - Uses a single Ingress controller (e.g., NGINX, Traefik) with one LoadBalancer.
  - Routes traffic to multiple Services based on host (e.g., app1.example.com) or path (e.g., /api).
  - **Advantages**:
    - Cost-effective (one LB for many Services).
    - Advanced routing, TLS termination, rewrites.
    - Works with various backends (ALB, NLB, etc.).
  - **Example**:
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: my-ingress
    spec:
      rules:
        - host: app1.example.com
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: app1-svc
                    port:
                      number: 80
        - host: app2.example.com
          http:
            paths:
              - path: /
                backend:
                  service:
                    name: app2-svc
                    port:
                      number: 80
    ```

- **Recommendation**: Use **Ingress** for production to save costs and enable advanced routing.


### 22. Your app works with ClusterIP but fails with Ingress — how do you troubleshoot it?

**Question**: Your app works with ClusterIP but fails with Ingress — how do you troubleshoot it?

**Answer**:
When an application works fine via ClusterIP but fails through Ingress, follow this troubleshooting sequence:

1. **Verify Ingress Controller is installed and healthy**  
   - Check if the Ingress controller pods (nginx-ingress, traefik, ALB Ingress, etc.) are running:  
     ```bash
     kubectl get pods -n ingress-nginx
     ```
   - The controller creates/manages the external Load Balancer — if it's not running, Ingress won't work.

2. **Check Ingress controller logs**  
   - Look for errors related to your Ingress resource:  
     ```bash
     kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
     ```
   - Common issues: invalid annotation, backend service not found, TLS misconfiguration.

3. **Verify ingressClassName is correctly set**  
   - If you are using NGINX Ingress, the Ingress resource must specify:  
     ```yaml
     spec:
       ingressClassName: nginx
     ```
   - Without the correct class name, the controller ignores the Ingress.

4. **Validate Ingress YAML**  
   - Wrong backend service name
   - Wrong service port
   - Missing pathType (Prefix/Exact/ImplementationSpecific)
   - Incorrect host name

5. **Check the backend service and pod**  
   - Ensure the service has valid endpoints:  
     ```bash
     kubectl get endpoints my-service
     ```
   - If endpoints are `<none>`, selector mismatch → fix labels.

6. **Check DNS / external LB**  
   - Verify the Load Balancer IP/DNS is reachable and points to the Ingress controller.
  
7. **Check Backend Pod Probes**
   - Ensure readiness probes pass on backend Pods
   - Ingress won't route to Pods failing readiness probe
   - Check: kubectl describe pod <pod-name> | grep -A 5 "Readiness"

### 23. Why do I need to setup ingress controller after creating ingress?

**Question**: Why do I need to set up an Ingress controller after creating an Ingress resource?

**Answer**:
The `Ingress` resource is just a **declarative API object** (like a Pod or Service). It describes the desired routing rules (host, path → service), but **no component in core Kubernetes** acts on it.

You need an **Ingress Controller** (a separate component) that:

- Watches for Ingress resources
- Reads the rules
- Configures an actual load balancer / proxy (NGINX, Traefik, HAProxy, AWS ALB, etc.)
- Exposes the application externally

Popular Ingress controllers:
- NGINX Ingress Controller
- Traefik
- Contour
- AWS Load Balancer Controller (ALB)
- HAProxy Ingress
- Istio Ingress Gateway

Without a controller watching the Ingress resource, the object exists in etcd but does **nothing**.

### 24. We have an in-house LB, can we use ingress with our load balancer?

**Question**: We have an in-house load balancer. Can we use it with Kubernetes Ingress?

**Answer**:
Yes — but you need an **Ingress Controller** that can configure your in-house load balancer.

Kubernetes does **not** natively know how to talk to your custom/internal LB.

Options:

1. **Use an existing controller that supports your LB** (if available).

2. **Develop a custom Ingress Controller**  
   - Use **controller-runtime** (Go) or **Operator SDK**.
   - Watch Ingress resources.
   - Call your load balancer’s API to create/update virtual servers, pools, health checks, etc.

3. **Use MetalLB** (bare-metal) + standard Ingress controller.

Bottom line:  
You need a controller that translates Kubernetes Ingress objects into configuration for **your specific load balancer**.

### 25. Your deployment has replicas: 3 but only 1 pod is running — what could be wrong?

**Question**: A Deployment specifies replicas: 3 but only 1 pod is running. What could be wrong?

**Answer**:
Most common causes:

1. **Insufficient cluster resources**  
   - Nodes don’t have enough CPU/memory → scheduler cannot place additional pods.
   - Check: `kubectl describe pod <pod-name>` → "Insufficient cpu/memory".

2. **Node taints / missing tolerations**  
   - Nodes have taints → pods without tolerations cannot be scheduled.

3. **Image pull issues**  
   - Image not found, private registry without pull secret → `ImagePullBackOff`.

4. **Pod failing startup**  
   - CrashLoopBackOff — check `kubectl logs --previous`.

5. **Pod stuck in Pending**  
   - Resource requests too high, PVC not bound, node selector/affinity not satisfied.

**Diagnostic commands**:
```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl describe deployment <name>
kubectl get events
```

### 26. Your pod mounts a ConfigMap but changes to the ConfigMap are not reflected?

**Question**: A Pod mounts a ConfigMap as volume but updates to the ConfigMap are not visible inside the container. Why?

**Answer**:
There are two ways to consume a ConfigMap:

1. **As environment variables** → values are injected at pod creation → changes **not** reflected without restart.

2. **As volume mount** → files are mounted → Kubernetes **automatically updates** the files when ConfigMap changes (usually within ~1 minute).

**Correct behavior**:
```yaml
volumeMounts:
  - name: config-volume
    mountPath: /etc/config
volumes:
  - name: config-volume
    configMap:
      name: my-config
```

If using env vars → restart pod required.  
If using volume → changes propagate automatically.

**Note**: The automatic update depends on:
- kubelet syncing (default ~60 seconds)
- The mounted ConfigMap files are updated, but the application 
  must read them again (not loaded in memory once)

### 27. Explain concept of node affinity?

**Question**: Explain node affinity in Kubernetes.

**Answer**:
Node affinity is a way to **influence** where Kubernetes schedules Pods — it allows you to express preferences or hard requirements about which nodes a Pod should run on, based on node labels.

There are two types:

1. **requiredDuringSchedulingIgnoredDuringExecution** (hard requirement)  
   → Pod **will not** be scheduled unless the rule is satisfied.

2. **preferredDuringSchedulingIgnoredDuringExecution** (soft preference)  
   → Scheduler tries to satisfy the rule, but can schedule elsewhere if no node matches.

**Example (hard)**:
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
```

**Example (soft)**:
```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 80
      preference:
        matchExpressions:
        - key: zone
          operator: In
          values:
          - zone-a
```

### 28. What is the difference between node affinity and node label selector?

**Question**: What is the difference between node affinity and nodeSelector?

**Answer**:

| Feature                     | nodeSelector (old)                  | Node Affinity (new)                                      |
|-----------------------------|-------------------------------------|----------------------------------------------------------|
| Syntax                      | Simple key:value                    | Rich expressions (In, NotIn, Exists, Gt, Lt, etc.)        |
| Hard requirement            | Yes                                 | Yes (requiredDuringSchedulingIgnoredDuringExecution)    |
| Soft preference             | No                                  | Yes (preferredDuringSchedulingIgnoredDuringExecution)   |
| Weighting                   | No                                  | Yes (soft rules have weights 1–100)                      |
| Multiple terms (OR logic)   | No                                  | Yes                                                      |

**Recommendation**: Use **node affinity** for new workloads — more powerful and future-proof.

### 29. What is container runtime in Kubernetes?

**Question**: What is a container runtime in Kubernetes?

**Answer**:
The **container runtime** is the software that actually runs containers on a node.

It handles:
- Pulling images
- Unpacking them
- Creating cgroups, namespaces, network interfaces
- Starting/stopping containers

Kubernetes uses the **Container Runtime Interface (CRI)** to abstract different runtimes.

Common runtimes:
- **containerd** (default in recent versions)
- **CRI-O** (lightweight, Kubernetes-native)
- **Docker** (deprecated/removed in newer versions)

**Flow**: kubelet → CRI → container runtime → container

### 30. What is k8s QoS (Quality of Service)?

**Question**: What is Kubernetes Quality of Service (QoS) class and why is it important?

**Answer**:
Kubernetes assigns a **QoS class** to every Pod based on its resource requests and limits. QoS helps the kubelet decide which Pods to evict when a node runs out of resources.

Three classes:

| QoS Class     | Requests | Limits      | Eviction Priority | Use Case                             |
|---------------|----------|-------------|-------------------|--------------------------------------|
| **Guaranteed** | Set      | Set & equal | Lowest            | Critical workloads (databases)       |
| **Burstable**  | Set      | Higher      | Medium            | Most applications                    |
| **BestEffort** | Not set  | Not set     | Highest           | Low-priority / batch jobs            |

**Eviction order**:
1. BestEffort
2. Burstable
3. Guaranteed

**Check QoS**:
```bash
kubectl describe pod <pod-name> | grep -i qos
```

### 31. K8s request and limits

**Question**: Explain Kubernetes requests and limits.

**Answer**:
Requests and limits are defined under `resources` in a container spec to manage CPU and memory usage.

- **Requests**: Minimum guaranteed resources used by the scheduler for pod placement.
  ```yaml
  resources:
    requests:
      memory: "4096Mi"
      cpu: "4"
  ```
  The scheduler ensures a node has at least these resources available before scheduling the Pod.

- **Limits**: Maximum allowed resources enforced by the kubelet at runtime.
  ```yaml
  resources:
    limits:
      memory: "8Gi"
      cpu: "6"
  ```
  If a container exceeds limits:
  - Memory: Pod is evicted/OOMKilled.
  - CPU: Throttled (no eviction).

**Key Difference**:
- **Requests**: Used for scheduling (guaranteed minimum).
- **Limits**: Runtime enforcement (maximum cap) by kubelet.
- Exceeding limits can lead to pod termination to protect node stability and other workloads.

### 32. Can we use k8s master for scheduling the pods?

**Question**: Can we use Kubernetes master nodes for scheduling Pods?

**Answer**:
Yes, it is technically possible, but **not recommended** in production.

- **Default Behavior**: Master nodes have a taint `node-role.kubernetes.io/master:NoSchedule` (or `control-plane` in newer versions), preventing the scheduler from placing Pods.
- **How to Allow**:
  - Remove the taint:
    ```bash
    kubectl taint nodes <master-node-name> node-role.kubernetes.io/master:NoSchedule-
    ```
  - Or add tolerations to Pods:
    ```yaml
    tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
    ```
- **Newer versions (1.25+)**: Master nodes have a taint `node-role.kubernetes.io/control-plane:NoSchedule`
- **Older versions**: Master nodes have a taint `node-role.kubernetes.io/master:NoSchedule`

- **Why Avoid**:
  - Isolation: Control plane components (API server, etcd, scheduler) need dedicated resources for cluster stability.
  - Risk: Workload Pods could starve control plane processes, causing cluster instability.

**Best Practice**: Keep master nodes dedicated to control plane duties.

### 33. Explain horizontal vs vertical scaling?

**Question**: Explain horizontal vs vertical scaling.

**Answer**:

- **Horizontal Scaling** (Scale Out/In):
  - Add or remove instances/nodes/Pods.
  - **Cluster Level**: Add worker nodes to the cluster.
  - **Application Level**: Increase replica count in Deployment/HPA:
    ```yaml
    replicas: 5  # Or use HorizontalPodAutoscaler
    ```
  - **Advantages**: Better fault tolerance, no downtime, handles traffic spikes.
  - Common in Kubernetes via HPA.

- **Vertical Scaling** (Scale Up/Down):
  - Increase resources (CPU/memory) on existing instances.
  - **Application Level**: Update requests/limits → Pod restart required for changes to take effect:
    ```yaml
    resources:
      requests:
        cpu: "4"
        memory: "8Gi"
      limits:
        cpu: "6"
        memory: "12Gi"
    ```
  - **Advantages**: Simpler for stateful/monolithic apps.
  - **Disadvantages**: Requires Pod restart, limited by node capacity.

**Kubernetes Preference**: Horizontal scaling is preferred due to immutability and easier automation.

### 34. Types of secrets in k8s? And why are we have different types?

**Question**: What are the types of Secrets in Kubernetes, and why do we have different types?

**Answer**:
Kubernetes Secrets store sensitive data (passwords, tokens, keys). Different types provide specific structure and use cases.

| Type                              | Use Case                                      | Example Content                              |
|-----------------------------------|-----------------------------------------------|----------------------------------------------|
| **Opaque** (default)              | Generic key-value secrets                     | Arbitrary data (DB credentials, API keys)    |
| **kubernetes.io/dockerconfigjson**| Image pull secrets from private registries    | `.docker/config.json` for auth                |
| **kubernetes.io/tls**             | TLS certificates and keys                     | `tls.crt` and `tls.key` for Ingress          |
| **kubernetes.io/service-account-token** | Service account tokens (deprecated)      | JWT for service account authentication       |
| **bootstrap.kubernetes.io/token** | Bootstrap tokens for node joining              | Used during cluster bootstrap                |

**Why Different Types**:
- **Structure Validation**: Kubernetes validates content (e.g., TLS secrets require valid cert/key).
- **Automation**: Specific types trigger built-in behaviors (e.g., dockerconfigjson used for ImagePullSecrets).
- **Security & Flexibility**: Opaque for custom data, specialized types for common patterns.

**Creation Example**:
```bash
kubectl create secret generic my-secret --from-literal=user=admin --from-literal=password=secret
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key
kubectl create secret docker-registry my-reg --docker-server=... --docker-username=... --docker-password=...
```


## Conclusion
Understanding Kubernetes architecture, component interaction, Services, kube-proxy, Network Policies, probes, and deployment strategies is fundamental for any Kubernetes interview. These answers cover the most frequently asked conceptual and practical questions, helping you confidently explain how Kubernetes achieves reliability, service discovery, security, and zero-downtime deployments.

