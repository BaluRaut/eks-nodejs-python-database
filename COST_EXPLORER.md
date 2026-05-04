# AWS Cost Explorer Guide for EKS Learners

> **This project:** Node.js (Hono) + Python (FastAPI) on EKS, cluster `learning-cluster` in `us-east-1`, 2 x `t3.medium` nodes, ECR for image storage, ELB for ingress traffic.

---

## The Hotel Analogy

AWS billing is like a hotel. You do not pay one flat rate — you pay for each service independently, and the charges accumulate whether or not you are actively using them.

| Hotel Item | AWS Equivalent |
|---|---|
| Room rental (per night) | EC2 instances (per hour) |
| Parking garage | ELB load balancer (always-on fee) |
| Minibar and room service | Data transfer (egress charges) |
| Luggage storage | ECR / S3 storage |
| Concierge desk | EKS control plane (managed service fee) |
| Resort fee (unavoidable) | EKS control plane — no free tier |

The hotel charges you for parking even if you did not use your car. AWS charges you for a load balancer even if zero requests came in. This guide exists so you do not check out of the hotel with a $300 surprise on your bill.

---

## Table of Contents

1. [Understanding Your AWS Bill for This Project](#1-understanding-your-aws-bill-for-this-project)
2. [AWS Cost Explorer UI Walkthrough](#2-aws-cost-explorer-ui-walkthrough)
3. [Setting Up Budget Alerts](#3-setting-up-budget-alerts-critical-for-learners)
4. [Cost Tagging Strategy](#4-cost-tagging-strategy)
5. [Cost Optimization Strategies](#5-cost-optimization-strategies)
6. [The "Leave It Running" Risk](#6-the-leave-it-running-risk)
7. [Quick Cost Check Commands](#7-quick-cost-check-commands)

---

## 1. Understanding Your AWS Bill for This Project

### Line-by-Line Cost Breakdown

#### EKS Control Plane

- **Price:** $0.10 per hour, charged continuously while the cluster exists
- **This is not optional.** You cannot run EKS without the control plane fee.
- **There is no free tier for EKS.** The moment you run `eksctl create cluster`, the clock starts.
- Monthly cost at constant uptime: `$0.10 x 24 x 30 = $72.00`

#### EC2 Worker Nodes — 2 x t3.medium

- **On-demand price:** ~$0.0416 per hour per instance (us-east-1, Linux)
- 2 nodes running continuously: `$0.0416 x 2 x 24 = ~$2.00/day`
- Monthly: `~$60.00` for both nodes
- These are regular EC2 instances. EKS does not add a surcharge on top of EC2 pricing.
- The nodes also consume EBS storage (default 20 GB gp2 per node):
  - EBS cost: `$0.10/GB/month x 20 GB x 2 = $4.00/month`

#### ELB — Classic/Application Load Balancer

When Kubernetes creates a `LoadBalancer` service or an `Ingress`, AWS provisions a real load balancer.

- **ALB hourly fee:** $0.008 per hour
- **LCU (Load Balancer Capacity Unit):** $0.008 per LCU-hour (covers new connections, active connections, processed bytes, rule evaluations)
- At very low traffic (learning workload), LCU charges will be minimal — budget $0.01–$0.05/day
- The base hourly charge applies even with zero traffic: `$0.008 x 24 = $0.19/day`, `~$5.76/month`
- **Critical:** If you delete your cluster without deleting the `LoadBalancer` service first, the ELB persists and continues charging.

#### ECR — Container Image Storage

- **Storage:** $0.10 per GB per month
- **Data transfer into ECR:** Free
- **Data transfer out of ECR to EKS (same region):** Free
- **Data transfer out to the internet:** $0.09/GB (first 10 TB/month)
- A typical Node.js or Python image is 100–300 MB. For 5–10 images, expect less than $0.10/month in storage.
- ECR is rarely a meaningful cost at this scale.

#### Data Transfer

Data transfer is the hidden charge that surprises people.

| Transfer Type | Price |
|---|---|
| Inbound (to AWS) | Free |
| Between services in the same AZ | Free |
| Between services in different AZs (same region) | $0.01/GB each direction |
| Outbound to the internet (first 100 GB/month) | $0.09/GB |
| EC2 to ECR (same region) | Free |

For a learning workload with low external traffic, data transfer costs are negligible. They become significant at scale.

---

### Realistic Cost Table: This Exact Project

Assumptions: cluster running 24/7, 2 x t3.medium nodes, 1 ALB, ~5 small container images in ECR, minimal external traffic.

| Service | Unit | Hourly | Daily | Monthly |
|---|---|---|---|---|
| EKS Control Plane | 1 cluster | $0.100 | $2.40 | $72.00 |
| EC2 t3.medium x2 | On-demand | $0.083 | $2.00 | $60.00 |
| EBS storage (2 x 20 GB) | gp2 | — | $0.13 | $4.00 |
| ALB (base fee) | 1 LB | $0.008 | $0.19 | $5.76 |
| ALB LCU (light traffic) | estimate | — | $0.03 | $0.90 |
| ECR storage (~1 GB) | 1 GB | — | $0.003 | $0.10 |
| Data transfer (outbound) | minimal | — | $0.05 | $1.50 |
| **Total (estimated)** | | **~$0.19/hr** | **~$4.80** | **~$144** |

> **The EKS control plane alone costs $72/month.** This is the single largest cost for a small learning cluster and is often underestimated.

---

### What Free Tier Covers (and Does NOT Cover)

| Item | Free Tier? | Notes |
|---|---|---|
| EKS control plane | **No** | $0.10/hr from the first second |
| EC2 t2.micro (not t3.medium) | Yes, 750 hrs/month for 12 months | t3.medium is NOT free tier eligible |
| EC2 t3.medium | **No** | You pay full on-demand rate |
| EBS storage (30 GB) | Yes, for 12 months | Covers the default node storage |
| ALB | **No** | No free tier |
| ECR (500 MB storage) | Yes, always free | Only 500 MB; exceeded easily |
| Data transfer (1 GB outbound) | Yes, always free | Very small amount |
| CloudWatch (basic metrics) | Yes | Basic EC2 metrics included |
| CloudWatch detailed metrics | **No** | 1-minute granularity costs extra |

**Bottom line for this project:** The EC2 free tier does not apply to `t3.medium`. You are paying for all EC2, all EKS control plane, and all ALB charges from day one.

---

## 2. AWS Cost Explorer UI Walkthrough

### Finding Cost Explorer

1. Log into the AWS Management Console
2. In the top search bar, type `Cost Explorer`
3. Select **AWS Cost Explorer** under Billing and Cost Management
4. If this is your first visit, click **Enable Cost Explorer** — it takes up to 24 hours to populate historical data

Direct URL: `https://us-east-1.console.aws.amazon.com/cost-management/home#/cost-explorer`

---

### Filtering by Service

Cost Explorer opens on a bar chart showing total spend. To isolate specific services:

1. In the right panel, find **Filters**
2. Click **Service** from the filter dimension dropdown
3. Check the boxes for: `Amazon Elastic Kubernetes Service`, `Amazon EC2`, `Elastic Load Balancing`, `Amazon EC2 Container Registry (ECR)`
4. Click **Apply**

The chart now shows only those services. You can toggle individual services on and off using the legend.

---

### Grouping by Usage Type

Usage type is the most granular view — it shows exactly what you are being charged for within each service.

1. In the **Group by** dropdown, select `Usage Type`
2. You will see entries like:
   - `USE1-BoxUsage:t3.medium` — EC2 compute hours in us-east-1
   - `USE1-LoadBalancerUsage` — ALB base hours
   - `USE1-LCUUsage` — ALB capacity unit charges
   - `AmazonEKS-Hours:perCluster` — control plane fee
   - `USE1-EBS:VolumeUsage.gp2` — EBS disk storage

This view answers the question: "I know EC2 is expensive this month — but what exactly within EC2 is costing me?"

---

### Using the Forecasting Feature

Cost Explorer can project your spend for the remainder of the current month.

1. Set the date range to include the current month (e.g., May 1 – May 31)
2. Look for the **Forecasted costs** line on the chart (shown as a dashed line extending to the end of the month)
3. The **Monthly forecast** card shows the projected total and a confidence range

For a more explicit forecast:

1. Go to **Forecasting** in the left navigation panel
2. Set the forecast period (up to 12 months)
3. The model uses your historical spending pattern — it is more accurate after you have 2+ months of data

At the start of a project, the forecast is based on the current daily burn rate projected forward.

---

### Identifying the Single Biggest Cost Driver

1. Set the date range to the current month
2. Set **Group by** to `Service`
3. Sort the data table (below the chart) by the **Cost** column, descending
4. The top row is your biggest cost driver

For this project, EKS control plane or EC2 will be at the top. Once identified, drill deeper:

1. Click the service name in the table
2. Change **Group by** to `Usage Type` to see what within that service is driving cost

---

## 3. Setting Up Budget Alerts (Critical for Learners)

Budget alerts are the single most important thing a learner can set up. They prevent the scenario where you forget about a running cluster for two weeks and receive a $300 bill.

### Create a Monthly Budget via AWS Console

1. Go to **AWS Budgets**: `https://console.aws.amazon.com/billing/home#/budgets`
2. Click **Create budget**
3. Select **Use a template** > **Monthly cost budget**
4. Set the budgeted amount (e.g., `$50`)
5. Add email recipients for alerts
6. Click **Create budget**

---

### Create a Budget via AWS CLI

The following example creates a $50/month budget for the `learning-cluster` project with alerts at 50%, 80%, and 100%.

**Step 1: Create the budget JSON file**

```bash
cat > /tmp/budget.json << 'EOF'
{
  "BudgetName": "learning-cluster-monthly",
  "BudgetLimit": {
    "Amount": "50",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST",
  "CostFilters": {},
  "CostTypes": {
    "IncludeTax": true,
    "IncludeSubscription": true,
    "UseBlended": false,
    "IncludeRefund": false,
    "IncludeCredit": false,
    "IncludeUpfront": true,
    "IncludeRecurring": true,
    "IncludeOtherSubscription": true,
    "IncludeSupport": true,
    "UseAmortized": false
  }
}
EOF
```

**Step 2: Create the notifications JSON file**

Replace `your-email@example.com` with your actual address.

```bash
cat > /tmp/notifications.json << 'EOF'
[
  {
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 50,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [
      {
        "SubscriptionType": "EMAIL",
        "Address": "your-email@example.com"
      }
    ]
  },
  {
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [
      {
        "SubscriptionType": "EMAIL",
        "Address": "your-email@example.com"
      }
    ]
  },
  {
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 100,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [
      {
        "SubscriptionType": "EMAIL",
        "Address": "your-email@example.com"
      }
    ]
  }
]
EOF
```

**Step 3: Create the budget**

```bash
# Get your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws budgets create-budget \
  --account-id "$ACCOUNT_ID" \
  --budget file:///tmp/budget.json \
  --notifications-with-subscribers file:///tmp/notifications.json
```

**Step 4: Verify the budget was created**

```bash
aws budgets describe-budgets \
  --account-id "$ACCOUNT_ID" \
  --query 'Budgets[*].{Name:BudgetName,Limit:BudgetLimit.Amount,Type:BudgetType}'
```

---

### SNS Topic Setup for Programmatic Alerts

For alerts beyond email (e.g., triggering a Lambda function or sending to Slack), use SNS.

```bash
# Create an SNS topic
aws sns create-topic \
  --name learning-cluster-cost-alerts \
  --region us-east-1

# Subscribe your email to the topic
SNS_ARN=$(aws sns list-topics \
  --query "Topics[?contains(TopicArn,'learning-cluster-cost-alerts')].TopicArn" \
  --output text)

aws sns subscribe \
  --topic-arn "$SNS_ARN" \
  --protocol email \
  --notification-endpoint your-email@example.com

# You must confirm the subscription via the email AWS sends before alerts will fire
```

To wire the budget to SNS instead of (or in addition to) direct email, use `"SubscriptionType": "SNS"` and set `"Address"` to the SNS topic ARN in your notifications JSON.

---

### Recommended Budget Structure for This Project

| Budget Name | Amount | Alert Thresholds |
|---|---|---|
| `learning-cluster-monthly` | $50 | 50%, 80%, 100% actual |
| `learning-cluster-forecast` | $50 | 100% forecasted |

Set up both: one for actual spend and one for forecasted spend. The forecast alert fires before you hit the limit.

---

## 4. Cost Tagging Strategy

### Why Tags Are Essential

AWS does not automatically know which charges belong to which project. Without tags, your bill is a single number. With tags, you can answer:

- "How much did the Node.js service cost this month vs. the Python service?"
- "What does the dev environment cost vs. staging?"
- "Which team's workloads drove the EC2 overage?"

Tags are key-value pairs you attach to AWS resources. They have no effect on functionality — they are purely for organization and billing.

---

### Recommended Tag Schema for This Project

```
Environment = dev | staging | prod
Project     = learning-cluster
Team        = your-name (or team name)
Service     = node-hono | python-fastapi | postgres | shared
ManagedBy   = eksctl | terraform | manual
```

Apply these tags consistently across every resource. A resource without tags is invisible in cost reports.

---

### Tagging EKS Node Groups

When creating or updating a managed node group with `eksctl`, add tags in the nodegroup section of your cluster config:

```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: learning-cluster
  region: us-east-1
  tags:
    Environment: dev
    Project: learning-cluster
    Team: your-name

nodeGroups:
  - name: ng-1
    instanceType: t3.medium
    desiredCapacity: 2
    tags:
      Environment: dev
      Project: learning-cluster
      Service: shared
      ManagedBy: eksctl
```

Apply with:

```bash
eksctl create cluster -f cluster-config.yaml
```

For an existing node group, update tags via CLI:

```bash
# Get the Auto Scaling Group name for your node group
ASG_NAME=$(aws eks describe-nodegroup \
  --cluster-name learning-cluster \
  --nodegroup-name ng-1 \
  --query 'nodegroup.resources.autoScalingGroups[0].name' \
  --output text)

# Tag the Auto Scaling Group (tags propagate to EC2 instances)
aws autoscaling create-or-update-tags \
  --tags \
    "ResourceId=$ASG_NAME,ResourceType=auto-scaling-group,Key=Environment,Value=dev,PropagateAtLaunch=true" \
    "ResourceId=$ASG_NAME,ResourceType=auto-scaling-group,Key=Project,Value=learning-cluster,PropagateAtLaunch=true" \
    "ResourceId=$ASG_NAME,ResourceType=auto-scaling-group,Key=Service,Value=shared,PropagateAtLaunch=true"
```

---

### Tagging ECR Repositories

```bash
# Get the repository ARN
REPO_ARN=$(aws ecr describe-repositories \
  --repository-names node-hono-app \
  --query 'repositories[0].repositoryArn' \
  --output text)

# Apply tags
aws ecr tag-resource \
  --resource-arn "$REPO_ARN" \
  --tags \
    Key=Environment,Value=dev \
    Key=Project,Value=learning-cluster \
    Key=Service,Value=node-hono
```

Repeat for each repository (e.g., `python-fastapi-app`).

---

### Enabling Cost Allocation Tags in the Billing Console

Tags do not appear in Cost Explorer automatically. You must activate them.

1. Go to **AWS Billing Console** > **Cost allocation tags**
   - URL: `https://console.aws.amazon.com/billing/home#/tags`
2. Find your tag keys in the **User-defined cost allocation tags** section
3. Select each key you want to appear in Cost Explorer: `Environment`, `Project`, `Team`, `Service`
4. Click **Activate**

It takes up to 24 hours for activated tags to appear in Cost Explorer. Tags only apply to costs incurred after activation — they do not backfill historical data.

---

### Creating a Tag-Filtered Report in Cost Explorer

1. Open Cost Explorer
2. Set date range to current month
3. Under **Filters**, click **Tag**
4. Select the tag key (e.g., `Project`) and set the value to `learning-cluster`
5. Click **Apply**
6. Under **Group by**, select `Tag: Service`

This shows a breakdown of costs per service within your project. Save this as a named report using **Save to report library** so you can return to it each month.

---

## 5. Cost Optimization Strategies

### Spot Instances: 60–90% Savings

Spot instances are unused EC2 capacity that AWS sells at a discount. The trade-off is that AWS can reclaim them with a 2-minute warning when capacity is needed.

**Potential savings on t3.medium:**
- On-demand: ~$0.0416/hr
- Spot (typical): ~$0.008–$0.015/hr
- Savings: ~60–80%

**Adding a Spot node group with eksctl:**

```yaml
# Add this to your cluster-config.yaml nodeGroups section
  - name: ng-spot
    instanceTypes:
      - t3.medium
      - t3.large
      - t3a.medium
    spot: true
    desiredCapacity: 2
    minSize: 1
    maxSize: 4
    tags:
      Environment: dev
      Project: learning-cluster
      Service: shared
```

Always specify multiple instance types for spot node groups. If `t3.medium` spot capacity is unavailable, EKS can fall back to `t3.large` or `t3a.medium`.

**What happens when spot is interrupted:**

1. AWS sends a 2-minute interruption notice
2. The node taint is updated to `node.kubernetes.io/unschedulable`
3. Kubernetes evicts pods from the node
4. The pod is rescheduled on a remaining node (if capacity exists)
5. The instance is terminated

For learning workloads, a brief interruption is acceptable. For production workloads, configure pod disruption budgets and run a mix of on-demand and spot nodes.

---

### Savings Plans vs Reserved Instances

| Option | Commitment | Flexibility | Discount | Best For |
|---|---|---|---|---|
| On-demand | None | Full | 0% | Experimentation, short-term |
| Savings Plans (Compute) | 1 or 3 year spend/hr | Instance family, region, OS | ~40–66% | Consistent workloads, any instance type |
| Savings Plans (EC2 Instance) | 1 or 3 year spend/hr | Specific family in specific region | ~40–72% | Stable instance types |
| Reserved Instances | 1 or 3 year instance | Specific instance type | ~40–72% | Predictable, unchanging workloads |

**For this learning project:** Do not purchase Savings Plans or Reserved Instances. The cluster likely does not run 24/7, and committing to a 1-year term on a learning environment rarely makes financial sense. Use on-demand or spot.

**When to consider Savings Plans:** You have been running the same workload consistently for 2+ months and have Cost Explorer data showing a stable baseline. Purchase enough to cover your baseline, leave spikes to on-demand.

---

### Right-Sizing with AWS Compute Optimizer

Compute Optimizer analyzes CloudWatch metrics and recommends whether you are over- or under-provisioned.

```bash
# Enable Compute Optimizer for your account (one-time setup)
aws compute-optimizer update-enrollment-status \
  --status Active

# Get recommendations for your EC2 instances
aws compute-optimizer get-ec2-instance-recommendations \
  --region us-east-1 \
  --query 'instanceRecommendations[*].{
    Instance:instanceArn,
    Current:currentInstanceType,
    Recommended:recommendationOptions[0].instanceType,
    Savings:recommendationOptions[0].estimatedMonthlySavings.value
  }'
```

If Compute Optimizer recommends downsizing from `t3.medium` to `t3.small`, it means your actual CPU and memory utilization is consistently low. For a learning workload with two services and light traffic, this is likely.

---

### Deleting Idle Resources

The number one cause of unexpected AWS bills is idle resources. Resources that are not running but still cost money:

| Resource | Cost When Idle | How to Check |
|---|---|---|
| ELB | $0.008/hr base charge | AWS Console > EC2 > Load Balancers |
| EBS volumes (unattached) | $0.10/GB/month | AWS Console > EC2 > Volumes, filter "available" |
| Elastic IPs (unassociated) | $0.005/hr | AWS Console > EC2 > Elastic IPs |
| NAT Gateway | $0.045/hr + $0.045/GB | AWS Console > VPC > NAT Gateways |
| RDS (stopped) | Storage only | Stopped instances resume auto-starting after 7 days |

**Check for idle resources with CLI:**

```bash
# Unattached EBS volumes
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].{ID:VolumeId,Size:Size,Created:CreateTime}' \
  --region us-east-1

# Unassociated Elastic IPs
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==null].{AllocationId:AllocationId,IP:PublicIp}' \
  --region us-east-1

# Load balancers with no registered targets
aws elbv2 describe-target-groups \
  --query 'TargetGroups[*].{ARN:TargetGroupArn,Name:TargetGroupName,LB:LoadBalancerArns}' \
  --region us-east-1
```

---

### Scaling Down During Off-Hours

If you are not using the cluster overnight or on weekends, scaling down the node group to zero saves ~100% of EC2 and EBS costs during that period. The EKS control plane still charges $0.10/hr regardless.

```bash
# Scale node group to 0 (stops EC2 billing, EKS control plane still charges)
aws eks update-nodegroup-config \
  --cluster-name learning-cluster \
  --nodegroup-name ng-1 \
  --scaling-config minSize=0,maxSize=2,desiredSize=0 \
  --region us-east-1

# Scale back up when needed
aws eks update-nodegroup-config \
  --cluster-name learning-cluster \
  --nodegroup-name ng-1 \
  --scaling-config minSize=1,maxSize=2,desiredSize=2 \
  --region us-east-1
```

Alternatively, use `eksctl scale nodegroup`:

```bash
# Scale down
eksctl scale nodegroup \
  --cluster learning-cluster \
  --name ng-1 \
  --nodes 0 \
  --nodes-min 0 \
  --region us-east-1

# Scale back up
eksctl scale nodegroup \
  --cluster learning-cluster \
  --name ng-1 \
  --nodes 2 \
  --nodes-min 1 \
  --region us-east-1
```

For automated scale-down, consider AWS EventBridge rules that trigger Lambda functions on a schedule to call these CLI commands.

---

### Karpenter for Efficient Bin-Packing

Karpenter is an open-source cluster autoscaler built by AWS that can replace the default Kubernetes cluster autoscaler. Its key advantage is **consolidation**: it continuously monitors whether pods can be moved to fewer nodes, then terminates the emptied nodes.

Standard cluster autoscaler: scales out when pods are pending, but is slow to scale in.

Karpenter consolidation: actively moves pods from underutilized nodes to pack them onto fewer nodes, terminating the empties within minutes.

For a learning environment with two services, consolidation can reduce your node count from 2 to 1 during quiet periods, halving your EC2 cost.

Basic Karpenter NodePool configuration with consolidation enabled:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["t3.medium", "t3.large", "t3a.medium"]
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
```

`consolidateAfter: 30s` means Karpenter will attempt to consolidate as soon as a node has been underutilized for 30 seconds. Be cautious with very aggressive values in production — use `300s` or longer for workloads sensitive to pod restarts.

---

## 6. The "Leave It Running" Risk

### Cost Per Hour/Day/Month If You Forget

| Scenario | Hourly | Daily | Monthly |
|---|---|---|---|
| Full cluster (2 nodes + EKS + ALB) | ~$0.19 | ~$4.80 | ~$144 |
| Cluster deleted but ALB left running | $0.008 | $0.19 | $5.76 |
| Cluster deleted but NAT Gateway left | $0.045 | $1.08 | $32.40 |
| EKS control plane only (no nodes) | $0.10 | $2.40 | $72.00 |
| 2 idle EC2 t3.medium (no cluster) | $0.083 | $2.00 | $60.00 |
| Unattached 100 GB EBS volume | — | $0.33 | $10.00 |

A two-week vacation with the cluster left running: `$4.80 x 14 = $67.20`.

---

### The Load Balancer Problem

This is the most common mistake when deleting an EKS cluster.

When Kubernetes provisions an ELB via a `LoadBalancer` service, the load balancer is created in your AWS account as a separate resource. It is associated with the cluster by tags, but it is not owned by the cluster.

**What happens if you run `eksctl delete cluster` without cleaning up:**

1. `eksctl` deletes the EKS cluster and node group
2. The EC2 instances are terminated — EC2 billing stops
3. The ELB, however, was created by the in-cluster AWS Load Balancer Controller
4. `eksctl` does not always clean up ELBs created by in-cluster controllers
5. **The ELB continues running and charging $0.008/hr indefinitely**

**Safe cluster deletion procedure:**

```bash
# Step 1: Delete Kubernetes LoadBalancer services first
kubectl delete svc --all -n default
kubectl delete svc --all -n your-app-namespace

# Step 2: Wait for ELBs to be deprovisioned (30–60 seconds)
# Verify in console: EC2 > Load Balancers — should show no active LBs for this cluster

# Step 3: Delete the cluster
eksctl delete cluster --name learning-cluster --region us-east-1

# Step 4: Verify no orphaned ELBs remain
aws elbv2 describe-load-balancers \
  --region us-east-1 \
  --query 'LoadBalancers[*].{Name:LoadBalancerName,State:State.Code,DNS:DNSName}'
```

---

### CloudWatch Alarm for Unexpected Spend

Budget alerts fire based on daily billing data updates. For faster detection, set a CloudWatch alarm on the `EstimatedCharges` metric.

```bash
# Create an alarm that fires if estimated charges exceed $10
# (useful for catching a runaway resource early in the billing cycle)
aws cloudwatch put-metric-alarm \
  --alarm-name "learning-cluster-spend-alarm" \
  --alarm-description "Alert when estimated AWS charges exceed $10" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 86400 \
  --evaluation-periods 1 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=Currency,Value=USD \
  --alarm-actions "$SNS_ARN" \
  --region us-east-1
```

Note: `EstimatedCharges` billing metrics are only published in `us-east-1`, regardless of where your resources run. Always use `--region us-east-1` for this alarm.

AWS updates billing metrics approximately every 6–8 hours. This alarm will not catch charges in real time, but it is faster than waiting for the monthly bill.

---

## 7. Quick Cost Check Commands

### Current Month Spend (Total)

```bash
aws ce get-cost-and-usage \
  --time-period Start=$(date -u +%Y-%m-01),End=$(date -u +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --region us-east-1 \
  --query 'ResultsByTime[0].Total.BlendedCost.{Amount:Amount,Unit:Unit}'
```

---

### Current Month Spend — Broken Down by Service

```bash
aws ce get-cost-and-usage \
  --time-period Start=$(date -u +%Y-%m-01),End=$(date -u +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --region us-east-1 \
  --query 'ResultsByTime[0].Groups[*].{Service:Keys[0],Cost:Metrics.BlendedCost.Amount}' \
  --output table
```

---

### Today's Spend Only

```bash
aws ce get-cost-and-usage \
  --time-period Start=$(date -u +%Y-%m-%d),End=$(date -u -d "+1 day" +%Y-%m-%d 2>/dev/null || date -u -v+1d +%Y-%m-%d) \
  --granularity DAILY \
  --metrics BlendedCost \
  --region us-east-1 \
  --query 'ResultsByTime[0].Total.BlendedCost.{Amount:Amount,Unit:Unit}'
```

> Note: Cost Explorer data lags by approximately 24 hours. Today's charges may not appear until tomorrow.

---

### EKS-Specific Costs Only

```bash
aws ce get-cost-and-usage \
  --time-period Start=$(date -u +%Y-%m-01),End=$(date -u +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --filter '{
    "Dimensions": {
      "Key": "SERVICE",
      "Values": [
        "Amazon Elastic Kubernetes Service",
        "Amazon EC2",
        "Elastic Load Balancing",
        "Amazon EC2 Container Registry (ECR)"
      ]
    }
  }' \
  --group-by Type=DIMENSION,Key=SERVICE \
  --region us-east-1 \
  --query 'ResultsByTime[0].Groups[*].{Service:Keys[0],Cost:Metrics.BlendedCost.Amount}' \
  --output table
```

---

### Check Running EC2 Instances (Quick Sanity Check)

```bash
# See all running instances and their types
aws ec2 describe-instances \
  --filters Name=instance-state-name,Values=running \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,Type:InstanceType,Launch:LaunchTime,Name:Tags[?Key==`Name`]|[0].Value}' \
  --region us-east-1 \
  --output table
```

---

### Check for Orphaned Load Balancers

```bash
# List all ALBs/NLBs
aws elbv2 describe-load-balancers \
  --region us-east-1 \
  --query 'LoadBalancers[*].{Name:LoadBalancerName,Type:Type,State:State.Code,Created:CreatedTime}' \
  --output table

# List all Classic Load Balancers
aws elb describe-load-balancers \
  --region us-east-1 \
  --query 'LoadBalancerDescriptions[*].{Name:LoadBalancerName,Created:CreatedTime}' \
  --output table
```

---

### Check NAT Gateways (Expensive if Left Running)

```bash
aws ec2 describe-nat-gateways \
  --filter Name=state,Values=available \
  --query 'NatGateways[*].{ID:NatGatewayId,VPC:VpcId,State:State,Created:CreateTime}' \
  --region us-east-1 \
  --output table
```

---

### Projected Month-End Cost

```bash
aws ce get-cost-forecast \
  --time-period Start=$(date -u +%Y-%m-%d),End=$(date -u +%Y-%m-01 | xargs -I{} date -d "{} +1 month" +%Y-%m-01 2>/dev/null || date -u +%Y-%m-01 | xargs -I{} date -j -v+1m -f %Y-%m-%d {} +%Y-%m-%d) \
  --granularity MONTHLY \
  --metric BLENDED_COST \
  --region us-east-1 \
  --query 'Total.{MeanForecast:MeanValue,LowerBound:PredictionIntervalLowerBound,UpperBound:PredictionIntervalUpperBound}'
```

---

## Summary: The Most Important Actions for This Project

| Priority | Action | Why |
|---|---|---|
| 1 | Set up a $50 monthly budget with email alerts | Catches runaway costs before they compound |
| 2 | Always delete `LoadBalancer` services before deleting the cluster | Prevents orphaned ELBs that cost money indefinitely |
| 3 | Check Cost Explorer weekly | Spot trends before they become surprises |
| 4 | Scale nodes to 0 when not learning | Saves ~$60/month in EC2 costs |
| 5 | Enable cost allocation tags | Makes the bill readable and attributable |
| 6 | Run the quick cost check commands before and after each session | Confirms you did not accidentally leave something running |

The EKS control plane charges $0.10/hr whether or not any pods are scheduled. If you are done with the cluster for more than a few days, delete it entirely and recreate it when needed. Cluster creation with `eksctl` takes 15–20 minutes and is fully repeatable if you have your config files in version control.
