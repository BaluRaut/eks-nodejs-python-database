# Kubernetes Bottlenecks: A Practical Guide for EKS

This guide covers the most common performance and reliability bottlenecks in Kubernetes, with a focus on AWS EKS running Node.js and Python microservices. Every section includes the kubectl commands and YAML patterns you need to diagnose and fix each issue.

---

## Table of Contents

1. [Resource Bottlenecks](#1-resource-bottlenecks)
2. [Network Bottlenecks](#2-network-bottlenecks)
3. [Storage Bottlenecks](#3-storage-bottlenecks)
4. [Application-Level Bottlenecks](#4-application-level-bottlenecks)
5. [EKS-Specific Bottlenecks](#5-eks-specific-bottlenecks)
6. [Observability for Bottleneck Detection](#6-observability-for-bottleneck-detection)
7. [Quick Diagnosis Checklist](#7-quick-diagnosis-checklist)

---

## 1. Resource Bottlenecks

### 1.1 CPU Throttling

**How it happens**

Kubernetes enforces CPU limits using Linux CFS (Completely Fair Scheduler) bandwidth control. When a container's CPU limit is set, the kernel allows it to use that many CPU-milliseconds per 100ms window. If the container exhausts its quota early in the window, it is throttled — stalled — for the remainder. This is invisible to the process; it does not see an error, it just runs slowly.

A container with `limits.cpu: 500m` can use 50ms of CPU time every 100ms window. A burst of work that needs 80ms in one window will be throttled for the remaining 20ms even if the node has spare capacity.

**Why your Node Hono and FastAPI services are vulnerable**

The manifests in this repo (`k8s/multi-api.yaml`) define no resource requests or limits at all. This means:
- The pods run in the `BestEffort` QoS class, the lowest priority.
- During node pressure they are the first to be evicted.
- There is no CPU limit, so there is also no throttling — but if you add limits without careful tuning, you will introduce throttling.

**Detecting CPU throttling**

Install metrics-server first if it is not present:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Then check live usage:

```bash
# Live CPU and memory for all pods
kubectl top pods -n default

# Live usage per container (useful for multi-container pods)
kubectl top pods -n default --containers

# Live node utilization
kubectl top nodes
```

`kubectl top` shows current usage, not throttling. For throttling you need the kernel cgroup metric `container_cpu_cfs_throttled_periods_total`. With Prometheus:

```promql
# Throttling ratio per container (0 = no throttling, 1 = fully throttled)
rate(container_cpu_cfs_throttled_periods_total{container!=""}[5m])
/
rate(container_cpu_cfs_periods_total{container!=""}[5m])
```

Alert when this ratio exceeds 0.25 (25% of periods throttled) for any production container.

**Fixing CPU throttling**

```yaml
resources:
  requests:
    cpu: "250m"      # What the scheduler reserves on the node
  limits:
    cpu: "1000m"     # 4x the request — gives burst headroom
```

For latency-sensitive services (like the Node Hono API), consider removing the CPU limit entirely and relying only on requests. This prevents throttling at the cost of less predictable neighbor behavior.

```yaml
resources:
  requests:
    cpu: "500m"
  # No limits.cpu — pod can burst freely
  limits:
    memory: "512Mi"  # Always set memory limits
```

---

### 1.2 Memory Pressure and OOMKilled Pods

**How it happens**

When a container exceeds its `limits.memory`, the Linux OOM killer terminates it immediately. The pod restarts, you see `OOMKilled` in the status, and `CrashLoopBackOff` follows if this happens repeatedly.

Memory limits are hard caps — unlike CPU, there is no throttling. The process dies.

**Detecting OOMKilled pods**

```bash
# See pod restart counts and current status
kubectl get pods -n default

# Full detail including last termination reason
kubectl describe pod <pod-name> -n default

# Filter events for OOMKilled across the namespace
kubectl get events -n default --sort-by='.lastTimestamp' | grep -i oom

# Check previous container exit code (137 = OOMKilled)
kubectl get pod <pod-name> -n default -o jsonpath=\
  '{.status.containerStatuses[0].lastState.terminated.reason}'
```

Example output when a pod was OOMKilled:

```
Last State:  Terminated
  Reason:    OOMKilled
  Exit Code: 137
  Started:   Mon, 04 May 2026 09:00:00 +0000
  Finished:  Mon, 04 May 2026 09:00:45 +0000
```

**Fixing memory pressure**

Set requests based on observed steady-state usage. Set limits at 2x the request for most stateless services. For Postgres or other stateful services, measure carefully — setting limits too low on a database is dangerous.

```yaml
# Example for the python-fastapi container
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "500m"
```

Use Vertical Pod Autoscaler (VPA) in recommendation mode to get data-driven suggestions:

```bash
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vertical-pod-autoscaler.yaml

# Create a VPA object in recommendation mode (does not change pods)
```

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: node-hono-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: node-hono
  updatePolicy:
    updateMode: "Off"   # Recommend only, do not mutate pods
```

```bash
# Read recommendations after 24 hours of traffic
kubectl describe vpa node-hono-vpa -n default
```

---

### 1.3 Setting Requests vs Limits Correctly

The ratio between request and limit is critical.

| Scenario | Effect |
|---|---|
| No requests, no limits | BestEffort QoS — evicted first under pressure |
| Requests = Limits | Guaranteed QoS — never throttled or evicted, but wastes capacity |
| Requests < Limits | Burstable QoS — best for most workloads, allows overcommit |
| Limits much larger than requests | Node overcommit, potential cascading OOM on all pods |
| CPU limit very close to request | Frequent throttling even at moderate load |

**Recommended ratio for this repo's services**

For the Node Hono and Python FastAPI microservices, which are HTTP request handlers:

```yaml
resources:
  requests:
    cpu: "100m"      # Baseline reservation
    memory: "128Mi"
  limits:
    cpu: "500m"      # 5x burst headroom for spikes
    memory: "256Mi"  # 2x headroom for memory
```

For Postgres, requests and limits should be close to equal to ensure the scheduler always finds a node with enough real capacity:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "1000m"
    memory: "2Gi"
```

---

### 1.4 Node Resource Exhaustion

**How the scheduler fails**

When all nodes are full (allocatable CPU and memory consumed), new pods remain in `Pending` state. The scheduler cannot place them. This is distinct from container-level OOM — the pod never starts.

```bash
# See pending pods
kubectl get pods --all-namespaces | grep Pending

# Understand why a pod is pending
kubectl describe pod <pod-name> -n default
# Look for: "0/3 nodes are available: 3 Insufficient cpu."
```

**Cluster Autoscaler**

Cluster Autoscaler watches for unschedulable pods and requests new nodes from the EC2 Auto Scaling Group.

```bash
# Check Cluster Autoscaler logs for scaling decisions
kubectl logs -n kube-system deployment/cluster-autoscaler | tail -50

# Check current node group status
kubectl get nodes -o wide
```

Key Cluster Autoscaler tuning parameters:

```yaml
# In the cluster-autoscaler deployment args:
- --scale-down-delay-after-add=10m      # Wait 10m before scaling down after a scale-up
- --scale-down-unneeded-time=10m        # Node must be unneeded for 10m before removal
- --max-node-provision-time=15m         # Treat a node as failed if not ready in 15m
- --balance-similar-node-groups=true    # Spread across node groups evenly
- --skip-nodes-with-local-storage=false # Allow scale-down of nodes with emptyDir
```

**Karpenter** (the modern alternative)

Karpenter is an open-source node provisioner that provisions nodes in response to unschedulable pods without pre-defined node groups. It is faster than Cluster Autoscaler (typically under 60 seconds for a new node) and uses EC2 spot and on-demand intelligently.

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.large", "m5.xlarge", "m5.2xlarge"]
  limits:
    cpu: "100"
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
```

---

## 2. Network Bottlenecks

### 2.1 DNS Resolution Bottlenecks

**The ndots problem**

Kubernetes sets `ndots: 5` in `/etc/resolv.conf` by default. This means any DNS name with fewer than 5 dots triggers up to 5 search-domain lookups before the resolver tries the absolute name.

When your Node Hono pod queries `postgres-service`:

1. Try `postgres-service.default.svc.cluster.local` — resolves, done.

But when it queries an external name like `api.example.com`:

1. Try `api.example.com.default.svc.cluster.local` — NXDOMAIN
2. Try `api.example.com.svc.cluster.local` — NXDOMAIN
3. Try `api.example.com.cluster.local` — NXDOMAIN
4. Try `api.example.com.us-east-1.compute.internal` — NXDOMAIN
5. Try `api.example.com` — resolves

That is 5 DNS round trips for every external call. At high QPS this floods CoreDNS.

**Detecting DNS bottlenecks**

```bash
# Check CoreDNS pod health
kubectl get pods -n kube-system -l k8s-app=kube-dns

# CoreDNS error rate
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100 | grep -i error

# Check if CoreDNS is CPU-throttled
kubectl top pods -n kube-system -l k8s-app=kube-dns
```

With Prometheus (if installed):

```promql
# CoreDNS request rate
rate(coredns_dns_requests_total[5m])

# CoreDNS error rate
rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m])
```

**Fix 1: Reduce ndots per pod**

```yaml
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"    # Only 2 dots needed before trying absolute names
      - name: single-request-reopen
        value: ""     # Avoid UDP race conditions
```

With `ndots: 2`, `postgres-service.default` resolves in one lookup. External names like `api.example.com` (2 dots) resolve directly.

**Fix 2: NodeLocal DNSCache**

NodeLocal DNSCache runs a caching DNS agent on every node as a DaemonSet. Pods hit a local cache (169.254.20.10) instead of CoreDNS for every request, eliminating cross-node DNS traffic for cached entries and reducing CoreDNS load by 60-80%.

```bash
# Deploy NodeLocal DNSCache (EKS 1.25+)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
```

After deployment, patch CoreDNS to use TCP for upstream (avoids UDP timeouts):

```yaml
# In CoreDNS ConfigMap, add to the forward block:
forward . 169.254.20.10 {
  prefer_udp
}
```

---

### 2.2 Service Mesh Overhead

If you add Istio or Linkerd to this repo's services, each pod gets a sidecar proxy (Envoy for Istio, linkerd-proxy for Linkerd). This adds:

- **Latency**: 0.5ms–2ms per hop for the proxy traversal
- **CPU**: 50m–200m per pod for the sidecar
- **Memory**: 50Mi–150Mi per pod for the sidecar

**Detecting mesh overhead**

```bash
# Count sidecar containers
kubectl get pods -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{","}{end}{"\n"}{end}'

# Check sidecar resource usage specifically
kubectl top pods -n default --containers | grep -E "envoy|linkerd"
```

**Mitigation**: Disable the sidecar for non-critical services or batch jobs using annotations:

```yaml
metadata:
  annotations:
    sidecar.istio.io/inject: "false"     # Istio
    linkerd.io/inject: disabled          # Linkerd
```

---

### 2.3 Inter-Pod Latency: Within vs Across AZs

AWS charges for cross-AZ data transfer and adds network latency. In EKS, pods on nodes in different AZs communicate over the VPC backbone — typically 1-5ms additional latency compared to same-AZ pod-to-pod (sub-millisecond).

**Detecting cross-AZ traffic**

```bash
# See which AZ each node is in
kubectl get nodes -o custom-columns=\
  NAME:.metadata.name,\
  AZ:.metadata.labels.'topology\.kubernetes\.io/zone',\
  TYPE:.metadata.labels.'node\.kubernetes\.io/instance-type'
```

**Fix: Topology-aware routing**

Kubernetes topology-aware hints (1.23+) tell kube-proxy to prefer endpoints in the same zone:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-hono-service
  annotations:
    service.kubernetes.io/topology-mode: "Auto"
spec:
  selector:
    app: node-hono
  ports:
  - port: 80
    targetPort: 3000
```

For pod-to-pod affinity, use pod topology spread constraints:

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: node-hono
```

---

### 2.4 Network Policies That Block Traffic Silently

A NetworkPolicy that blocks traffic produces no error message on the sending side — the connection simply times out. This is one of the hardest bottlenecks to diagnose.

**Detecting blocked traffic**

```bash
# List all NetworkPolicies in the namespace
kubectl get networkpolicies -n default

# Describe a specific policy to see its rules
kubectl describe networkpolicy <policy-name> -n default

# Test connectivity from inside a pod
kubectl exec -it <pod-name> -n default -- curl -v http://postgres-service:5432 --max-time 5

# Use a debug pod with network tools
kubectl run nettest --rm -it --image=nicolaka/netshoot -- bash
# Inside: nmap -p 5432 postgres-service
# Inside: dig postgres-service
```

**Example: Allow node-hono to reach postgres**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-node-hono-to-postgres
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: node-hono
    - podSelector:
        matchLabels:
          app: python-fastapi
    ports:
    - protocol: TCP
      port: 5432
```

---

## 3. Storage Bottlenecks

### 3.1 PVC IOPS Limits: gp2 vs gp3

The Postgres deployment in this repo (`k8s/postgres.yaml`) uses the default storage class, which in most EKS clusters is `gp2`. This matters enormously for database performance.

| Volume Type | Baseline IOPS | Max IOPS | Throughput | Cost |
|---|---|---|---|---|
| gp2 | 3 IOPS/GB (min 100) | 3,000 burst | 250 MB/s | Lower |
| gp3 | 3,000 baseline | 16,000 | 1,000 MB/s | ~20% less than gp2 |
| io1/io2 | Provisioned | 64,000 | 1,000 MB/s | Higher |

A 20GB gp2 volume gets only 100 IOPS baseline. Under load, Postgres will throttle.

**Detecting storage throttling**

```bash
# Check if a pod is experiencing I/O wait
kubectl exec -it <postgres-pod> -- iostat -x 1 5

# Check the storage class in use
kubectl get pvc -n default
kubectl describe pvc <pvc-name> -n default

# Check current storage class
kubectl get storageclass
```

In CloudWatch, look for `VolumeQueueLength` and `VolumeReadOps`/`VolumeWriteOps` exceeding provisioned IOPS.

**Creating a gp3 storage class**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"        # Baseline, no extra cost vs gp2
  throughput: "125"   # MB/s
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

Install the EBS CSI driver if not present:

```bash
aws eks create-addon \
  --cluster-name <your-cluster> \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::<account>:role/AmazonEKS_EBS_CSI_DriverRole
```

**Updated Postgres PVC using gp3**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 20Gi
```

---

### 3.2 ReadWriteOnce vs ReadWriteMany Limitations

EBS volumes (gp2, gp3, io1) only support `ReadWriteOnce` (RWO) — they can be mounted by a single node at a time. This has two implications:

1. If you scale a Postgres Deployment to 2 replicas, the second pod will fail to start because the EBS volume is already attached to another node.
2. During a rolling update, the new pod cannot start on a different node until the old pod (and its EBS attachment) is terminated.

For shared storage across multiple pods, use Amazon EFS with the EFS CSI driver, which supports `ReadWriteMany` (RWX):

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-xxxxxxxxx
  directoryPerms: "700"
```

---

### 3.3 StatefulSet Slow Rolling Updates Due to PVC Binding

When you convert the Postgres Deployment to a StatefulSet (the correct approach for production), rolling updates can stall because of PVC binding behavior.

**How it stalls**

StatefulSets use `OrderedReady` pod management by default. During a rolling update:
1. Pod `postgres-0` is terminated.
2. The EBS volume detaches (takes 10-30 seconds).
3. New `postgres-0` starts, waits for volume to attach.
4. Only after `postgres-0` is Ready does `postgres-1` begin updating.

For a 3-replica StatefulSet, this can take 5-10 minutes.

**Mitigation: Parallel pod management**

For stateless workloads using StatefulSets (e.g., for stable network identity), use `Parallel` pod management:

```yaml
spec:
  podManagementPolicy: Parallel
```

For actual databases, keep `OrderedReady` — the slow update is correct behavior that ensures data safety.

**Mitigation: WaitForFirstConsumer binding**

Ensure the StorageClass uses `WaitForFirstConsumer` to avoid cross-AZ volume/pod mismatches that cause pods to be stuck in `ContainerCreating`:

```yaml
volumeBindingMode: WaitForFirstConsumer
```

---

## 4. Application-Level Bottlenecks

### 4.1 Replica Count vs Resource Availability

The Node Hono deployment runs 1 replica, Python FastAPI runs 1 replica. Under load, a single replica is a single point of failure and a bottleneck.

**Too few replicas under load**

- All traffic hits one pod
- If the pod is on a node being updated or drained, there is downtime
- No ability to use multiple CPU cores across pods

**Too many replicas without enough resources**

- Pods compete for the same node's CPU and memory
- Scheduler may spread pods across expensive large nodes
- Each replica consumes memory even when idle

**Right-sizing replicas**

Start with a minimum of 2 replicas for any production service. Use HPA (see next section) to scale beyond that.

```yaml
spec:
  replicas: 2    # Minimum always-on replicas
```

Also use pod disruption budgets to ensure at least 1 replica is always available during node drains:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: node-hono-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: node-hono
```

---

### 4.2 Horizontal Pod Autoscaler (HPA)

**Basic CPU-based HPA**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: node-hono-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: node-hono
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60   # Scale when average CPU > 60% of request
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # Scale up after 1 minute of high load
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60              # Add at most 2 pods per minute
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 minutes before scaling down
```

**Custom metrics HPA** (using KEDA, which is far more flexible than native HPA)

KEDA extends HPA to support 60+ event sources including SQS queue depth, HTTP request rate, and Prometheus metrics:

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace
```

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: node-hono-scaler
spec:
  scaleTargetRef:
    name: node-hono
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-server.monitoring:9090
      metricName: http_requests_total
      threshold: "100"    # Scale up when >100 req/s per replica
      query: |
        sum(rate(http_requests_total{app="node-hono"}[1m]))
```

**HPA cool-down period misconfiguration**

The most common HPA mistake is setting `scaleDown.stabilizationWindowSeconds` too short, causing yo-yo scaling:

- Load spike -> scale up to 10 pods
- Load drops -> scale down immediately to 2 pods
- Next spike -> scale up again
- Each scale-up takes 1-2 minutes, causing repeated latency spikes

Keep scale-down at 300 seconds (5 minutes) minimum for stateless HTTP services.

**Checking HPA status**

```bash
kubectl get hpa -n default
kubectl describe hpa node-hono-hpa -n default

# See why HPA is not scaling
kubectl get events -n default | grep HorizontalPodAutoscaler
```

---

### 4.3 Liveness and Readiness Probe Misconfiguration

Misconfigured probes are one of the most common causes of cascading restarts in production.

**Liveness probe**: If this fails, the container is killed and restarted. Never use a liveness probe that hits a database or external dependency — if the database is slow, all your pods restart simultaneously.

**Readiness probe**: If this fails, the pod is removed from the Service's endpoint list (no traffic sent to it). The pod is not restarted.

**Startup probe**: Gives a slow-starting container time to initialize before liveness kicks in.

**Example: Correct probe configuration for Node Hono**

```yaml
containers:
- name: node-hono
  livenessProbe:
    httpGet:
      path: /health/live    # Must only check if the process is alive, NOT DB
      port: 3000
    initialDelaySeconds: 10
    periodSeconds: 10
    failureThreshold: 3     # 3 consecutive failures before restart
    timeoutSeconds: 2

  readinessProbe:
    httpGet:
      path: /health/ready   # CAN check DB connectivity
      port: 3000
    initialDelaySeconds: 5
    periodSeconds: 5
    failureThreshold: 2
    successThreshold: 1
    timeoutSeconds: 3

  startupProbe:
    httpGet:
      path: /health/live
      port: 3000
    failureThreshold: 30    # Allow 30 * 10s = 5 minutes to start
    periodSeconds: 10
```

**Diagnosing probe failures**

```bash
# See probe failure events
kubectl describe pod <pod-name> -n default | grep -A 10 "Liveness\|Readiness"

# Live probe events
kubectl get events -n default --sort-by='.lastTimestamp' | grep -i probe
```

**Common mistakes**

| Mistake | Consequence |
|---|---|
| `initialDelaySeconds` too short | Pod killed before it finishes starting |
| `timeoutSeconds` too short | Slow DB query causes false probe failure |
| Liveness checks DB | DB blip kills all pods simultaneously |
| `failureThreshold: 1` | Single transient error causes restart |
| Missing readiness probe | Traffic sent to starting pods, causing errors |

---

### 4.4 Init Container Delays

Init containers run sequentially before the main container starts. If an init container is slow or retrying, the pod stays in `Init:0/1` state and serves no traffic.

```bash
# Check init container status
kubectl get pods -n default
# STATUS column shows: Init:0/1, Init:1/2, etc.

# Get logs from a specific init container
kubectl logs <pod-name> -n default -c init-migrations

# Describe to see init container events
kubectl describe pod <pod-name> -n default
```

**Common init container bottlenecks**

- Database migration scripts that run on every pod start (use a Job instead)
- `initContainer` waiting for a service that is itself starting
- Pulling a large init image that is not cached on the node

**Fix: Use a Job for migrations, not an init container**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrations
spec:
  template:
    spec:
      containers:
      - name: migrations
        image: 425848620191.dkr.ecr.us-east-1.amazonaws.com/node-hono:latest
        command: ["npm", "run", "migrate"]
        env:
        - name: DB_HOST
          value: postgres-service
      restartPolicy: OnFailure
  backoffLimit: 3
```

---

## 5. EKS-Specific Bottlenecks

### 5.1 ENI/IP Exhaustion (VPC CNI)

**How it happens**

EKS uses the AWS VPC CNI plugin, which assigns real VPC IP addresses to each pod. Every EC2 instance has a limited number of ENIs, and each ENI has a limited number of secondary IPs. When all IPs are consumed, new pods enter `Pending` state with the error:

```
Failed to create pod sandbox: ... no available addresses in this subnet
```

IP limits per instance type (examples):
- `t3.small`: 3 ENIs x 4 IPs = 12 pods max
- `m5.large`: 3 ENIs x 10 IPs = 30 pods max
- `m5.4xlarge`: 8 ENIs x 30 IPs = 240 pods max

**Detecting IP exhaustion**

```bash
# Check pods stuck in Pending
kubectl get pods --all-namespaces | grep Pending

# Check the VPC CNI status on a node
kubectl describe node <node-name> | grep -A 5 "Allocated resources"

# Check available IPs per node
kubectl exec -n kube-system ds/aws-node -- /app/aws-cni-support.sh 2>/dev/null | grep eni
```

**Fix: Enable prefix delegation**

With prefix delegation, the VPC CNI assigns a /28 CIDR prefix (16 IPs) to each ENI slot instead of individual IPs. This increases pod density by 16x.

```bash
# Enable prefix delegation
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true

# Set warm prefix target
kubectl set env daemonset aws-node -n kube-system WARM_PREFIX_TARGET=1
```

After enabling, existing nodes need to be recycled (terminate and let the ASG replace them) to pick up the new IP allocation mode.

For new clusters, enable it at creation time:

```bash
eksctl create cluster \
  --name my-cluster \
  --vpc-private-subnets subnet-xxxx,subnet-yyyy \
  ...
```

Then patch the aws-node ConfigMap:

```bash
kubectl patch daemonset aws-node -n kube-system \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/initContainers/0/env/-", "value": {"name": "ENABLE_PREFIX_DELEGATION", "value": "true"}}]'
```

---

### 5.2 Node Group Scaling Lag: Cluster Autoscaler vs Karpenter

**Cluster Autoscaler scaling timeline**

1. Pod becomes unschedulable: ~0s
2. Cluster Autoscaler polling interval: 10s (default)
3. ASG scaling request to EC2: ~10-30s
4. EC2 instance launch: 1-3 minutes
5. Node bootstrap and join: 1-3 minutes
6. **Total: 3-7 minutes** before the pod can start

During that window, the service is degraded.

**Karpenter scaling timeline**

1. Pod becomes unschedulable: ~0s
2. Karpenter detects via watch (not polling): ~2s
3. EC2 RunInstances API call: ~5s
4. Instance launch: 1-2 minutes
5. Node bootstrap: 30-60s
6. **Total: 1.5-3 minutes**

Karpenter is roughly 2x faster because it uses watches instead of polling and calls EC2 directly instead of through ASG.

**Mitigation: Overprovisioning buffer**

For latency-sensitive production systems, pre-provision capacity using low-priority "placeholder" pods:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overprovisioning
spec:
  replicas: 2
  selector:
    matchLabels:
      app: overprovisioning
  template:
    metadata:
      labels:
        app: overprovisioning
    spec:
      priorityClassName: cluster-overprovisioner  # Low priority
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.9
        resources:
          requests:
            cpu: "1000m"
            memory: "1Gi"
```

These placeholder pods hold capacity. When a real pod needs scheduling, it preempts the placeholder, and Cluster Autoscaler scales up to replace the placeholder — all without the real pod waiting.

---

### 5.3 API Server Throttling

**How it happens**

Every controller, operator, and kubectl call goes to the Kubernetes API server. AWS EKS manages the API server, and it enforces rate limits per user-agent. When too many LIST/WATCH calls pile up (e.g., from a misconfigured controller that re-lists the entire pod list every 10 seconds), the API server returns HTTP 429 and controllers fall behind.

**Detecting throttling**

```bash
# Check for throttling in component logs
kubectl logs -n kube-system deployment/cluster-autoscaler | grep -i throttle
kubectl logs -n kube-system deployment/coredns | grep -i throttle

# Check API server audit logs in CloudWatch
# Filter for: responseStatus.code = 429
```

With Prometheus:

```promql
# API server request rate by verb and resource
rate(apiserver_request_total[5m])

# Error rate (429 and 5xx)
rate(apiserver_request_total{code=~"4..|5.."}[5m])
```

**Mitigation**

- Use `--watch` (informers) instead of polling with periodic LIST calls in custom controllers
- Set appropriate `resync-period` in informer caches (default 10-12 hours is fine; do not set to minutes)
- Limit the number of operator instances — run 1 replica with leader election, not 3
- Use `resourceVersion` in LIST calls to avoid full list scans

---

### 5.4 ECR Image Pull Rate Limits and Pull-Through Cache

**How it happens**

ECR does not have hard rate limits the way Docker Hub does. However, image pulls have two latency sources:

1. **Large image layers**: Pulling a 1GB Docker image on a cold node takes 60-90 seconds. This delays pod startup.
2. **Docker Hub pulls through ECR pull-through cache**: If you pull public images (like `postgres:15`) from Docker Hub on many nodes simultaneously, Docker Hub's unauthenticated rate limit (100 pulls/6 hours per IP) can block pulls on NAT gateway IPs.

**Detecting slow image pulls**

```bash
# See image pull duration in pod events
kubectl describe pod <pod-name> -n default | grep -A 3 "Pulling image\|Pulled image"

# Example output showing 45-second pull:
# Normal  Pulling  2m    kubelet  Pulling image "postgres:15"
# Normal  Pulled   75s   kubelet  Successfully pulled image "postgres:15" in 45.2s
```

**Fix 1: Use ECR pull-through cache for public images**

Instead of pulling `postgres:15` from Docker Hub, mirror it through ECR. ECR caches the image in your region after the first pull.

```bash
# Create a pull-through cache rule for Docker Hub
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix dockerhub \
  --upstream-registry-url registry-1.docker.io \
  --region us-east-1
```

Then reference the image as:

```yaml
image: 425848620191.dkr.ecr.us-east-1.amazonaws.com/dockerhub/library/postgres:15
```

**Fix 2: Minimize image size to reduce pull time**

For this repo's Node Hono and Python FastAPI services:

```dockerfile
# Use slim base images
FROM node:20-alpine   # ~50MB vs ~300MB for node:20
FROM python:3.12-slim # ~90MB vs ~400MB for python:3.12
```

**Fix 3: Pre-pull images on nodes (DaemonSet approach)**

For images you know every node will need, pre-pull them when the node joins:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: image-prepuller
spec:
  selector:
    matchLabels:
      app: prepuller
  template:
    metadata:
      labels:
        app: prepuller
    spec:
      initContainers:
      - name: prepull-node-hono
        image: 425848620191.dkr.ecr.us-east-1.amazonaws.com/node-hono:latest
        command: ["sh", "-c", "echo pre-pulled"]
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.9
```

---

## 6. Observability for Bottleneck Detection

### 6.1 Essential kubectl Commands

```bash
# ---- Pod health overview ----
kubectl get pods -n default -o wide
kubectl get pods -n default --sort-by='.status.containerStatuses[0].restartCount'

# ---- Resource usage ----
kubectl top pods -n default --containers
kubectl top nodes

# ---- Events (sorted by time, most recent last) ----
kubectl get events -n default --sort-by='.lastTimestamp'

# ---- Describe for full context ----
kubectl describe pod <pod-name> -n default
kubectl describe node <node-name>

# ---- Logs ----
kubectl logs <pod-name> -n default --tail=100
kubectl logs <pod-name> -n default -p          # Previous (crashed) container logs
kubectl logs -n default -l app=node-hono --tail=50  # All pods matching label

# ---- Live log streaming ----
kubectl logs -f <pod-name> -n default

# ---- Debug a running pod ----
kubectl exec -it <pod-name> -n default -- sh

# ---- Debug without modifying the pod (ephemeral container, K8s 1.23+) ----
kubectl debug -it <pod-name> -n default --image=nicolaka/netshoot --target=node-hono
```

---

### 6.2 CloudWatch Container Insights

Container Insights collects cluster, node, pod, and container metrics and ships them to CloudWatch automatically.

**Setup**

```bash
# Attach the policy to the node IAM role
aws iam attach-role-policy \
  --role-name <NodeInstanceRole> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# Deploy CloudWatch agent as a DaemonSet
ClusterName=<your-cluster-name>
RegionName=us-east-1

curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml \
  | sed "s/{{cluster_name}}/${ClusterName}/;s/{{region_name}}/${RegionName}/" \
  | kubectl apply -f -
```

**Key CloudWatch metrics to monitor**

| Metric | Namespace | What it tells you |
|---|---|---|
| `pod_cpu_utilization` | ContainerInsights | CPU usage vs limit |
| `pod_memory_utilization` | ContainerInsights | Memory usage vs limit |
| `pod_cpu_reserved_capacity` | ContainerInsights | Node overcommit level |
| `node_cpu_utilization` | ContainerInsights | Node-level CPU |
| `pod_number_of_container_restarts` | ContainerInsights | Crash/OOM restarts |
| `cluster_node_count` | ContainerInsights | Current node count |
| `cluster_failed_node_count` | ContainerInsights | Node failures |

---

### 6.3 Prometheus + Grafana on EKS

**Quick setup with kube-prometheus-stack**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.retention=7d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=gp3 \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.storageClassName=gp3 \
  --set grafana.persistence.size=10Gi
```

This installs:
- Prometheus (scrapes all pods with `prometheus.io/scrape: "true"`)
- Alertmanager
- Grafana (with built-in dashboards for nodes, pods, and namespaces)
- kube-state-metrics (exposes Kubernetes object state as metrics)
- node-exporter (exposes OS-level metrics per node)

**Access Grafana**

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# Username: admin
# Password:
kubectl get secret -n monitoring kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

**Instrument your Node Hono service**

Add Prometheus annotations so the scraper picks up your app metrics:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "3000"
    prometheus.io/path: "/metrics"
```

---

### 6.4 Key Metrics to Watch

**CPU throttling rate** — the single most important signal for CPU-related slowness:

```promql
sum by (pod, container) (
  rate(container_cpu_cfs_throttled_periods_total{namespace="default",container!=""}[5m])
)
/
sum by (pod, container) (
  rate(container_cpu_cfs_periods_total{namespace="default",container!=""}[5m])
)
> 0.1
```

Alert threshold: > 10% throttling for 5 minutes.

**Memory working set** — actual memory in active use (vs RSS which includes cache):

```promql
container_memory_working_set_bytes{namespace="default",container!=""}
```

Alert threshold: > 85% of `limits.memory`.

**OOMKilled container count**:

```promql
increase(kube_pod_container_status_last_terminated_reason{reason="OOMKilled",namespace="default"}[1h]) > 0
```

**Network packet drops per node**:

```promql
rate(node_network_transmit_drop_total{device!~"lo|veth.*|docker.*"}[5m]) > 10
```

**DNS error rate from CoreDNS**:

```promql
rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m]) > 1
```

**Pod restart rate**:

```promql
increase(kube_pod_container_status_restarts_total{namespace="default"}[1h]) > 5
```

**HPA at max replicas** (means HPA cannot scale further):

```promql
kube_horizontalpodautoscaler_status_current_replicas
==
kube_horizontalpodautoscaler_spec_max_replicas
```

**Pending pods**:

```promql
kube_pod_status_phase{phase="Pending",namespace="default"} > 0
```

---

## 7. Quick Diagnosis Checklist

Use this checklist when pods are slow, crashing, or traffic is degraded.

### When pods are crashing or restarting

1. Run `kubectl get pods -n default` — check RESTARTS column and STATUS field.
2. Run `kubectl describe pod <pod-name> -n default` — check Events section and Last State.
3. If STATUS is `OOMKilled`: increase `limits.memory` and check for memory leaks with `kubectl top pods --containers`.
4. If STATUS is `CrashLoopBackOff`: run `kubectl logs <pod-name> -n default -p` for previous container logs.
5. If STATUS is `Error`: check liveness probe configuration — `initialDelaySeconds` may be too short.
6. Check `kubectl get events -n default --sort-by='.lastTimestamp'` for context.

### When pods are stuck in Pending

7. Run `kubectl describe pod <pod-name> -n default` — look at Events for scheduler failure reason.
8. If "Insufficient cpu/memory": run `kubectl top nodes` — nodes may be full.
9. If "no available addresses": ENI IP exhaustion — enable prefix delegation (see section 5.1).
10. If "did not match node selector": check node labels with `kubectl get nodes --show-labels`.
11. If "unbound PersistentVolumeClaims": check `kubectl get pvc -n default` for Pending PVCs.
12. Check if Cluster Autoscaler is running and its logs: `kubectl logs -n kube-system deployment/cluster-autoscaler | tail -30`.

### When the application is slow under load

13. Run `kubectl top pods -n default --containers` — identify which container is near its CPU limit.
14. Check CPU throttling ratio in Prometheus: `container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total`.
15. If throttling > 25%: increase `limits.cpu` or remove the CPU limit entirely.
16. Check if HPA is active: `kubectl get hpa -n default` — is `CURRENT` near `MAX`?
17. If HPA is maxed: increase `maxReplicas` and ensure nodes have capacity (or Karpenter/CA will scale).
18. Run `kubectl get events -n default | grep -i "probe"` — readiness probe failures remove pods from the Service endpoint list.
19. Check DNS: `kubectl exec -it <pod> -- time nslookup postgres-service` — should be under 5ms.
20. If DNS is slow (>50ms): add `ndots: 2` to pod dnsConfig (see section 2.1) and consider NodeLocal DNSCache.

### When a specific service cannot reach another service

21. Test connectivity: `kubectl exec -it <source-pod> -- curl -v http://<service-name>:<port>/health --max-time 5`.
22. If connection refused: check `kubectl get endpoints <service-name> -n default` — no endpoints means all pods fail readiness probes.
23. If connection times out: check `kubectl get networkpolicies -n default` — a NetworkPolicy may be silently blocking traffic.
24. Check that service selector matches pod labels: `kubectl describe svc <service-name>` vs `kubectl get pods --show-labels`.

### When new nodes are not joining the cluster fast enough

25. Check Cluster Autoscaler logs: `kubectl logs -n kube-system deployment/cluster-autoscaler | grep -E "scale-up|error"`.
26. Check EC2 console for instances in "pending" state in the Auto Scaling Group.
27. Check node bootstrap logs via Systems Manager or EC2 serial console for cloud-init failures.
28. If using Karpenter: `kubectl get nodeclaim -n kube-system` and `kubectl describe nodeclaim <name>`.
29. Verify the node IAM role has required policies: AmazonEKSWorkerNodePolicy, AmazonEKS_CNI_Policy, AmazonEC2ContainerRegistryReadOnly.

### When ECR image pulls are failing

30. Verify the node IAM role has `ecr:GetAuthorizationToken` and `ecr:BatchGetImage` permissions.
31. Check the image URI in the deployment spec matches the exact ECR repository URI and tag.
32. Run `kubectl describe pod <pod-name>` — look for "ImagePullBackOff" or "ErrImagePull" in Events.
33. If pulling from a private ECR in a different account, ensure a cross-account resource policy is attached to the ECR repository.
34. Test the pull manually: `aws ecr get-login-password | docker login --username AWS --password-stdin <account>.dkr.ecr.us-east-1.amazonaws.com`.

---

## Reference: Resource Configuration for This Repo

The manifests in this repo currently have no resource requests or limits. Below are starting values based on typical Node.js and Python FastAPI workload profiles. Tune these values after observing real traffic with `kubectl top` and Prometheus.

**`k8s/multi-api.yaml` — recommended resource additions**

```yaml
# node-hono container
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

# python-fastapi container
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

**`k8s/postgres.yaml` — recommended resource additions**

```yaml
# postgres container
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
  limits:
    cpu: "1000m"
    memory: "1Gi"
```

These values target the `Burstable` QoS class — predictable baseline with burst headroom. Monitor for OOMKilled events (increase memory limits) and CPU throttling (increase CPU limits or remove the limit) after deploying.
