# Kubernetes (Container Orchestration)

> **Phase 10 — Containerization & Deployment → 10.2 Kubernetes**
> Goal: Understand Kubernetes — the architecture (control plane / worker nodes), core resources (Pod, ReplicaSet, Deployment, Service, ConfigMap, Secret, Namespace, Ingress), workloads (StatefulSet, DaemonSet, Job, CronJob), storage (PV/PVC/StorageClass), health checks, resources & autoscaling (HPA/VPA), `kubectl`, Helm, local clusters (Minikube/kind/k3s), and deployment strategies (Rolling, Blue-Green, Canary).

---

## 0. The Big Picture

Docker (10.1) runs **one** container on **one** host. Real systems need many containers across many machines, self-healing, scaling, rolling updates, and service discovery. **Kubernetes (K8s)** is the **container orchestrator** that does this — you declare the *desired state* ("run 3 replicas of this image"), and K8s continuously **reconciles** reality to match it.

```
   You: kubectl apply -f deployment.yaml   (DESIRED state, declarative)
                  │
                  ▼
   ┌──────────────── Control Plane ────────────────┐
   │ API Server ─ etcd ─ Scheduler ─ Controllers    │  ◄ reconciliation loop
   └───────────────────────┬────────────────────────┘
                           │ schedules Pods onto...
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ Worker 1 │  │ Worker 2 │  │ Worker 3 │   (nodes run kubelet + your Pods)
   │  [Pod]   │  │  [Pod]   │  │  [Pod]   │
   └──────────┘  └──────────┘  └──────────┘
```

> K8s runs the **Docker images** you built (10.1), consumes **health probes** (9.2), respects the **memory limits** the JVM reads (`MaxRAMPercentage` — 10.1 §7.1), and is deployed by **CI/CD** (10.3). It's the runtime for **microservices** (Phase 12).

### ⭐ The core idea: declarative + reconciliation
You describe *what* you want (YAML manifests); **controllers** loop forever making it true. A pod dies → controller starts a new one. This **self-healing**, declarative model is the whole philosophy.

---

## 1. Architecture

### 1.1 Control Plane (the brain)
| Component | Role |
|-----------|------|
| **API Server** | Front door — all `kubectl`/component traffic goes through it (REST) |
| **etcd** | Distributed key-value store = the cluster's **source of truth** (all state) |
| **Scheduler** | Decides **which node** a new Pod runs on (resources, affinity, taints) |
| **Controller Manager** | Runs controllers (reconciliation loops: Deployment, ReplicaSet, Node...) |
| **Cloud Controller Manager** | Integrates with the cloud (LBs, volumes) |

### 1.2 Worker Nodes (the muscle)
| Component | Role |
|-----------|------|
| **kubelet** | Agent on each node; starts/monitors Pods, reports to API server |
| **kube-proxy** | Networking — implements Service routing/load-balancing |
| **Container runtime** | Runs containers (containerd/CRI-O — Docker images still run) |

> ⭐ Everything flows through the **API Server** and is stored in **etcd**. Controllers watch for differences between desired (in etcd) and actual state and act. Lose etcd = lose the cluster's memory → back it up.

---

## 2. Core Resources

### 2.1 Pod — the smallest unit
A **Pod** wraps one (usually) or more tightly-coupled containers that share network (same IP/localhost) and storage. ⚠️ **You rarely create Pods directly** — they're ephemeral (no self-healing). Let a Deployment manage them.

### 2.2 ReplicaSet — keeps N copies
Ensures a specified **number of identical Pods** are running (self-healing/scaling). You don't manage it directly either — a **Deployment** owns it.

### 2.3 Deployment ⭐ — the workhorse for stateless apps
Manages ReplicaSets → declarative updates, **rolling deploys** (§9), rollbacks, scaling. This is what you use for a Spring Boot service.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: order-service }
spec:
  replicas: 3                                  # desired Pods
  selector: { matchLabels: { app: order-service } }
  template:
    metadata: { labels: { app: order-service } }
    spec:
      containers:
        - name: app
          image: registry/order-service:1.4.0   # your Docker image (10.1)
          ports: [ { containerPort: 8080 } ]
          envFrom:
            - configMapRef: { name: order-config }   # config (§2.5)
            - secretRef:    { name: order-secrets }  # secrets (§2.6)
          resources:                               # §6 — CRITICAL for the JVM
            requests: { cpu: "250m", memory: "512Mi" }
            limits:   { cpu: "1",    memory: "1Gi" }
          readinessProbe: { httpGet: { path: /actuator/health/readiness, port: 8080 } }  # 9.2
          livenessProbe:  { httpGet: { path: /actuator/health/liveness,  port: 8080 } }
```

### 2.4 Service — stable networking for Pods
Pods are ephemeral (IPs change). A **Service** gives a **stable virtual IP + DNS name** and **load-balances** across the matching Pods.
| Service type | Exposes | Use |
|--------------|---------|-----|
| **ClusterIP** (default) | Internal-only virtual IP | Service-to-service (in-cluster) |
| **NodePort** | A port on every node | Basic external access (dev) |
| **LoadBalancer** | Cloud LB → the Service | External access (prod, cloud) |
| **(Headless)** | No virtual IP; direct Pod DNS | StatefulSets (§3) |
```yaml
kind: Service
metadata: { name: order-service }
spec:
  selector: { app: order-service }      # routes to Pods with this label
  ports: [ { port: 80, targetPort: 8080 } ]
  # type: ClusterIP (default)  → reachable in-cluster as http://order-service
```
> ⭐ In-cluster **DNS**: a Service named `order-service` is reachable at `http://order-service` (or `order-service.namespace.svc.cluster.local`) — this is K8s-native **service discovery** (Phase 12.2).

### 2.5 ConfigMap — non-secret config
External config (env vars, files) decoupled from the image (12-factor; recall Spring profiles/externalized config — Phase 5.1/5.2).
```yaml
kind: ConfigMap
metadata: { name: order-config }
data:
  SPRING_PROFILES_ACTIVE: prod
  APP_FEATURE_X_ENABLED: "true"
```

### 2.6 Secret — sensitive config
Like a ConfigMap but for passwords/keys/tokens. ⚠️ **Base64-encoded, NOT encrypted by default** — enable encryption-at-rest and/or use external secret managers (Vault — Phase 12.5/15.3).
```yaml
kind: Secret
metadata: { name: order-secrets }
type: Opaque
data:
  SPRING_DATASOURCE_PASSWORD: c2VjcmV0   # base64("secret") — ⚠️ not real encryption
```

### 2.7 Namespace — virtual cluster partition
Logical isolation/grouping (e.g., `dev`, `staging`, `prod`, per-team) with separate quotas/RBAC.

### 2.8 Ingress — HTTP routing into the cluster
Routes external HTTP(S) traffic to Services by host/path, terminates TLS — one entry point instead of many LoadBalancers. Needs an **Ingress Controller** (nginx, Traefik).
```yaml
kind: Ingress
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - { path: /orders, pathType: Prefix, backend: { service: { name: order-service, port: { number: 80 } } } }
```
> Ingress is the K8s-level edge router; a richer **API Gateway** (Spring Cloud Gateway — Phase 12.4) often sits here too.

---

## 3. Workload Types Beyond Deployment

| Workload | Use for |
|----------|---------|
| **Deployment** | Stateless apps (your typical Spring Boot service) |
| **StatefulSet** | **Stateful** apps needing stable identity & storage (databases, Kafka — Phase 4/11): stable Pod names (`db-0`, `db-1`), ordered start, per-Pod PVCs |
| **DaemonSet** | One Pod **per node** (log/metrics agents — 9.2/9.3, CNI) |
| **Job** | Run-to-completion task (a batch job, a **DB migration** — Flyway/Liquibase, Phase 8 §7!) |
| **CronJob** | Scheduled Jobs (cron syntax — like Spring `@Scheduled`, Phase 5.8, but at cluster level) |

> ⭐ **DB migrations as a Job** is the production pattern from Phase 8: run Flyway/Liquibase as a K8s **Job** *before* rolling out new app Pods, rather than every replica migrating at startup.

---

## 4. Storage

Containers are ephemeral; persistent data needs explicit storage.
| Resource | Meaning |
|----------|---------|
| **PersistentVolume (PV)** | A piece of cluster storage (disk) provisioned by admin/cloud |
| **PersistentVolumeClaim (PVC)** | A Pod's *request* for storage (size/access mode) → bound to a PV |
| **StorageClass** | Defines **dynamic provisioning** (e.g., "AWS gp3") — creates PVs on demand |
| **Access modes** | RWO (one node), ROX (read-only many), RWX (read-write many) |
```
Pod ──uses──► PVC ──binds to──► PV ◄──provisioned by── StorageClass (dynamic)
```
> ⭐ For databases/stateful workloads (StatefulSet §3), each Pod gets its **own PVC** (stable storage that survives restarts). Most stateless Spring apps need **no** persistent storage (state lives in the DB — Phase 4 — and they stay stateless for scaling, Phase 13.3).

---

## 5. Health Checks (Probes)

K8s uses the **liveness/readiness/startup** probes you exposed via Actuator (9.2 §6):
| Probe | If it fails |
|-------|-------------|
| **livenessProbe** | **Restart** the Pod (app is broken/deadlocked) |
| **readinessProbe** | Remove from Service endpoints (**stop traffic**), don't restart |
| **startupProbe** | Guards slow-starting apps before liveness kicks in (JVM warmup — Phase 1.11) |
> ⚠️ Same rule as 9.2: **liveness must NOT depend on the DB** (a DB blip would restart all Pods). DB readiness → `readinessProbe`. Probes + rolling updates (§9) + graceful shutdown (PID 1/`SIGTERM` — 10.1) give zero-downtime deploys.

---

## 6. Resources & Autoscaling

### 6.1 Requests & limits ⭐ (critical for the JVM)
```yaml
resources:
  requests: { cpu: "250m", memory: "512Mi" }   # guaranteed; used for scheduling
  limits:   { cpu: "1",    memory: "1Gi" }      # hard ceiling
```
- **request** = reserved (scheduler places the Pod where this fits).
- **limit** = max; exceed **memory** → Pod is **OOMKilled**; exceed **CPU** → throttled.
- ⚠️ ⭐ The JVM reads the **memory limit** to size the heap (with `MaxRAMPercentage` — 10.1 §7.1). Set the limit *and* `MaxRAMPercentage` together, leaving headroom for non-heap (metaspace, threads, off-heap — Phase 1.11) — else **OOMKilled**.

### 6.2 Autoscaling
| Autoscaler | Scales |
|------------|--------|
| **HPA (Horizontal Pod Autoscaler)** | **Number of Pods** based on CPU/memory/custom metrics (e.g., Prometheus — 9.2) ⭐ |
| **VPA (Vertical Pod Autoscaler)** | A Pod's **requests/limits** (right-sizing) |
| **Cluster Autoscaler** | **Number of nodes** (adds machines when Pods can't schedule) |
```yaml
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef: { kind: Deployment, name: order-service }
  minReplicas: 3
  maxReplicas: 20
  metrics: [ { type: Resource, resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } } } ]
```
> ⭐ **HPA = horizontal scaling** (Phase 13.3) — add Pods under load. Works only if your app is **stateless** (no in-memory session → use Redis sessions, Phase 13.3). Don't combine HPA + VPA on the same CPU/memory metric (they conflict).

---

## 7. `kubectl` (the CLI)

| Command | Does |
|---------|------|
| `kubectl apply -f file.yaml` | Create/update resources (declarative) ⭐ |
| `kubectl get pods/deploy/svc` | List resources |
| `kubectl describe pod <name>` | Detailed status/events (debugging) |
| `kubectl logs -f <pod>` | Stream logs (stdout — 9.1) |
| `kubectl exec -it <pod> -- sh` | Shell into a Pod |
| `kubectl scale deploy/x --replicas=5` | Scale manually |
| `kubectl rollout status/undo deploy/x` | Watch / **roll back** a deploy |
| `kubectl port-forward svc/x 8080:80` | Tunnel a Service to localhost (debug) |
| `kubectl get events` | Cluster events (why a Pod won't start) |
> ⭐ Prefer **`apply` (declarative)** over imperative `create`/`edit` — keep YAML in Git (GitOps; Phase 0.4/10.3). `describe` + `logs` + `get events` are your debugging trio.

---

## 8. Helm (Package Manager for K8s)

Raw YAML gets repetitive across services/environments. **Helm** packages manifests into reusable, parameterized **charts** with templating + values.
```
mychart/
├── Chart.yaml
├── values.yaml            # default values (overridable per env)
└── templates/
    ├── deployment.yaml    # {{ .Values.image.tag }} templated
    └── service.yaml
```
```bash
helm install order ./mychart -f values-prod.yaml
helm upgrade order ./mychart --set image.tag=1.5.0
helm rollback order 1
```
> ⭐ Helm = **templating + release management + versioned upgrades/rollbacks** for K8s. One chart, many environments via different `values.yaml` (recall Spring profiles — Phase 5.1). Alternatives: **Kustomize** (overlay-based, no templating).

---

## 9. Deployment Strategies ⭐

| Strategy | How | Pros / Cons |
|----------|-----|-------------|
| **Rolling Update** (K8s default) | Replace Pods gradually (new up, old down, controlled by `maxSurge`/`maxUnavailable`) | ✅ Zero-downtime, simple; ⚠️ both versions run briefly (need backward-compat — recall expand/contract, Phase 8 §6.1) |
| **Recreate** | Kill all old, then start new | ⚠️ Downtime; only if versions can't coexist |
| **Blue-Green** | Run v2 (green) alongside v1 (blue); switch traffic at once; instant rollback | ✅ Instant cutover/rollback; ⚠️ 2× resources |
| **Canary** | Send a small % of traffic to v2, watch metrics (9.2), then ramp up | ✅ Limits blast radius, data-driven; ⚠️ needs traffic-splitting (Ingress/mesh) + good monitoring |

```
Rolling:  [v1 v1 v1] → [v2 v1 v1] → [v2 v2 v1] → [v2 v2 v2]   (gradual, zero-downtime)
Canary:   95% → v1,  5% → v2   (watch errors/latency — 9.2)  →  ramp to 100% v2
```
> ⭐ **Rolling** is the default and fine for most. **Canary** (often via a service mesh / Argo Rollouts / Flagger) is the gold standard for risky changes — release to 5%, watch the golden signals (9.2 §3), auto-rollback on errors. All depend on **backward-compatible DB changes** (expand/contract — Phase 8) since versions overlap.

---

## 10. Local Kubernetes (Dev)

| Tool | Note |
|------|------|
| **Minikube** | Single-node local cluster (VM/driver); full-featured, classic choice |
| **kind** (Kubernetes-in-Docker) | Clusters as Docker containers; fast, great for CI |
| **k3s** | Lightweight production-capable K8s (edge/IoT/dev); small footprint |
| (Docker Desktop) | Built-in single-node K8s toggle |
> ⭐ Use **kind**/Minikube to learn and to test manifests locally before CI/staging. **k3s** is a real lightweight distro (not just for dev).

---

## 11. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Creating bare Pods | Use a Deployment (self-healing/rolling) (§2) |
| No resource requests/limits | Set them; size JVM heap to the limit (§6.1, 10.1) |
| `-Xmx` ignoring the container limit | `MaxRAMPercentage` + memory limit (§6.1) |
| Liveness depends on DB | Put deps in readiness (§5, 9.2) |
| Treating Secrets as encrypted | Base64 ≠ encryption; use Vault/encryption-at-rest (§2.6, Phase 12.5) |
| HPA on a stateful app | Make app stateless (Redis sessions — Phase 13.3) |
| Migrating from every replica at startup | Run migrations as a Job (§3, Phase 8) |
| Imperative kubectl edits not in Git | Declarative `apply` + GitOps (§7) |
| Breaking changes with rolling updates | Backward-compatible (expand/contract — Phase 8 §6.1) |
| Copy-pasting YAML per env | Helm/Kustomize (§8) |
| StatefulSet where Deployment suffices | Use Deployment unless you truly need identity/storage (§3) |

---

## 12. Connection to Backend / Spring (Why This Matters Later)

- Runs your **Docker images** (10.1); JVM heap sizing depends on **limits** (§6, 10.1 §7.1, Phase 1.11).
- **Actuator probes** (9.2) → liveness/readiness/startup; graceful shutdown via `SIGTERM` (10.1).
- **ConfigMaps/Secrets** externalize config (Phase 5.1/5.2) → deeper in **config management** (Phase 12.5).
- **Service DNS** = native **service discovery** for **microservices** (Phase 12.2); **Ingress** ↔ **API Gateway** (Phase 12.4).
- **DB migrations as Jobs** (Phase 8); **StatefulSets** for stateful infra (Phase 4/11).
- **HPA** = horizontal scaling (Phase 13.3) on **Prometheus metrics** (9.2).
- **CI/CD** (10.3) deploys via `kubectl`/Helm; **deployment strategies** depend on backward-compat (Phase 8).
- Core of **Project 6 (Deployment Pipeline)** & **Project 7 (Microservices E-Commerce)**.

---

## 13. Quick Self-Check Questions

1. What problem does K8s solve over plain Docker, and what is the declarative/reconciliation model?
2. Name control-plane components (API Server, etcd, Scheduler, Controllers) and worker components (kubelet, kube-proxy, runtime).
3. Pod vs ReplicaSet vs Deployment — what manages what, and which do you create?
4. What does a Service do, and how does in-cluster DNS/service discovery work? Service types?
5. ConfigMap vs Secret — and why are Secrets not truly secure by default?
6. When use StatefulSet, DaemonSet, Job, CronJob instead of Deployment?
7. Explain PV/PVC/StorageClass and dynamic provisioning.
8. Why are resource requests/limits critical for the JVM, and how do they interact with `MaxRAMPercentage`?
9. HPA vs VPA vs Cluster Autoscaler — what does each scale, and what does HPA require of the app?
10. Compare Rolling, Blue-Green, and Canary — and why all need backward-compatible DB changes.

---

## 14. Key Terms Glossary

- **Kubernetes / orchestration:** declarative system to run containers at scale, self-healing.
- **Control plane (API Server, etcd, Scheduler, Controller Manager):** the brain.
- **Node / kubelet / kube-proxy:** worker machine / agent / network proxy.
- **Pod / ReplicaSet / Deployment:** smallest unit / N copies / declarative manager.
- **Service (ClusterIP/NodePort/LoadBalancer) / Ingress:** stable networking / HTTP edge routing.
- **ConfigMap / Secret / Namespace:** config / sensitive config / partition.
- **StatefulSet / DaemonSet / Job / CronJob:** stateful / per-node / run-once / scheduled.
- **PV / PVC / StorageClass:** storage / claim / dynamic provisioner.
- **Liveness / readiness / startup probe:** restart / traffic-gate / slow-start checks.
- **Requests / limits / OOMKilled:** reserved / ceiling / killed for exceeding memory.
- **HPA / VPA / Cluster Autoscaler:** scale Pods / Pod size / nodes.
- **kubectl / Helm / Kustomize:** CLI / package manager / overlay tool.
- **Rolling / Blue-Green / Canary:** deployment strategies.
- **Minikube / kind / k3s:** local/lightweight clusters.

---

*This is the note for **Section 10.2 — Kubernetes**.*
*Previous section in roadmap: **10.1 Docker**.*
*Next section in roadmap: **10.3 CI/CD**.*
