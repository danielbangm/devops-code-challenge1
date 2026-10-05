# Building a Production-Style CI/CD Platform on AWS ECS

## Why I Built This

I built this project as part of my hands-on journey into Cloud and DevOps engineering.

I wanted to go beyond tutorials and build an environment where I could connect the tools I had been learning individually — Docker, Terraform, Jenkins, AWS, GitHub Actions, networking, IAM, load balancing, and auto scaling — into one complete deployment workflow.

The goal was simple:

> Push code, build containers, ship them to AWS, deploy them automatically, and let the infrastructure handle scaling.

The application itself is intentionally simple: a React frontend communicating with a Node.js/Express backend. The interesting part of this project is everything around the application.

I containerized both services, provisioned the AWS infrastructure with Terraform, deployed the containers to ECS Fargate, configured an Application Load Balancer for path-based routing, added CPU-based auto scaling, and built two different CI/CD workflows using Jenkins and GitHub Actions.

This repo is one of my cloud labs — build it, break it, troubleshoot it, automate it, understand why it works, and eventually tear it all back down.

---

## Architecture

```text
                              Internet
                                 |
                                 v
                     Application Load Balancer
                          /              \
                         /                \
                    /api/*                 /*
                       |                    |
                       v                    v
                 Backend ECS          Frontend ECS
                 Fargate Task         Fargate Task
                  Port 8080             Port 80
                       |                    |
                       +---------+----------+
                                 |
                            Amazon ECR
                                 ^
                                 |
                    +------------+------------+
                    |                         |
                 Jenkins               GitHub Actions
                    ^                         ^
                    |                         |
                    +----------- GitHub ------+
```

The Application Load Balancer is the public entry point into the environment.

Traffic is routed based on the request path:

- `/` → React frontend
- `/api/*` → Express backend

Both services run independently as Docker containers on AWS ECS Fargate.

---

## Tech Stack

### Cloud

- AWS ECS
- AWS Fargate
- Amazon ECR
- Application Load Balancer
- AWS IAM
- Amazon VPC

### Infrastructure & Automation

- Terraform
- Jenkins
- GitHub Actions
- Docker
- Git

### Application

- React
- Node.js
- Express
- Nginx

---

## What I Built

Instead of treating this as just an application deployment, I broke the environment into several layers:

```text
Application
     ↓
Containers
     ↓
Container Registry
     ↓
Compute / Orchestration
     ↓
Networking
     ↓
Load Balancing
     ↓
Auto Scaling
     ↓
CI/CD Automation
     ↓
Infrastructure as Code
```

That helped me understand how the individual DevOps tools fit together rather than learning each one in isolation.

---

## Repository Structure

```text
.
├── backend/
│   ├── Dockerfile
│   ├── config.js
│   ├── index.js
│   └── package.json
│
├── frontend/
│   ├── Dockerfile
│   ├── src/
│   └── package.json
│
├── terraform/
│   ├── alb.tf
│   ├── autoscaling.tf
│   ├── ecr.tf
│   ├── ecs.tf
│   ├── iam.tf
│   ├── network.tf
│   ├── provider.tf
│   └── security-groups.tf
│
├── Jenkinsfile
└── README.md
```

---

# Running the Application Locally

The application was tested using Node.js 16.

## Backend

```bash
cd backend
npm ci
npm start
```

The backend listens on:

```text
http://localhost:8080
```

## Frontend

From another terminal:

```bash
cd frontend
npm ci
npm start
```

The React development server runs on:

```text
http://localhost:3000
```

The frontend calls the backend and displays a GUID returned by the API.

---

# Containerization

Both application components are containerized independently.

## Backend

```bash
docker build -t challenge-backend ./backend
```

The Express application listens on port `8080`.

## Frontend

```bash
docker build -t challenge-frontend ./frontend
```

The frontend uses a multi-stage Docker build.

Node.js builds the React production bundle, then Nginx serves the resulting static files on port `80`.

```text
React Source
     ↓
Node.js Build
     ↓
Static Production Files
     ↓
Nginx
     ↓
Port 80
```

The resulting images are pushed to private Amazon ECR repositories before being deployed to ECS.

---

# Infrastructure as Code with Terraform

I wanted the application infrastructure to be reproducible instead of clicking through the AWS console every time I rebuilt the environment.

Terraform provisions the AWS application stack.

The configuration creates:

- VPC
- Two public subnets across separate Availability Zones
- Internet Gateway
- Public route table
- Security groups
- Application Load Balancer
- Frontend target group
- Backend target group
- Amazon ECR repositories
- ECS cluster
- ECS task definitions
- ECS services
- IAM ECS task execution role
- ECS Service Auto Scaling

The Terraform workflow is:

```bash
cd terraform
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

I intentionally review `terraform plan` before applying changes so I can see exactly what Terraform intends to create, modify, or destroy.

---

# ECS Fargate

I chose AWS Fargate so the application containers can run without managing the underlying ECS worker servers.

The frontend and backend run as separate ECS services.

Each task is configured with:

```text
CPU:            512 units (0.5 vCPU)
Memory:         1024 MB (1 GB)

Minimum tasks:  1
Desired tasks:  1
Maximum tasks:  4
```

This gives each service its own lifecycle while still allowing both to be exposed through the same load balancer.

---

# Auto Scaling

Both ECS services use target-tracking auto scaling.

The target is:

```text
50% average CPU utilization
```

The services can scale between:

```text
1 → 4 tasks
```

Conceptually:

```text
Normal Traffic
     ↓
   1 Task

CPU increases
     ↓
Target Tracking Policy
     ↓
ECS increases DesiredCount
     ↓
Additional Fargate Tasks
     ↓
ALB distributes traffic
```

As demand falls, ECS can scale the service back toward its minimum capacity.

---

# Application Load Balancer

I use one public Application Load Balancer as the entry point for both services.

```text
                       ALB
                        |
             +----------+----------+
             |                     |
          /api/*                   /*
             |                     |
             v                     v
      Backend Target Group   Frontend Target Group
             |                     |
             v                     v
       ECS Port 8080          ECS Port 80
```

The React application uses:

```text
/api/
```

for API requests.

That means the browser can access both services through the same public endpoint while the ALB handles routing internally.

---

# Jenkins CI/CD

One of my goals with this project was to eliminate the manual deployment process.

Before automation, deploying a change would mean doing something like:

```text
Build Docker Image
        ↓
Login to ECR
        ↓
Tag Image
        ↓
Push Image
        ↓
Update ECS
        ↓
Wait for Deployment
```

I automated that workflow with Jenkins.

Jenkins runs on a dedicated EC2 instance and uses an EC2 IAM role to communicate with AWS.

The deployment pipeline is defined as code in:

```text
Jenkinsfile
```

The pipeline performs:

```text
GitHub
   |
   v
Checkout Source
   |
   v
Build Frontend + Backend Images
   |
   v
Authenticate to Amazon ECR
   |
   v
Push Images
   |
   v
Trigger ECS Deployments
   |
   v
Wait for Services to Stabilize
   |
   v
Deployment Complete
```

This was also a useful troubleshooting exercise.

While building the pipeline, the Jenkins EC2 instance ran out of memory during the React build. Rather than treating the failure as just a pipeline problem, I traced the Linux logs, identified an OOM kill, resized the instance, corrected Jenkins resource thresholds, and reran the deployment successfully.

That experience was one of my favorite parts of the project because it connected CI/CD with actual Linux and infrastructure troubleshooting.

---

# GitHub Actions

After getting the Jenkins pipeline working, I wanted to implement the same deployment workflow using another CI/CD platform.

The `gitops` branch contains a GitHub Actions implementation:

```text
.github/workflows/deploy.yml
```

A push to that branch triggers:

```text
Git Push
   |
   v
GitHub Actions Runner
   |
   v
Authenticate to AWS
   |
   v
Build Docker Images
   |
   v
Push Images to ECR
   |
   v
Update ECS Services
   |
   v
Wait for Services to Stabilize
   |
   v
Deployment Complete
```

This gave me a chance to compare a self-managed CI/CD server such as Jenkins with a managed CI/CD platform such as GitHub Actions.

The two implementations ultimately deploy to the same AWS infrastructure:

| Branch | Automation | Deployment Path |
|---|---|---|
| `main` | Jenkins | GitHub → Jenkins → Docker → ECR → ECS |
| `gitops` | GitHub Actions | GitHub → Actions → Docker → ECR → ECS |

---

# IAM & Security

I tried to avoid solving automation problems by simply hardcoding credentials.

Some of the security decisions in the project include:

- AWS credentials are not committed to Git.
- Jenkins authenticates to AWS through an EC2 IAM role.
- GitHub Actions secrets are stored outside the source code.
- Amazon ECR repositories are private.
- ECS application ports only accept inbound traffic from the ALB security group.
- Terraform state files are excluded from Git.
- SSH/private keys are excluded from Git.
- Application containers are not directly exposed as the primary public entry point.

The `.gitignore` prevents common sensitive files from accidentally entering version control.

---

# End-to-End Flow

Putting everything together:

```text
Developer
    |
    v
  GitHub
    |
    +----------------------+
    |                      |
    v                      v
 Jenkins             GitHub Actions
    |                      |
    +----------+-----------+
               |
               v
            Docker
               |
               v
          Amazon ECR
               |
               v
          ECS Fargate
          /         \
         /           \
   Frontend        Backend
      |               |
      +-------+-------+
              |
              v
     Application Load Balancer
              |
              v
           Internet
```

Terraform manages the AWS application infrastructure underneath the deployment workflow.

---

# What I Learned

The biggest takeaway from this project wasn't learning another individual AWS service or DevOps command.

It was understanding how the pieces connect.

Docker makes the application portable.

ECR stores the images.

ECS runs them.

Fargate provides the compute.

The ALB exposes and routes traffic.

Auto Scaling adjusts capacity.

IAM controls what the automation is allowed to do.

Terraform describes the infrastructure.

Jenkins and GitHub Actions automate the path from source code to deployment.

Git ties the entire workflow together.

Building the complete environment made those relationships much clearer than learning each technology independently.

---

# Tearing It Down

Because this is a lab environment, I don't leave the infrastructure running when I'm not using it.

Terraform-managed resources can be removed with:

```bash
cd terraform
terraform destroy
```

The manually created Jenkins EC2 resources are removed separately.

Then I can rebuild the environment again when I want to experiment with something new.

Build. Break. Troubleshoot. Automate. Destroy. Repeat.

---

## Author

**Daniel BM**

Cloud & DevOps Engineering
