# Cloud & AWS

> **Phase 16 — Advanced Topics → 16.7 Cloud / AWS**
> Goal: Understand cloud computing fundamentals and the core AWS services a Java backend engineer uses — compute, storage, database, networking, messaging, serverless, security/IAM — plus infrastructure as code.

---

## 0. The Big Picture

The **cloud** is on-demand, pay-as-you-go computing resources over the internet — no buying/racking servers. **AWS** (Amazon Web Services) is the largest provider; the concepts transfer to Azure/GCP. For a backend engineer, the cloud is where your containers (Phase 10), databases (Phase 4/16.6), and messaging (Phase 11) actually run, with managed services that offload operations.

```
   Your app (Spring Boot container — 10.1)
        runs on cloud COMPUTE (EC2 / ECS / EKS / Lambda)
        + managed DATABASE (RDS/Aurora — 16.6)
        + STORAGE (S3) + MESSAGING (SQS/SNS/MSK — 11) + IAM security + networking (VPC)
        all defined as code (Terraform/CloudFormation)
```

> Where everything you've learned is deployed: containers/K8s (Phase 10), databases (4/16.6), messaging (11), observability (9), security (15). Serverless ties to GraalVM cold starts (16.5); DBaaS to 16.6.

---

## 1. Cloud Fundamentals

| Concept | Meaning |
|---------|---------|
| **IaaS / PaaS / SaaS** | Infrastructure (VMs) / Platform (managed runtime) / Software (apps) — increasing abstraction |
| **Regions & Availability Zones (AZs)** ⭐ | Geographic regions, each with multiple isolated data centers (AZs) for HA |
| **Elasticity** | Scale resources up/down on demand (autoscaling — 13.3/10.2) |
| **Pay-as-you-go** | Pay for what you use (cost is an engineering concern) |
| **Shared responsibility** ⭐ | Provider secures *the cloud*; **you** secure *what's in it* (your app/data/config — 15) |
> ⭐ **Deploy across multiple AZs** for high availability (a data center can fail). The **shared responsibility model** is crucial for security (15): AWS secures the hardware/hypervisor; **you** are responsible for your app code, IAM config, data encryption (15.3), and patching (15.4). Cost-awareness (right-sizing, autoscaling) is part of the job (FinOps).

---

## 2. Compute (where your app runs) ⭐

| Service | Model | Use |
|---------|-------|-----|
| **EC2** | Virtual machines | Full control; run anything (IaaS) |
| **ECS** | Managed container orchestration | Run containers (10.1) without full K8s |
| **EKS** ⭐ | Managed **Kubernetes** (10.2) | K8s without managing the control plane |
| **Fargate** | Serverless containers | Run containers without managing servers |
| **Lambda** ⭐ | **Serverless functions (FaaS)** | Event-driven, auto-scaling, pay-per-invocation; ⚠️ **cold starts** (→ GraalVM native — 16.5) |
| **Elastic Beanstalk** | PaaS | Quick app deploy (abstracts infra) |
> ⭐ For containerized Spring Boot (10.1), **EKS** (managed K8s — 10.2) or **ECS/Fargate** are typical. **Lambda** suits event-driven/spiky workloads — but the JVM's **cold start** hurts (a Lambda may sleep then must start fast) → **GraalVM native images** (16.5) or SnapStart mitigate this. Choose compute by control-vs-convenience and workload shape.

---

## 3. Storage & Database

| Service | Type | Use |
|---------|------|-----|
| **S3** ⭐ | Object storage | Files, backups, static assets, data lake; **pre-signed URLs** for uploads (7.1 §7) |
| **EBS** | Block storage | Disks for EC2 (≈ K8s PV — 10.2) |
| **EFS** | Network file system | Shared file storage |
| **RDS** ⭐ | Managed relational DB | Postgres/MySQL (DBaaS — 16.6) |
| **Aurora** | Cloud-native RDBMS | Scalable Postgres/MySQL-compatible (16.6) |
| **DynamoDB** | Managed NoSQL key-value | Serverless, high-scale (AP — 16.6/4.8) |
| **ElastiCache** | Managed Redis/Memcached | Caching/sessions (4.8/13.1/13.3) |
| **OpenSearch** | Managed Elasticsearch (16.4) | Search/log analytics |
> ⭐ **S3** is the workhorse for files/backups — and the scalable file-upload pattern (**pre-signed URLs** — 7.1 §7) bypasses your app. **RDS/Aurora** = managed databases (16.6 DBaaS); **ElastiCache** = managed Redis (4.8). All support **encryption at rest** (15.3).

---

## 4. Networking & Delivery

| Service | Role |
|---------|------|
| **VPC** | Your isolated virtual network (subnets, route tables) |
| **Security Groups** ⭐ | Virtual firewalls (allow/deny traffic per instance) — least privilege (15) |
| **ELB / ALB** | Load balancer (L7 ALB ≈ Ingress/Gateway — 10.2/12.4) |
| **Route 53** | DNS |
| **CloudFront** | CDN (cache static/edge content — perf 13.1) |
| **API Gateway** | Managed API front door (≈ 12.4 concepts) |
> ⭐ Put resources in a **VPC** with **private subnets** (DBs not internet-facing) and tight **security groups** (least privilege — 15). **ALB** load-balances; **CloudFront** (CDN) speeds static delivery (13.1). This mirrors the gateway/ingress/LB concepts from 10.2/12.4.

---

## 5. Messaging & Integration

| Service | ≈ |
|---------|---|
| **SQS** | Managed message **queue** (≈ RabbitMQ point-to-point — 11.1/11.3) |
| **SNS** | Managed **pub/sub** notifications (≈ topic fan-out — 11.1) |
| **EventBridge** | Event bus / routing (event-driven — 11.4) |
| **MSK** | Managed **Kafka** (11.2) |
| **Kinesis** | Managed streaming (≈ Kafka streams) |
| **Step Functions** | Workflow orchestration (≈ Saga orchestration — 11.4/12.6) |
> ⭐ AWS provides managed versions of everything in Phase 11: **SQS** (queues), **SNS/EventBridge** (pub-sub/events), **MSK** (Kafka). **Step Functions** orchestrate workflows (saga orchestration — 11.4). Managed = no broker ops, at the cost of some portability.

---

## 6. Security & Identity (IAM) ⭐

| Service | Role |
|---------|------|
| **IAM** ⭐ | Identities, roles, and **fine-grained permissions** (who can do what) |
| **IAM Roles** | Grant an app/service permissions **without static credentials** (instance/pod roles) |
| **KMS** | Managed encryption keys (envelope encryption — 15.3) |
| **Secrets Manager / Parameter Store** | Managed secrets (≈ Vault — 12.5) |
| **Cognito** | Managed user auth/identity (OAuth2/OIDC — 15.2) |
| **WAF / Shield** | Web app firewall / DDoS protection (15.1) |
> ⭐ **IAM is the core of AWS security** — apply **least privilege** (15): grant the minimum permissions, prefer **IAM roles** (temporary, auto-rotated credentials) over long-lived access keys (don't bake keys into code/images — 15.3/12.5). **KMS** manages encryption keys (15.3); **Secrets Manager** ≈ Vault (12.5). This is the shared-responsibility "your part" (§1).

---

## 7. Infrastructure as Code (IaC) ⭐

Define cloud infrastructure in **version-controlled code** instead of clicking the console — reproducible, reviewable, automatable (like DB migrations for infra — Phase 8 mindset).
| Tool | Note |
|------|------|
| **Terraform** ⭐ | Cloud-agnostic, declarative (HCL); the de facto standard |
| **AWS CloudFormation / CDK** | AWS-native (YAML / real code via CDK) |
| **Pulumi** | IaC in general-purpose languages |
```hcl
resource "aws_db_instance" "app" {     # Terraform — declarative infra (version-controlled, reviewed)
  engine = "postgres"; instance_class = "db.t3.medium"; allocated_storage = 20
}
```
> ⭐ **IaC makes infrastructure reproducible and reviewable** (PRs — Phase 0.4) and is the foundation of **GitOps**/CD (10.3). Treat infra like application code: version it, review it, apply it in CI/CD. Avoid "click-ops" (manual console changes that drift and can't be reproduced).

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Misunderstanding shared responsibility | Secure your app/data/IAM — provider only secures the cloud (§1, 15) |
| Single-AZ deployment | Multi-AZ for HA (§1) |
| Long-lived access keys in code/images | IAM roles + Secrets Manager (§6, 15.3/12.5) |
| Over-broad IAM permissions | Least privilege (§6, 15) |
| Public databases | Private subnets + security groups (§4) |
| JVM Lambda cold starts | GraalVM native / SnapStart (§2, 16.5) |
| Click-ops (manual console changes) | Infrastructure as Code (§7) |
| Ignoring cost | Right-size, autoscale, monitor spend (§1) |
| Vendor lock-in unmanaged | Prefer portable layers (containers/K8s, Terraform) where it matters |
| No encryption at rest/in transit | Enable KMS/TLS (§3/§6, 15.3) |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- Runs **containers** (10.1) on **EKS/ECS/Fargate** (≈ K8s — 10.2); **Lambda** ties to **GraalVM native** cold starts (16.5).
- **RDS/Aurora/DynamoDB/ElastiCache/OpenSearch** = managed Phase 4/16.6/16.4 stores; **S3** for files (pre-signed URLs — 7.1).
- **SQS/SNS/EventBridge/MSK** = managed Phase 11 messaging; **Step Functions** ≈ Saga (11.4).
- **IAM/KMS/Secrets Manager** = security/secrets (15.2/15.3, ≈ Vault — 12.5); **VPC/SG/ALB/CloudFront** = networking (10.2/12.4/13.1).
- **IaC (Terraform)** ↔ CI/CD/GitOps (10.3), version control (0.4).
- **Spring Cloud AWS** integrates these into Spring apps.
- The deployment target for **Projects 6 & 7**.

---

## 10. Quick Self-Check Questions

1. What are IaaS/PaaS/SaaS, regions/AZs, and the shared responsibility model?
2. Why deploy across multiple AZs?
3. Compare EC2, ECS, EKS, Fargate, and Lambda — when use each?
4. Why are Lambda cold starts a problem for the JVM, and what mitigates them?
5. What is S3 used for, and how do pre-signed URLs help uploads?
6. Which AWS services map to RDS/Redis/Elasticsearch/Kafka/queues?
7. What is IAM, and why prefer roles over access keys (least privilege)?
8. How do VPC, security groups, ALB, and CloudFront secure/serve traffic?
9. What is Infrastructure as Code, and why is it better than click-ops?
10. How does cloud relate to cost as an engineering concern?

---

## 11. Key Terms Glossary

- **Cloud / IaaS / PaaS / SaaS:** on-demand computing at increasing abstraction.
- **Region / Availability Zone (AZ):** geographic area / isolated data center.
- **Shared responsibility model:** provider secures cloud; you secure your part.
- **EC2 / ECS / EKS / Fargate / Lambda:** VMs / containers / managed K8s / serverless containers / FaaS.
- **Cold start:** serverless startup latency (16.5).
- **S3 / EBS / EFS:** object / block / file storage.
- **RDS / Aurora / DynamoDB / ElastiCache / OpenSearch:** managed DBs/cache/search.
- **VPC / Security Group / ALB / CloudFront / Route 53:** network / firewall / LB / CDN / DNS.
- **SQS / SNS / EventBridge / MSK / Step Functions:** queue / pub-sub / event bus / Kafka / workflow.
- **IAM / KMS / Secrets Manager / Cognito / WAF:** identity / keys / secrets / user auth / firewall.
- **Infrastructure as Code (Terraform/CloudFormation/CDK):** version-controlled infra.

---

*This is the note for **Section 16.7 — Cloud / AWS**.*
*Previous section in roadmap: **16.6 Advanced Database**.*
*Next section in roadmap: **16.8 System Design**.*
