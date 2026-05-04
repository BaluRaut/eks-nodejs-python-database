# Monitoring and Logging on EKS: A Practical Guide

This guide covers observability for the `learning-cluster` EKS cluster running in `us-east-1`. It walks through logs, metrics, traces, and alerting — from the basics of why observability matters to the exact commands you need to get it working.

---

## Table of Contents

1. [Why Observability Matters](#1-why-observability-matters)
2. [Logs: Getting Application Logs Out of Kubernetes](#2-logs-getting-application-logs-out-of-kubernetes)
3. [Structured Logging Best Practices](#3-structured-logging-best-practices)
4. [Metrics: Understanding What Is Happening](#4-metrics-understanding-what-is-happening)
5. [Prometheus and Grafana Stack](#5-prometheus-and-grafana-stack)
6. [Traces: Following a Request Across Services](#6-traces-following-a-request-across-services)
7. [Alerting](#7-alerting)
8. [Quick kubectl Log Commands](#8-quick-kubectl-log-commands)
9. [Log Retention and Cost](#9-log-retention-and-cost)

---

## 1. Why Observability Matters

### The "Flying Blind" Problem

Running a production application without monitoring is like flying a plane with no instruments. You might know the plane is in the air, but you have no idea how fast it is going, whether the engine temperature is rising, how much fuel is left, or whether you are drifting off course. Everything feels fine right up until it does not.

Without observability:
- A memory leak grows slowly for six hours and takes down your Node Hono service at 2 AM.
- A database connection pool exhausts itself under load, causing requests to silently time out.
- A bad deployment increases error rates by 30%, but you only find out when a user emails you.
- You get paged for an outage and have no data to tell you which service failed first.

With observability, you see these problems forming and can act before users are impacted.

### The Three Pillars

Observability is built on three complementary data types. Each answers a different question:

| Pillar | Question | Example |
|---|---|---|
| **Logs** | What happened, and when? | "Request 8a3f failed with 500 at 14:23:01 because the DB query timed out" |
| **Metrics** | How much, and how fast? | "The node-hono service handled 2,400 requests in the last minute with a p99 latency of 340ms" |
| **Traces** | Where did time get spent? | "This request took 820ms: 5ms in the gateway, 200ms in node-hono, 610ms waiting on Postgres" |

You need all three. Metrics tell you *that* something is wrong. Logs tell you *what* went wrong. Traces tell you *where* in the request lifecycle the problem lives.

---

## 2. Logs: Getting Application Logs Out of Kubernetes

### How Container Logging Works

When your Node Hono or Python FastAPI app writes to stdout (which is what any well-behaved containerized app should do), Kubernetes captures that output and writes it to a log file on the node at:

```
/var/log/containers/<pod-name>_<namespace>_<container-name>-<container-id>.log
```

By default those files stay on the node. When the node is terminated — which happens whenever EKS scales down, replaces an unhealthy node, or you update the node group — those logs are gone. You need a log shipper to forward logs to a durable store before that happens.

### Fluent Bit: The Log Shipper

Fluent Bit is a lightweight log processor and forwarder written in C. In EKS it runs as a **DaemonSet**, meaning one Fluent Bit pod runs on every node in the cluster. Its job is simple:

1. Tail the log files written by the container runtime on the node.
2. Parse and enrich each log record (add pod name, namespace, container name, cluster name).
3. Ship the records to CloudWatch Logs in real time.

Because it runs as a DaemonSet, every node is covered automatically — including new nodes that join when the cluster scales out.

```
Container (stdout)
        |
        v
/var/log/containers/... (node disk)
        |
        v
Fluent Bit DaemonSet (reads, parses, enriches)
        |
        v
CloudWatch Logs (/aws/containerinsights/learning-cluster/application)
```

### Installing Fluent Bit via EKS Add-on

The easiest production-ready installation method is through the EKS managed add-on. The add-on handles installation, upgrades, and configuration for you.

**Step 1: Verify your node role has the required policy**

Fluent Bit needs permission to create log groups and write log events. Attach the `CloudWatchAgentServerPolicy` managed policy to the IAM role used by your EKS node group.

```bash
# Find your node instance role
aws eks describe-nodegroup \
  --cluster-name learning-cluster \
  --nodegroup-name <your-nodegroup-name> \
  --region us-east-1 \
  --query 'nodegroup.nodeRole' \
  --output text

# Attach the policy (replace ROLE_NAME with the role name extracted above)
aws iam attach-role-policy \
  --role-name <ROLE_NAME> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```

**Step 2: Install the add-on**

```bash
aws eks create-addon \
  --cluster-name learning-cluster \
  --addon-name aws-for-fluent-bit \
  --region us-east-1
```

**Step 3: Verify it is running**

```bash
kubectl get daemonset -n amazon-cloudwatch
kubectl get pods -n amazon-cloudwatch
```

You should see one Fluent Bit pod per node. All pods should be in `Running` state.

**Step 4: Confirm logs are arriving in CloudWatch**

```bash
aws logs describe-log-groups \
  --log-group-name-prefix /aws/containerinsights/learning-cluster \
  --region us-east-1
```

### Log Group Naming Convention

Container Insights creates log groups with a predictable structure:

| Log Group | Contents |
|---|---|
| `/aws/containerinsights/learning-cluster/application` | Application stdout/stderr from all pods |
| `/aws/containerinsights/learning-cluster/host` | OS-level logs from the EC2 nodes (syslog, etc.) |
| `/aws/containerinsights/learning-cluster/dataplane` | Kubernetes control plane logs from the node (kubelet, kube-proxy) |
| `/aws/containerinsights/learning-cluster/performance` | Container Insights performance data (CPU, memory, network) |

For debugging your apps, you will almost always be looking in the `application` log group.

### CloudWatch Logs Insights Query Examples

Open the CloudWatch console, go to **Logs Insights**, select the log group `/aws/containerinsights/learning-cluster/application`, and run these queries.

**Find all ERROR-level logs in the last hour**

```
fields @timestamp, @message, kubernetes.pod_name, kubernetes.namespace_name
| filter @message like /ERROR|error|Error/
| sort @timestamp desc
| limit 100
```

**Count requests per minute (useful for spotting traffic spikes)**

```
fields @timestamp, @message
| filter kubernetes.container_name = "node-hono"
| stats count() as request_count by bin(1m)
| sort @timestamp asc
```

**Search logs from a specific pod**

```
fields @timestamp, @message
| filter kubernetes.pod_name like /node-hono-7d9f4b/
| sort @timestamp desc
| limit 200
```

**Find 5xx errors with status codes**

```
fields @timestamp, @message, kubernetes.pod_name
| filter @message like /"status":5/
| sort @timestamp desc
| limit 50
```

**Count errors grouped by container**

```
fields kubernetes.container_name
| filter @message like /ERROR|Exception|panic/
| stats count() as error_count by kubernetes.container_name
| sort error_count desc
```

---

## 3. Structured Logging Best Practices

### Why JSON Logs Beat Plain Text

Plain text logs look like this:

```
2024-01-15 14:23:01 INFO  POST /api/users 201 45ms request_id=abc123
2024-01-15 14:23:02 ERROR GET /api/orders 500 12ms request_id=def456 error=connection timeout
```

JSON logs look like this:

```json
{"timestamp":"2024-01-15T14:23:01.443Z","level":"info","method":"POST","path":"/api/users","status":201,"duration_ms":45,"request_id":"abc123"}
{"timestamp":"2024-01-15T14:23:02.112Z","level":"error","method":"GET","path":"/api/orders","status":500,"duration_ms":12,"request_id":"def456","error":"connection timeout"}
```

CloudWatch Logs Insights can parse and filter JSON fields directly. With plain text you are limited to regex matching against the full string. With JSON you can write queries like:

```
filter status >= 500
| stats avg(duration_ms) as avg_latency by path
| sort avg_latency desc
```

You cannot write that query against unstructured text.

### Configuring Structured Logging in Node.js (Hono) with Pino

[Pino](https://getpino.io) is the fastest JSON logger for Node.js. It writes newline-delimited JSON to stdout by default, which is exactly what you want.

**Install pino**

```bash
npm install pino pino-http
```

**Configure in your Hono app (`index.js`)**

```javascript
import { Hono } from 'hono'
import pino from 'pino'
import pinoHttp from 'pino-http'

const logger = pino({
  // In production, use level 'info'. In development you can use 'debug'.
  level: process.env.LOG_LEVEL || 'info',
  // Remove pino's default 'pid' and 'hostname' fields to reduce noise
  base: null,
  // ISO timestamp strings are easier to read in CloudWatch than epoch milliseconds
  timestamp: pino.stdTimeFunctions.isoTime,
})

const app = new Hono()

// Attach the HTTP request logger as middleware
// This automatically logs every request with method, url, status, response time
app.use('*', async (c, next) => {
  const start = Date.now()
  await next()
  logger.info({
    method: c.req.method,
    path: c.req.path,
    status: c.res.status,
    duration_ms: Date.now() - start,
    request_id: c.req.header('x-request-id'),
  })
})

// Log errors with full context
app.onError((err, c) => {
  logger.error({
    err: err.message,
    stack: err.stack,
    method: c.req.method,
    path: c.req.path,
  })
  return c.json({ error: 'Internal Server Error' }, 500)
})

export default app
```

Every request now produces structured JSON output like:

```json
{"time":"2024-01-15T14:23:01.443Z","level":"info","method":"GET","path":"/health","status":200,"duration_ms":3,"request_id":"a1b2c3"}
```

### Configuring Structured Logging in Python (FastAPI) with python-json-logger

**Install the library**

```bash
pip install python-json-logger
```

**Configure in your FastAPI app (`main.py`)**

```python
import logging
import time
import uuid
from pythonjsonlogger import jsonlogger
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

# Set up the JSON formatter on the root logger
def configure_logging():
    logger = logging.getLogger()
    handler = logging.StreamHandler()
    formatter = jsonlogger.JsonFormatter(
        fmt='%(asctime)s %(name)s %(levelname)s %(message)s',
        datefmt='%Y-%m-%dT%H:%M:%S',
        rename_fields={'asctime': 'timestamp', 'levelname': 'level', 'name': 'logger'}
    )
    handler.setFormatter(formatter)
    logger.addHandler(handler)
    logger.setLevel(logging.INFO)

configure_logging()
logger = logging.getLogger(__name__)

app = FastAPI()

# Request logging middleware
@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()
    request_id = request.headers.get("x-request-id", str(uuid.uuid4()))

    response = await call_next(request)

    duration_ms = round((time.time() - start_time) * 1000, 2)
    logger.info(
        "request handled",
        extra={
            "method": request.method,
            "path": request.url.path,
            "status": response.status_code,
            "duration_ms": duration_ms,
            "request_id": request_id,
        }
    )
    return response

# Exception handler
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.error(
        "unhandled exception",
        extra={
            "error": str(exc),
            "method": request.method,
            "path": request.url.path,
        },
        exc_info=True
    )
    return JSONResponse(status_code=500, content={"error": "Internal Server Error"})
```

Every request now produces:

```json
{"timestamp": "2024-01-15T14:23:01", "logger": "main", "level": "INFO", "message": "request handled", "method": "GET", "path": "/health", "status": 200, "duration_ms": 4.21, "request_id": "a1b2c3"}
```

---

## 4. Metrics: Understanding What Is Happening

### CloudWatch Container Insights

Container Insights collects CPU, memory, network, and disk metrics at the cluster, node, pod, and container level. It uses a CloudWatch agent DaemonSet that reads metrics from the Kubernetes metrics API and cAdvisor.

**Step 1: Attach the required policy to your node IAM role**

If you followed the Fluent Bit section, you already attached `CloudWatchAgentServerPolicy`. If not, do it now:

```bash
aws iam attach-role-policy \
  --role-name <NODE_INSTANCE_ROLE_NAME> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```

**Step 2: Enable Container Insights on the cluster**

```bash
aws eks update-cluster-config \
  --name learning-cluster \
  --region us-east-1 \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```

**Step 3: Install the CloudWatch agent and Fluent Bit using the quick-start script**

AWS provides a one-command setup that installs both the CloudWatch agent (for metrics) and Fluent Bit (for logs) together. This is the recommended approach for a complete setup:

```bash
ClusterName=learning-cluster
RegionName=us-east-1
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off' || FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'

curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | \
sed "s/{{cluster_name}}/${ClusterName}/;s/{{region_name}}/${RegionName}/;s/{{http_server_toggle}}/${FluentBitHttpServer}/;s/{{http_server_port}}/${FluentBitHttpPort}/;s/{{read_from_head}}/${FluentBitReadFromHead}/;s/{{read_from_tail}}/${FluentBitReadFromTail}/" | \
kubectl apply -f -
```

**Step 4: View the dashboards**

In the CloudWatch console, go to **Insights > Container Insights > Performance Monitoring**. Use the dropdown to switch between views:

| Dashboard view | What it shows |
|---|---|
| EKS Clusters | Cluster-wide CPU, memory, and pod count |
| EKS Nodes | Per-node CPU and memory utilization |
| EKS Pods | Per-pod CPU and memory, restart counts |
| EKS Services | Request rate and latency per service (requires App Mesh or additional instrumentation) |

These dashboards update every minute and require no additional configuration once the agent is running.

---

## 5. Prometheus and Grafana Stack

CloudWatch Container Insights is convenient but limited for custom application metrics. The Prometheus + Grafana stack gives you far more power: custom metrics, flexible dashboards, and fine-grained alerting rules.

### Installing kube-prometheus-stack via Helm

`kube-prometheus-stack` is a Helm chart that installs Prometheus, Grafana, Alertmanager, and a set of pre-built dashboards and alert rules in one shot.

**Step 1: Add the Helm repo**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

**Step 2: Install the stack**

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=gp2 \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=20Gi \
  --set grafana.adminPassword=changeme-use-a-secret \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.size=5Gi \
  --set alertmanager.alertmanagerSpec.retention=120h
```

This installs everything into the `monitoring` namespace. It takes a couple of minutes for all pods to come up.

**Step 3: Verify the installation**

```bash
kubectl get pods -n monitoring
# You should see pods for:
# prometheus-kube-prometheus-stack-prometheus-0
# alertmanager-kube-prometheus-stack-alertmanager-0
# kube-prometheus-stack-grafana-<hash>
# kube-prometheus-stack-kube-state-metrics-<hash>
# kube-prometheus-stack-prometheus-node-exporter-<hash> (one per node)
```

### Accessing Grafana via Port-Forward

Grafana does not have a public load balancer by default. Use port-forward to access it locally:

```bash
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
```

Open `http://localhost:3000` in your browser. Log in with:
- Username: `admin`
- Password: whatever you set in `--set grafana.adminPassword` above

### Pre-Built Dashboards

The kube-prometheus-stack comes with these dashboards already loaded:

| Dashboard | Description |
|---|---|
| Kubernetes / Compute Resources / Cluster | Cluster-wide CPU and memory usage and limits |
| Kubernetes / Compute Resources / Namespace (Pods) | Per-namespace breakdown of all pods |
| Kubernetes / Compute Resources / Node (Pods) | Which pods are using which nodes' resources |
| Kubernetes / Networking / Namespace (Pods) | Network traffic per pod |
| Node Exporter / Nodes | Detailed OS-level metrics per EC2 node |

These dashboards cover the majority of day-to-day operational needs without any custom configuration.

### Scraping Custom Metrics from Node Hono

To expose application-level metrics from your Node Hono service, add the `prom-client` library:

```bash
npm install prom-client
```

Add a `/metrics` endpoint to your Hono app:

```javascript
import { Registry, collectDefaultMetrics, Counter, Histogram } from 'prom-client'

const registry = new Registry()

// Collect default Node.js metrics (event loop lag, GC, memory, etc.)
collectDefaultMetrics({ register: registry })

// Custom counter: total HTTP requests
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'path', 'status'],
  registers: [registry],
})

// Custom histogram: request duration
const httpRequestDurationMs = new Histogram({
  name: 'http_request_duration_ms',
  help: 'HTTP request duration in milliseconds',
  labelNames: ['method', 'path', 'status'],
  buckets: [5, 10, 25, 50, 100, 250, 500, 1000, 2500],
  registers: [registry],
})

// Middleware to record metrics
app.use('*', async (c, next) => {
  const start = Date.now()
  await next()
  const duration = Date.now() - start
  httpRequestsTotal.inc({ method: c.req.method, path: c.req.path, status: c.res.status })
  httpRequestDurationMs.observe({ method: c.req.method, path: c.req.path, status: c.res.status }, duration)
})

// Metrics endpoint for Prometheus to scrape
app.get('/metrics', async (c) => {
  c.header('Content-Type', registry.contentType)
  return c.text(await registry.metrics())
})
```

Tell Prometheus to scrape your pod by adding annotations to the pod spec in your Kubernetes manifest:

```yaml
# In k8s/multi-api.yaml, add to the node-hono pod template metadata
template:
  metadata:
    annotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "3000"
      prometheus.io/path: "/metrics"
```

Prometheus automatically discovers and scrapes any pod with `prometheus.io/scrape: "true"` — no other configuration required.

### 5 Essential Alerts to Configure

Save the following as `prometheus-alerts.yaml` and apply it:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: learning-cluster-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: pod-health
      rules:
        # Alert 1: OOMKilled - pod was killed because it ran out of memory
        - alert: PodOOMKilled
          expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: "Pod OOMKilled: {{ $labels.pod }}"
            description: "Container {{ $labels.container }} in pod {{ $labels.pod }} was killed due to out-of-memory. Increase memory limits or investigate memory leak."

        # Alert 2: CrashLoopBackOff - pod is repeatedly crashing
        - alert: PodCrashLoopBackOff
          expr: kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "CrashLoopBackOff: {{ $labels.pod }}"
            description: "Container {{ $labels.container }} in pod {{ $labels.pod }} is in CrashLoopBackOff. Check logs immediately."

        # Alert 3: Pod stuck in Pending - likely a scheduling or resource issue
        - alert: PodStuckPending
          expr: kube_pod_status_phase{phase="Pending"} == 1
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Pod pending for 15+ minutes: {{ $labels.pod }}"
            description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} has been Pending for over 15 minutes. Check node capacity or resource requests."

        # Alert 4: High CPU throttling - CPU limits are too tight
        - alert: HighCPUThrottling
          expr: |
            rate(container_cpu_cfs_throttled_periods_total{container!=""}[5m])
            / rate(container_cpu_cfs_periods_total{container!=""}[5m]) > 0.5
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High CPU throttling: {{ $labels.pod }}/{{ $labels.container }}"
            description: "Container {{ $labels.container }} is being CPU throttled more than 50% of the time. Consider raising the CPU limit."

    - name: node-health
      rules:
        # Alert 5: Node not ready - EC2 node is unhealthy
        - alert: NodeNotReady
          expr: kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Node not ready: {{ $labels.node }}"
            description: "Node {{ $labels.node }} has been NotReady for more than 5 minutes. Check EC2 instance health in the AWS console."
```

```bash
kubectl apply -f prometheus-alerts.yaml
```

Verify the rules loaded:

```bash
kubectl get prometheusrule -n monitoring
```

---

## 6. Traces: Following a Request Across Services

### What Distributed Tracing Is

When a user hits your API, the request might travel through multiple services: an ingress controller, the Node Hono service, then to the Python FastAPI service, then to Postgres. A single user-visible error could have originated in any one of them.

Think of distributed tracing like passport stamps. Each service the request passes through adds a stamp — a **span** — with its name and how long it took. When you collect all those stamps into a single **trace**, you see the complete journey: where the request spent its time, and exactly which service introduced latency or the error.

Without tracing, you have to manually correlate log entries across services by timestamp, which is slow and error-prone. With tracing, you click on a trace and immediately see a waterfall diagram of every service call.

### AWS X-Ray

X-Ray is AWS's distributed tracing service. It integrates directly with CloudWatch and works well on EKS.

**How it works:**
1. Your app SDK generates trace data (spans) for each inbound and outbound request.
2. The SDK sends span data to the X-Ray daemon (a small process running alongside your app).
3. The daemon batches and forwards spans to the X-Ray API.
4. The X-Ray console shows service maps and trace timelines.

### Instrumenting Node Hono with X-Ray

**Install the AWS X-Ray SDK**

```bash
npm install aws-xray-sdk
```

**Configure in your app**

```javascript
import AWSXRay from 'aws-xray-sdk'
import { Hono } from 'hono'

// The X-Ray SDK patches the built-in http/https modules to auto-trace outbound requests
AWSXRay.captureHTTPsGlobal(require('https'))

const app = new Hono()

// Middleware to start a new segment for each incoming request
app.use('*', async (c, next) => {
  const segment = new AWSXRay.Segment('node-hono')
  AWSXRay.setSegment(segment)

  try {
    await next()
    segment.close()
  } catch (err) {
    segment.addError(err)
    segment.close()
    throw err
  }
})
```

### Instrumenting Python FastAPI with X-Ray

**Install the SDK**

```bash
pip install aws-xray-sdk
```

**Configure in your app**

```python
from aws_xray_sdk.core import xray_recorder, patch_all
from aws_xray_sdk.ext.fastapi.middleware import XRayMiddleware
from fastapi import FastAPI

# Patch all supported libraries (requests, boto3, psycopg2, etc.)
patch_all()

app = FastAPI()

# Add X-Ray middleware — this automatically creates a segment for each request
app.add_middleware(XRayMiddleware, recorder=xray_recorder)

xray_recorder.configure(
    service='python-fastapi',
    daemon_address='127.0.0.1:2000',  # X-Ray daemon address (localhost when using sidecar)
)
```

### Running the X-Ray Daemon

The X-Ray SDK in your app sends trace data to a local daemon, not directly to AWS. The daemon handles batching and retries. You can run it as a **sidecar** container in the same pod as your app.

Add this to your pod spec in `k8s/multi-api.yaml`:

```yaml
containers:
  - name: node-hono
    image: 425848620191.dkr.ecr.us-east-1.amazonaws.com/node-hono:latest
    # ... your existing container spec ...

  # X-Ray daemon sidecar
  - name: xray-daemon
    image: amazon/aws-xray-daemon:latest
    ports:
      - containerPort: 2000
        protocol: UDP
    resources:
      limits:
        memory: 64Mi
        cpu: 100m
      requests:
        memory: 32Mi
        cpu: 50m
    env:
      - name: AWS_REGION
        value: us-east-1
```

The X-Ray daemon needs permission to call `xray:PutTraceSegments` and `xray:PutTelemetryRecords`. Attach the `AWSXRayDaemonWriteAccess` policy to your node IAM role:

```bash
aws iam attach-role-policy \
  --role-name <NODE_INSTANCE_ROLE_NAME> \
  --policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
```

### Viewing Traces in the X-Ray Console

In the AWS console, go to **CloudWatch > X-Ray traces > Service map**. You will see:
- A visual graph of every service and its connections.
- Latency percentiles (p50, p95, p99) on each edge.
- Error rates (4xx, 5xx, fault) per service.

Click any node on the map to drill into individual traces. Click any trace to see the full waterfall diagram of all spans.

---

## 7. Alerting

### CloudWatch Alarms

CloudWatch Alarms watch a single metric and trigger actions when the metric crosses a threshold. They are simpler to set up than Prometheus alerts but less flexible.

**Set up an SNS topic for notifications first**

```bash
# Create the topic
aws sns create-topic --name learning-cluster-alerts --region us-east-1

# Subscribe your email address
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:425848620191:learning-cluster-alerts \
  --protocol email \
  --notification-endpoint your-email@example.com \
  --region us-east-1

# Check your email and confirm the subscription
```

**Example 1: Alert when node CPU exceeds 80% for 5 minutes**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EKS-NodeCPUHigh-learning-cluster" \
  --alarm-description "EKS node CPU utilization above 80% for 5 minutes" \
  --metric-name node_cpu_utilization \
  --namespace ContainerInsights \
  --dimensions Name=ClusterName,Value=learning-cluster \
  --statistic Average \
  --period 60 \
  --evaluation-periods 5 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:425848620191:learning-cluster-alerts \
  --ok-actions arn:aws:sns:us-east-1:425848620191:learning-cluster-alerts \
  --region us-east-1
```

**Example 2: Alert when any pod restarts more than 5 times in an hour**

Container Insights publishes a `pod_number_of_container_restarts` metric. This alarm fires when a pod has accumulated more than 5 restarts within a 1-hour evaluation window.

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EKS-PodRestartHigh-learning-cluster" \
  --alarm-description "A pod in learning-cluster has restarted more than 5 times in 1 hour" \
  --metric-name pod_number_of_container_restarts \
  --namespace ContainerInsights \
  --dimensions Name=ClusterName,Value=learning-cluster \
  --statistic Sum \
  --period 3600 \
  --evaluation-periods 1 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-1:425848620191:learning-cluster-alerts \
  --region us-east-1
```

### Routing Alerts to Slack

Instead of email, you can route SNS notifications to Slack using an AWS Lambda function or an SNS HTTPS subscription pointed at a Slack incoming webhook.

The simplest approach is a Lambda function that SNS invokes:

```python
# lambda_handler.py
import json
import urllib.request

SLACK_WEBHOOK_URL = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

def handler(event, context):
    message = event['Records'][0]['Sns']['Message']
    alarm = json.loads(message)

    text = (
        f"*CloudWatch Alarm*\n"
        f"Alarm: `{alarm.get('AlarmName')}`\n"
        f"State: `{alarm.get('NewStateValue')}`\n"
        f"Reason: {alarm.get('NewStateReason')}"
    )

    payload = json.dumps({"text": text}).encode()
    req = urllib.request.Request(
        SLACK_WEBHOOK_URL,
        data=payload,
        headers={"Content-Type": "application/json"}
    )
    urllib.request.urlopen(req)
```

Subscribe this Lambda to the SNS topic:

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:425848620191:learning-cluster-alerts \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:us-east-1:425848620191:function:slack-notifier \
  --region us-east-1
```

---

## 8. Quick kubectl Log Commands

These commands cover the most common log inspection scenarios. Run them against your cluster after setting the correct context:

```bash
kubectl config use-context <your-learning-cluster-context>
```

**Tail logs from all pods of a deployment (useful when you have multiple replicas)**

```bash
# Tail logs from every pod in the node-hono deployment simultaneously
kubectl logs -f deployment/node-hono -n default

# Same for python-fastapi
kubectl logs -f deployment/python-fastapi -n default
```

**Search logs for errors across all pods in a namespace**

```bash
# Print all lines containing "error" or "Error" from all pods currently in the namespace
kubectl logs --selector '' -n default --tail=500 | grep -i error

# More targeted: search only pods with a specific label
kubectl logs -l app=node-hono -n default --tail=1000 | grep -iE "error|exception|panic"
```

**Get logs from a crashed (previous) container**

When a container restarts, its logs are gone from `kubectl logs` by default. Use `-p` to retrieve the logs from the last terminated instance:

```bash
# Get logs from the previous (crashed) container instance
kubectl logs <pod-name> -n default -p

# With a specific container name (for multi-container pods)
kubectl logs <pod-name> -c node-hono -n default -p

# Find pods that have restarted
kubectl get pods -n default | grep -v "0/\|Running\|Completed" 
```

**Follow logs in real time from a specific pod**

```bash
# Follow logs as they arrive
kubectl logs -f <pod-name> -n default

# Follow logs and add timestamps (useful when app logs do not include timestamps)
kubectl logs -f <pod-name> -n default --timestamps

# Follow the most recent 50 lines first, then stream new lines
kubectl logs -f <pod-name> -n default --tail=50
```

**Get logs from all pods matching a label selector**

```bash
# Stream logs from every pod with app=node-hono, with pod name prefix in output
kubectl logs -f -l app=node-hono -n default --prefix=true

# Get the last 100 lines from all matching pods
kubectl logs -l app=node-hono -n default --tail=100
```

**Describe a pod to see events (useful for crash diagnosis)**

```bash
# Events section shows OOMKilled, image pull errors, scheduling failures, etc.
kubectl describe pod <pod-name> -n default
```

---

## 9. Log Retention and Cost

### CloudWatch Logs Pricing

CloudWatch Logs charges on three dimensions:

| Activity | Price (us-east-1, as of 2024) |
|---|---|
| Log ingestion | $0.50 per GB |
| Log storage | $0.03 per GB per month |
| Logs Insights queries | $0.005 per GB scanned |

For a small cluster running a few services, ingestion costs are usually a few dollars per month. For clusters handling significant traffic with verbose logging, ingestion costs can grow quickly. Structured JSON logs that include only relevant fields are smaller than verbose plain text, so the logging best practices in Section 3 also reduce your bill.

### Setting Retention Periods

By default, CloudWatch log groups retain logs **forever**. That means logs from two years ago keep accumulating storage charges. For most debugging purposes, 30 days is sufficient. After 30 days, the problem has either been resolved or it is a historical curiosity that probably does not require raw logs to investigate.

**Set retention on a specific log group**

```bash
aws logs put-retention-policy \
  --log-group-name /aws/containerinsights/learning-cluster/application \
  --retention-in-days 30 \
  --region us-east-1
```

**Set 30-day retention on all EKS log groups for this cluster at once**

```bash
# List all log groups for this cluster and set 30-day retention on each
aws logs describe-log-groups \
  --log-group-name-prefix /aws/containerinsights/learning-cluster \
  --region us-east-1 \
  --query 'logGroups[*].logGroupName' \
  --output text | tr '\t' '\n' | while read log_group; do
    echo "Setting 30-day retention on: $log_group"
    aws logs put-retention-policy \
      --log-group-name "$log_group" \
      --retention-in-days 30 \
      --region us-east-1
  done
```

**Common retention strategies**

| Log type | Suggested retention | Rationale |
|---|---|---|
| Application logs | 30 days | Enough for incident investigation and post-mortems |
| Kubernetes control plane logs | 90 days | Useful for audit and compliance |
| Security / audit logs | 1 year or more | Compliance requirements often mandate longer retention |
| Performance / metrics logs | 15 days | Already summarized in CloudWatch metrics; raw logs have diminishing value after a week |

**Archive to S3 for long-term storage**

If you need logs beyond 30 days for compliance but do not want to pay CloudWatch storage rates, set up a CloudWatch Logs subscription filter to stream logs to S3 via Kinesis Data Firehose. S3 Standard-IA storage costs roughly $0.0125 per GB per month — less than half the CloudWatch rate — and Glacier drops it further to around $0.004 per GB per month.

```bash
# Create an S3 bucket for log archives
aws s3 mb s3://learning-cluster-log-archive-425848620191 --region us-east-1

# (See AWS documentation for the full Firehose + subscription filter setup)
```

---

## Summary: Observability Stack for learning-cluster

| Layer | Tool | Where data lives |
|---|---|---|
| Logs | Fluent Bit DaemonSet + CloudWatch Logs | `/aws/containerinsights/learning-cluster/application` |
| Metrics (infra) | CloudWatch Container Insights | CloudWatch metrics namespace `ContainerInsights` |
| Metrics (app) | Prometheus + Grafana | In-cluster, `monitoring` namespace |
| Traces | AWS X-Ray + X-Ray daemon sidecar | X-Ray service map in CloudWatch console |
| Alerts (infra) | CloudWatch Alarms + SNS | Email or Slack |
| Alerts (app) | PrometheusRule + Alertmanager | Routed via Alertmanager to Slack or PagerDuty |

Start with logs and CloudWatch Container Insights — they give you the most value with the least setup. Add Prometheus and Grafana once you need custom application metrics or more flexible dashboards. Add X-Ray tracing when you have more than one service and need to understand cross-service latency.
