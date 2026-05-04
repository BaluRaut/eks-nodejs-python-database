# The "Quick Restart" Guide: How to rebuild your cluster in 15 minutes

If you have deleted your cluster to save money and want to start learning again, follow these exact steps to get back to where you were.

---

## 1. Prepare your environment
Open your terminal and make sure you are in the project folder:
```bash
cd /Users/balasahebraut/learning/aws-eks-ecr-postgres-node
```

---

## 2. Re-create the EKS Cluster
This is the longest step (takes ~15-20 minutes). It builds the servers and the "brain" of your cluster.

```bash
eksctl create cluster \
  --name learning-cluster \
  --region us-east-1 \
  --nodegroup-name standard-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --managed
```

---

## 3. Re-deploy the Database & Apps
Once the cluster is "Ready", run this one command to launch everything:

```bash
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/multi-api.yaml
```

---

## 4. Verify everything is running
Check your pods:
```bash
kubectl get pods
```
*Wait until all pods say `Running`. If you see `ErrImagePull`, make sure you haven't deleted your ECR repositories!*

---

## 5. Get your New URLs
Since we deleted the old LoadBalancers, AWS has given you new addresses. Run this to find them:

```bash
kubectl get svc
```
Look for the **EXTERNAL-IP** for `node-hono-service` and `python-fastapi-service`.

---

## 6. Pro Tip: Connecting PGAdmin again
Every time you restart the cluster, you must restart the "Tunnel" on your Mac:

```bash
kubectl port-forward svc/postgres-service 5433:5432
```
*Note: Your database will be empty again! Your Node.js app will automatically create the `users` table, but any other data you added manually will be gone (because we are using "Ephemeral" storage for this learning project).*

---

### ⚠️ Don't forget to delete again when done!
```bash
eksctl delete cluster --name learning-cluster --region us-east-1
```

**Happy Learning!** 🚀
