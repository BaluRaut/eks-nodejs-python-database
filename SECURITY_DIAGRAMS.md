# Security Architecture Diagrams

Reference diagrams for WAF, DDoS protection, authentication, rate limiting, mTLS, RBAC, and secrets management.

---

## 1. WAF + CloudFront + ALB + EKS — Full Request Path

```mermaid
sequenceDiagram
    actor Client as Internet Client
    participant Shield as AWS Shield Advanced
    participant CF as CloudFront CDN
    participant WAF as AWS WAF
    participant ALB as Application Load Balancer
    participant Ingress as K8s Ingress Controller
    participant Pod as App Pod

    Client->>Shield: HTTPS request
    Shield->>Shield: DDoS pattern detection
    alt DDoS detected
        Shield-->>Client: Drop / absorb attack traffic
    end

    Shield->>CF: Clean traffic forwarded
    CF->>WAF: Evaluate WAF rules
    alt WAF rule match - SQLi / XSS / geo-block / IP block
        WAF-->>Client: 403 Forbidden
    end

    CF->>CF: Check edge cache
    alt Cache HIT
        CF-->>Client: Cached response 200
    end

    CF->>ALB: Cache MISS - forward to origin over HTTPS
    ALB->>ALB: Check listener rules and SSL cert
    ALB->>Ingress: Route to target group port 80
    Ingress->>Pod: Match host path rules, forward request
    Pod-->>Ingress: Application response
    Ingress-->>ALB: Response
    ALB-->>CF: Response
    CF->>CF: Cache response if cacheable
    CF-->>Client: HTTPS 200 response
```

---

## 2. JWT Authentication — Login and Token Validation

```mermaid
sequenceDiagram
    actor User as User / Client App
    participant API as API Gateway
    participant Auth as Auth Service
    participant DB as User DB Postgres
    participant Redis as Redis Token Store
    participant App as Protected Service

    note over User,App: Phase 1 - Login and get JWT

    User->>API: POST /auth/login with email and password
    API->>Auth: Forward login request
    Auth->>DB: SELECT user WHERE email matches
    DB-->>Auth: User record with hashed password
    Auth->>Auth: bcrypt.compare password with hash
    alt Password invalid
        Auth-->>User: 401 Unauthorized
    end
    Auth->>Auth: Sign JWT access token 15min + refresh token 7d
    Auth->>Redis: Store refresh token with user ID TTL 7d
    Auth-->>User: 200 access_token + refresh_token

    note over User,App: Phase 2 - Call protected endpoint

    User->>API: GET /api/orders with Bearer access_token
    API->>API: Extract JWT from Authorization header
    API->>API: Verify JWT signature with secret key
    alt JWT expired or invalid signature
        API-->>User: 401 Unauthorized
    end
    API->>App: Forward request with decoded user claims
    App-->>User: 200 Protected data

    note over User,App: Phase 3 - Refresh expired token

    User->>API: POST /auth/refresh with refresh_token
    API->>Auth: Validate refresh token
    Auth->>Redis: GET refresh token by value
    alt Token not found or expired
        Auth-->>User: 401 Session expired - login again
    end
    Auth->>Auth: Sign new access token
    Auth-->>User: 200 new access_token
```

---

## 3. OAuth2 Authorization Code Flow with PKCE

```mermaid
sequenceDiagram
    actor User as Browser / Mobile App
    participant App as Your Application
    participant IDP as Identity Provider (Google / GitHub)
    participant API as Your API

    User->>App: Click Login with Google

    App->>App: Generate code_verifier random 64 bytes
    App->>App: code_challenge = SHA256(code_verifier) base64url
    App->>App: Store code_verifier in session / localStorage

    App->>IDP: Redirect to /authorize with client_id scope redirect_uri code_challenge
    User->>IDP: Enter Google credentials + consent screen
    IDP-->>App: Redirect back with authorization_code

    App->>IDP: POST /token with authorization_code + code_verifier
    IDP->>IDP: Verify SHA256(code_verifier) matches code_challenge
    alt PKCE check fails
        IDP-->>App: 400 invalid_grant
    end
    IDP-->>App: access_token + id_token + refresh_token

    App->>IDP: GET /userinfo with access_token
    IDP-->>App: User profile name email picture

    App->>App: Create or update local user record
    App->>App: Issue your own app JWT for the user
    App-->>User: Logged in - set session cookie

    User->>API: Subsequent requests with app JWT
    API-->>User: Protected resources
```

---

## 4. API Rate Limiting by User ID — Redis Sliding Window

```mermaid
sequenceDiagram
    participant Client as API Client
    participant GW as API Gateway / Middleware
    participant Redis as Redis Rate Limit Store
    participant App as Application Service
    participant DB as Database

    Client->>GW: POST /api/orders with JWT Bearer token

    GW->>GW: Decode JWT extract user_id = u123
    GW->>GW: Build key = ratelimit:u123:orders

    GW->>Redis: ZADD key current_timestamp score=timestamp member=uuid
    GW->>Redis: ZREMRANGEBYSCORE key 0 window_start (remove old entries)
    GW->>Redis: ZCARD key (count requests in window)
    Redis-->>GW: count = 47

    alt count exceeds limit (e.g. 50 per minute)
        GW->>Redis: TTL key (seconds until window resets)
        Redis-->>GW: ttl = 23 seconds
        GW-->>Client: 429 Too Many Requests Retry-After 23s
    end

    GW->>Redis: EXPIRE key window_seconds (reset TTL)
    GW->>App: Forward request with user context
    App->>DB: Execute business logic
    DB-->>App: Result
    App-->>GW: 200 Response
    GW-->>Client: 200 Response with X-RateLimit-Remaining header
```

---

## 5. API Rate Limiting by API Token — Token Bucket Algorithm

```mermaid
sequenceDiagram
    participant Client as API Consumer App
    participant GW as API Gateway
    participant Redis as Redis Bucket Store
    participant App as Backend Service

    note over Client,App: Token bucket: max 100 tokens, refill 10 per second

    Client->>GW: GET /api/data with X-API-Key abc123

    GW->>GW: Validate API key exists in key registry
    alt API key not found or revoked
        GW-->>Client: 401 Invalid API Key
    end

    GW->>GW: Build bucket key = bucket:apikey:abc123
    GW->>Redis: HGETALL bucket:apikey:abc123
    Redis-->>GW: tokens=85 last_refill=timestamp

    GW->>GW: elapsed = now - last_refill
    GW->>GW: refill = elapsed x 10 tokens per second
    GW->>GW: new_tokens = min(100, 85 + refill) - 1 cost

    alt new_tokens less than 0
        GW-->>Client: 429 Rate limit exceeded bucket empty
    end

    GW->>Redis: HSET bucket tokens=new_tokens last_refill=now EXPIRE 3600
    GW->>App: Forward request
    App-->>GW: Response
    GW-->>Client: 200 with X-RateLimit-Tokens remaining header
```

---

## 6. Multi-Tier Rate Limiting — IP + User + Tenant

```mermaid
sequenceDiagram
    participant Client as Client
    participant GW as API Gateway
    participant Redis as Redis
    participant App as Service

    Client->>GW: Request with JWT and client IP

    note over GW,Redis: Tier 1 - IP rate limit (abuse protection)
    GW->>Redis: INCR ip:1.2.3.4 EXPIRE 60
    Redis-->>GW: count 12
    alt IP count exceeds 1000 per minute
        GW-->>Client: 429 IP rate limited
    end

    note over GW,Redis: Tier 2 - User rate limit (fairness)
    GW->>Redis: INCR user:u123:api EXPIRE 60
    Redis-->>GW: count 45
    alt User count exceeds 100 per minute
        GW-->>Client: 429 User rate limited
    end

    note over GW,Redis: Tier 3 - Tenant quota (billing plan)
    GW->>Redis: INCR tenant:t456:daily EXPIRE 86400
    Redis-->>GW: count 4891
    alt Tenant daily quota exceeds plan limit
        GW-->>Client: 429 Tenant quota exceeded upgrade plan
    end

    GW->>App: All checks passed forward request
    App-->>Client: 200 Response
```

---

## 7. mTLS — Mutual TLS Between Microservices

```mermaid
sequenceDiagram
    participant SvcA as Service A (client)
    participant CA as Certificate Authority (cert-manager / Istio CA)
    participant SvcB as Service B (server)

    note over SvcA,CA: One-time setup - certificate issuance

    SvcA->>CA: Generate CSR for service-a.namespace.svc
    CA->>CA: Sign cert with cluster root CA
    CA-->>SvcA: TLS cert + private key (auto-rotated)

    SvcB->>CA: Generate CSR for service-b.namespace.svc
    CA->>CA: Sign cert
    CA-->>SvcB: TLS cert + private key

    note over SvcA,SvcB: Every request - mutual authentication

    SvcA->>SvcB: ClientHello TLS 1.3
    SvcB-->>SvcA: ServerHello + server cert
    SvcA->>SvcA: Verify server cert signed by trusted CA
    SvcA-->>SvcB: Client cert
    SvcB->>SvcB: Verify client cert signed by trusted CA
    alt Either cert invalid or untrusted
        SvcB-->>SvcA: TLS alert handshake failure
    end
    SvcA->>SvcB: Encrypted request - identity verified both sides
    SvcB-->>SvcA: Encrypted response

    note over SvcA,SvcB: Neither service can be impersonated.<br/>No app-level auth tokens needed between services.
```

---

## 8. RBAC Authorization — Role Based Access Control

```mermaid
sequenceDiagram
    actor User as Authenticated User
    participant API as API Endpoint
    participant AuthZ as Authorization Middleware
    participant DB as Permissions DB
    participant Cache as Redis Permission Cache
    participant Resource as Resource Handler

    User->>API: DELETE /api/projects/p99 with JWT

    API->>AuthZ: Check permission for user=u123 action=delete resource=project:p99

    AuthZ->>Cache: GET perm:u123:delete:project:p99
    alt Cache HIT
        Cache-->>AuthZ: allowed=true TTL remaining
    end

    AuthZ->>DB: SELECT roles FROM user_roles WHERE user_id=u123
    DB-->>AuthZ: roles = [viewer, editor]

    AuthZ->>DB: SELECT permissions FROM role_permissions WHERE role IN roles
    DB-->>AuthZ: permissions = [read, write] - delete NOT included

    alt User lacks required permission
        AuthZ-->>User: 403 Forbidden - delete permission required
    end

    AuthZ->>Cache: SET perm:u123:delete:project:p99 allowed=false TTL 300s
    AuthZ-->>User: 403 Forbidden
```

---

## 9. AWS Secrets Manager — Secret Rotation Flow

```mermaid
sequenceDiagram
    participant Lambda as Rotation Lambda
    participant SM as Secrets Manager
    participant RDS as RDS Database
    participant Pod as K8s App Pod
    participant ESO as External Secrets Operator

    note over Lambda,SM: Automatic rotation triggered (every 30 days)

    SM->>Lambda: Invoke with rotation token and secret ARN
    Lambda->>SM: GetSecretValue - fetch current secret
    Lambda->>Lambda: Generate new strong password

    Lambda->>SM: PutSecretValue AWSPENDING stage with new password
    Lambda->>RDS: ALTER USER app_user PASSWORD new_password
    RDS-->>Lambda: Password changed

    Lambda->>SM: Test connection with new password
    alt Connection test fails
        Lambda->>SM: Mark AWSPENDING as failed
        Lambda->>RDS: Revert to old password
        SM-->>Lambda: Rotation failed alert
    end

    Lambda->>SM: RotateSecretVersion - promote AWSPENDING to AWSCURRENT
    SM-->>Lambda: Rotation complete

    note over ESO,Pod: K8s picks up new secret automatically

    ESO->>SM: Poll for secret changes every 1 minute
    SM-->>ESO: New secret version available
    ESO->>ESO: Update K8s Secret object
    Pod->>Pod: Reload env vars or detect secret file change
    Pod-->>Pod: Connected with new credentials
```

---

## 10. Zero Trust — Service Identity with SPIFFE / SPIRE

```mermaid
sequenceDiagram
    participant WorkloadA as Workload A (Pod)
    participant Agent as SPIRE Agent (DaemonSet)
    participant Server as SPIRE Server
    participant CA as Root CA
    participant WorkloadB as Workload B

    note over WorkloadA,Server: One-time registration

    Server->>Server: Admin registers node + workload selectors
    note over Server: Selector example - k8s:pod-label:app=orders

    note over WorkloadA,Server: Runtime - SVID issuance

    WorkloadA->>Agent: Request SVID via Unix socket
    Agent->>Agent: Verify workload identity via kernel attestation
    Agent->>Server: Node + workload attestation data
    Server->>Server: Validate selectors match policy
    Server->>CA: Issue X.509 SVID for spiffe://cluster/ns/app-orders
    CA-->>Server: Signed SVID cert
    Server-->>Agent: SVID cert + key valid 1 hour
    Agent-->>WorkloadA: SVID delivered via Workload API

    note over WorkloadA,WorkloadB: Request with SPIFFE identity

    WorkloadA->>WorkloadB: mTLS using SVID spiffe://cluster/ns/app-orders
    WorkloadB->>WorkloadB: Validate SVID audience and trust domain
    WorkloadB-->>WorkloadA: Response - identity verified

    note over WorkloadA,WorkloadB: No long-lived passwords or API keys.<br/>Short-lived certs auto-rotate every hour.
```

---

## Summary — When to Use Each Pattern

| Pattern | Best For | Key Tool |
|---------|----------|----------|
| WAF + Shield | Public APIs, DDoS protection | AWS WAF, CloudFront |
| JWT Auth | Stateless API auth, microservices | jsonwebtoken, Auth0 |
| OAuth2 PKCE | Third-party login, mobile apps | Cognito, Auth0, Google |
| User rate limit | Fair usage, abuse prevention | Redis ZADD sliding window |
| API key bucket | Paid API tiers, B2B consumers | Redis HSET token bucket |
| Multi-tier limits | Production APIs with billing | Redis composite keys |
| mTLS | Service-to-service internal auth | cert-manager, Istio |
| RBAC | Feature access control | OPA, Casbin, DB roles |
| Secrets rotation | Credential hygiene, compliance | AWS Secrets Manager |
| SPIFFE / SPIRE | Zero-trust, no static secrets | SPIRE, Istio SPIFFE |
