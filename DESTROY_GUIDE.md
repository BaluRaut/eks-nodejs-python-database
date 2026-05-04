# The "Kill Switch": Safely Destroying Your AWS EKS Resources

To avoid unexpected charges from AWS, you must ensure that all resources are completely removed. While `eksctl delete cluster` does 90% of the work, follow this checklist to be 100% sure.

---

## 1. The Main Command (Destroys 90%)
This command removes the EKS Control Plane, the EC2 Worker Nodes, and the VPC.
```bash
eksctl delete cluster --name learning-cluster --region us-east-1
```
*Note: This takes about 10–15 minutes.*

---

## 2. Cleanup ECR Repositories (The "Storage")
AWS ECR repositories are not deleted by the command above. They cost a small amount for storage. To delete them:

```bash
# Delete the Node.js repository
aws ecr delete-repository --repository-name node-app --region us-east-1 --force

# Delete the Hono repository
aws ecr delete-repository --repository-name node-hono --region us-east-1 --force

# Delete the Python repository
aws ecr delete-repository --repository-name python-app --region us-east-1 --force
```

---

## 3. Verify Load Balancers (The "Orphans")
Sometimes AWS Load Balancers (ELB) get stuck if the cluster is deleted while they are still active. Check for any remaining ones:

```bash
aws elb describe-load-balancers --query "LoadBalancerDescriptions[*].LoadBalancerName"
```
**If it returns `[]`, you are safe.** If you see a name in the list, delete it via the AWS Console (EC2 > Load Balancers).

---

## 4. Local Environment Cleanup
Clean up your local `kubectl` configuration so it doesn't try to connect to a dead cluster:

```bash
# List your contexts
kubectl config get-contexts

# Delete the specific context for this cluster
kubectl config delete-context arn:aws:eks:us-east-1:425848620191:cluster/learning-cluster
```

---

## 5. Final AWS Console Check (The "Pro" Verification)
Log into the AWS Console and check these 3 places manually to ensure they are empty:
1. **CloudFormation:** Ensure stacks starting with `eksctl-learning-cluster` are gone.
2. **EC2 > Instances:** Ensure no instances are "Running".
3. **EKS > Clusters:** Ensure the list is empty.

---

### 💡 Why do we do this?
In the cloud, you only pay for what you **use**. By destroying these resources, you "Turn off the lights" and stop the billing clock immediately.
