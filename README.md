# AWS EKS/ECR Learning Path (Node.js + Postgres)

This project is a step-by-step guide to deploying a Node.js application with a Postgres database to AWS EKS (Elastic Kubernetes Service) using ECR (Elastic Container Registry).

## Goal
Learn how to:
1. Containerize a Node.js application.
2. Push images to AWS ECR.
3. Deploy a Postgres database on Kubernetes (or use AWS RDS).
4. Deploy the Node.js app to EKS.
5. Automate the process with CircleCI/GitHub.

## Cost Warning (AWS)
- **EKS Cluster:** ~$0.10 per hour ($72/month). Always delete the cluster when not in use!
- **ECR:** Very low cost for storage.
- **NAT Gateway:** If used, can be expensive (~$32/month). We will try to avoid this by using public subnets for learning.

## Roadmap
1. [x] **Step 1: Local Node.js App & Docker** - Build and run locally.
2. [x] **Step 2: AWS Setup** - Configure CLI and ECR.
3. [x] **Step 3: EKS Cluster Creation** - Create a minimal cluster.
4. [x] **Step 4: Kubernetes Deployment** - Deploy Postgres and Node.js.
5. [x] **Step 5: CI/CD with CircleCI** - Automate the build and deploy.

## Current Progress
- [x] Step 1: Local Node.js App & Docker
- [x] Step 2: AWS Setup & ECR
- [x] Step 3: EKS Cluster Setup
- [x] Step 4: Kubernetes Deployment
- [x] Step 5: CI/CD with CircleCI
