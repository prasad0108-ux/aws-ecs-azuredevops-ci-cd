# 🚀 ECS Deployment with Azure DevOps CI/CD

This project demonstrates **end-to-end CI/CD** for a Node.js application using **Azure DevOps Pipelines** to deploy on **AWS ECS (Fargate)** with **Application Load Balancer (ALB)** and **Amazon ECR**.

---

## 📌 Architecture Overview

```
Developer → GitHub → Azure DevOps Pipeline
                ↓
            Docker Build
                ↓
              Amazon ECR
                ↓
         Amazon ECS (Fargate)
                ↓
       Application Load Balancer
                ↓
              End Users
```

---

## 🧰 Tech Stack

| Layer                   | Technology                      |
| ----------------------- | ------------------------------- |
| Language                | Node.js (Express)               |
| Containerization        | Docker                          |
| CI/CD                   | Azure DevOps Pipelines          |
| Cloud Provider          | AWS                             |
| Container Orchestration | Amazon ECS (Fargate)            |
| Image Registry          | Amazon ECR                      |
| Load Balancer           | Application Load Balancer (ALB) |
| Networking              | VPC, Subnets, Security Groups   |
| Logging                 | CloudWatch Logs                 |

---

## 📁 Repository Structure

```
aws-ecs-azuredevops-ci-cd/
│
├── app/
│   ├── app.js
│   └── package.json
│
├── docker/
│   └── Dockerfile
│
├── pipelines/
│   └── azure-pipelines-ecs.yml
│
└── README.md
```

---

## 🧩 Application Code

### `app/app.js`

```js
const express = require('express');
const app = express();

const PORT = 3000;

app.get('/', (req, res) => {
  res.send('Hello from ECS deployed via Azure DevOps 🚀');
});

app.listen(PORT, () => {
  console.log(`App running on port ${PORT}`);
});
```

---

## 🐳 Docker Configuration

### `docker/Dockerfile`

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY app/package*.json ./
RUN npm install --omit=dev

COPY app/ .

EXPOSE 3000

CMD ["node", "app.js"]
```

---

## 🔐 AWS Configuration (One-Time Setup)

### 1️⃣ Amazon ECR

* Create repository: `ecs-sample-app`
* Used to store Docker images built by Azure DevOps

### 2️⃣ ECS Cluster

* Launch type: **Fargate**
* Networking: default VPC
* Subnets: minimum 2 AZs

### 3️⃣ Task Definition

* Container image:

  ```
  <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/ecs-sample-app:latest
  ```
* Container port: `3000`
* Network mode: `awsvpc`

### 4️⃣ ECS Service

* Desired tasks: `1`
* Load balancer: **Application Load Balancer**
* Target group type: `IP`
* Container port: `3000`

### 5️⃣ Load Balancer

* Listener: `HTTP : 80`
* Forward traffic → ECS Target Group

---

## 🔒 Security Group Configuration (IMPORTANT)

Using **default security group** for both ALB and ECS:

| Type       | Port | Source                        |
| ---------- | ---- | ----------------------------- |
| HTTP       | 80   | 0.0.0.0/0                     |
| Custom TCP | 3000 | Default Security Group (self) |

✔ No restart required after SG changes.

---

## 🔑 IAM Configuration

### IAM User (Azure DevOps)

* `AmazonECS_FullAccess`
* `AmazonEC2ContainerRegistryFullAccess`

### ECS Task Execution Role

* `ecsTaskExecutionRole`
* Allows ECS to pull images from ECR

---

## 🔁 Azure DevOps Pipeline

### `pipelines/azure-pipelines-ecs.yml`

```yaml
trigger:
- main

pool:
  vmImage: ubuntu-latest

variables:
  AWS_REGION: us-east-1
  ECR_REPO: ecs-sample-app
  ECS_CLUSTER: ecs-sample-cluster
  ECS_SERVICE: ecs-sample-app-service-0b82ybyb

steps:
- checkout: self

- task: Docker@2
  displayName: Build and Push Image to ECR
  inputs:
    containerRegistry: ecr-docker-connection
    repository: $(ECR_REPO)
    command: buildAndPush
    Dockerfile: docker/Dockerfile
    tags: |
      latest

- task: AWSCLI@1
  displayName: Deploy to ECS
  inputs:
    awsCredentials: aws-ecs-connection
    regionName: $(AWS_REGION)
    awsCommand: ecs
    awsSubCommand: update-service
    awsArguments: >
      --cluster $(ECS_CLUSTER)
      --service $(ECS_SERVICE)
      --force-new-deployment
```

---

## 🌐 Accessing the Application

Use **only the ALB DNS name**:

```
http://<alb-dns-name>
```

❌ Do NOT use:

* `localhost`
* task IP
* `:3000` in browser

---

## 🧠 Key Concepts (Why This Architecture)

* **Docker** → consistent builds across environments
* **ECR** → secure, IAM-integrated image storage
* **ECS Fargate** → no server management, auto-healing
* **ALB** → internal service routing + health checks
* **Azure DevOps** → enterprise-grade CI/CD

---

# 🎤 Interview Questions & Answers (Spoken, Real-World)

### 1️⃣ *“In real production, how does traffic reach an ECS service?”*

**Spoken Answer:**

> In production, users never access the ALB directly.
> Traffic usually flows from Route 53 to CloudFront for TLS, WAF, and caching.
> CloudFront forwards traffic to the ALB, and the ALB routes it to ECS tasks.
>
> ECS containers stay private inside the VPC.
> For this POC, I exposed the ALB directly, but in production Route 53 and CloudFront always sit in front.

---

### 2️⃣ *“Your ECS task is running but the site is down. How do you troubleshoot?”*

**Spoken Answer:**

> I troubleshoot layer by layer.
> First DNS and Route 53, then CloudFront if present.
> After that I check ALB listeners and target groups.
>
> If targets are unhealthy or anomalous, I validate health check paths and ports.
> Then I check ECS task logs and security group rules.
>
> This approach avoids guessing and helps isolate the problem quickly.

---

### 3️⃣ *“Why don’t we expose container ports directly?”*

**Spoken Answer:**

> Containers are ephemeral and restart frequently.
> Exposing them directly bypasses TLS, WAF, and traffic controls.
>
> In production, containers are always private and traffic flows through controlled entry points like ALB or CloudFront.

---

### 4️⃣ *“What does ‘target is anomalous’ mean?”*

**Spoken Answer:**

> It means the ALB can reach the container, but responses are inconsistent.
>
> Common reasons are wrong health check paths, the app listening on localhost, slow startup, or intermittent crashes.
>
> I usually fix it by checking health checks and container logs.

---

### 5️⃣ *“How do you deploy changes without downtime?”*

**Spoken Answer:**

> ECS uses rolling deployments.
> New tasks start first, health checks pass, traffic shifts, and old tasks are drained.
>
> This ensures zero downtime without manual intervention.

---

### 6️⃣ *“What happens if a container crashes?”*

**Spoken Answer:**

> ECS service scheduler automatically replaces the task.
> ALB only sends traffic to healthy tasks, so users don’t notice the failure.

---

### 7️⃣ *“How are secrets handled securely?”*

**Spoken Answer:**

> Secrets are stored in AWS Secrets Manager or SSM Parameter Store.
> ECS tasks retrieve them using IAM roles — never hard-coded in images or repos.

---

### 8️⃣ *“How does CI/CD integrate with ECS in real projects?”*

**Spoken Answer:**

> Code is pushed, pipeline builds a Docker image, pushes it to ECR, and updates the ECS service.
>
> ECS handles rolling deployment automatically, and rollback is just redeploying an older image.

---

## ✅ Final Outcome

✔ End-to-end CI/CD
✔ Production-aligned ECS deployment
✔ Real troubleshooting experience
✔ Interview-ready project

---

## 🏁 Summary

This project reflects **real enterprise deployment patterns**, not just a demo.
It demonstrates **how production systems are designed, deployed, and troubleshot**.

---

