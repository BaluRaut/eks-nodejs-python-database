# Route 53, Custom Domains, and ACM with EKS

This guide covers everything you need to expose your Node-Hono and Python-FastAPI services on a real domain name with HTTPS, using AWS Route 53, the AWS Load Balancer Controller, External-DNS, and AWS Certificate Manager (ACM).

---

## Table of Contents

1. [How Route 53 Works with EKS](#1-how-route-53-works-with-eks)
2. [Prerequisites](#2-prerequisites)
3. [AWS Load Balancer Controller](#3-aws-load-balancer-controller)
4. [ACM: Requesting and Validating a Certificate](#4-acm-requesting-and-validating-a-certificate)
5. [External-DNS Controller](#5-external-dns-controller)
6. [Kubernetes Ingress with Custom Domain and TLS](#6-kubernetes-ingress-with-custom-domain-and-tls)
7. [Full End-to-End Example](#7-full-end-to-end-example)
8. [Wildcard Certificates for Multiple Services](#8-wildcard-certificates-for-multiple-services)
9. [Common Pitfalls](#9-common-pitfalls)
10. [Architecture Summary](#10-architecture-summary)

---

## 1. How Route 53 Works with EKS

### The Big Picture

When Kubernetes creates a Service of type `LoadBalancer`, AWS provisions an Elastic Load Balancer (ELB or ALB) and assigns it a long, auto-generated DNS name like:

```
a1b2c3d4e5f6g7h8.us-east-1.elb.amazonaws.com
```

Nobody can remember that. Route 53 lets you map a friendly domain name like `api.yourdomain.com` to that load balancer automatically.

### Route 53 Key Concepts

**Hosted Zone**

A hosted zone is a container for DNS records for a single domain. When you register or transfer a domain to Route 53, a public hosted zone is created automatically. You can also create one manually for a domain registered elsewhere.

```
yourdomain.com  →  Hosted Zone ID: Z1D633PJN98FT9
```

**Record Types Used in EKS**

| Type | Use Case | Example |
|------|----------|---------|
| `A` | Maps a name directly to an IPv4 address | `api.yourdomain.com → 54.12.34.56` |
| `CNAME` | Maps a name to another DNS name | `api.yourdomain.com → elb.amazonaws.com` |
| `A (Alias)` | AWS-native alias to an AWS resource (no extra charge, supports zone apex) | `yourdomain.com → ALB DNS name` |

**Why Alias Records Matter**

A regular `CNAME` cannot be used for the zone apex (the root domain, e.g. `yourdomain.com` itself). Route 53 Alias records solve this — they behave like `A` records but resolve dynamically to the current IPs of the target AWS resource (ALB, CloudFront, etc.). Always prefer Alias records when pointing to AWS-managed load balancers.

### The Flow

```
Browser
  |
  | DNS query: api.yourdomain.com
  v
Route 53 Hosted Zone
  |
  | Returns: ALB DNS name (alias) or IP
  v
AWS Application Load Balancer (ALB)
  |
  | Routes by Host header / path
  v
Kubernetes Ingress
  |
  | Routes to correct Service
  v
node-hono-service  OR  python-fastapi-service
```

---

## 2. Prerequisites

Before starting, ensure you have:

- An EKS cluster running (e.g., `learning-cluster` in `us-east-1`)
- `kubectl`, `eksctl`, `aws`, and `helm` CLIs installed and configured
- A domain name (registered in Route 53 or elsewhere)
- OIDC provider enabled on your cluster (required for IRSA)

**Enable OIDC if not already done:**

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster learning-cluster \
  --region us-east-1 \
  --approve
```

**Verify OIDC is active:**

```bash
aws eks describe-cluster \
  --name learning-cluster \
  --region us-east-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

---

## 3. AWS Load Balancer Controller

The AWS Load Balancer Controller watches Kubernetes Ingress resources and provisions real AWS ALBs. Without it, Kubernetes only creates classic ELBs (Layer 4), which lack path-based routing and TLS termination via ACM.

### 3a. Create IAM Policy for the Controller

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

### 3b. Create IAM Role and Service Account (IRSA)

Replace `111122223333` with your actual AWS account ID.

```bash
eksctl create iamserviceaccount \
  --cluster learning-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

### 3c. Install via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=learning-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-0123456789abcdef0
```

Find your VPC ID with:

```bash
aws eks describe-cluster \
  --name learning-cluster \
  --region us-east-1 \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text
```

**Verify the controller is running:**

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Expected output:
```
NAME                           READY   UP-TO-DATE   AVAILABLE
aws-load-balancer-controller   2/2     2            2
```

---

## 4. ACM: Requesting and Validating a Certificate

ACM is AWS's free TLS certificate service. Certificates issued through ACM can be attached directly to ALBs — the private key never leaves AWS.

### 4a. Create a Hosted Zone (if needed)

If your domain is registered outside Route 53, you still create a hosted zone in Route 53:

```bash
aws route53 create-hosted-zone \
  --name yourdomain.com \
  --caller-reference "$(date +%s)"
```

Note the four nameservers returned. Go to your domain registrar and update the nameservers to these four values. DNS propagation can take up to 48 hours, though it usually completes in under an hour.

If your domain is already registered in Route 53, the hosted zone was created automatically.

### 4b. Request a Certificate

```bash
aws acm request-certificate \
  --domain-name "yourdomain.com" \
  --subject-alternative-names "*.yourdomain.com" \
  --validation-method DNS \
  --region us-east-1
```

The `*.yourdomain.com` SAN (Subject Alternative Name) means one certificate covers all subdomains (`api.yourdomain.com`, `app.yourdomain.com`, etc.).

Note the returned `CertificateArn`. You will need it later.

### 4c. DNS Validation vs Email Validation

| Method | How It Works | When to Use |
|--------|-------------|-------------|
| DNS | Add a CNAME record to your hosted zone | Preferred — can be automated, renews automatically |
| Email | Click a link in an email sent to domain admin addresses | Use only if you cannot modify DNS records |

Always use DNS validation with Route 53. ACM can auto-renew the certificate as long as the validation CNAME record stays in place.

### 4d. Complete DNS Validation

Retrieve the CNAME record that ACM requires:

```bash
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:us-east-1:111122223333:certificate/CERT-ID \
  --region us-east-1 \
  --query "Certificate.DomainValidationOptions"
```

Output will show something like:

```json
[
  {
    "DomainName": "yourdomain.com",
    "ValidationStatus": "PENDING_VALIDATION",
    "ResourceRecord": {
      "Name": "_abc123.yourdomain.com.",
      "Type": "CNAME",
      "Value": "_xyz789.acm-validations.aws."
    }
  }
]
```

Add this CNAME record to Route 53. If your hosted zone is in Route 53, AWS can do this automatically:

```bash
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:us-east-1:111122223333:certificate/CERT-ID \
  --region us-east-1 \
  --query "Certificate.DomainValidationOptions[0].ResourceRecord"
```

Then create the record via the CLI (replace the `Name` and `Value` from the output above):

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1D633PJN98FT9 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "_abc123.yourdomain.com.",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "_xyz789.acm-validations.aws."}]
      }
    }]
  }'
```

Wait for the certificate status to become `ISSUED`:

```bash
aws acm wait certificate-validated \
  --certificate-arn arn:aws:acm:us-east-1:111122223333:certificate/CERT-ID \
  --region us-east-1

# Or poll manually:
watch -n 10 "aws acm describe-certificate \
  --certificate-arn arn:aws:acm:us-east-1:111122223333:certificate/CERT-ID \
  --region us-east-1 \
  --query 'Certificate.Status'"
```

---

## 5. External-DNS Controller

External-DNS automatically creates and updates Route 53 records whenever a Kubernetes Service or Ingress resource appears or changes. Without it, you must manually update DNS records every time an ALB gets a new DNS name.

### 5a. IAM Policy for External-DNS

Create a file named `externaldns-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets",
        "route53:ListTagsForResource"
      ],
      "Resource": ["*"]
    }
  ]
}
```

Apply it:

```bash
aws iam create-policy \
  --policy-name ExternalDNSPolicy \
  --policy-document file://externaldns-policy.json
```

### 5b. Create IRSA for External-DNS

```bash
eksctl create iamserviceaccount \
  --cluster learning-cluster \
  --namespace external-dns \
  --name external-dns \
  --attach-policy-arn arn:aws:iam::111122223333:policy/ExternalDNSPolicy \
  --override-existing-serviceaccounts \
  --approve
```

### 5c. Install External-DNS via Helm

```bash
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm repo update

helm install external-dns external-dns/external-dns \
  --namespace external-dns \
  --create-namespace \
  --set provider=aws \
  --set aws.region=us-east-1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=external-dns \
  --set policy=sync \
  --set txtOwnerId=learning-cluster \
  --set domainFilters[0]=yourdomain.com
```

Key parameters:

| Parameter | Purpose |
|-----------|---------|
| `policy=sync` | Creates and deletes records. Use `upsert-only` if you want External-DNS to never delete records. |
| `txtOwnerId` | A unique identifier written to TXT records so External-DNS knows which records it manages. Set this to your cluster name. |
| `domainFilters[0]` | Restricts External-DNS to only touch records under `yourdomain.com`. Important in shared accounts. |

### 5d. How External-DNS Works

When you create a Kubernetes Service or Ingress with the annotation `external-dns.alpha.kubernetes.io/hostname: api.yourdomain.com`, External-DNS:

1. Detects the annotation on the resource
2. Resolves the associated load balancer DNS name or IP
3. Creates or updates an Alias `A` record (for ALBs) or `CNAME` record in Route 53
4. Writes an ownership TXT record alongside it so it can track what it created

When you delete the Service or Ingress, External-DNS removes the DNS records (if `policy=sync`).

---

## 6. Kubernetes Ingress with Custom Domain and TLS

Now that the controller and DNS automation are in place, change the Services from `type: LoadBalancer` to `type: ClusterIP`, and front them with a single Ingress resource. This is the preferred production pattern — one ALB serves all services, routing by hostname or path.

### 6a. Update Services to ClusterIP

```yaml
# k8s/services.yaml
apiVersion: v1
kind: Service
metadata:
  name: node-hono-service
spec:
  type: ClusterIP        # Changed from LoadBalancer
  ports:
  - port: 80
    targetPort: 3000
  selector:
    app: node-hono
---
apiVersion: v1
kind: Service
metadata:
  name: python-fastapi-service
spec:
  type: ClusterIP        # Changed from LoadBalancer
  ports:
  - port: 80
    targetPort: 8000
  selector:
    app: python-fastapi
```

### 6b. Create the Ingress Resource

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: default
  annotations:
    # Tell Kubernetes to use the AWS Load Balancer Controller
    kubernetes.io/ingress.class: alb

    # Create an internet-facing ALB (use "internal" for private VPC-only access)
    alb.ingress.kubernetes.io/scheme: internet-facing

    # Target pods directly (IP mode) rather than going through NodePort
    alb.ingress.kubernetes.io/target-type: ip

    # The ACM certificate ARN — enables HTTPS on port 443
    alb.ingress.kubernetes.io/certificate-arn: >
      arn:aws:acm:us-east-1:111122223333:certificate/CERT-ID

    # Redirect all HTTP traffic to HTTPS
    alb.ingress.kubernetes.io/ssl-redirect: "443"

    # Tell External-DNS which hostname to register
    external-dns.alpha.kubernetes.io/hostname: >
      api.yourdomain.com,app.yourdomain.com

    # Optional: set ALB access log bucket
    # alb.ingress.kubernetes.io/load-balancer-attributes: >
    #   access_logs.s3.enabled=true,access_logs.s3.bucket=my-alb-logs

spec:
  rules:
  - host: api.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: node-hono-service
            port:
              number: 80

  - host: app.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: python-fastapi-service
            port:
              number: 80
```

Apply it:

```bash
kubectl apply -f k8s/services.yaml
kubectl apply -f k8s/ingress.yaml
```

Watch the ALB get created:

```bash
kubectl get ingress main-ingress -w
```

After 2-3 minutes you will see an ADDRESS appear:

```
NAME           CLASS   HOSTS                                    ADDRESS                                          PORTS   AGE
main-ingress   alb     api.yourdomain.com,app.yourdomain.com   k8s-default-mainingr-abc123.us-east-1.elb.amazonaws.com   80      3m
```

External-DNS will then create Route 53 alias records pointing to that ALB address automatically.

### 6c. Verify DNS Records in Route 53

```bash
aws route53 list-resource-record-sets \
  --hosted-zone-id Z1D633PJN98FT9 \
  --query "ResourceRecordSets[?Name=='api.yourdomain.com.']"
```

### 6d. Test the Endpoints

```bash
curl -v https://api.yourdomain.com/
curl -v https://app.yourdomain.com/health
```

---

## 7. Full End-to-End Example

Here is the complete sequence of commands to go from zero to a live HTTPS endpoint.

```bash
# ---- Step 1: Cluster prerequisites ----
eksctl utils associate-iam-oidc-provider \
  --cluster learning-cluster \
  --region us-east-1 \
  --approve

# ---- Step 2: Install AWS Load Balancer Controller ----
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
  --cluster learning-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve

helm repo add eks https://aws.github.io/eks-charts && helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=learning-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$(aws eks describe-cluster \
    --name learning-cluster \
    --region us-east-1 \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)

# ---- Step 3: Request ACM certificate ----
CERT_ARN=$(aws acm request-certificate \
  --domain-name "yourdomain.com" \
  --subject-alternative-names "*.yourdomain.com" \
  --validation-method DNS \
  --region us-east-1 \
  --query CertificateArn \
  --output text)

echo "Certificate ARN: $CERT_ARN"

# ---- Step 4: Add validation CNAME to Route 53 ----
# (Retrieve name/value from ACM and add to hosted zone as shown in Section 4d)
# Wait for validation:
aws acm wait certificate-validated --certificate-arn $CERT_ARN --region us-east-1
echo "Certificate is ISSUED"

# ---- Step 5: Install External-DNS ----
cat > externaldns-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["route53:ChangeResourceRecordSets"],
      "Resource": ["arn:aws:route53:::hostedzone/*"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets",
        "route53:ListTagsForResource"
      ],
      "Resource": ["*"]
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name ExternalDNSPolicy \
  --policy-document file://externaldns-policy.json

eksctl create iamserviceaccount \
  --cluster learning-cluster \
  --namespace external-dns \
  --name external-dns \
  --attach-policy-arn arn:aws:iam::111122223333:policy/ExternalDNSPolicy \
  --override-existing-serviceaccounts \
  --approve

helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/ && helm repo update

helm install external-dns external-dns/external-dns \
  --namespace external-dns \
  --create-namespace \
  --set provider=aws \
  --set aws.region=us-east-1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=external-dns \
  --set policy=sync \
  --set txtOwnerId=learning-cluster \
  --set domainFilters[0]=yourdomain.com

# ---- Step 6: Deploy services and ingress ----
kubectl apply -f k8s/services.yaml
# Edit k8s/ingress.yaml and insert your real $CERT_ARN
kubectl apply -f k8s/ingress.yaml

# ---- Step 7: Watch everything come up ----
kubectl get ingress main-ingress -w
kubectl logs -n external-dns deploy/external-dns --tail=30 -f

# ---- Step 8: Test ----
curl https://api.yourdomain.com/
curl https://app.yourdomain.com/
```

---

## 8. Wildcard Certificates for Multiple Services

A wildcard certificate (`*.yourdomain.com`) covers any single-level subdomain:

- `api.yourdomain.com` — covered
- `app.yourdomain.com` — covered
- `db.internal.yourdomain.com` — NOT covered (two levels deep)

### Requesting a Wildcard Certificate

You already did this in Step 3 by passing `--subject-alternative-names "*.yourdomain.com"`. The certificate covers both the apex (`yourdomain.com`) and all first-level subdomains.

### Using One Certificate Across Multiple Ingresses

If you have services in different namespaces, each with their own Ingress, they can all reference the same certificate ARN:

```yaml
# In namespace: api-team
annotations:
  alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:111122223333:certificate/CERT-ID
  alb.ingress.kubernetes.io/group.name: shared-alb   # <-- groups multiple Ingresses onto one ALB
```

```yaml
# In namespace: data-team
annotations:
  alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:111122223333:certificate/CERT-ID
  alb.ingress.kubernetes.io/group.name: shared-alb   # <-- same group = same ALB
```

The `alb.ingress.kubernetes.io/group.name` annotation is an AWS Load Balancer Controller feature that merges multiple Ingress resources into a single ALB, sharing cost and listener rules.

### Multiple Certificates on One ALB

ALBs support up to 25 certificates. Add them as a comma-separated list or reference multiple ARNs with additional `alb.ingress.kubernetes.io/certificate-arn` values if needed via the `alb.ingress.kubernetes.io/listen-ports` and ACM association.

---

## 9. Common Pitfalls

### 9a. Certificate Stuck in PENDING_VALIDATION

**Symptom:** `aws acm describe-certificate` shows `PENDING_VALIDATION` after 30+ minutes.

**Causes and fixes:**

1. The CNAME validation record was not added to Route 53. Double-check with:
   ```bash
   aws route53 list-resource-record-sets \
     --hosted-zone-id Z1D633PJN98FT9 \
     --query "ResourceRecordSets[?Type=='CNAME']"
   ```

2. The domain's nameservers do not point to Route 53. Verify with:
   ```bash
   dig NS yourdomain.com +short
   ```
   The output must match the four NS records in your Route 53 hosted zone.

3. The CNAME value was entered with a trailing dot in one place but not the other. Route 53 adds trailing dots automatically; your registrar may or may not.

### 9b. DNS Propagation Delays

**Symptom:** `curl https://api.yourdomain.com/` gets a `could not resolve host` error right after applying the Ingress.

**What to expect:**

- External-DNS syncs every 1-3 minutes after the ALB address is assigned
- New Route 53 records have a default TTL of 300 seconds (5 minutes) — resolvers cache for this long
- Global propagation is typically under 10 minutes for Route 53, but can take up to 48 hours if your TTL is high or upstream resolvers are slow

**Check if Route 53 has the record:**
```bash
dig api.yourdomain.com @8.8.8.8
```

**Check External-DNS logs for errors:**
```bash
kubectl logs -n external-dns deploy/external-dns --tail=50
```

### 9c. ALB Not Created / Ingress Has No Address

**Symptom:** `kubectl get ingress` shows no ADDRESS after 5+ minutes.

**Diagnosis:**
```bash
kubectl describe ingress main-ingress
kubectl logs -n kube-system deploy/aws-load-balancer-controller --tail=50
```

**Common causes:**

1. The IAM role for the AWS Load Balancer Controller is missing permissions. Verify the service account has the correct annotation:
   ```bash
   kubectl get serviceaccount aws-load-balancer-controller -n kube-system -o yaml
   ```
   Look for `eks.amazonaws.com/role-arn` in the annotations.

2. The Ingress class annotation is wrong. Verify:
   ```yaml
   annotations:
     kubernetes.io/ingress.class: alb
   ```
   Or use the `spec.ingressClassName: alb` field for newer Kubernetes versions.

3. The VPC ID passed to the Helm chart is incorrect. The controller uses it to discover subnets.

4. Your public subnets are missing the tag `kubernetes.io/role/elb: 1`. The AWS Load Balancer Controller requires this tag to discover which subnets to create the ALB in:
   ```bash
   aws ec2 create-tags \
     --resources subnet-0abc123 subnet-0def456 \
     --tags Key=kubernetes.io/role/elb,Value=1
   ```

### 9d. HTTPS Works but HTTP Does Not Redirect

**Symptom:** `http://api.yourdomain.com/` returns a connection timeout or the raw HTTP response instead of a 301 redirect.

**Fix:** Ensure the SSL redirect annotation is present on the Ingress:
```yaml
alb.ingress.kubernetes.io/ssl-redirect: "443"
```

Also confirm the ALB listener for port 80 exists. The AWS Load Balancer Controller creates it automatically when `ssl-redirect` is set.

### 9e. External-DNS Not Creating Records

**Symptom:** ALB has an address, but no Route 53 record appears.

**Check External-DNS logs:**
```bash
kubectl logs -n external-dns deploy/external-dns | grep -E "error|Error|unable"
```

**Common causes:**

1. The IAM role for External-DNS is missing `route53:ChangeResourceRecordSets` permission on the hosted zone.

2. The `domainFilters` value does not match the hostname in your annotation. If `domainFilters=yourdomain.com` but your annotation says `hostname: api.other-domain.com`, External-DNS will skip it silently.

3. The `txtOwnerId` changed between installs. External-DNS will not manage records whose TXT ownership record has a different owner ID. Use a consistent value tied to the cluster name.

### 9f. Certificate Not Trusted by Browser

**Symptom:** Browser shows "Your connection is not private" or `curl` returns `SSL certificate problem: unable to get local issuer certificate`.

**Causes:**

1. The certificate ARN in the Ingress annotation has a typo or points to a certificate in a different region. ACM certificates for ALBs must be in the same region as the ALB.

2. The certificate has not yet reached `ISSUED` status. Check with:
   ```bash
   aws acm describe-certificate \
     --certificate-arn $CERT_ARN \
     --region us-east-1 \
     --query "Certificate.Status"
   ```

3. The domain in the certificate does not match the hostname being accessed. A cert for `*.yourdomain.com` does not cover `yourdomain.com` (the apex) unless you explicitly added it as a SAN.

---

## 10. Architecture Summary

After completing this guide, your infrastructure looks like this:

```
Internet
   |
   | HTTPS request to api.yourdomain.com
   v
Route 53 (yourdomain.com hosted zone)
   |
   | Alias A record → ALB DNS name
   v
AWS Application Load Balancer (ALB)
   |
   | TLS terminated here using ACM certificate
   | Port 443 listener with rules
   v
   |--- Host: api.yourdomain.com  -->  node-hono-service:80  -->  node-hono pods (port 3000)
   |
   `--- Host: app.yourdomain.com  -->  python-fastapi-service:80  -->  python-fastapi pods (port 8000)

Automation layer (running in-cluster):
  AWS Load Balancer Controller  →  Creates and configures the ALB from Ingress specs
  External-DNS                  →  Creates and updates Route 53 records from Ingress/Service annotations
```

**IAM roles in play:**

| Component | IAM Role | Permissions Needed |
|-----------|----------|--------------------|
| AWS Load Balancer Controller | IRSA on `kube-system/aws-load-balancer-controller` | EC2, ELB, ACM, IAM read, WAF |
| External-DNS | IRSA on `external-dns/external-dns` | Route 53 ChangeResourceRecordSets, List |

**Cost considerations:**

- Each ALB costs roughly $0.008/hour plus $0.008/LCU-hour. Use `alb.ingress.kubernetes.io/group.name` to consolidate multiple Ingresses onto one ALB.
- ACM certificates are free when used with ALBs.
- Route 53 hosted zones cost $0.50/month each. Additional query charges apply above 1 billion queries/month.

---

## Cleanup

When you are done learning, delete resources to avoid charges:

```bash
# Remove Ingress (this deletes the ALB)
kubectl delete ingress main-ingress

# Uninstall Helm releases
helm uninstall aws-load-balancer-controller -n kube-system
helm uninstall external-dns -n external-dns

# Delete the cluster
eksctl delete cluster --name learning-cluster --region us-east-1

# Optionally delete the ACM certificate (after the ALB is gone)
aws acm delete-certificate \
  --certificate-arn arn:aws:acm:us-east-1:111122223333:certificate/CERT-ID \
  --region us-east-1

# Optionally delete the hosted zone (removes all DNS records)
aws route53 delete-hosted-zone --id Z1D633PJN98FT9
```

> Note: Delete the Ingress before deleting the cluster. If you delete the cluster first while an ALB exists, the ALB remains provisioned and continues to incur charges because CloudFormation/eksctl cannot clean it up without the controller running.
