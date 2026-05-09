# System Architecture Diagrams

Reference diagrams for microservices, event-driven systems, resilience patterns, deployments, real-time, and multi-region architectures.

---

## 1. API Gateway + Microservices — Request Fan-Out

```mermaid
sequenceDiagram
    actor User as Mobile / Web Client
    participant GW as API Gateway (Kong / AWS APIGW)
    participant Auth as Auth Service
    participant Order as Order Service
    participant Product as Product Service
    participant Notif as Notification Service
    participant DB1 as Orders DB
    participant DB2 as Products DB

    User->>GW: GET /checkout-summary with JWT

    GW->>Auth: Validate JWT token
    Auth-->>GW: Token valid user_id=u123

    note over GW,DB2: Parallel calls to downstream services

    par Fetch order data
        GW->>Order: GET /orders?user=u123
        Order->>DB1: SELECT orders WHERE user_id=u123
        DB1-->>Order: Order rows
        Order-->>GW: Order list
    and Fetch product data
        GW->>Product: GET /products/featured
        Product->>DB2: SELECT featured products
        DB2-->>Product: Products
        Product-->>GW: Product list
    end

    GW->>GW: Aggregate responses into single payload
    GW-->>User: 200 Combined checkout summary

    GW->>Notif: Async fire-and-forget - log page visit event
    note over GW,Notif: Non-critical async calls do not block user response
```

---

## 2. Event-Driven Architecture — SNS Fan-Out to SQS

```mermaid
sequenceDiagram
    participant Producer as Producer Service (Order Service)
    participant SNS as AWS SNS Topic
    participant SQS1 as SQS Queue - Email Service
    participant SQS2 as SQS Queue - Analytics Service
    participant SQS3 as SQS Queue - Inventory Service
    participant Email as Email Lambda
    participant Analytics as Analytics Lambda
    participant Inventory as Inventory Worker

    Producer->>SNS: Publish OrderPlaced event JSON
    SNS->>SNS: Fan-out to all subscribed queues

    par SNS fan-out
        SNS->>SQS1: Deliver OrderPlaced message
    and
        SNS->>SQS2: Deliver OrderPlaced message
    and
        SNS->>SQS3: Deliver OrderPlaced message
    end

    note over SQS1,Inventory: Each consumer processes independently and at own pace

    SQS1->>Email: Trigger Lambda with message batch
    Email->>Email: Send order confirmation email
    Email->>SQS1: Delete message (success)

    SQS2->>Analytics: Trigger Lambda
    Analytics->>Analytics: Write event to Redshift or S3
    Analytics->>SQS2: Delete message

    SQS3->>Inventory: Worker polls every 10 seconds
    Inventory->>Inventory: Decrement stock
    Inventory->>SQS3: Delete message

    note over Producer,Inventory: Producer does not know or care about consumers.<br/>New consumers just subscribe to SNS - no producer changes.
```

---

## 3. Kafka Producer-Consumer — At-Least-Once Delivery

```mermaid
sequenceDiagram
    participant Prod as Producer (Payment Service)
    participant Broker as Kafka Broker Cluster
    participant CG1 as Consumer Group - Fulfillment
    participant CG2 as Consumer Group - Fraud Detection
    participant DB as Consumer DB

    note over Prod,Broker: Producer sends with acks=all (strongest guarantee)

    Prod->>Broker: Produce PaymentProcessed to topic payments partition 2 offset 441
    Broker->>Broker: Write to leader and 2 replicas
    Broker-->>Prod: ACK after all in-sync replicas confirm

    note over Broker,CG2: Two consumer groups - each gets all messages independently

    CG1->>Broker: Fetch from payments partition 2 offset 441
    Broker-->>CG1: PaymentProcessed message batch
    CG1->>DB: INSERT fulfillment_job
    DB-->>CG1: Committed
    CG1->>Broker: Commit offset 442 for group fulfillment

    CG2->>Broker: Fetch from payments partition 2 offset 441 (own offset)
    Broker-->>CG2: Same message (not consumed by CG2 yet)
    CG2->>CG2: Run fraud rules
    CG2->>Broker: Commit offset 442 for group fraud-detection

    note over CG1,DB: Consumer crash before offset commit = message redelivered.<br/>Consumers must be idempotent (use payment_id as dedup key).
```

---

## 4. Circuit Breaker Pattern — Closed / Open / Half-Open States

```mermaid
sequenceDiagram
    participant Client as Calling Service
    participant CB as Circuit Breaker (Resilience4j / custom)
    participant Dep as Downstream Service
    participant Fallback as Fallback Handler

    note over Client,Fallback: State CLOSED - normal operation

    Client->>CB: Call downstream
    CB->>Dep: Forward request
    Dep-->>CB: 200 OK
    CB-->>Client: Success (failure count = 0)

    note over Client,Fallback: Multiple failures - threshold reached

    Client->>CB: Call downstream
    CB->>Dep: Forward request
    Dep-->>CB: 503 error or timeout
    CB->>CB: failure_count++ now 5 of 5 threshold

    CB->>CB: STATE CHANGE closed to OPEN
    note over CB: Open state - stop calling downstream for 30s cool-down

    Client->>CB: Call downstream (during open window)
    CB->>Fallback: Short-circuit - do not call downstream
    Fallback-->>Client: Cached response or default value

    note over Client,Fallback: After 30 seconds - half-open probe

    CB->>CB: STATE CHANGE open to HALF-OPEN
    Client->>CB: Call downstream
    CB->>Dep: Allow one probe request
    Dep-->>CB: 200 OK - service recovered
    CB->>CB: STATE CHANGE half-open to CLOSED
    CB-->>Client: Success - normal traffic resumes

    note over CB,Dep: If probe fails in half-open state go back to OPEN.<br/>Prevents thundering herd when downstream recovers.
```

---

## 5. Retry with Exponential Backoff and Jitter

```mermaid
sequenceDiagram
    participant Client as Client Service
    participant Dep as Unstable Downstream Service
    participant Log as Structured Logger

    note over Client,Log: Initial request

    Client->>Dep: Request attempt 1
    Dep-->>Client: 503 Service Unavailable

    Client->>Log: Log warning attempt 1 failed 503
    Client->>Client: Wait base_delay=1s + jitter=rand(0,0.5)s = 1.3s

    Client->>Dep: Request attempt 2 (after 1.3s)
    Dep-->>Client: Timeout after 5s

    Client->>Log: Log warning attempt 2 failed timeout
    Client->>Client: Wait 2s + jitter=rand(0,1)s = 2.7s

    Client->>Dep: Request attempt 3 (after 2.7s)
    Dep-->>Client: 429 Too Many Requests

    note over Client,Log: 429 with Retry-After header - use that value instead of backoff

    Client->>Client: Wait Retry-After=10s from response header

    Client->>Dep: Request attempt 4 (after 10s)
    Dep-->>Client: 200 OK

    Client->>Log: Log info success after 4 attempts total_elapsed=14s

    note over Client,Dep: Max retries = 4. Only retry on 5xx and timeout.<br/>Never retry on 4xx except 429 with Retry-After.<br/>Jitter prevents thundering herd after outage.
```

---

## 6. Blue-Green Deployment

```mermaid
sequenceDiagram
    participant CI as CI/CD Pipeline
    participant LB as Load Balancer / ALB
    participant Blue as Blue Environment (v1 live)
    participant Green as Green Environment (v2 new)
    participant Monitor as Health Monitoring
    participant Rollback as Rollback Mechanism

    note over CI,Blue: Initial state - 100 percent traffic to Blue v1

    LB->>Blue: 100 percent of traffic
    Blue-->>LB: Serving requests

    note over CI,Green: Deploy v2 to Green - no live traffic

    CI->>Green: Deploy new version v2 to Green environment
    CI->>Green: Run smoke tests and integration tests
    Green-->>CI: All tests pass

    note over LB,Green: Traffic switch - instant cutover

    CI->>LB: Update routing 100 percent to Green v2
    LB->>Green: 100 percent traffic now on Green v2
    LB->>Blue: 0 percent traffic - Blue still running as standby

    Monitor->>Green: Check error rate latency p99
    alt Green metrics healthy after 10 minutes
        Monitor-->>CI: Green is stable
        CI->>Blue: Terminate Blue environment (or keep for next deploy)
    else Error rate spikes on Green
        Monitor-->>CI: Alert Green unhealthy
        CI->>LB: Rollback - switch 100 percent back to Blue v1
        LB->>Blue: Restore full traffic
        LB->>Green: 0 percent - Green taken offline
        CI-->>CI: Investigate and fix v2
    end

    note over Blue,Green: Rollback is instant - just a routing change.<br/>No code redeployment needed to roll back.
```

---

## 7. Canary Deployment — Weighted Traffic Routing

```mermaid
sequenceDiagram
    participant CI as CI/CD Pipeline
    participant LB as ALB or Ingress with weighted routing
    participant Stable as Stable v1 (95 percent)
    participant Canary as Canary v2 (5 percent)
    participant Monitor as Monitoring Prometheus / Datadog
    participant Rollout as Progressive Rollout Controller

    note over CI,Canary: Stage 1 - deploy canary at 5 percent traffic

    CI->>Canary: Deploy v2 to canary pod set
    CI->>LB: Set weights stable=95 canary=5
    LB->>Stable: 95 percent requests
    LB->>Canary: 5 percent requests

    Monitor->>Canary: Collect error rate p99 latency
    Monitor->>Stable: Collect same metrics for baseline

    alt Canary metrics match baseline after 10 min
        Monitor-->>Rollout: Canary healthy - proceed
        Rollout->>LB: Increase canary to 20 percent
        Rollout->>LB: Increase canary to 50 percent
        Rollout->>LB: Increase canary to 100 percent
        CI->>Stable: Remove old version
    else Canary error rate elevated
        Monitor-->>Rollout: Alert canary degraded
        Rollout->>LB: Route 0 percent to canary
        CI->>Canary: Scale down canary pods
        CI-->>CI: Investigate v2 issues
    end

    note over LB,Monitor: Real users test v2 with limited blast radius.<br/>Automated rollback on threshold breach protects most users.
```

---

## 8. Serverless — Lambda Invocation Patterns

```mermaid
sequenceDiagram
    participant Client as Client
    participant APIGW as API Gateway
    participant Lambda as Lambda Function
    participant WarmLambda as Warm Lambda Container
    participant ColdLambda as Cold Lambda Container
    participant SQS as SQS Queue
    participant AsyncLambda as Async Lambda Consumer

    note over Client,ColdLambda: Synchronous invocation - cold start path

    Client->>APIGW: POST /process
    APIGW->>Lambda: Invoke (no warm container available)
    Lambda->>ColdLambda: Init new container - download code - load runtime
    note over ColdLambda: Cold start 200ms to 2000ms depending on runtime
    ColdLambda->>ColdLambda: Execute handler
    ColdLambda-->>APIGW: 200 Response
    APIGW-->>Client: 200 Response (total 500ms)

    note over Client,WarmLambda: Synchronous invocation - warm path

    Client->>APIGW: POST /process (second request soon after)
    APIGW->>Lambda: Invoke (warm container reused)
    Lambda->>WarmLambda: Reuse existing container - skip init
    WarmLambda->>WarmLambda: Execute handler only
    WarmLambda-->>APIGW: 200 Response
    APIGW-->>Client: 200 Response (total 50ms)

    note over Client,AsyncLambda: Asynchronous invocation via SQS

    Client->>SQS: Publish message - order to process
    SQS-->>Client: 200 MessageId (client does not wait for processing)
    SQS->>AsyncLambda: Trigger Lambda with batch of 10 messages
    AsyncLambda->>AsyncLambda: Process each message
    AsyncLambda-->>SQS: Delete successfully processed messages
    note over AsyncLambda: Failed messages go to DLQ after maxReceiveCount
```

---

## 9. CDN — Cache Flow with Origin Miss and Purge

```mermaid
sequenceDiagram
    actor User as End User (Singapore)
    participant Edge as CloudFront Edge (Singapore PoP)
    participant Regional as Regional Cache (Tokyo)
    participant Origin as Origin ALB + App (us-east-1)

    note over User,Origin: Cache MISS all the way to origin

    User->>Edge: GET /static/logo.png
    Edge->>Edge: Cache miss at Singapore edge
    Edge->>Regional: Check regional cache Tokyo
    Regional->>Regional: Cache miss at regional level
    Regional->>Origin: Fetch from origin us-east-1
    Origin-->>Regional: logo.png Cache-Control max-age=31536000
    Regional->>Regional: Store in regional cache 1 year
    Regional-->>Edge: logo.png
    Edge->>Edge: Store at Singapore edge
    Edge-->>User: logo.png (latency 200ms - went to origin)

    note over User,Edge: Cache HIT - subsequent requests

    User->>Edge: GET /static/logo.png (same user or different user nearby)
    Edge->>Edge: Cache HIT - Singapore edge
    Edge-->>User: logo.png (latency 5ms - served from edge)

    note over Origin,Edge: Cache INVALIDATION after deploy

    Origin->>Edge: CloudFront CreateInvalidation for path /static/*
    Edge->>Edge: Purge matching cache entries from all edges globally
    Edge->>Regional: Propagate invalidation
    note over Edge: Next request for /static/* will go to origin again
```

---

## 10. WebSocket — Real-Time Bidirectional Communication

```mermaid
sequenceDiagram
    actor Client as Browser Client
    participant LB as ALB with WebSocket support
    participant WS as WebSocket Server (Node.js)
    participant Redis as Redis Pub/Sub
    participant WS2 as WebSocket Server 2 (other pod)
    actor Client2 as Browser Client 2 (on WS2)

    note over Client,WS: WebSocket handshake

    Client->>LB: HTTP GET /ws with Upgrade: websocket header
    LB->>WS: Forward upgrade request (sticky sessions by cookie)
    WS-->>LB: 101 Switching Protocols
    LB-->>Client: 101 Switching Protocols - connection upgraded
    note over Client,WS: Connection held open - no request/response cycle anymore

    note over Client,WS2: Real-time message flow with Redis fan-out

    Client->>WS: WebSocket frame - send message to room:chat-42
    WS->>Redis: PUBLISH channel:chat-42 message payload
    Redis->>WS: Deliver to all subscribers on WS pod
    Redis->>WS2: Deliver to subscribers on WS2 pod
    WS->>Client: Broadcast message to all sockets in chat-42 on this pod
    WS2->>Client2: Broadcast to sockets in chat-42 on WS2 pod
    Client2-->>Client2: Receives message in real time

    note over Client,WS: Heartbeat to detect dead connections

    WS->>Client: Ping frame every 30 seconds
    Client->>WS: Pong frame
    alt No pong within 10 seconds
        WS->>WS: Close connection and clean up room membership
    end
```

---

## 11. gRPC — Unary and Server Streaming

```mermaid
sequenceDiagram
    participant Client as gRPC Client Service
    participant LB as gRPC Load Balancer (L7)
    participant Server as gRPC Server Service
    participant DB as Database

    note over Client,DB: Unary RPC - single request single response

    Client->>LB: HTTP2 POST /orders.OrderService/GetOrder frame with protobuf
    LB->>Server: Route to healthy backend (connection reuse HTTP2)
    Server->>DB: Query
    DB-->>Server: Data
    Server-->>Client: Protobuf response with trailer status=OK

    note over Client,Server: Server streaming RPC - one request many responses

    Client->>Server: StreamOrders request user_id=u1 since=2024-01-01
    note over Server: Server sends frames as data becomes available
    Server-->>Client: OrderEvent stream frame 1
    Server-->>Client: OrderEvent stream frame 2
    Server-->>Client: OrderEvent stream frame 3
    Server-->>Client: EOF trailer status=OK stream complete

    note over Client,Server: Client streaming RPC - many requests one response

    Client->>Server: UploadEvents stream start
    Client->>Server: Event frame 1
    Client->>Server: Event frame 2
    Client->>Server: Event frame 3 + EOF
    Server->>Server: Process all events in batch
    Server-->>Client: Single summary response status=OK processed=3

    note over Client,DB: gRPC advantages: binary protobuf (smaller than JSON),<br/>strongly typed contracts, HTTP2 multiplexing, bi-directional streaming.
```

---

## 12. Service Mesh — Istio Sidecar Proxy Flow

```mermaid
sequenceDiagram
    participant SvcA as Service A Pod
    participant ProxyA as Envoy Sidecar A
    participant ControlPlane as Istio Control Plane (Istiod)
    participant ProxyB as Envoy Sidecar B
    participant SvcB as Service B Pod
    participant Jaeger as Distributed Tracing (Jaeger)

    note over ControlPlane,ProxyB: Control plane pushes config to sidecars

    ControlPlane->>ProxyA: xDS config - routes retries mTLS policy
    ControlPlane->>ProxyB: xDS config - routes retries mTLS policy

    note over SvcA,SvcB: Data plane - all traffic goes through sidecars

    SvcA->>ProxyA: HTTP request to service-b (iptables intercept)
    ProxyA->>ProxyA: Apply outbound policies - retries circuit breaker
    ProxyA->>ProxyB: mTLS encrypted request with tracing headers
    ProxyB->>ProxyB: Apply inbound policies - auth policy rate limit
    ProxyB->>SvcB: Decrypted request with tracing context
    SvcB-->>ProxyB: Response
    ProxyB-->>ProxyA: mTLS encrypted response
    ProxyA-->>SvcA: Decrypted response

    note over ProxyA,Jaeger: Observability - automatic without app code changes

    ProxyA->>Jaeger: Span - service-a to service-b duration latency status
    ProxyB->>Jaeger: Span - inbound to service-b
    Jaeger->>Jaeger: Assemble distributed trace tree

    note over SvcA,SvcB: App code has zero knowledge of mTLS, retries, or tracing.<br/>All handled by Envoy sidecar transparently.
```

---

## 13. Multi-Region Active-Active Architecture

```mermaid
sequenceDiagram
    actor UserUS as User in USA
    actor UserEU as User in Europe
    participant DNS as Route 53 Latency Routing
    participant APIUS as API Region us-east-1
    participant APIEU as API Region eu-west-1
    participant DDB as DynamoDB Global Tables
    participant Conflict as Conflict Resolver

    note over UserUS,DDB: Each region serves local users with local writes

    UserUS->>DNS: API request
    DNS->>DNS: Latency routing - US user gets US region
    DNS-->>UserUS: Route to us-east-1
    UserUS->>APIUS: POST /data write operation
    APIUS->>DDB: Write to us-east-1 replica
    DDB-->>APIUS: Written with timestamp and region=US
    APIUS-->>UserUS: 200 Write accepted

    UserEU->>DNS: API request
    DNS-->>UserEU: Route to eu-west-1
    UserEU->>APIEU: POST /data write operation
    APIEU->>DDB: Write to eu-west-1 replica
    DDB-->>APIEU: Written with timestamp and region=EU
    APIEU-->>UserEU: 200 Write accepted

    note over DDB,Conflict: Async cross-region replication

    DDB->>DDB: Replicate US writes to EU region within 1 second
    DDB->>DDB: Replicate EU writes to US region within 1 second

    alt Same item written in both regions simultaneously
        DDB->>Conflict: Conflict detected - last writer wins by default
        Conflict->>DDB: Apply last-write-wins based on timestamp
        note over Conflict: Custom conflict resolution via Lambda trigger possible
    end

    note over UserUS,DDB: If US region fails Route53 health check removes US endpoint.<br/>All users routed to EU - no manual intervention needed.
```

---

## 14. Disaster Recovery — RPO and RTO Planning

```mermaid
sequenceDiagram
    participant Primary as Primary Region (us-east-1)
    participant Backup as DR Region (us-west-2)
    participant S3 as S3 Cross-Region Replication
    participant RDS as RDS Automated Snapshots
    participant Runbook as DR Runbook / Automation
    participant DNS as Route 53 Health Check

    note over Primary,RDS: Continuous backup during normal operation

    Primary->>S3: App state and config synced every hour
    S3->>Backup: Cross-region replication within 15 minutes
    Primary->>RDS: Automated snapshot every 24 hours
    RDS->>Backup: Copy snapshot to DR region

    note over Primary,Runbook: Disaster strikes primary region

    Primary->>Primary: Region outage at T=0

    DNS->>DNS: Health check fails after 3 consecutive failures (30s)
    DNS->>DNS: Remove primary region from routing policy

    Runbook->>Backup: Trigger DR activation at T+5 minutes
    Backup->>RDS: Restore latest snapshot (last 24h = RPO)
    Backup->>Backup: Launch EKS cluster from saved config
    Backup->>S3: Restore app state from last S3 sync (RPO 1h)
    Backup->>Backup: Run smoke tests

    DNS->>DNS: Add DR region to routing at T+60 minutes
    DNS-->>DNS: Traffic flows to DR region (RTO = 1 hour)

    note over Primary,Runbook: RPO (data loss) = last snapshot age up to 24 hours.<br/>RTO (recovery time) = 60 minutes.<br/>Use Aurora Global DB to reduce RPO to 1 second.
```

---

## 15. Strangler Fig Migration — Monolith to Microservices

```mermaid
sequenceDiagram
    participant Client as Client
    participant Proxy as API Proxy / Router (nginx or APIGW)
    participant Monolith as Monolith App
    participant NewSvc as New Microservice
    participant SharedDB as Shared DB (transitional)
    participant NewDB as New Service DB

    note over Client,NewDB: Phase 1 - proxy in front of monolith, no change

    Client->>Proxy: All requests
    Proxy->>Monolith: 100 percent passthrough
    Monolith-->>Client: Response as before

    note over Client,NewDB: Phase 2 - extract orders into new service

    Client->>Proxy: POST /orders
    Proxy->>Proxy: Route rule - orders path goes to new service
    Proxy->>NewSvc: Forward /orders request
    NewSvc->>SharedDB: Write to orders table (same DB as monolith still)
    SharedDB-->>NewSvc: Committed
    NewSvc-->>Client: Response from new service

    Client->>Proxy: GET /users (not yet migrated)
    Proxy->>Monolith: Route to monolith as before
    Monolith-->>Client: Response from monolith

    note over Client,NewDB: Phase 3 - migrate data ownership to new DB

    NewSvc->>NewSvc: Run dual-write period - write to both SharedDB and NewDB
    NewSvc->>NewDB: New writes go to dedicated DB
    NewSvc->>SharedDB: Legacy reads still work during transition
    NewSvc->>NewSvc: Verify data parity then cut off SharedDB reads
    Proxy->>Monolith: Remove orders routes from monolith

    note over Monolith,NewDB: Repeat for each domain until monolith is empty and retired.<br/>At no point does client experience downtime or API changes.
```

---

## Summary — Architecture Pattern Reference

| Pattern | Problem Solved | Complexity |
|---------|---------------|------------|
| API Gateway fan-out | Single entry, multiple backends, parallel calls | Low |
| SNS + SQS fan-out | One producer, many independent consumers | Low |
| Kafka producer-consumer | High throughput, durable, replayable event stream | Medium |
| Circuit breaker | Cascade failure prevention | Medium |
| Retry with backoff | Transient failure recovery without thundering herd | Low |
| Blue-green deploy | Zero-downtime deploy with instant rollback | Medium |
| Canary deploy | Gradual rollout with automated rollback on metrics | High |
| Lambda cold/warm | Serverless compute with variable latency | Low |
| CDN cache flow | Global edge delivery, reduced origin load | Low |
| WebSocket + Redis pub/sub | Real-time multi-server broadcast | Medium |
| gRPC streaming | Efficient binary RPC with streaming | Medium |
| Istio service mesh | Zero-trust, observability, traffic management without code changes | High |
| Multi-region active-active | Zero-downtime regional failover, global low latency | High |
| Disaster recovery RPO/RTO | Business continuity planning | High |
| Strangler fig migration | Incrementally replace monolith with zero downtime | High |
