# CircleCI CI/CD Guide for This Project

A practical, beginner-friendly guide to understanding and operating the CI/CD pipeline in this repository. Every concept is paired with a real-world analogy before the technical detail.

---

## Table of Contents

1. [What is CI/CD?](#1-what-is-cicd)
2. [CircleCI Concepts with Analogies](#2-circleci-concepts-with-analogies)
3. [The .circleci/config.yml Deep Dive](#3-the-circleciconfigyml-deep-dive)
4. [Git Tagging Strategy](#4-git-tagging-strategy)
5. [Setting Up the CircleCI Context (aws-context)](#5-setting-up-the-circleci-context-aws-context)
6. [Adding Tests to the Pipeline](#6-adding-tests-to-the-pipeline)
7. [Approval Gates for Production](#7-approval-gates-for-production)
8. [Rollback Strategy](#8-rollback-strategy)

---

## 1. What is CI/CD?

### The Assembly Line Analogy

Before modern car factories existed, each car was built by a single craftsman from start to finish. Every car was slightly different. A craftsman might forget a bolt, measure a part wrong, or deliver a car that looked identical to yesterday's but mysteriously broke after a week. There was no consistent process.

Modern car manufacturing replaced that with an assembly line: standardized stations, automated checks, and a conveyor belt that moves the product forward only when each step passes inspection.

Software deployment has the same problem that hand-crafted cars had. Without automation:

- A developer runs `docker build` on their laptop, which has slightly different software installed than the server.
- They manually SSH into the production server and run commands from memory.
- They forget one step. Or they run the right steps in the wrong order.
- A bug ships that a five-second automated test would have caught.

**CI/CD is the assembly line for your software.**

### Continuous Integration (CI)

**Analogy:** The quality-control station at the entrance of the factory floor. Before any part is allowed onto the line, a machine checks it: is this bolt the right thread? Is this panel the right gauge of steel? Parts that fail the check never make it onto the assembly line.

In software, CI means: every time a developer pushes code, an automated system immediately runs checks — linting, unit tests, build verification. If those checks fail, the code never reaches deployment. The problem is caught at the source, not in production.

### Continuous Deployment (CD)

**Analogy:** The conveyor belt that, once a car passes every quality check, automatically moves it to the loading dock and ships it to the dealership. No human has to manually load the truck. The process is deterministic and repeatable.

In software, CD means: once the CI checks pass, the system automatically packages the application (builds a Docker image), stores it in a registry (ECR), and tells Kubernetes to update the running service with the new version.

### How This Repository Uses CI/CD

The flow in this project, from a single command on your terminal to a live update in EKS:

```
git tag node-v1.1
git push origin node-v1.1
       |
       v
CircleCI detects the new tag
       |
       v
Job 1: aws-ecr/build-and-push-image
  - Clones the repo
  - Runs: docker build --platform linux/amd64 ./node-hono
  - Logs into ECR: 425848620191.dkr.ecr.us-east-1.amazonaws.com
  - Pushes image tagged as node-hono:node-v1.1
       |
       v
Job 2: deploy-k8s  (only runs after Job 1 succeeds)
  - Configures kubectl to talk to the learning-cluster EKS cluster
  - Runs: kubectl set image deployment/node-hono node-hono=425848620191.dkr.ecr.us-east-1.amazonaws.com/node-hono:node-v1.1
  - Kubernetes performs a rolling update: new pods start before old pods stop
  - CircleCI watches the rollout and reports success or failure
```

No SSH. No manual `docker build`. No human error in the deployment commands. The same sequence runs identically every time.

---

## 2. CircleCI Concepts with Analogies

### Pipeline

**Analogy:** The entire factory floor. When you push code (or a tag) to GitHub, the factory starts up. Everything that happens from that moment until the product ships or fails inspection is the pipeline.

Technically: a pipeline is the top-level object in CircleCI. It is triggered by a push event and contains all workflows defined in `.circleci/config.yml`. One push = one pipeline run.

### Workflow

**Analogy:** A sequence of departments the product passes through. A car passes through the frame assembly department, then the paint department, then the interior department. Departments can run in parallel (two paint booths painting simultaneously) or in sequence (interior only starts after paint is dry).

Technically: a workflow is a named list of jobs with ordering rules (`requires`) and conditions (`filters`). This project has three workflows: `node-deploy`, `python-deploy`, and `postgres-deploy`. Each workflow runs independently and only when the right tag pattern is detected.

### Job

**Analogy:** One department. The paint department. It has its own team, its own tools, and one clear responsibility. When the car enters, it leaves painted. If the paint fails quality control, the car does not move to the next department.

Technically: a job is a named unit of work. It runs on a specific executor (a machine or container) and contains a list of steps. Jobs in this project: `build-push-node`, `deploy-node`, `build-push-python`, `deploy-python`, `deploy-postgres`.

### Step

**Analogy:** One action performed by a worker inside a department. In the paint department: (1) sand the surface, (2) apply primer, (3) apply color coat, (4) inspect. Each action is distinct and ordered.

Technically: a step is a single instruction inside a job. It is either a `run` step (a shell command) or a named orb step (like `aws-eks/update-kubeconfig-with-authenticator`). If any step fails (exits with a non-zero code), the job fails and subsequent jobs in the workflow do not run.

### Executor

**Analogy:** The workbench where the job runs. Some jobs need a heavy industrial press (a full virtual machine). Some jobs only need a small, clean table (a lightweight Docker container). You choose the right workbench for the task.

Technically: an executor defines the environment where a job runs. This project's deploy jobs use `docker: cimg/base:stable` — a minimal Ubuntu-based container. The ECR build jobs use the executor defined inside the `aws-ecr` orb, which includes Docker-in-Docker capabilities needed to build and push images.

### Orbs

**Analogy:** Pre-built machinery you lease instead of fabricate yourself. Instead of welding your own robotic arm for windshield installation, you rent a proven commercial one. It does the job correctly, it is already tested, and if the manufacturer improves it you get the update.

Technically: orbs are reusable packages of CircleCI configuration published to the CircleCI registry. They bundle jobs, commands, and executors that other teams have already written and tested. This project uses:

| Orb | Version | What it does |
|-----|---------|--------------|
| `circleci/aws-ecr` | 9.0.2 | Builds a Docker image and pushes it to Amazon ECR, handling ECR authentication automatically |
| `circleci/aws-eks` | 2.2.0 | Configures `kubectl` to authenticate with an EKS cluster using AWS credentials |
| `circleci/kubernetes` | 1.3.1 | Provides the `kubernetes/update-container-image` command that runs `kubectl set image` and waits for the rollout |

Without orbs, you would write 50-100 lines of shell script to do what each orb does in 3-4 lines of YAML.

### Context

**Analogy:** The factory's secure key cabinet. Workers need access to the building, the paint supply room, and the machinery — but you do not write those access codes on a whiteboard in the break room. They are stored in a locked cabinet. Workers are granted access to the cabinet for their shift, they use the keys, and the keys go back in the cabinet when they're done.

Technically: a CircleCI Context is a named group of environment variables stored encrypted in CircleCI's servers. Jobs that declare `context: aws-context` receive those variables injected as environment variables at runtime. The values are never visible in logs or config files. This project's `aws-context` context stores the AWS credentials that allow CircleCI to push to ECR and update EKS.

---

## 3. The .circleci/config.yml Deep Dive

Here is the complete config with line-by-line annotations, followed by a discussion of the most important decisions.

### Full Annotated Config

```yaml
version: 2.1
# version: 2.1 is required to use orbs. Never use version 2 if you want orbs.

orbs:
  # These three lines "import" external orb packages.
  # Think of it like: import boto3 in Python, or require('express') in Node.
  # The alias on the left (aws-ecr, aws-eks, kubernetes) is how you reference
  # the orb's jobs and commands later in this file.
  aws-ecr: circleci/aws-ecr@9.0.2
  aws-eks: circleci/aws-eks@2.2.0
  kubernetes: circleci/kubernetes@1.3.1

jobs:
  # --------------------------------------------------------------------------
  # deploy-k8s: A reusable job that updates a Kubernetes deployment.
  # It is called twice: once for node-hono, once for python-fastapi.
  # Using parameters makes it generic so you don't repeat the same logic twice.
  # --------------------------------------------------------------------------
  deploy-k8s:
    parameters:
      deployment-name:
        type: string        # e.g., "node-hono"
      container-name:
        type: string        # e.g., "node-hono" (must match the container name in the YAML spec)
      image-url:
        type: string        # The full ECR image URL including the tag
    docker:
      - image: cimg/base:stable
      # cimg/base:stable is a small, stable Ubuntu image maintained by CircleCI.
      # It is enough for running kubectl commands. We do not need a full VM here.
    steps:
      - checkout
        # Downloads the source code from the repository into the job's workspace.
        # This is needed because the deploy-postgres job applies YAML files from the repo.

      - aws-eks/update-kubeconfig-with-authenticator:
          cluster-name: learning-cluster
          install-kubectl: true
          # This orb step does two things:
          #   1. Downloads and installs the aws-eks-auth binary (needed for EKS authentication)
          #   2. Runs: aws eks update-kubeconfig --name learning-cluster
          #      which writes a kubeconfig entry so that subsequent kubectl commands
          #      are authenticated and pointed at the right cluster.
          # The AWS credentials from the context (AWS_ACCESS_KEY_ID, etc.) are what
          # authorize this call to the AWS API.

      - kubernetes/update-container-image:
          container-image-updates: << parameters.container-name >>=<< parameters.image-url >>
          deployment-name: << parameters.deployment-name >>
          get-rollout-status: true
          # This orb step runs the equivalent of:
          #   kubectl set image deployment/<deployment-name> <container-name>=<image-url>
          #
          # "kubectl set image" triggers a Kubernetes rolling update:
          #   - Kubernetes starts a new pod with the new image
          #   - Waits for the new pod to be healthy (readiness probe passes)
          #   - Then terminates one old pod
          #   - Repeats until all pods are running the new image
          #   - At no point is the service completely down (zero-downtime deploy)
          #
          # get-rollout-status: true means the step also runs:
          #   kubectl rollout status deployment/<deployment-name>
          # This watches the rollout until it completes (or fails), so CircleCI
          # reports the correct pass/fail result instead of exiting immediately.

  # --------------------------------------------------------------------------
  # deploy-postgres: Applies the postgres Kubernetes manifest directly.
  # Postgres is infrastructure, not application code, so we just apply the YAML
  # rather than building a custom image and rolling it out via tags.
  # --------------------------------------------------------------------------
  deploy-postgres:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - aws-eks/update-kubeconfig-with-authenticator:
          cluster-name: learning-cluster
          install-kubectl: true
      - run:
          name: Deploy Postgres
          command: kubectl apply -f k8s/postgres.yaml
          # kubectl apply -f is idempotent: if the resource already exists,
          # it updates it with any changes. If it does not exist, it creates it.

workflows:
  version: 2

  # ============================================================
  # Workflow 1: node-deploy
  # Triggers only when a tag matching /^node-v.*/ is pushed.
  # Example tags that trigger this: node-v1.0, node-v1.1, node-v2.0-beta
  # Example tags that do NOT trigger this: python-v1.0, v1.0, node-1.0 (no v)
  # ============================================================
  node-deploy:
    jobs:
      - aws-ecr/build-and-push-image:
          # This is a job provided by the aws-ecr orb.
          # It builds a Docker image and pushes it to ECR.
          name: build-push-node
          context: aws-context
          # Injects AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_DEFAULT_REGION,
          # AWS_ACCOUNT_ID, and AWS_REGION as environment variables.

          account-id: ${AWS_ACCOUNT_ID}
          # Resolves to 425848620191 at runtime from the context.
          # Used to construct the ECR registry URL:
          #   425848620191.dkr.ecr.us-east-1.amazonaws.com

          region: ${AWS_REGION}
          # The AWS region where your ECR repository lives: us-east-1

          repo: node-hono
          # The ECR repository name. The full image path becomes:
          #   425848620191.dkr.ecr.us-east-1.amazonaws.com/node-hono

          tag: << pipeline.git.tag >>
          # << pipeline.git.tag >> is a CircleCI pipeline value that resolves
          # to the git tag that triggered this pipeline.
          # If you pushed "node-v1.1", this becomes "node-v1.1".
          # The final image stored in ECR becomes:
          #   425848620191.dkr.ecr.us-east-1.amazonaws.com/node-hono:node-v1.1

          path: node-hono
          # The build context directory passed to docker build.
          # Equivalent to: docker build ./node-hono
          # The Dockerfile at node-hono/Dockerfile is used.

          platform: linux/amd64
          # Forces the image to be built for the x86_64 architecture.
          # Critical if you are developing on an Apple Silicon (M1/M2) Mac,
          # which uses arm64 by default. EKS nodes run on x86_64 (amd64).
          # Without this flag, your image would fail to run on EKS nodes
          # with an "exec format error".
          # Internally uses: docker buildx build --platform linux/amd64

          filters:
            branches:
              ignore: /.*/
              # ignore: /.*/ means "ignore ALL branch pushes".
              # This job will NOT run when you push to main, feature branches, etc.
              # This is intentional: we only deploy when a tag is pushed.
            tags:
              only: /^node-v.*/
              # only: /^node-v.*/ is a regular expression.
              # ^ means "starts with"
              # node-v means the literal string "node-v"
              # .* means "followed by anything"
              # So: tags starting with "node-v" trigger this job.
              # node-v1.0 YES | node-v2.0-hotfix YES | python-v1.0 NO | v1.0 NO

      - deploy-k8s:
          name: deploy-node
          context: aws-context
          deployment-name: node-hono
          # Must match the deployment name in k8s/multi-api.yaml:
          #   metadata: name: node-hono
          container-name: node-hono
          # Must match the container name inside that deployment spec:
          #   containers: - name: node-hono
          image-url: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/node-hono:<< pipeline.git.tag >>
          # For tag node-v1.1, this resolves to:
          #   425848620191.dkr.ecr.us-east-1.amazonaws.com/node-hono:node-v1.1
          requires:
            - build-push-node
            # This job only starts after build-push-node finishes successfully.
            # If the build fails, the deploy never runs. The assembly line stops.
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^node-v.*/
              # The filters must be repeated on every job in a workflow that
              # involves tag-based filtering. This is a CircleCI requirement.
              # If you omit filters from a downstream job, CircleCI will try to
              # run it on every branch push and fail because pipeline.git.tag
              # would be empty.

  # ============================================================
  # Workflow 2: python-deploy
  # Identical structure to node-deploy, different service.
  # Triggers only on tags matching /^python-v.*/
  # ============================================================
  python-deploy:
    jobs:
      - aws-ecr/build-and-push-image:
          name: build-push-python
          context: aws-context
          account-id: ${AWS_ACCOUNT_ID}
          region: ${AWS_REGION}
          repo: python-app
          # ECR repo name for the Python service: python-app
          # Full image: 425848620191.dkr.ecr.us-east-1.amazonaws.com/python-app
          tag: << pipeline.git.tag >>
          path: python-fastapi
          # Build context: the python-fastapi/ directory and its Dockerfile
          platform: linux/amd64
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^python-v.*/

      - deploy-k8s:
          name: deploy-python
          context: aws-context
          deployment-name: python-fastapi
          container-name: python-fastapi
          image-url: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/python-app:<< pipeline.git.tag >>
          requires:
            - build-push-python
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^python-v.*/

  # ============================================================
  # Workflow 3: postgres-deploy
  # Triggers on tags matching /^postgres-v.*/
  # Applies the postgres.yaml manifest. Does not build an image.
  # ============================================================
  postgres-deploy:
    jobs:
      - deploy-postgres:
          context: aws-context
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^postgres-v.*/
```

### How the Tag Version Becomes the Image Tag

This is the key mechanism that ties git to ECR to Kubernetes.

When you run:
```bash
git tag node-v1.1
git push origin node-v1.1
```

CircleCI receives the event and exposes the value `node-v1.1` as `<< pipeline.git.tag >>`. Every reference to that value in the config expands to the literal string `node-v1.1` during the pipeline run.

The image built and pushed to ECR is:
```
425848620191.dkr.ecr.us-east-1.amazonaws.com/node-hono:node-v1.1
```

The `kubectl set image` command that Kubernetes receives is:
```
node-hono=425848620191.dkr.ecr.us-east-1.amazonaws.com/node-hono:node-v1.1
```

This means every deployed version of your application is traceable. You can look at what is running in EKS, read the image tag (`node-v1.1`), and find the exact commit in your git history that produced it. There is a direct, unambiguous chain: `git tag -> CircleCI pipeline -> ECR image -> Kubernetes pod`.

### The `platform: linux/amd64` Decision

The node-hono `Dockerfile` starts with `FROM node:18-slim`. The python-fastapi `Dockerfile` starts with `FROM python:3.9-slim`. Both base images are available for multiple CPU architectures.

If you are on an Apple Silicon Mac (M1, M2, M3), Docker defaults to building for `linux/arm64`. EKS worker nodes (unless you specifically configure ARM instances) run on Intel/AMD hardware: `linux/amd64`. An arm64 image on an amd64 node produces:

```
standard_init_linux.go:228: exec user process caused: exec format error
```

The `platform: linux/amd64` flag in the orb forces Docker to cross-compile for the correct architecture regardless of the machine running the build. CircleCI machines are typically amd64, so in practice this flag is a safety guarantee more than a requirement in this environment.

---

## 4. Git Tagging Strategy

### Why Tags Instead of Every Commit?

**Analogy:** A car factory does not ship every car the moment it comes off the line, including the test runs, the half-finished models used to validate new equipment, and the cars built during the break-in period when a new machine is being calibrated. The factory decides which cars are ready to ship. Tags are that decision.

If you triggered deployment on every commit to `main`:
- A commit that says "fix typo in README" would trigger a full build and deploy cycle.
- A commit that says "WIP: half-implemented feature" (that somehow got merged) would deploy broken code.
- Multiple engineers committing in rapid succession would create a queue of deployments, each overwriting the last, with no human decision point in between.

Tags represent an explicit human decision: "I am releasing this. I have decided it is ready."

### Semantic Versioning

Semantic versioning (SemVer) is the convention: `MAJOR.MINOR.PATCH`.

| Component | When to increment | Example |
|-----------|------------------|---------|
| `MAJOR` | Breaking change — existing users must change how they use the API | `1.0.0` -> `2.0.0` |
| `MINOR` | New feature added, fully backward compatible | `1.0.0` -> `1.1.0` |
| `PATCH` | Bug fix, no new features, fully backward compatible | `1.0.0` -> `1.0.1` |

This project uses a simplified two-part version (`v1.0`, `v1.1`) which is common for learning environments. For production systems, the full three-part `v1.0.0` format is recommended.

Tag format in this project: `<service>-v<version>`

| Tag | Triggers | Deploys |
|-----|----------|---------|
| `node-v1.0` | `node-deploy` workflow | Node Hono service |
| `node-v1.1` | `node-deploy` workflow | Node Hono service |
| `python-v1.0` | `python-deploy` workflow | Python FastAPI service |
| `python-v2.0` | `python-deploy` workflow | Python FastAPI service |
| `postgres-v1.0` | `postgres-deploy` workflow | Postgres manifest |

### The Exact Git Commands

**Create and push a tag for Node Hono:**
```bash
# Make sure you are on the commit you want to release
git log --oneline -5    # Verify you are where you think you are

# Create a lightweight tag pointing to HEAD
git tag node-v1.1

# Push the tag to GitHub (tags are not pushed by default with git push)
git push origin node-v1.1
```

**Create and push a tag for Python FastAPI:**
```bash
git tag python-v1.1
git push origin python-v1.1
```

**Create and push a tag for Postgres:**
```bash
git tag postgres-v1.0
git push origin postgres-v1.0
```

**Create a tag on a specific past commit (not HEAD):**
```bash
# Find the commit hash
git log --oneline

# Tag that specific commit
git tag node-v1.0 a1b2c3d
git push origin node-v1.0
```

### Listing, Deleting, and Managing Tags

```bash
# List all local tags
git tag

# List tags with a filter pattern
git tag -l "node-*"

# Show the commit a tag points to
git show node-v1.1 --no-patch

# Delete a local tag (does not affect GitHub)
git tag -d node-v1.1

# Delete a tag on GitHub (remote)
git push origin --delete node-v1.1

# Delete both local and remote in one sequence
git tag -d node-v1.1
git push origin --delete node-v1.1
```

### What Happens If You Push a Bad Tag?

Scenario: you push `node-v1.2` and discover it contains a breaking bug.

**Step 1: Rollback the Kubernetes deployment immediately** (do not wait for CI/CD — see Section 8 for kubectl rollback).

**Step 2: Fix the code and create a new tag:**
```bash
# Fix the bug in your code
git add .
git commit -m "fix: correct the broken behavior introduced in v1.2"

# Tag the fix as the next patch version
git tag node-v1.3
git push origin node-v1.3
```

**Step 3 (optional): Delete the bad tag** to prevent confusion:
```bash
git tag -d node-v1.2
git push origin --delete node-v1.2
```

Note: do not reuse a version number. Do not delete `node-v1.2` and then create a new `node-v1.2` pointing at a different commit. This destroys the traceability property. Always increment the version.

---

## 5. Setting Up the CircleCI Context (aws-context)

### What Variables to Store

The `aws-context` context in this project must contain these environment variables:

| Variable | Example Value | Purpose |
|----------|--------------|---------|
| `AWS_ACCESS_KEY_ID` | `AKIAIOSFODNN7EXAMPLE` | Authenticates the CircleCI IAM user with AWS |
| `AWS_SECRET_ACCESS_KEY` | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` | The secret half of the IAM credentials |
| `AWS_DEFAULT_REGION` | `us-east-1` | Default region for AWS CLI calls |
| `AWS_REGION` | `us-east-1` | Used explicitly by the aws-ecr orb for ECR operations |
| `AWS_ACCOUNT_ID` | `425848620191` | Your AWS account ID, used to build the ECR registry URL |

`AWS_DEFAULT_REGION` and `AWS_REGION` both need to be set because different orbs and AWS SDK versions check different variable names. Setting both is safe and avoids subtle failures.

### How to Create the Context in the CircleCI UI

1. Log in to [circleci.com](https://circleci.com).
2. In the left sidebar, click your organization name.
3. Navigate to **Organization Settings** (gear icon).
4. Click **Contexts** in the left menu.
5. Click **Create Context**.
6. Name it exactly: `aws-context` (case-sensitive — it must match what is written in `config.yml`).
7. Click the new context to open it.
8. Click **Add Environment Variable** for each variable in the table above.
9. Paste the variable name in the Name field and the value in the Value field. The value is encrypted on save and cannot be read back — only overwritten.
10. Click **Save**.

To verify the context is accessible: in your CircleCI project settings, confirm the context appears under the organization that owns the project.

### Why Contexts Are Better Than Per-Project Environment Variables

CircleCI also lets you set environment variables at the project level (Project Settings > Environment Variables). Contexts are preferable for several reasons:

**Reusability:** If you have five services in five different CircleCI projects and they all deploy to the same AWS account, you store the credentials once in the context and all five projects reference `context: aws-context`. If the credentials change (key rotation), you update one place.

**Access control:** CircleCI lets you restrict which projects or which people can use a context. You can require approval before a new project can use a security-sensitive context.

**Auditability:** Context usage is logged. You can see which pipeline runs used a context and when.

**Separation of concerns:** Application-specific config (e.g., a feature flag URL) goes in project environment variables. Infrastructure credentials (AWS keys) go in a shared context.

### Minimum IAM Permissions for the CircleCI IAM User

Create a dedicated IAM user for CircleCI. Do not use root credentials or a personal IAM user. Attach a policy with these permissions at minimum:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAuthAndPush",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ECRRepositoryAccess",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:CreateRepository",
        "ecr:DescribeRepositories"
      ],
      "Resource": "arn:aws:ecr:us-east-1:425848620191:repository/*"
    },
    {
      "Sid": "EKSDescribeCluster",
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster"
      ],
      "Resource": "arn:aws:eks:us-east-1:425848620191:cluster/learning-cluster"
    }
  ]
}
```

For `kubectl` to work, the CircleCI IAM user must also be added to the EKS cluster's `aws-auth` ConfigMap. Run this on a machine with admin access to the cluster:

```bash
# Check existing aws-auth entries
kubectl describe configmap aws-auth -n kube-system

# Edit the configmap to add the CircleCI user
kubectl edit configmap aws-auth -n kube-system
```

Add an entry under `mapUsers`:
```yaml
mapUsers: |
  - userarn: arn:aws:iam::425848620191:user/circleci-deployer
    username: circleci-deployer
    groups:
      - system:masters
```

Replace `system:masters` with a more restrictive group in production. For this learning cluster, `system:masters` grants full access.

---

## 6. Adding Tests to the Pipeline

### Where Tests Fit in the Workflow

**Analogy:** Quality inspection happens before the car goes into the painting booth, not after the finished car is loaded onto the truck. You do not repaint a shipped car because a test found a dent. You find the dent before painting.

In a CI/CD pipeline: tests run before the build, or at minimum before the deploy. The order must be:

```
lint/test -> build image -> push to ECR -> deploy to Kubernetes
```

Never run tests after deployment. By then, bad code is live.

### Adding Tests for Node Hono

First, add test dependencies to `node-hono/package.json`:

```json
{
  "name": "node-hono",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "test": "jest",
    "lint": "eslint index.js"
  },
  "dependencies": {
    "@hono/node-server": "^1.10.1",
    "hono": "^4.2.7",
    "knex": "^3.1.0",
    "pg": "^8.11.3"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "eslint": "^8.0.0"
  }
}
```

Then add a test job to `.circleci/config.yml`:

```yaml
jobs:
  test-node:
    docker:
      - image: cimg/node:18.20
    working_directory: ~/project/node-hono
    steps:
      - checkout:
          path: ~/project
      - run:
          name: Install dependencies
          command: npm ci
      - run:
          name: Run linter
          command: npm run lint
      - run:
          name: Run unit tests
          command: npm test
```

Update the `node-deploy` workflow to run the test job first:

```yaml
workflows:
  node-deploy:
    jobs:
      - test-node:
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^node-v.*/

      - aws-ecr/build-and-push-image:
          name: build-push-node
          context: aws-context
          requires:
            - test-node          # Build only starts if tests pass
          account-id: ${AWS_ACCOUNT_ID}
          region: ${AWS_REGION}
          repo: node-hono
          tag: << pipeline.git.tag >>
          path: node-hono
          platform: linux/amd64
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^node-v.*/

      - deploy-k8s:
          name: deploy-node
          context: aws-context
          deployment-name: node-hono
          container-name: node-hono
          image-url: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/node-hono:<< pipeline.git.tag >>
          requires:
            - build-push-node
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^node-v.*/
```

### Adding Tests for Python FastAPI

Add a test file to the `python-fastapi/` directory:

```python
# python-fastapi/test_main.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_read_root():
    response = client.get("/")
    assert response.status_code == 200
```

Add test dependencies to `python-fastapi/requirements.txt`:

```
fastapi
uvicorn
psycopg2-binary
pytest
httpx
```

Add a test job to `.circleci/config.yml`:

```yaml
jobs:
  test-python:
    docker:
      - image: cimg/python:3.9
    working_directory: ~/project/python-fastapi
    steps:
      - checkout:
          path: ~/project
      - run:
          name: Install dependencies
          command: pip install -r requirements.txt
      - run:
          name: Run pytest
          command: pytest test_main.py -v
```

Update the `python-deploy` workflow:

```yaml
workflows:
  python-deploy:
    jobs:
      - test-python:
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^python-v.*/

      - aws-ecr/build-and-push-image:
          name: build-push-python
          context: aws-context
          requires:
            - test-python
          account-id: ${AWS_ACCOUNT_ID}
          region: ${AWS_REGION}
          repo: python-app
          tag: << pipeline.git.tag >>
          path: python-fastapi
          platform: linux/amd64
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^python-v.*/

      - deploy-k8s:
          name: deploy-python
          context: aws-context
          deployment-name: python-fastapi
          container-name: python-fastapi
          image-url: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/python-app:<< pipeline.git.tag >>
          requires:
            - build-push-python
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^python-v.*/
```

### How a Failing Test Stops the Pipeline

If `pytest` exits with a non-zero code (which it does when any test fails), the `test-python` job fails. CircleCI marks the job red and does not start `build-push-python`. The `deploy-k8s` job, which `requires: build-push-python`, also never starts. Bad code never reaches ECR. No deployment happens. The assembly line stops at the first failed inspection station.

---

## 7. Approval Gates for Production

### What an Approval Step Is

**Analogy:** In some factories, moving a car from the assembly line to the loading dock for shipment requires a sign-off from a quality manager — not just passing automated checks. The line stops and waits. A human physically walks over, inspects the car, and presses a button to authorize shipment. This is the human checkpoint.

An approval step in CircleCI is a job of type `approval`. The workflow pauses at that job and waits. A human logs into the CircleCI UI, reviews the pipeline (checks test results, looks at what changed), and clicks **Approve**. Only then does the downstream deploy job start.

This is particularly useful when deploying to a production environment, where automatic deployment on every tag is too risky.

### Example: Workflow with an Approval Gate

This example extends the `node-deploy` workflow with a staging deployment (automatic) and a production deployment (requires human approval):

```yaml
workflows:
  node-deploy:
    jobs:
      - test-node:
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^node-v.*/

      - aws-ecr/build-and-push-image:
          name: build-push-node
          context: aws-context
          requires:
            - test-node
          account-id: ${AWS_ACCOUNT_ID}
          region: ${AWS_REGION}
          repo: node-hono
          tag: << pipeline.git.tag >>
          path: node-hono
          platform: linux/amd64
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^node-v.*/

      # Staging deploys automatically after a successful build
      - deploy-k8s:
          name: deploy-node-staging
          context: aws-context
          deployment-name: node-hono-staging
          container-name: node-hono
          image-url: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/node-hono:<< pipeline.git.tag >>
          requires:
            - build-push-node
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^node-v.*/

      # A human must approve before production receives the update
      - hold-for-production-approval:
          type: approval
          requires:
            - deploy-node-staging
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^node-v.*/
          # The workflow pauses here. CircleCI sends a notification (if configured).
          # The pipeline shows a yellow "On Hold" badge.
          # Someone navigates to the pipeline in the CircleCI UI and clicks Approve.

      # Production only deploys after approval
      - deploy-k8s:
          name: deploy-node-production
          context: aws-context
          deployment-name: node-hono
          container-name: node-hono
          image-url: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/node-hono:<< pipeline.git.tag >>
          requires:
            - hold-for-production-approval
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /^node-v.*/
```

### How to Approve in the CircleCI UI

1. Navigate to the CircleCI dashboard.
2. Click on the pipeline that is paused (it shows "On Hold").
3. Click on the workflow graph.
4. Find the yellow `hold-for-production-approval` job.
5. Click it.
6. Click **Approve**.

The `deploy-node-production` job starts immediately.

### Setting a Timeout on the Approval

By default, a pipeline waits indefinitely for approval. To automatically cancel the pipeline if no one approves within a set window, add the `no_output_timeout` field. However, for approval jobs the more appropriate mechanism is to configure an inactivity timeout at the project level in CircleCI settings, or to treat an unapproved pipeline as expired and simply create a new one when you are ready to deploy.

---

## 8. Rollback Strategy

### Kubernetes Rolling Updates: How They Work and How to Undo Them

**Analogy:** Imagine a restaurant chain with 10 locations. When the menu changes, you do not close all 10 locations simultaneously, update them, and reopen. Instead, you update one location, verify customers like the new menu, then update another. If you discover the new menu has a problem, you pull it from the locations that received it and go back to the old one. No customer ever went to a closed restaurant.

Kubernetes rolling updates work the same way. When you run `kubectl set image`, Kubernetes does not kill all existing pods and start new ones. It:

1. Starts one new pod with the new image.
2. Waits for that pod to pass its readiness probe (the health check endpoint returns 200 OK).
3. Only after the new pod is healthy, terminates one old pod.
4. Repeats until all pods are on the new image.

At no point is the service down. At every point, at least some pods are serving traffic.

Kubernetes also keeps a history of rollouts. By default, it retains the last 10 revisions.

### Rolling Back with kubectl

**See rollout history:**
```bash
kubectl rollout history deployment/node-hono
```

Output:
```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```

**Inspect a specific revision:**
```bash
kubectl rollout history deployment/node-hono --revision=2
```

**Roll back to the previous revision (one step back):**
```bash
kubectl rollout undo deployment/node-hono
```

**Roll back to a specific revision:**
```bash
kubectl rollout undo deployment/node-hono --to-revision=1
```

**Watch the rollback progress:**
```bash
kubectl rollout status deployment/node-hono
```

**Verify what image is now running:**
```bash
kubectl get deployment node-hono -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### How to Roll Back via CircleCI (The Preferred Production Method)

`kubectl rollout undo` works in an emergency. But for audited, tracked deployments, the preferred approach is to deploy a known-good tag through CircleCI so there is a pipeline record of what happened.

**Scenario:** `node-v1.3` introduced a bug. You want to redeploy `node-v1.2`.

```bash
# The image for node-v1.2 is still in ECR:
# 425848620191.dkr.ecr.us-east-1.amazonaws.com/node-hono:node-v1.2
# You do not need to rebuild it. Just re-tag and re-push the git tag.

# Step 1: Find the commit that was tagged node-v1.2
git log --oneline --decorate | grep node-v1.2

# Step 2: Delete the bad tag from remote (optional, for cleanliness)
git tag -d node-v1.3
git push origin --delete node-v1.3

# Step 3: Create a new deployment tag pointing at the node-v1.2 commit
# (or a fixed commit)
git tag node-v1.4 <commit-hash-of-node-v1.2>
git push origin node-v1.4
```

CircleCI triggers the `node-deploy` workflow for `node-v1.4`. The `aws-ecr/build-and-push-image` job builds a fresh image from that commit and pushes it as `node-hono:node-v1.4`. The deploy job updates Kubernetes to use `node-v1.4`. Your system is now running code equivalent to what was in `node-v1.2`, but under a new tag, so the timeline is fully auditable.

**Why create a new tag instead of re-pushing the old one?**

Re-using a tag (`git tag -f node-v1.2`) on a different commit and force-pushing it is strongly discouraged. ECR already has an image tagged `node-v1.2`. CircleCI might cache the pipeline. Other engineers looking at the tag history will see a tag that points at a different commit than it used to. New tags (`node-v1.4`) preserve a clear, linear history of what was deployed and when.

### Emergency kubectl Rollback vs. CI/CD Rollback

| Method | When to use | Downside |
|--------|-------------|----------|
| `kubectl rollout undo` | Production is actively broken, every second matters, you need rollback in under 60 seconds | No CI/CD audit trail, manual step not reproducible via automation |
| New tag through CircleCI | Non-urgent rollback, or after the emergency has been stabilized | Takes 3-5 minutes for the pipeline to run |

In practice: use `kubectl rollout undo` to stop the bleeding immediately, then create a proper new tag and push it through CI/CD to re-establish a clean deployment record.
