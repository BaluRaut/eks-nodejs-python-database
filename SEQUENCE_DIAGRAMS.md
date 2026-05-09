# Sequence Diagrams — AWS EKS/ECR Node.js + Postgres

All diagrams use [Mermaid](https://mermaid.js.org/) syntax and render automatically on GitHub.

---

## 1. High-Level End-to-End Flow

```mermaid
sequenceDiagram
    actor Dev as Developer
    participant GH as GitHub
    participant CI as CircleCI
    participant ECR as AWS ECR
    participant EKS as AWS EKS
    participant RDS as RDS/Postgres
    actor User as End User
    participant R53 as Route 53 + ACM

    Dev->>GH: git push (main branch)
    GH-->>CI: Webhook trigger

    rect rgb(30, 50, 80)
        note over CI,ECR: Build & Publish Phase
        CI->>CI: Checkout code, run tests
        CI->>ECR: docker build + docker push (image:SHA)
    end

    rect rgb(30, 60, 50)
        note over CI,EKS: Deploy Phase
        CI->>EKS: kubectl set image / apply manifests
        EKS->>ECR: Pull new image
        EKS->>EKS: Rolling update (zero downtime)
        EKS->>RDS: Verify DB connection
    end

    rect rgb(60, 30, 50)
        note over User,R53: Request Phase
        User->>R53: HTTPS request (api.yourdomain.com)
        R53->>EKS: Resolve to ALB / LoadBalancer IP
        EKS->>RDS: App queries database
        RDS-->>EKS: Query result
        EKS-->>User: HTTP 200 response
    end
```

---

## 2. CI/CD Pipeline — CircleCI Detail

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub (main)
    participant CI as CircleCI
    participant ECR as AWS ECR
    participant EKS as EKS Cluster

    Dev->>GH: git push

    GH->>CI: Trigger pipeline (webhook)

    CI->>CI: job: build-and-push
    CI->>CI: Checkout source code
    CI->>CI: docker build -t app:$CIRCLE_SHA1 .

    CI->>ECR: aws ecr get-login-password | docker login
    CI->>ECR: docker tag app:SHA <account>.dkr.ecr.<region>.amazonaws.com/app:SHA
    CI->>ECR: docker push <account>.dkr.ecr.<region>.amazonaws.com/app:SHA
    ECR-->>CI: Push confirmed ✓

    CI->>CI: job: deploy-to-eks
    CI->>EKS: aws eks update-kubeconfig
    CI->>EKS: kubectl set image deployment/app app=<ecr-url>:SHA
    EKS-->>CI: Deployment triggered

    CI->>EKS: kubectl rollout status deployment/app
    loop Wait for rollout
        EKS-->>CI: Pods: 1/3 → 2/3 → 3/3 ready
    end
    EKS-->>CI: Rollout complete ✓
    CI-->>Dev: Pipeline passed ✅
```

---

## 3. Docker Image Build & ECR Push

```mermaid
sequenceDiagram
    participant Dev as Developer / CI
    participant Docker as Docker Daemon
    participant ECR as AWS ECR Registry

    Dev->>Docker: docker build -t app:latest .
    Docker->>Docker: Read Dockerfile layers
    Docker->>Docker: npm install (cached layer)
    Docker->>Docker: COPY app source
    Docker->>Docker: EXPOSE 3000
    Docker-->>Dev: Image built ✓ (e.g. 180 MB)

    Dev->>ECR: aws ecr get-login-password --region us-east-1
    ECR-->>Dev: Auth token (12h TTL)

    Dev->>Docker: docker login -u AWS -p <token> <ecr-url>
    Docker-->>Dev: Login succeeded ✓

    Dev->>Docker: docker tag app:latest <ecr-url>/app:SHA
    Dev->>Docker: docker push <ecr-url>/app:SHA
    Docker->>ECR: Upload layers (only delta layers)
    ECR-->>Docker: Layer already exists / Pushed
    ECR-->>Dev: Digest: sha256:abc123 ✓

    note over ECR: Image immutable once pushed.\nTag :latest can be overwritten;\ntag :SHA is permanent.
```

---

## 4. EKS Pod Scheduling & Deployment

```mermaid
sequenceDiagram
    participant CI as CircleCI / kubectl
    participant API as K8s API Server
    participant Sched as K8s Scheduler
    participant Node as Worker Node
    participant Kubelet as Kubelet (on Node)
    participant ECR as AWS ECR
    participant Pod as App Pod

    CI->>API: kubectl apply -f deployment.yaml
    API->>API: Validate manifest, store in etcd

    API->>Sched: New ReplicaSet detected (3 replicas)
    Sched->>Sched: Score nodes (CPU, memory, affinity)
    Sched->>API: Bind Pod → Node

    API->>Kubelet: PodSpec assigned
    Kubelet->>ECR: Pull image <ecr-url>/app:SHA
    ECR-->>Kubelet: Image layers downloaded

    Kubelet->>Pod: Start container (entrypoint: node server.js)
    Pod->>Pod: Load env from ConfigMap / Secret
    Pod->>Pod: Connect to Postgres (DB_HOST, DB_PASS)

    Kubelet->>API: Pod status = Running
    API-->>CI: rollout status: 3/3 ready ✓

    note over API,Pod: Old pods terminated after\nnew pods pass readinessProbe.
```

---

## 5. DNS Resolution & HTTPS via Route 53 + ACM

```mermaid
sequenceDiagram
    actor User as Browser / Client
    participant R53 as Route 53
    participant ACM as AWS ACM (TLS Cert)
    participant ALB as Application Load Balancer
    participant Ingress as K8s Ingress Controller
    participant Svc as K8s Service (ClusterIP)
    participant Pod as App Pod

    User->>R53: DNS query: api.yourdomain.com
    R53-->>User: CNAME → <alb-dns>.elb.amazonaws.com

    User->>ALB: HTTPS :443 (TLS ClientHello)
    ALB->>ACM: Fetch certificate for api.yourdomain.com
    ACM-->>ALB: Certificate (auto-renewed by AWS)
    ALB-->>User: TLS handshake complete ✓

    User->>ALB: GET /api/users (HTTPS)
    ALB->>Ingress: Forward HTTP to Ingress (port 80 internally)
    Ingress->>Ingress: Match host/path rules
    Ingress->>Svc: Route to Service: app-service:3000
    Svc->>Pod: Load balance across pods (round-robin)
    Pod-->>Svc: Response JSON
    Svc-->>Ingress: Response
    Ingress-->>ALB: Response
    ALB-->>User: HTTPS 200 OK ✓
```

---

## 6. Database Connection Flow (App → RDS Postgres)

```mermaid
sequenceDiagram
    participant Pod as App Pod
    participant Secret as K8s Secret
    participant SG as AWS Security Group
    participant RDS as RDS Postgres Instance
    participant PG as pg (node-postgres)

    Pod->>Secret: Mount env vars (DB_HOST, DB_USER, DB_PASS, DB_PORT)
    Secret-->>Pod: Injected at container start

    Pod->>PG: new Pool({ host, user, password, database, port:5432 })
    PG->>RDS: TCP SYN → <rds-endpoint>:5432

    RDS->>SG: Inbound rule check (port 5432, source: EKS node SG)
    alt Security group allows
        SG-->>RDS: Allow ✓
        RDS-->>PG: TCP connection established
        PG->>RDS: PostgreSQL handshake + auth (md5/scram)
        RDS-->>PG: Auth OK
        PG-->>Pod: Pool ready (min 2, max 10 connections)
    else Security group denies
        SG-->>PG: Connection refused ✗
        PG-->>Pod: Error: connect ECONNREFUSED
        note over Pod: Fix: add EKS node SG to\nRDS inbound rules on port 5432
    end

    Pod->>PG: pool.query('SELECT * FROM users WHERE id=$1', [id])
    PG->>RDS: Execute query
    RDS-->>PG: Result rows
    PG-->>Pod: { rows: [...] }
```

---

## 7. Kubernetes Auto-Scaling Flow (HPA)

```mermaid
sequenceDiagram
    participant User as Incoming Traffic
    participant Pod as App Pods (initial: 2)
    participant Metrics as Metrics Server
    participant HPA as HPA Controller
    participant API as K8s API Server
    participant Sched as Scheduler
    participant NewPod as New Pod

    User->>Pod: High traffic spike
    Pod->>Pod: CPU usage rises > 70%

    loop Every 15s
        Metrics->>Pod: Scrape CPU/memory metrics
        Metrics-->>HPA: Current CPU: 85% (target: 50%)
    end

    HPA->>HPA: Calculate desired replicas\n= ceil(2 * 85/50) = 4
    HPA->>API: Scale Deployment replicas: 2 → 4

    API->>Sched: Schedule 2 new pods
    Sched->>NewPod: Create Pod on available node
    NewPod->>NewPod: Pull image, start container
    NewPod-->>API: Status: Running ✓

    User->>Pod: Traffic distributed across 4 pods
    note over HPA: Scale-down waits 5 min\n(stabilization window) before\nremoving pods.
```

---

## 8. Secret & ConfigMap Injection Flow

```mermaid
sequenceDiagram
    participant Dev as Developer / CI
    participant API as K8s API Server
    participant etcd as etcd (encrypted)
    participant Kubelet as Kubelet
    participant Pod as App Pod
    participant App as Node.js App

    Dev->>API: kubectl create secret generic db-secret\n--from-literal=DB_PASS=***
    API->>etcd: Store secret (encrypted at rest)
    etcd-->>API: Stored ✓

    Dev->>API: kubectl apply -f configmap.yaml\n(DB_HOST, DB_PORT, DB_NAME)
    API->>etcd: Store configmap
    etcd-->>API: Stored ✓

    note over API,Pod: At Pod startup:
    API->>Kubelet: PodSpec with envFrom references
    Kubelet->>API: Fetch Secret: db-secret
    Kubelet->>API: Fetch ConfigMap: app-config
    API-->>Kubelet: Secret + ConfigMap values

    Kubelet->>Pod: Inject env vars into container\n(DB_HOST, DB_PORT, DB_NAME, DB_PASS)
    Pod->>App: process.env.DB_HOST, process.env.DB_PASS
    App-->>Pod: Connected to database ✓

    note over etcd: Secrets are base64 encoded,\nnot encrypted by default in K8s.\nUse AWS KMS envelope encryption\nor Sealed Secrets for production.
```

---

## Reading These Diagrams

| Symbol | Meaning |
|--------|---------|
| `->>` | Synchronous request / call |
| `-->>` | Response / return |
| `rect` | Grouped phase with background color |
| `loop` | Repeated action |
| `alt/else` | Conditional branch |
| `note` | Explanatory annotation |

> Render these diagrams on GitHub (automatic), in VS Code with the Mermaid Preview extension, or at [mermaid.live](https://mermaid.live).
