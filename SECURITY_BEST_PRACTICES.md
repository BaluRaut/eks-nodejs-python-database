# Security Best Practices for EKS: A Practical Guide

**Cluster:** `learning-cluster` | **Region:** `us-east-1`
**Services:** Node.js (Hono) on port 3000, Python (FastAPI) on port 8000, Postgres on port 5432

---

## Table of Contents

1. [The Security Mindset: The "Least Privilege Vault" Analogy](#1-the-security-mindset-the-least-privilege-vault-analogy)
2. [Secret Management: Stop Storing Passwords in YAML](#2-secret-management-stop-storing-passwords-in-yaml)
3. [IAM: Principle of Least Privilege](#3-iam-principle-of-least-privilege)
4. [Kubernetes RBAC](#4-kubernetes-rbac)
5. [Network Policies: The Firewall Inside the Cluster](#5-network-policies-the-firewall-inside-the-cluster)
6. [Container Security](#6-container-security)
7. [TLS Everywhere](#7-tls-everywhere)
8. [Audit Logging](#8-audit-logging)
9. [Security Checklist](#9-security-checklist)

---

## 1. The Security Mindset: The "Least Privilege Vault" Analogy

Think of your Kubernetes cluster as a bank vault system. The vault holds everything valuable: database credentials, API keys, customer data, internal service endpoints. In a well-run bank:

- The teller gets a key to their own drawer. They do not get a master key to the entire vault.
- The security guard can open the main door. They cannot open individual safety deposit boxes.
- Nobody writes the vault combination on a sticky note taped to the front door.

In Kubernetes, "least privilege" means the same thing: every pod, every service account, every IAM role, every network connection gets the minimum access required to do its specific job — and nothing more.

**Current state of this repo: the vault door is open.**

Right now, `k8s/node-app.yaml`, `k8s/multi-api.yaml`, and `k8s/postgres.yaml` all contain this:

```yaml
- name: DB_PASSWORD
  value: "password"
```

This means:
- Anyone who clones this git repository sees the database password immediately.
- Anyone who runs `kubectl describe pod <node-hono-pod>` sees the password in plain text in the output.
- Any pod in the cluster that can call the Kubernetes API can read this value.
- CI/CD logs that echo environment variables will leak it automatically.
- The password itself (`password`) is in every dictionary attack wordlist ever created.

There is no network policy, so a compromised frontend pod can directly query Postgres on port 5432. There is no RBAC restriction, so pods run as the `default` service account, which has broader API access than it should. There is no non-root user in the Dockerfiles, so a container escape gives an attacker root-equivalent access on the node.

The sections below fix each of these gaps, in order from highest impact to lowest friction.

---

## 2. Secret Management: Stop Storing Passwords in YAML

### What is wrong right now

Three files commit secrets to git in plaintext:

| File | Secret |
|---|---|
| `k8s/node-app.yaml` | `DB_PASSWORD: password` |
| `k8s/multi-api.yaml` | `DB_PASSWORD: password` (both deployments) |
| `k8s/postgres.yaml` | `POSTGRES_PASSWORD: password` |

**Rule:** Never store a secret in a file that is committed to version control. Ever. Not even in a private repo. Git history is forever — even if you delete the file in a later commit, the secret remains in the commit history and can be extracted with `git log -p`.

---

### Fix 1: Kubernetes Secrets (minimum baseline)

Kubernetes Secrets store values base64-encoded and keep them out of your YAML manifests. This is not encryption, but it is better than plaintext in git.

**Step 1: Create the secret from the command line (never from a YAML file)**

```bash
kubectl create secret generic postgres-credentials \
  --from-literal=password=your-actual-strong-password \
  --from-literal=username=postgres \
  --namespace=default
```

Confirm it exists:

```bash
kubectl get secret postgres-credentials -o yaml
```

The `data` field will show base64-encoded values. Anyone with `kubectl get secret` permission can decode them with `base64 -d`, which is why this is called a baseline, not a solution.

**Step 2: Reference the secret in your pod spec**

Replace the plaintext `value:` entries in `k8s/node-app.yaml` and `k8s/multi-api.yaml`:

```yaml
# Before (insecure — remove this)
env:
- name: DB_PASSWORD
  value: "password"

# After (references the Kubernetes Secret)
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: postgres-credentials
      key: password
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: postgres-credentials
      key: username
```

Apply the same change to the Postgres deployment in `k8s/postgres.yaml`:

```yaml
env:
- name: POSTGRES_PASSWORD
  valueFrom:
    secretKeyRef:
      name: postgres-credentials
      key: password
```

**Why this is still not perfect:**

- Secrets are stored base64-encoded in etcd, which is not the same as encrypted. Anyone with direct etcd access (or a backup of etcd) can read them.
- By default, any pod in the same namespace can request the secret if it has `get` permission on the secret resource.
- The secret value is visible in memory inside the pod process.

This approach is acceptable for a learning cluster. It is not acceptable for production without the additions described in Fix 2 and Fix 3.

---

### Fix 2: AWS Secrets Manager + Secrets Store CSI Driver (production)

This approach stores the actual secret value in AWS Secrets Manager (outside the cluster entirely) and mounts it into pods at runtime using the Secrets Store CSI Driver. The secret never touches etcd.

**Step 1: Store the credential in AWS Secrets Manager**

```bash
aws secretsmanager create-secret \
  --name "learning-cluster/postgres/credentials" \
  --region us-east-1 \
  --secret-string '{"username":"postgres","password":"your-strong-password-here"}'
```

Note the ARN that is returned. You will need it.

**Step 2: Install the Secrets Store CSI Driver and AWS provider**

```bash
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts

helm install csi-secrets-store \
  secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true

helm repo add aws-secrets-manager \
  https://aws.github.io/secrets-store-csi-driver-provider-aws

helm install secrets-provider-aws \
  aws-secrets-manager/secrets-store-csi-driver-provider-aws \
  --namespace kube-system
```

**Step 3: Create a SecretProviderClass**

This resource tells the CSI driver which secret to fetch and how to mount it:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: postgres-credentials-provider
  namespace: default
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "learning-cluster/postgres/credentials"
        objectType: "secretsmanager"
        jmesPath:
          - path: username
            objectAlias: db-username
          - path: password
            objectAlias: db-password
  # This block syncs the secret into a Kubernetes Secret object as well,
  # which lets you use secretKeyRef in env vars (optional but convenient)
  secretObjects:
  - secretName: postgres-credentials-synced
    type: Opaque
    data:
    - objectName: db-username
      key: username
    - objectName: db-password
      key: password
```

Apply it:

```bash
kubectl apply -f secretproviderclass.yaml
```

**Step 4: Mount the secret in your pod**

```yaml
spec:
  serviceAccountName: node-hono-sa  # must have IRSA permission to read Secrets Manager
  volumes:
  - name: secrets-store-vol
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: postgres-credentials-provider
  containers:
  - name: node-hono
    image: 425848620191.dkr.ecr.us-east-1.amazonaws.com/node-hono:latest
    volumeMounts:
    - name: secrets-store-vol
      mountPath: "/mnt/secrets"
      readOnly: true
    env:
    # Option A: read from the synced Kubernetes Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: postgres-credentials-synced
          key: password
    # Option B: read the mounted file directly in application code
    # The file /mnt/secrets/db-password will contain the password value
```

The service account (`node-hono-sa`) must have an IAM policy that allows `secretsmanager:GetSecretValue` on the specific ARN. See Section 3 for how to set that up with IRSA.

---

### Fix 3: Enable Envelope Encryption for Kubernetes Secrets at Rest (KMS)

By default, Kubernetes Secrets are stored unencrypted in etcd (base64 is encoding, not encryption). Envelope encryption wraps each secret with a data encryption key, which is itself encrypted by an AWS KMS key. An attacker with raw etcd access cannot read secrets without also compromising KMS.

**Step 1: Create a KMS key**

```bash
aws kms create-key \
  --description "EKS learning-cluster etcd encryption" \
  --region us-east-1
```

Note the KeyId from the output.

**Step 2: Enable encryption on the existing cluster with eksctl**

```bash
eksctl utils enable-secrets-encryption \
  --cluster=learning-cluster \
  --key-arn=arn:aws:kms:us-east-1:425848620191:key/YOUR-KEY-ID \
  --region=us-east-1
```

This command updates the EKS control plane configuration to encrypt all new and existing Secrets using the KMS key. It may take several minutes.

**Step 3: Rotate existing secrets to ensure they are encrypted**

```bash
kubectl get secrets --all-namespaces -o json | \
  kubectl replace -f -
```

This forces Kubernetes to re-write every secret, which triggers encryption with the new KMS key.

**To enable at cluster creation time with eksctl**, add to your cluster config:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: learning-cluster
  region: us-east-1
secretsEncryption:
  keyARN: arn:aws:kms:us-east-1:425848620191:key/YOUR-KEY-ID
```

---

## 3. IAM: Principle of Least Privilege

### IAM Roles for Service Accounts (IRSA)

By default, every pod on an EKS worker node inherits the IAM role attached to that EC2 node. If the node role has `AmazonS3FullAccess`, then every pod on that node — including a compromised one — can read and write every S3 bucket in your account.

IRSA solves this by linking a specific Kubernetes ServiceAccount to a specific IAM Role. Only the pods using that ServiceAccount get those IAM permissions. The node role gets nothing beyond what it needs to function as a worker.

**How IRSA works:**

1. EKS clusters have an OIDC (OpenID Connect) identity provider.
2. When a pod starts with a ServiceAccount that has an IAM role annotation, the pod receives a signed JWT token from the OIDC provider.
3. AWS STS validates that token and issues temporary credentials scoped to the annotated IAM role.
4. The pod's AWS SDK automatically uses those credentials. No access keys are needed.

**Full example: give the node-hono pod permission to read from one S3 bucket only**

**Step 1: Get your cluster's OIDC issuer URL**

```bash
aws eks describe-cluster \
  --name learning-cluster \
  --region us-east-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

Output looks like: `https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE`

**Step 2: Create the OIDC provider in IAM (one-time per cluster)**

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster learning-cluster \
  --region us-east-1 \
  --approve
```

**Step 3: Create the IAM policy**

```bash
cat > node-hono-s3-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-learning-bucket",
        "arn:aws:s3:::my-learning-bucket/*"
      ]
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name NodeHonoS3ReadPolicy \
  --policy-document file://node-hono-s3-policy.json
```

**Step 4: Create the IAM role with a trust policy scoped to this cluster and service account**

```bash
# Replace OIDC_ID with the ID portion of your OIDC URL (the part after /id/)
OIDC_ID="EXAMPLED539D4633E53DE1B71EXAMPLE"
ACCOUNT_ID="425848620191"

cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/${OIDC_ID}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/${OIDC_ID}:sub": "system:serviceaccount:default:node-hono-sa",
          "oidc.eks.us-east-1.amazonaws.com/id/${OIDC_ID}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name NodeHonoServiceRole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name NodeHonoServiceRole \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/NodeHonoS3ReadPolicy
```

**Step 5: Create the Kubernetes ServiceAccount with the role annotation**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-hono-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::425848620191:role/NodeHonoServiceRole
```

```bash
kubectl apply -f node-hono-serviceaccount.yaml
```

**Step 6: Reference the ServiceAccount in the deployment**

```yaml
spec:
  serviceAccountName: node-hono-sa
  containers:
  - name: node-hono
    ...
```

The pod now has credentials scoped only to that one S3 bucket. Nothing else.

---

### Dangerous IAM Anti-Patterns to Avoid

**Anti-pattern 1: EC2 node IAM role with AdministratorAccess**

If your worker node role has `AdministratorAccess`, every pod on every node in your cluster has full AWS admin access. A single pod escape or container vulnerability becomes a full account compromise.

Check your current node role:

```bash
# Find the node group role
aws eks describe-nodegroup \
  --cluster-name learning-cluster \
  --nodegroup-name <your-nodegroup-name> \
  --region us-east-1 \
  --query "nodegroup.nodeRole"

# List policies attached to that role
aws iam list-attached-role-policies --role-name <node-role-name>
```

The node role should only have the AWS-managed policies required for EKS worker nodes:
- `AmazonEKSWorkerNodePolicy`
- `AmazonEKS_CNI_Policy`
- `AmazonEC2ContainerRegistryReadOnly`

Nothing else.

**Anti-pattern 2: Hardcoded IAM credentials in environment variables**

```yaml
# Never do this
env:
- name: AWS_ACCESS_KEY_ID
  value: "AKIAIOSFODNN7EXAMPLE"
- name: AWS_SECRET_ACCESS_KEY
  value: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

Static IAM credentials do not rotate automatically. They appear in `kubectl describe pod` output. They appear in CI/CD logs. They end up in bug reports. Use IRSA instead — it provides temporary, automatically-rotated credentials with no access keys at all.

**Anti-pattern 3: Sharing one IAM user across multiple CI/CD pipelines**

Each pipeline (CircleCI, GitHub Actions, a developer's local machine) should have its own IAM identity with the minimum permissions for that specific job. Sharing credentials means:
- You cannot audit which pipeline performed which action.
- Rotating credentials for one pipeline breaks all pipelines.
- A leaked credential from one pipeline compromises all pipelines.

Use OIDC federation with your CI/CD provider instead of IAM users with static keys.

---

## 4. Kubernetes RBAC

### Default Behavior

By default, every pod runs under the `default` service account in its namespace. That service account has a token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. Any process inside the pod can use that token to call the Kubernetes API.

On many clusters (especially learning clusters), this gives pods more API access than they should have. A compromised application could enumerate secrets, list other pods, or even modify deployments.

### Creating a Minimal Role

A Role grants permissions within a single namespace. A ClusterRole grants permissions across all namespaces (or to cluster-scoped resources).

```yaml
# Allow the node-hono app to list and get pods in its own namespace only
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

```yaml
# Bind the role to the node-hono service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: node-hono-pod-reader
  namespace: default
subjects:
- kind: ServiceAccount
  name: node-hono-sa
  namespace: default
roleRef:
  kind: Role
  apiGroupName: rbac.authorization.k8s.io
  name: pod-reader
```

Apply both:

```bash
kubectl apply -f pod-reader-role.yaml
kubectl apply -f node-hono-rolebinding.yaml
```

### RoleBinding vs ClusterRoleBinding

| Resource | Scope | When to use |
|---|---|---|
| `Role` | Single namespace | Application-level permissions |
| `ClusterRole` | All namespaces + cluster resources | Cluster-wide tooling (monitoring, ingress controllers) |
| `RoleBinding` | Grants a Role or ClusterRole within one namespace | Narrowing a ClusterRole to one namespace |
| `ClusterRoleBinding` | Grants a ClusterRole across the entire cluster | Cluster administrators and global operators only |

Prefer RoleBinding over ClusterRoleBinding. If a ClusterRole defines useful permissions but your app only operates in one namespace, bind it with RoleBinding to limit scope.

### Never Use the Default Service Account for Applications

Create a dedicated ServiceAccount for each application:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: python-fastapi-sa
  namespace: default
  automountServiceAccountToken: false  # disable API token unless the app actually needs it
```

If your application does not need to call the Kubernetes API at all (most applications do not), set `automountServiceAccountToken: false`. This removes the token from the pod filesystem entirely, eliminating that attack surface.

Reference it in the deployment:

```yaml
spec:
  serviceAccountName: python-fastapi-sa
  automountServiceAccountToken: false
```

### Auditing Permissions with kubectl auth can-i

```bash
# Can the node-hono service account list secrets?
kubectl auth can-i list secrets \
  --namespace default \
  --as system:serviceaccount:default:node-hono-sa

# Can the default service account create deployments?
kubectl auth can-i create deployments \
  --namespace default \
  --as system:serviceaccount:default:default

# See all permissions for a service account
kubectl auth can-i --list \
  --namespace default \
  --as system:serviceaccount:default:node-hono-sa
```

Run these after any RBAC change to verify the permissions are exactly what you intended.

---

## 5. Network Policies: The Firewall Inside the Cluster

### Default Behavior

By default, Kubernetes allows all pods to communicate with all other pods on all ports. This is a flat network with no internal firewall.

In this repo, that means:

- The node-hono pod can connect directly to the python-fastapi pod on port 8000.
- The python-fastapi pod can connect directly to the Postgres pod on port 5432.
- The node-hono pod can also connect directly to Postgres on port 5432.
- A compromised node-hono pod can scan every other pod in the cluster on every port.

Network Policies let you define rules at the pod level: which pods can send traffic to which other pods, on which ports, using which protocols.

**Note:** Network Policies require a CNI plugin that supports them. Amazon VPC CNI supports Network Policies as of VPC CNI 1.14+. Alternatively, install Calico or Cilium.

### Enable Network Policy Support with Amazon VPC CNI

```bash
aws eks update-addon \
  --cluster-name learning-cluster \
  --addon-name vpc-cni \
  --configuration-values '{"enableNetworkPolicy": "true"}' \
  --region us-east-1
```

### Step 1: Deny all ingress by default

Apply this policy to every namespace that runs application workloads:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}  # matches all pods in this namespace
  policyTypes:
  - Ingress
```

After this policy is applied, no pod can receive any inbound connection unless another NetworkPolicy explicitly allows it.

### Step 2: Allow node-hono and python-fastapi to reach Postgres on port 5432 only

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-apps-to-postgres
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

Now only pods with the label `app: node-hono` or `app: python-fastapi` can reach Postgres. Anything else — including a compromised third pod — is blocked at the network layer.

### Step 3: Allow the ingress controller to reach app pods

If you are using the AWS Load Balancer Controller, allow traffic into your application pods from the ingress controller namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-node-hono
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: node-hono
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: TCP
      port: 3000
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-python-fastapi
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: python-fastapi
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: TCP
      port: 8000
```

### Verify the policies are working

```bash
# List all network policies in the default namespace
kubectl get networkpolicies -n default

# Describe a specific policy
kubectl describe networkpolicy allow-apps-to-postgres -n default

# Test connectivity from a debug pod (should be blocked by the deny-all policy)
kubectl run debug-pod --image=busybox --rm -it --restart=Never -- \
  nc -zv postgres-service 5432
```

---

## 6. Container Security

### ECR Image Scanning

Enable vulnerability scanning on every image push to ECR:

```bash
# Enable scan-on-push for the node-hono repository
aws ecr put-image-scanning-configuration \
  --repository-name node-hono \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1

# Enable scan-on-push for the python-app repository
aws ecr put-image-scanning-configuration \
  --repository-name python-app \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1
```

View scan results after a push:

```bash
aws ecr describe-image-scan-findings \
  --repository-name node-hono \
  --image-id imageTag=latest \
  --region us-east-1
```

The output lists CVEs by severity (CRITICAL, HIGH, MEDIUM, LOW). Address all CRITICAL and HIGH findings before deploying to production.

For continuous scanning with Amazon Inspector (scans images even after they are already in ECR), enable it at the account level:

```bash
aws inspector2 enable \
  --resource-types ECR \
  --region us-east-1
```

### Running as a Non-Root User

Both Dockerfiles in this repo run as root by default. If a vulnerability in the application allows code execution, the attacker gets root inside the container. Add a non-root user.

**For node-hono/Dockerfile:**

```dockerfile
FROM node:18-slim
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install --production
COPY . .

# Create a non-root user and switch to it
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 --ingroup nodejs hono
USER 1001

EXPOSE 3000
CMD ["node", "index.js"]
```

**For python-fastapi/Dockerfile:**

```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

# Create a non-root user and switch to it
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup appuser
USER 1001

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Read-Only Root Filesystem in Pod Spec

Prevent processes inside the container from writing to the container filesystem. Malware and exploit code often writes files to the root filesystem to persist. A read-only filesystem blocks this category of attack entirely.

```yaml
containers:
- name: node-hono
  image: 425848620191.dkr.ecr.us-east-1.amazonaws.com/node-hono:latest
  securityContext:
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
    runAsNonRoot: true
    runAsUser: 1001
  # If the app needs to write temp files, mount a writable emptyDir volume
  volumeMounts:
  - name: tmp-dir
    mountPath: /tmp
volumes:
- name: tmp-dir
  emptyDir: {}
```

### Dropping Linux Capabilities

Linux containers run with a set of default capabilities that most applications do not need. Drop them all and add back only what is required:

```yaml
securityContext:
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE  # only if the app binds to a port below 1024
```

For node-hono (port 3000) and python-fastapi (port 8000), both above 1024, you can drop ALL capabilities with no additions:

```yaml
securityContext:
  capabilities:
    drop:
    - ALL
```

### Pod Security Standards

Kubernetes has built-in Pod Security Standards that enforce security at the namespace level. There are three levels:

| Level | What it allows | Use case |
|---|---|---|
| `privileged` | Everything — no restrictions | System components that genuinely need root access |
| `baseline` | Prevents known privilege escalation. Allows most containers | General workloads that have not been hardened |
| `restricted` | Requires non-root user, read-only root FS, dropped capabilities | Production application workloads |

Enforce the `restricted` standard on the `default` namespace:

```bash
kubectl label namespace default \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

After applying this label, any pod that violates the restricted standard will be rejected at admission time. Use `warn` mode first to identify violations before enforcing.

---

## 7. TLS Everywhere

### Between the Internet and the ALB

Use an ACM certificate on the Application Load Balancer so that traffic from the user's browser to the ALB is encrypted. This is covered in detail in `ROUTE53_CUSTOM_DOMAIN_ACM.md`.

The short version: create a certificate in ACM, annotate the Ingress or Service with `service.beta.kubernetes.io/aws-load-balancer-ssl-cert`, and the ALB terminates TLS before forwarding to your pods.

### Between Pods (Internal Traffic)

After TLS terminates at the ALB, internal traffic from the ALB to your pods and between pods travels unencrypted by default. For most learning clusters, this is acceptable because the traffic stays within your VPC. For production:

**Option A: Kubernetes native TLS**

Generate a certificate for your service and mount it as a Secret. Configure your application server (for Hono, add HTTPS support; for Uvicorn, use `--ssl-keyfile` and `--ssl-certfile`) to serve TLS. Reference the cert Secret in the pod spec.

**Option B: Service mesh (Istio or AWS App Mesh)**

A service mesh automatically provisions mutual TLS (mTLS) between every pair of services with no application code changes. Istio injects a sidecar proxy alongside every pod and handles certificate rotation automatically.

```bash
# Install Istio (basic example)
istioctl install --set profile=default -y

# Enable automatic sidecar injection in the default namespace
kubectl label namespace default istio-injection=enabled
```

After injection, all service-to-service traffic is encrypted and mutually authenticated with zero changes to your application code.

### Database Connections: Always Require SSL

When connecting to RDS Postgres (see `DATABASE_CONSOLIDATION_RDS_VS_CONTAINER.md`), require SSL in the connection string:

```bash
# Node.js (pg library) connection string
postgresql://postgres:password@your-rds-endpoint:5432/postgres?sslmode=require

# Python (psycopg2)
psycopg2.connect(
    host="your-rds-endpoint",
    user="postgres",
    password="...",
    database="postgres",
    sslmode="require"
)
```

RDS enforces SSL by default for Postgres. Setting `sslmode=require` in the client ensures that if SSL ever fails (misconfiguration, certificate issue), the connection is refused rather than silently falling back to plaintext.

For in-cluster Postgres (as in this repo), add TLS to the Postgres service by generating a self-signed certificate and configuring `pg_hba.conf` to require SSL. For a learning cluster, this is optional; for production, use RDS with SSL enabled.

---

## 8. Audit Logging

### Enable EKS Control Plane Audit Logs

EKS control plane logging captures every API call made to your cluster: who ran `kubectl`, what command, from which identity, at what time. This is your primary forensic tool for detecting and investigating incidents.

```bash
eksctl utils update-cluster-logging \
  --cluster=learning-cluster \
  --region=us-east-1 \
  --enable-types=audit,api,authenticator,controllerManager,scheduler \
  --approve
```

Logs go to CloudWatch Logs under the log group `/aws/eks/learning-cluster/cluster`. You pay for CloudWatch storage, so set a retention policy:

```bash
aws logs put-retention-policy \
  --log-group-name /aws/eks/learning-cluster/cluster \
  --retention-in-days 90 \
  --region us-east-1
```

### CloudWatch Logs Insights: Finding Suspicious Activity

Open CloudWatch Logs Insights, select the log group `/aws/eks/learning-cluster/cluster`, and run these queries:

**Find all `kubectl exec` calls (someone shelled into a pod):**

```
fields @timestamp, @message
| filter @logStream like /kube-apiserver-audit/
| filter ispresent(requestURI)
| filter requestURI like /exec/
| sort @timestamp desc
| limit 50
```

`kubectl exec` into a pod is a significant event in production. It should be rare and intentional. Any `exec` by an unexpected user or into a sensitive pod (like Postgres) is a red flag.

**Find all secret reads:**

```
fields @timestamp, user.username, objectRef.name, objectRef.namespace
| filter @logStream like /kube-apiserver-audit/
| filter verb = "get"
| filter objectRef.resource = "secrets"
| sort @timestamp desc
| limit 100
```

**Find all failed authentication attempts:**

```
fields @timestamp, @message
| filter @logStream like /authenticator/
| filter @message like /authentication failed/
| sort @timestamp desc
| limit 50
```

**Find privilege escalation attempts (binding to cluster-admin):**

```
fields @timestamp, user.username, objectRef.name
| filter @logStream like /kube-apiserver-audit/
| filter verb in ["create", "update"]
| filter objectRef.resource in ["clusterrolebindings", "rolebindings"]
| filter requestObject.roleRef.name = "cluster-admin"
| sort @timestamp desc
```

### AWS GuardDuty for EKS

GuardDuty for EKS analyzes EKS audit logs and detects threats automatically. It identifies:

- Cryptocurrency mining activity (unusual CPU usage patterns + outbound connections to mining pools)
- Privilege escalation (pod attempting to bind to cluster-admin, unusual API calls from pods)
- Container escape attempts (host path mounts, privileged container requests)
- Unusual Kubernetes API activity (API calls from an unexpected IP, access outside business hours)
- Tor exit node connections (communication with known Tor network endpoints)

Enable it with one command:

```bash
aws guardduty create-detector \
  --enable \
  --features '[{"Name": "EKS_AUDIT_LOGS", "Status": "ENABLED"}]' \
  --region us-east-1
```

Verify it is running:

```bash
aws guardduty list-detectors --region us-east-1
```

GuardDuty findings appear in the GuardDuty console and can be sent to Security Hub, SNS, or EventBridge for alerting. Set up an SNS notification for HIGH and CRITICAL findings:

```bash
# Get your detector ID from the previous command
DETECTOR_ID="your-detector-id"

aws guardduty create-publishing-destination \
  --detector-id $DETECTOR_ID \
  --destination-type SNS \
  --destination-properties DestinationArn=arn:aws:sns:us-east-1:425848620191:security-alerts \
  --region us-east-1
```

---

## 9. Security Checklist

Use this checklist before declaring a cluster production-ready.

### Secrets

- [ ] 1. No plaintext passwords in any YAML file committed to git
- [ ] 2. All secrets created via `kubectl create secret` or stored in AWS Secrets Manager
- [ ] 3. All Deployments use `secretKeyRef` or the Secrets Store CSI Driver — no `value:` for credentials
- [ ] 4. KMS envelope encryption enabled for Kubernetes Secrets at rest
- [ ] 5. Git history audited and cleaned if secrets were ever committed (`git-secrets` or `truffleHog` scan)

### IAM

- [ ] 6. EC2 node IAM role has only the three required AWS-managed EKS policies — nothing else
- [ ] 7. OIDC provider associated with the cluster (`eksctl utils associate-iam-oidc-provider`)
- [ ] 8. Each application that needs AWS access has a dedicated IAM Role via IRSA — no shared roles
- [ ] 9. No IAM access keys stored in environment variables or Kubernetes Secrets
- [ ] 10. CI/CD pipelines use OIDC federation, not long-lived IAM user keys

### Kubernetes RBAC

- [ ] 11. No application pods use the `default` service account — each app has a named ServiceAccount
- [ ] 12. Applications that do not call the Kubernetes API have `automountServiceAccountToken: false`
- [ ] 13. No ClusterRoleBinding grants `cluster-admin` to any application service account
- [ ] 14. Permissions audited with `kubectl auth can-i --list` for each service account

### Network

- [ ] 15. Default-deny NetworkPolicy applied to all application namespaces
- [ ] 16. Explicit NetworkPolicies grant only the necessary pod-to-pod connections
- [ ] 17. Postgres is reachable only from node-hono and python-fastapi pods — not from the internet, not from other pods
- [ ] 18. Security groups on worker nodes restrict ingress to necessary ports only

### Container Security

- [ ] 19. ECR `scanOnPush` enabled for all repositories
- [ ] 20. No CRITICAL or HIGH CVEs in deployed images
- [ ] 21. All container images run as a non-root user (UID >= 1000)
- [ ] 22. Pod spec sets `runAsNonRoot: true` and `allowPrivilegeEscalation: false`
- [ ] 23. `readOnlyRootFilesystem: true` set on all containers
- [ ] 24. All unnecessary Linux capabilities dropped
- [ ] 25. Pod Security Standard `restricted` enforced on application namespaces

### TLS

- [ ] 26. All external-facing endpoints served over HTTPS with a valid ACM certificate
- [ ] 27. HTTP redirected to HTTPS at the load balancer level
- [ ] 28. Database connections use `sslmode=require`
- [ ] 29. Internal pod-to-pod traffic encrypted (service mesh or app-level TLS) if handling sensitive data

### Audit and Detection

- [ ] 30. EKS control plane audit logs enabled and shipping to CloudWatch
- [ ] 31. CloudWatch log retention policy set (90 days minimum)
- [ ] 32. GuardDuty enabled with EKS audit log analysis
- [ ] 33. Alerting configured for HIGH and CRITICAL GuardDuty findings
- [ ] 34. A process exists to review audit logs weekly and act on alerts

---

## Quick Reference: Commands to Check Your Current State

```bash
# Are any secrets stored in plaintext in current pod specs?
kubectl get pods -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.containers[*]}{range .env[*]}{.name}{": "}{.value}{"\n"}{end}{end}{end}'

# Which pods are running as root?
kubectl get pods -n default -o jsonpath='{range .items[*]}{.metadata.name}{" runAsUser: "}{.spec.containers[0].securityContext.runAsUser}{"\n"}{end}'

# Which service accounts have cluster-admin?
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'

# Is GuardDuty enabled?
aws guardduty list-detectors --region us-east-1

# Are EKS audit logs enabled?
aws eks describe-cluster \
  --name learning-cluster \
  --region us-east-1 \
  --query "cluster.logging.clusterLogging"
```

---

*This guide is specific to `learning-cluster` in `us-east-1`. Account ID: `425848620191`. For a complete walkthrough of cluster setup, see `EKS_FULL_GUIDE.md`. For RDS vs in-cluster Postgres tradeoffs (including SSL configuration), see `DATABASE_CONSOLIDATION_RDS_VS_CONTAINER.md`.*
