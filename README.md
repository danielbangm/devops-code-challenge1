# Containerized Application Platform on AWS ECS

A fully automated AWS environment for running a containerized React frontend and Node.js backend on ECS Fargate.

This project started with a simple idea: take a small application and build everything around it that makes cloud infrastructure interesting. containerization, networking, load balancing, infrastructure as code, CI/CD, IAM, auto scaling, and deployment automation.

The application runs as two independent ECS services behind an Application Load Balancer. Docker images are stored in Amazon ECR, the AWS infrastructure is defined with Terraform, and deployments can be driven through either Jenkins or GitHub Actions.

I like projects where I can build the entire stack, deliberately change things, troubleshoot failures, automate repetitive work, tear the environment down, and build it again.

## Architecture

                         Internet
                            |
                            v
                Application Load Balancer
                     /              \
                /api/*                /*
                   |                   |
                   v                   v
             Backend ECS          Frontend ECS
             Fargate Task         Fargate Task
             Port 8080            Port 80
                   \                 /
                    \               /
                       Amazon ECR
                           ^
                           |
                 +---------+---------+
                 |                   |
              Jenkins         GitHub Actions
                 ^                   ^
                 +------ GitHub -----+

## The Stack

**Cloud:** AWS ECS, Fargate, ECR, VPC, ALB, IAM  
**Infrastructure:** Terraform  
**Containers:** Docker, Nginx  
**CI/CD:** Jenkins, GitHub Actions  
**Application:** React, Node.js, Express  
**Source Control:** Git & GitHub

## Infrastructure

The AWS environment is defined in Terraform rather than being dependent on manually created application infrastructure.

Terraform provisions the networking, compute, container registry, load balancing, IAM, and scaling components:

- VPC with public subnets across two Availability Zones
- Internet Gateway and routing
- Application Load Balancer
- Frontend and backend target groups
- ECS Fargate cluster and services
- ECS task definitions
- Amazon ECR repositories
- IAM task execution role
- Security groups
- ECS Service Auto Scaling

The infrastructure lifecycle follows the usual Terraform workflow:

    terraform init
    terraform fmt
    terraform validate
    terraform plan
    terraform apply

And when I'm finished with the environment:

    terraform destroy

## Container Architecture

The frontend and backend are built and deployed independently.

### Frontend

The React frontend uses a multi-stage Docker build. Node.js builds the production bundle and Nginx serves the resulting static content on port 80.

    React
      ↓
    Node.js build
      ↓
    Static assets
      ↓
    Nginx :80

### Backend

The Node.js/Express API runs in its own container on port 8080.

    Express API
        ↓
    Docker container
        ↓
       :8080

Both images are stored in private Amazon ECR repositories and consumed by their respective ECS services.

## Traffic Flow

A single Application Load Balancer acts as the public entry point.

    Internet
       ↓
      ALB
       |
       +── /* ────────→ Frontend :80
       |
       └── /api/* ────→ Backend :8080

The frontend calls `/api/`, allowing frontend and API traffic to share the same public endpoint while the ALB handles service routing.

The ECS tasks themselves accept application traffic from the ALB security group rather than being the primary public entry point.

## ECS Fargate

Both services run on AWS Fargate, keeping the application layer focused on containers rather than managing ECS worker instances.

Each task runs with:

    CPU:     512 units / 0.5 vCPU
    Memory:  1024 MB / 1 GB

Each service starts with one task and can scale to four:

    Minimum:  1
    Desired:  1
    Maximum:  4

Target-tracking policies maintain approximately 50% average CPU utilization.

    Traffic increases
          ↓
    CPU utilization rises
          ↓
    Target tracking policy
          ↓
    ECS increases DesiredCount
          ↓
    New Fargate tasks start
          ↓
    ALB distributes traffic

## Jenkins Pipeline

The `main` branch uses Jenkins for CI/CD.

Jenkins runs on a dedicated EC2 instance and authenticates to AWS through an IAM instance role instead of static AWS credentials.

The pipeline is defined in the `Jenkinsfile`.

    Git Push
       ↓
    Jenkins
       ↓
    Checkout
       ↓
    Build Docker Images
       ↓
    Authenticate to ECR
       ↓
    Push Images
       ↓
    Force New ECS Deployment
       ↓
    Wait for Services to Stabilize
       ↓
    Deployment Complete

A deployment therefore takes the application from Git to running Fargate containers without manually building, tagging, pushing, or restarting services.

## GitHub Actions Pipeline

I also maintain a second deployment implementation on the `gitops` branch using GitHub Actions.

The workflow lives at:

    .github/workflows/deploy.yml

A push to that branch runs:

    Git Push
       ↓
    GitHub Actions
       ↓
    Authenticate to AWS
       ↓
    Build Docker Images
       ↓
    Push to ECR
       ↓
    Update ECS Services
       ↓
    Wait for Stability
       ↓
    Deployment Complete

This gives the project two independent paths to the same AWS environment:

| Branch | CI/CD | Flow |
|---|---|---|
| `main` | Jenkins | GitHub → Jenkins → ECR → ECS |
| `gitops` | GitHub Actions | GitHub → Actions → ECR → ECS |

## IAM & Security

Credentials and infrastructure access stay outside the application source.

- Jenkins uses an EC2 IAM role.
- AWS credentials are never committed to Git.
- GitHub Actions secrets are stored outside source control.
- ECR repositories are private.
- ECS application ports are restricted by security groups.
- Terraform state is excluded from Git.
- SSH private keys are excluded from Git.
- The ALB is the application's public entry point.

## Repository Structure

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

## Running Locally

Backend:

    cd backend
    npm ci
    npm start

The API listens on port `8080`.

Frontend:

    cd frontend
    npm ci
    npm start

The development frontend listens on port `3000`.

## Build. Automate. Break. Rebuild.

The part I enjoy most about cloud infrastructure is that none of it has to be permanent.

I can provision the network, deploy the containers, change the architecture, stress the services, watch them scale, break something, trace the failure, automate the fix, destroy the environment, and build something different the next time.

That's what this repository is for.

---

**Daniel BM**
