# Production-Ready Search Service on AWS (EKS + Aurora) — Architecture & Runbook

This takes the demo from the previous guide and turns it into something you'd actually run on-call for.
The app is the same Go document-search service; almost everything *around* it changes.

## 0. What "production-ready" actually changes

| Concern | Demo version | Production version | Why |
|---|---|---|---|
| **Database** | Postgres StatefulSet + EBS PVC | **Amazon Aurora PostgreSQL**, Multi-AZ, private subnets | You don't want to own DB failover, backups, patching, or storage |
| **Provisioning** | `eksctl` + `kubectl apply` by hand | **Terraform** (VPC, EKS, RDS, ECR, IAM) | Reproducible, reviewable, versioned; no click-ops |
| **Secrets** | plaintext `Secret` manifest | **Secrets Manager** + **External Secrets Operator** + rotation | Nothing secret in Git; automatic rotation |
| **DB connection** | `sslmode=disable`, one pool | TLS `verify-full`, **writer/reader split**, tuned pool | Encrypted in transit; reads scale on Aurora replicas |
| **Ingress** | HTTP-only ALB | **HTTPS** via ACM, HTTP→HTTPS redirect, Route 53, WAF | TLS is table stakes; WAF blocks common attacks |
| **App** | bare handlers | structured logs, **Prometheus metrics**, timeouts, middleware | You can't operate what you can't observe |
| **Container** | non-root distroless | + read-only rootfs, dropped caps, image scan (Trivy), SBOM | Minimize + prove the attack surface |
| **Workload** | Deployment + HPA | + **PDB**, topology spread, NetworkPolicy, securityContext | Survive node loss, AZ loss, and lateral movement |
| **Delivery** | manual | **GitHub Actions** (test/scan/sign/push) + **Argo CD** GitOps | Auditable, repeatable, rollback-able |
| **Scaling** | HPA only | HPA + **Karpenter** node autoscaling | Scale pods *and* nodes to demand/cost |
| **Observability** | `kubectl logs` | Prometheus/Grafana + CloudWatch logs + OTel traces | RED metrics, dashboards, alerts, tracing |

---

## 1. Target architecture

![Production architecture on AWS](diagrams-prod/prod-01-architecture.png)

**The tiers:** public subnets hold only the ALB + NAT; app pods run in private subnets with no public
IPs; the database lives in its own isolated private subnets reachable *only* from the app security
group. Egress to AWS APIs (ECR, Secrets Manager) goes via NAT or, better, **VPC endpoints**.

---

## 2. Layer 1 — Infrastructure as Code (Terraform)

Everything below is provisioned by Terraform (full files in `terraform/` in the scaffold). Pin every
module and provider version and use a **remote backend** (S3 + DynamoDB lock) so state is shared and
locked — never local state for shared infra.

### 2.1 Remote state + providers (`versions.tf`, `providers.tf`)

```hcl
terraform {
  required_version = ">= 1.7"
  required_providers {
    aws        = { source = "hashicorp/aws",        version = "~> 5.60" }
    kubernetes = { source = "hashicorp/kubernetes",  version = "~> 2.31" }
    helm       = { source = "hashicorp/helm",        version = "~> 2.14" }
  }
  backend "s3" {
    bucket         = "my-tfstate-bucket"
    key            = "search-service/prod.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "tf-locks"
    encrypt        = true
  }
}
```

### 2.2 Network (`vpc.tf`)

Use `terraform-aws-modules/vpc` with three AZs and the subnet tags EKS + the LB controller require:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.13"

  name = "search-prod"
  cidr = "10.0.0.0/16"
  azs  = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]

  private_subnets  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]  # app tier
  public_subnets   = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  database_subnets = ["10.0.201.0/24", "10.0.202.0/24", "10.0.203.0/24"]  # isolated data tier

  enable_nat_gateway   = true
  single_nat_gateway   = false   # one NAT per AZ = no cross-AZ SPOF (costs more; worth it in prod)
  enable_dns_hostnames = true

  # These tags are how the ALB controller discovers where to place load balancers.
  public_subnet_tags  = { "kubernetes.io/role/elb" = "1" }
  private_subnet_tags = { "kubernetes.io/role/internal-elb" = "1" }
}
```

### 2.3 EKS cluster (`eks.tf`)

`terraform-aws-modules/eks` **v21** — API-based **access entries** (no more `aws-auth` ConfigMap),
control-plane logging, private nodes, and core addons managed by the module:

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 21.0"

  name               = "search-prod"
  kubernetes_version = "1.33"

  endpoint_public_access                   = true   # lock to your CIDRs in real prod
  enable_cluster_creator_admin_permissions = true   # your Terraform identity becomes cluster admin

  vpc_id                   = module.vpc.vpc_id
  subnet_ids               = module.vpc.private_subnets   # nodes in private subnets
  control_plane_subnet_ids = module.vpc.private_subnets

  enabled_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]

  addons = {
    coredns                = {}
    kube-proxy             = {}
    vpc-cni                = { before_compute = true }
    eks-pod-identity-agent = { before_compute = true }
    aws-ebs-csi-driver     = {}   # still useful for Prometheus/Grafana PVCs
  }

  eks_managed_node_groups = {
    general = {
      ami_type       = "AL2023_x86_64_STANDARD"
      instance_types = ["m6i.large"]
      min_size       = 3      # one node per AZ minimum
      max_size       = 10
      desired_size   = 3
    }
  }

  tags = { Environment = "prod", Project = "search-service" }
}
```

> **IRSA vs Pod Identity:** v21 ships the `eks-pod-identity-agent` addon — **EKS Pod Identity** is the
> newer, simpler way to give pods IAM roles (no OIDC trust-policy juggling). This guide uses **IRSA**
> for the controllers because their docs still assume it, but Pod Identity is the forward path — pick
> one convention and stick to it.

### 2.4 Aurora PostgreSQL (`rds.tf`) — the biggest production change

```hcl
resource "random_password" "db" {
  length = 32
  special = false
}

resource "aws_secretsmanager_secret" "db" {
  name = "search-prod/db"
}
resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = jsonencode({
    username = "search"
    password = random_password.db.result
    dbname   = "searchdb"
    host     = module.aurora.cluster_endpoint          # writer
    reader   = module.aurora.cluster_reader_endpoint   # reader
    port     = 5432
  })
}

module "aurora" {
  source  = "terraform-aws-modules/rds-aurora/aws"
  version = "~> 9.10"

  name              = "search-prod"
  engine            = "aurora-postgresql"
  engine_version    = "16.4"
  instance_class    = "db.r6g.large"
  instances         = { writer = {}, reader = {} }   # Multi-AZ: writer + reader

  vpc_id               = module.vpc.vpc_id
  db_subnet_group_name = module.vpc.database_subnet_group_name
  security_group_rules = {
    from_app = { source_security_group_id = module.eks.node_security_group_id }  # only nodes can reach it
  }

  master_username             = "search"
  master_password             = random_password.db.result
  manage_master_user_password = false

  storage_encrypted   = true                 # KMS at rest
  deletion_protection = true
  backup_retention_period = 14
  performance_insights_enabled = true
  apply_immediately   = false
}
```

**Why Aurora, not the StatefulSet:** automated failover (writer↔reader promotion), continuous backup
to S3, point-in-time restore, encryption with KMS, `deletion_protection`, and a **reader endpoint** the
app can send searches to. Running that yourself in-cluster is a full-time job.

### 2.5 ECR + IRSA roles (`ecr.tf`, `irsa.tf`)

```hcl
resource "aws_ecr_repository" "app" {
  name                 = "search-service"
  image_tag_mutability = "IMMUTABLE"                 # can't overwrite a pushed tag
  image_scanning_configuration { scan_on_push = true }
  encryption_configuration { encryption_type = "KMS" }
}

# IRSA role letting the app pod read ONLY its own secret
module "irsa_app" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.44"
  role_name = "search-app"
  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["search:search-service"]
    }
  }
  role_policy_arns = { secrets = aws_iam_policy.read_db_secret.arn }
}
```

Apply order: `terraform init && terraform apply`. Then wire kubectl:
`aws eks update-kubeconfig --name search-prod --region ap-south-1`.

---

## 3. Layer 2 — Hardening the application

The app gains the things that make it operable. Full code in `internal/` in the scaffold; the load-
bearing decisions:

### 3.1 Config: validate and fail fast

Parse every env var at startup and **exit non-zero** if anything required is missing or malformed. A
pod that can't be configured should crash immediately (and loudly), not serve 500s.

### 3.2 Structured logging (`log/slog`, JSON)

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: level}))
slog.SetDefault(logger)
```

JSON logs to stdout → Fluent Bit ships them to CloudWatch Logs. Include a **request ID** on every line
(via middleware) so you can trace one request across the log stream.

### 3.3 Prometheus metrics — RED

Expose `/metrics` with request **R**ate, **E**rror rate, and **D**uration histogram, plus DB pool
gauges. This is what your Grafana dashboards and alerts read.

```go
var httpDuration = prometheus.NewHistogramVec(prometheus.HistogramOpts{
    Name:    "http_request_duration_seconds",
    Buckets: prometheus.DefBuckets,
}, []string{"method", "route", "status"})
// wrap handlers with middleware that observes this on every request
```

### 3.4 Writer/reader split + TLS to Aurora (`internal/db`)

Two pools: writes go to the **cluster (writer) endpoint**, searches to the **reader endpoint** so read
load spreads across Aurora replicas. Connect with `sslmode=verify-full` and the **RDS CA bundle** so
the client verifies the server cert (not just encrypts).

```go
writer, _ := pgxpool.New(ctx, dsn(cfg.WriterHost))  // POST /documents
reader, _ := pgxpool.New(ctx, dsn(cfg.ReaderHost))  // GET  /search

func dsn(host string) string {
    return fmt.Sprintf("postgres://%s:%s@%s:5432/%s?sslmode=verify-full&sslrootcert=/etc/ssl/rds/global-bundle.pem&pool_max_conns=20",
        cfg.User, cfg.Pass, host, cfg.DB)
}
```

Every query takes a `context` with a timeout — no unbounded DB calls that pile up under load.

### 3.5 Migrations with `golang-migrate`

Versioned `up`/`down` SQL run as a pre-deploy **Job** (or Argo CD PreSync hook), never from N racing
app replicas. Keeps schema changes auditable and reversible.

### 3.6 Graceful shutdown + `preStop`

The app already drains on SIGTERM. Add a small `preStop` sleep in the pod so the ALB deregisters the
target *before* the process exits → zero failed requests during rollouts and scale-downs.

---

## 4. Layer 3 — Container hardening (`Dockerfile`)

```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/server ./cmd/server

FROM alpine:3.20 AS certs
RUN apk add --no-cache ca-certificates curl && \
    curl -sSfo /global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=certs /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=certs /global-bundle.pem /etc/ssl/rds/global-bundle.pem
COPY --from=build  /out/server /server
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

Static binary on distroless-nonroot, RDS CA bundle baked in. In CI (§7): **Trivy** scans for CVEs,
**cosign** signs the image, and ECR blocks mutable tags. The Kubernetes securityContext (§5) enforces
read-only rootfs and dropped capabilities at runtime.

---

## 5. Layer 4 — Kubernetes workload hardening (`deploy/k8s/`)

### 5.1 ServiceAccount (IRSA) + External Secrets

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: search-service
  namespace: search
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT>:role/search-app   # from Terraform IRSA
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: search
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-secretsmanager, kind: ClusterSecretStore }
  target: { name: db-credentials }         # creates a k8s Secret from Secrets Manager
  dataFrom:
    - extract: { key: search-prod/db }     # pulls username/password/host/reader/dbname
```

The password never appears in Git — ESO pulls it from Secrets Manager at runtime and refreshes it,
picking up rotations automatically.

### 5.2 Deployment — the hardened spec

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: search-service
  namespace: search
spec:
  replicas: 3
  strategy:
    rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }   # never drop below capacity mid-rollout
  selector: { matchLabels: { app: search-service } }
  template:
    metadata:
      labels: { app: search-service }
    spec:
      serviceAccountName: search-service
      securityContext:
        runAsNonRoot: true
        seccompProfile: { type: RuntimeDefault }
      topologySpreadConstraints:                        # spread pods across AZs
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector: { matchLabels: { app: search-service } }
      containers:
        - name: search-service
          image: <ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/search-service@sha256:...  # digest, not tag
          ports: [{ containerPort: 8080, name: http }]
          envFrom:
            - secretRef: { name: db-credentials }
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          startupProbe:   { httpGet: { path: /healthz, port: http }, failureThreshold: 30, periodSeconds: 2 }
          readinessProbe: { httpGet: { path: /readyz,  port: http }, periodSeconds: 10 }
          livenessProbe:  { httpGet: { path: /healthz, port: http }, periodSeconds: 15 }
          lifecycle:
            preStop: { exec: { command: ["sleep", "10"] } }   # let ALB deregister first
          resources:
            requests: { cpu: 100m, memory: 96Mi }
            limits:   { cpu: 500m, memory: 192Mi }
```

### 5.3 PodDisruptionBudget, HPA, NetworkPolicy, ServiceMonitor

```yaml
# PDB — keep 2 up during voluntary disruptions (node drains, upgrades)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: search-service, namespace: search }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: search-service } }
---
# HPA — CPU baseline; add a custom RPS metric via the Prometheus adapter for real load-based scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: search-service, namespace: search }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: search-service }
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 65 } }
---
# NetworkPolicy — default-deny egress except DNS + Aurora:5432 (blocks lateral movement)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: search-service, namespace: search }
spec:
  podSelector: { matchLabels: { app: search-service } }
  policyTypes: ["Egress"]
  egress:
    - to: [{ ipBlock: { cidr: 10.0.201.0/24 } }, { ipBlock: { cidr: 10.0.202.0/24 } }, { ipBlock: { cidr: 10.0.203.0/24 } }]
      ports: [{ protocol: TCP, port: 5432 }]
    - ports: [{ protocol: UDP, port: 53 }, { protocol: TCP, port: 53 }]
---
# ServiceMonitor — tells Prometheus to scrape /metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: search-service, namespace: search }
spec:
  selector: { matchLabels: { app: search-service } }
  endpoints: [{ port: http, path: /metrics, interval: 15s }]
```

---

## 6. Layer 5 — Ingress with TLS (`deploy/k8s/ingress.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: search-service
  namespace: search
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'                       # force HTTPS
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:<REGION>:<ACCOUNT>:certificate/<id>
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:<REGION>:<ACCOUNT>:regional/webacl/<id>
    external-dns.alpha.kubernetes.io/hostname: search.example.com       # ExternalDNS writes Route 53
spec:
  ingressClassName: alb
  rules:
    - host: search.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: search-service, port: { number: 80 } } }
```

ACM issues/renews the cert; the LB controller wires it to the ALB HTTPS listener; **ExternalDNS**
manages the Route 53 record automatically; **WAF** sits in front with managed rule groups.

---

## 7. Layer 6 — CI/CD (GitHub Actions → ECR → Argo CD GitOps)

![CI/CD and GitOps pipeline](diagrams-prod/prod-02-cicd.png)

**Why GitOps:** the Git repo is the single source of truth for what's deployed. Argo CD continuously
reconciles the cluster to match Git — the same reconciliation model K8s uses internally, applied to
delivery. Rollback = `git revert`. Every change is a reviewed PR with an audit trail. The CI pipeline
never holds cluster credentials; it only pushes an image and bumps a digest.

Pipeline stages: `go test -race` + `go vet` + `golangci-lint` → build → **Trivy** (fail on HIGH/CRITICAL
CVEs) → **cosign** sign → push to ECR by **immutable git-SHA digest** → update the deploy repo → Argo CD
syncs. Migrations run as an Argo CD **PreSync hook** so schema changes land before the new pods.

---

## 8. Layer 7 — Observability

- **Metrics:** install `kube-prometheus-stack` (Prometheus + Grafana + Alertmanager) via Helm; the
  `ServiceMonitor` scrapes `/metrics`. Build a RED dashboard (rate, errors, p50/p95/p99 latency) plus
  DB pool saturation. Alert on error rate, p99 latency, pod restarts, and HPA maxed-out.
- **Logs:** **Fluent Bit** DaemonSet ships JSON stdout to **CloudWatch Logs** (or Loki). Structured
  fields + request IDs make queries actually useful.
- **Traces:** instrument with **OpenTelemetry**; export to **AWS X-Ray** (or Tempo) to see the
  request → handler → Aurora span breakdown and find slow queries.
- **Infra:** enable **CloudWatch Container Insights**; EKS control-plane logs already on (§2.3);
  Aurora **Performance Insights** on (§2.4) for slow-query analysis.

The request path, showing the read/write split you're now observing:

![Request path with read/write split](diagrams-prod/prod-03-request-flow.png)

---

## 9. Layer 8 — Autoscaling & resilience

- **Pods:** HPA on CPU now; add **RPS via the Prometheus adapter** for demand-based scaling.
- **Nodes:** install **Karpenter** — it provisions right-sized nodes (including **Spot**) in seconds
  when pods go Pending, and consolidates/removes them when idle. Cheaper and faster than the classic
  Cluster Autoscaler.
- **Disruption safety:** the **PDB** (§5.3) keeps ≥2 pods up during node drains/upgrades;
  topology spread keeps them across AZs so losing one AZ can't take the service down.
- **DB:** Aurora auto-fails-over writer→reader (~30s); the app's connection pools reconnect and the
  reader endpoint keeps serving searches throughout.

---

## 10. First-deploy runbook

```bash
# 1. Infra
cd terraform && terraform init && terraform apply
aws eks update-kubeconfig --name search-prod --region ap-south-1

# 2. Platform add-ons (once per cluster) — Helm
#    - AWS Load Balancer Controller (see previous guide §8.5, with IRSA)
#    - External Secrets Operator, ExternalDNS, kube-prometheus-stack, Karpenter, Argo CD
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
# ... (values files in deploy/helm/)

# 3. Register the app with Argo CD (GitOps takes over from here)
kubectl apply -f deploy/argocd/application.yaml

# 4. CI builds+pushes the first image; Argo CD syncs; watch it land
argocd app sync search-service
kubectl get pods -n search -w

# 5. Verify
curl -s https://search.example.com/search?q=container | jq
```

---

## 11. Day-2 operations

- **Deploy a change:** merge PR → CI pushes new digest → Argo CD auto-syncs (or manual sync on prod).
  Rollback: `git revert` or `argocd app rollback`.
- **Schema migration:** add a versioned file under `migrations/`; the PreSync Job applies it before
  new pods roll. Always ship backward-compatible migrations (expand/contract) so old+new pods coexist.
- **Backups/DR:** Aurora continuous backup + PITR (14-day window here). For cross-region DR, add an
  Aurora **global database**. Practice a restore — an untested backup isn't a backup.
- **Secret rotation:** Secrets Manager rotates the DB password on a schedule; ESO re-syncs within
  `refreshInterval`; connections pick up new creds on reconnect.
- **Incident basics:** Grafana for the RED dashboard, `kubectl logs`/CloudWatch for errors,
  X-Ray for slow spans, Aurora Performance Insights for DB hotspots.

---

## 12. Security checklist

- [x] Nodes + DB in **private subnets**; DB reachable only from the node security group
- [x] **TLS everywhere:** HTTPS at the ALB (ACM), `verify-full` app→Aurora, KMS encryption at rest
- [x] **No secrets in Git** — Secrets Manager + ESO + rotation
- [x] **Least-privilege IRSA** — app can read only its own secret
- [x] Container: non-root, **read-only rootfs**, all caps dropped, seccomp RuntimeDefault
- [x] **Immutable, scanned, signed** images (ECR IMMUTABLE + Trivy + cosign); deploy by digest
- [x] **NetworkPolicy** default-deny egress; **WAF** on the ALB
- [x] EKS **audit logging** + CloudTrail; access via **EKS access entries**, not shared creds
- [ ] Add: Pod Security Admission `restricted`, private cluster endpoint, VPC endpoints for AWS APIs

---

## 13. Cost notes

Biggest line items: EKS control plane (~$0.10/hr), nodes (right-size with Karpenter + Spot for
non-critical), Aurora instances (largest cost — start `db.r6g.large`, scale with load; consider
**Aurora Serverless v2** for spiky/low traffic), NAT gateways (one per AZ isn't free — VPC endpoints
cut NAT data cost), and ALB. Turn on **Cost Explorer** tags (`Project`, `Environment`) so spend is
attributable. Non-prod environments should scale to zero / smaller instance classes off-hours.

---

## 14. Production readiness checklist (the summary)

**Reliability:** multi-AZ nodes · PDB · topology spread · HPA + Karpenter · Aurora Multi-AZ · graceful drain
**Security:** private networking · TLS end-to-end · Secrets Manager + rotation · least-priv IRSA · hardened container · NetworkPolicy · WAF · signed images
**Observability:** RED metrics + Grafana · structured logs → CloudWatch · OTel traces → X-Ray · alerts · Aurora PI
**Delivery:** Terraform IaC + remote state · GitHub Actions (test/scan/sign) · Argo CD GitOps · immutable digests · reversible migrations
**Operability:** documented runbook · tested backup/restore · rollback path · cost tagging

If every box above is real (not aspirational), you can hand someone the pager without wincing.
