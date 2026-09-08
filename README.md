# DevOps Tech Challenge 1

## Node.js Frontend and Backend Deployment with AWS ECS, Terraform, and Jenkins CI/CD

## Overview

This repository contains a React frontend and an Express/Node.js backend. The frontend calls the backend and displays a `SUCCESS` message followed by a GUID when the connection is working correctly.

For this challenge, the application was containerized with Docker and deployed to AWS using:

- Amazon ECS with AWS Fargate
- Amazon ECR
- Application Load Balancer (ALB)
- Amazon VPC
- IAM
- Amazon CloudWatch
- Application Auto Scaling
- Terraform
- Jenkins
- GitHub

The final architecture uses an internet-facing Application Load Balancer as the public entry point. The ALB sends normal application traffic to the frontend service and `/api/*` traffic to the backend service. The frontend and backend run as separate ECS/Fargate services.

---

## Architecture

```text
                         Internet
                            |
                            v
              Application Load Balancer
                    (Public Subnets)
                   /                \
                  /                  \
          Default Route            /api/*
                |                     |
                v                     v
      Frontend Target Group   Backend Target Group
                |                     |
                v                     v
       Frontend ECS Service   Backend ECS Service
                |                     |
                v                     v
         Fargate Task            Fargate Task
             (Private Subnets / VPC)
```

### CI/CD Flow

```text
Developer
    |
    v
  GitHub
    |
    v
 Jenkins on EC2
    |
    +--> Build Frontend Docker Image
    |
    +--> Build Backend Docker Image
    |
    v
 Amazon ECR
    |
    v
 ECS Force New Deployment
    |
    v
 ECS / Fargate Services
```

---

## Technologies Used

| Area | Technology |
| --- | --- |
| Source Control | Git / GitHub |
| Frontend | React |
| Backend | Node.js / Express |
| Containers | Docker |
| Infrastructure as Code | Terraform |
| CI/CD | Jenkins |
| Container Registry | Amazon ECR |
| Container Orchestration | Amazon ECS |
| Compute | AWS Fargate |
| Networking | Amazon VPC / ALB |
| Permissions | AWS IAM |
| Logging / Metrics | Amazon CloudWatch |
| Auto Scaling | Application Auto Scaling |
| Load Testing | Siege |

**AWS Region:** `us-east-2`

---

## Repository Structure

```text
devops-code-challenge1/
├── backend/
│   ├── config.js
│   ├── dockerfile
│   ├── package.json
│   └── ...
├── frontend/
│   ├── src/
│   │   └── config.js
│   ├── Dockerfile
│   ├── package.json
│   └── ...
├── Terraform/
│   ├── provider.tf
│   ├── variables.tf
│   ├── vpc.tf
│   ├── ecr.tf
│   ├── ecs.tf
│   ├── IAM.tf
│   ├── alb.tf
│   ├── autoscaling.tf
│   ├── jenkins.tf
│   ├── outputs.tf
│   └── .terraform.lock.hcl
├── Jenkinsfile
├── .gitignore
└── README.md
```

Terraform state files and the local `.terraform/` directory are excluded from version control.

---

# Prerequisites

The following tools are needed to reproduce the deployment:

- Git
- Node.js / npm
- NVM (recommended for the provided application)
- Docker
- AWS CLI
- Terraform
- An AWS account with appropriate permissions
- An EC2 SSH key pair
- GitHub repository access
- Jenkins
- Siege (optional, for load testing)

> **Compatibility note:** The provided React/Webpack application was tested with Node.js 16 because newer Node versions caused an OpenSSL compatibility error. Node.js 16 is end-of-life and is used here only for compatibility with the challenge application.

---

# 1. Clone the Repository

```bash
git clone https://github.com/jhjeremy27-web/JCloud.git
cd JCloud
```

If working from a fork or another clone, use that repository URL instead.

---

# 2. Run the Application Locally

The backend and frontend run as separate processes. Start the backend first.

## Backend

```bash
cd backend
npm ci
npm start
```

The backend should respond to a GET request at:

```text
http://localhost:8080
```

A successful request returns a response containing:

```text
SUCCESS <GUID>
```

## Frontend

Open another terminal:

```bash
cd frontend
npm ci
npm start
```

The frontend is available at:

```text
http://localhost:3000
```

If the frontend successfully connects to the backend, the page displays `SUCCESS` followed by a GUID.

---

## Local Application Configuration

The frontend configuration is located at:

```text
frontend/src/config.js
```

For local development:

```javascript
const config = {
  backendUrl: "http://localhost:8080"
};

export default config;
```

The backend configuration is located at:

```text
backend/config.js
```

This defines the allowed frontend origin for CORS.

---

## Local Troubleshooting

### Node.js / OpenSSL Error

The frontend initially failed under a newer Node.js version with:

```text
ERR_OSSL_EVP_UNSUPPORTED
```

NVM was used to run the application with Node.js 16:

```bash
nvm install 16
nvm use 16
npm ci
npm start
```

### Frontend Response Property

The backend returns the successful response in the `message` property. The frontend was updated to read that property so the returned `SUCCESS <GUID>` message could be displayed correctly.

---

# 3. Dockerize the Application

## Backend Dockerfile

The backend Dockerfile is located at:

```text
backend/dockerfile
```

Example configuration:

```dockerfile
FROM node:16

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

EXPOSE 8080

CMD ["npm", "start"]
```

## Frontend Dockerfile

The frontend Dockerfile is located at:

```text
frontend/Dockerfile
```

```dockerfile
FROM node:16

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build

RUN npm install -g serve

EXPOSE 3000

CMD ["serve", "-s", "build"]
```

---

## Build the Docker Images

Run these commands from the repository root:

```bash
docker build -t backend-app ./backend
docker build -t frontend-app ./frontend
```

Run the containers:

```bash
docker run -d --name backend -p 8080:8080 backend-app
docker run -d --name frontend -p 3000:3000 frontend-app
```

Verify:

```text
Frontend: http://localhost:3000
Backend:  http://localhost:8080
```

---

## Docker Troubleshooting

### Build Context

Docker build paths are relative to the terminal's current directory.

From the repository root:

```bash
docker build -t frontend-app ./frontend
```

From inside the `frontend/` directory:

```bash
docker build -t frontend-app .
```

### `host.docker.internal`

During testing, the frontend was initially configured to call:

```text
http://host.docker.internal:8080
```

This did not work for the browser-executed React request. The React application executes its API request in the user's browser, so local testing worked correctly with:

```text
http://localhost:8080
```

---

# 4. Configure AWS CLI

Configure the AWS CLI:

```bash
aws configure
```

Verify authentication:

```bash
aws sts get-caller-identity
```

The command should return the authenticated AWS account and IAM identity.

> Never commit AWS access keys, secret access keys, passwords, session tokens, or credential CSV files to this repository.

---

# 5. Deploy the Infrastructure with Terraform

All Terraform configuration files are stored in:

```text
Terraform/
```

Terraform evaluates all `.tf` files in the current directory together.

Move into the Terraform directory:

```bash
cd Terraform
```

Initialize Terraform:

```bash
terraform init
```

Review the execution plan:

```bash
terraform plan
```

Create the infrastructure:

```bash
terraform apply
```

Review the proposed changes and approve the apply when ready.

---

## Infrastructure Provisioned by Terraform

The Terraform configuration creates:

- VPC
- Two public subnets
- Two private subnets
- Internet Gateway
- NAT Gateway
- Elastic IP
- Public and private route tables
- Security groups
- Two Amazon ECR repositories
- Amazon ECS cluster
- Frontend ECS task definition and service
- Backend ECS task definition and service
- ECS task execution/task IAM roles
- CloudWatch log groups
- Application Load Balancer
- ALB listener
- Frontend target group
- Backend target group
- Path-based listener rule
- Application Auto Scaling targets and policies
- Jenkins EC2 infrastructure

Although the challenge did not require Jenkins infrastructure to be provisioned with Terraform, this implementation includes `jenkins.tf`.

---

## ECS Configuration

```text
Cluster:
devops-challenge-cluster

Frontend Service:
devops-challenge-frontend-service

Backend Service:
devops-challenge-backend-service
```

Each application task is configured with:

```text
CPU:     512 CPU units (0.5 vCPU)
Memory:  1024 MB (1 GB)
```

Application Auto Scaling:

```text
Minimum capacity: 1
Desired capacity: 1
Maximum capacity: 4

Target metric:
ECSServiceAverageCPUUtilization

Target:
50% CPU
```

---

## Amazon ECR

Two repositories are used:

```text
devops-challenge-frontend
devops-challenge-backend
```

Jenkins builds and pushes the corresponding frontend and backend images to these repositories.

---

## Terraform Troubleshooting

### Terraform Initialized an Empty Directory

Terraform was initially run from the repository root and reported that the directory contained no Terraform configuration.

Terraform only evaluates `.tf` files in the current working directory.

Correct workflow:

```bash
cd Terraform
terraform init
terraform plan
terraform apply
```

### EC2 Key Pair Error

The Jenkins configuration originally referenced an EC2 key pair that did not exist in the deployment region.

AWS returned:

```text
InvalidKeyPair.NotFound
```

The Terraform configuration was updated to reference the existing key pair:

```text
terraform-key
```

Terraform then continued the deployment without requiring already-created resources to be rebuilt.

### `.gitignore`

Terraform state and local working files are excluded:

```gitignore
node_modules/
Terraform/.terraform/
Terraform/*.tfstate
Terraform/*.tfstate.*
```

The Terraform dependency lock file is retained:

```text
Terraform/.terraform.lock.hcl
```

---

# 6. Application Load Balancer

The Application Load Balancer is the public entry point for both services.

Routing configuration:

```text
Default route  -> Frontend Target Group
/api/*         -> Backend Target Group
```

This allows both the frontend and backend to be reached through a single public ALB hostname.

## Deployed Endpoints

### Frontend

```text
http://devops-challenge-alb-1542206865.us-east-2.elb.amazonaws.com
```

### Backend

```text
http://devops-challenge-alb-1542206865.us-east-2.elb.amazonaws.com/api/
```

---

# 7. Configure the Application for AWS

## Frontend

`frontend/src/config.js`

```javascript
const config = {
  backendUrl: "http://devops-challenge-alb-1542206865.us-east-2.elb.amazonaws.com/api/"
};

export default config;
```

## Backend

`backend/config.js`

```javascript
module.exports = {
  CORS_ORIGIN: "http://devops-challenge-alb-1542206865.us-east-2.elb.amazonaws.com"
};
```

---

## ALB Routing Troubleshooting

After deployment, the frontend initially produced an error similar to:

```text
Unexpected token '<', "<!doctype "... is not valid JSON
```

The frontend expected JSON from the backend but received the React application's HTML document.

The ALB backend rule was:

```text
/api/*
```

The frontend was requesting:

```text
/api
```

The `/api` request did not route to the backend as expected and instead reached the default frontend target group.

Testing:

```text
/api/
```

successfully returned the backend JSON response.

The frontend backend URL was therefore updated to include the trailing slash:

```text
http://devops-challenge-alb-1542206865.us-east-2.elb.amazonaws.com/api/
```

The `*` wildcard belongs to the ALB listener rule and should **not** be placed in the frontend URL.

---

# 8. Jenkins Setup

Jenkins runs inside a Docker container on an Amazon Linux 2023 EC2 instance.

Jenkins uses:

- Persistent Jenkins home storage
- Host Docker socket access
- Docker CLI
- AWS CLI
- GitHub credentials
- AWS credentials stored in Jenkins Credentials

## Jenkins URL

```text
http://3.150.212.162:8080/
```

No custom DNS name was configured for Jenkins. The server is accessed directly through the EC2 public IPv4 address on port `8080`.

> Jenkins login credentials are provided only through the approved challenge submission channel and are not stored in this repository.

---

## Jenkins Docker Permissions

The EC2 user must be able to communicate with Docker:

```bash
sudo usermod -aG docker ec2-user
```

Log out of the SSH session and reconnect so the new group membership takes effect.

Verify:

```bash
groups
docker ps
```

The Jenkins container also requires access to the host Docker socket.

Example pattern:

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -v /var/jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --group-add <DOCKER_GROUP_ID> \
  jenkins/jenkins:lts
```

---

## Jenkins Home Permissions

If Jenkins cannot write to its persistent home directory:

```bash
sudo mkdir -p /var/jenkins_home
sudo chown -R 1000:1000 /var/jenkins_home
```

Then recreate/start the Jenkins container.

---

# 9. Jenkins CI/CD Pipeline

The Jenkins job is configured as a **Pipeline from SCM** and reads the `Jenkinsfile` from this repository.

The pipeline performs the following workflow:

1. Checkout source code from GitHub.
2. Build the frontend Docker image.
3. Build the backend Docker image.
4. Authenticate Docker to Amazon ECR.
5. Tag the frontend and backend images.
6. Push the images to their ECR repositories.
7. Force a new deployment of the frontend ECS service.
8. Force a new deployment of the backend ECS service.
9. Clean the Jenkins workspace.

Example ECS deployment commands:

```bash
aws ecs update-service \
  --cluster devops-challenge-cluster \
  --service devops-challenge-frontend-service \
  --force-new-deployment \
  --region us-east-2
```

```bash
aws ecs update-service \
  --cluster devops-challenge-cluster \
  --service devops-challenge-backend-service \
  --force-new-deployment \
  --region us-east-2
```

ECR repositories used by the pipeline:

```text
devops-challenge-frontend
devops-challenge-backend
```

---

## Jenkins / AWS Authentication Troubleshooting

The Jenkins pipeline previously failed with:

```text
SignatureDoesNotMatch
```

The AWS secret access key had been transcribed from a screenshot, and a visually similar character was entered incorrectly.

The issue was resolved by replacing the credential using the original machine-readable AWS credentials source and testing authentication again.

### Lesson Learned

Secrets should never be reconstructed from screenshots or OCR.

Use:

- The original credential source
- A password manager
- Jenkins Credentials
- IAM roles or temporary credentials where possible

Do not store secrets directly in source code or documentation.

---

# 10. Load Testing and Auto Scaling

Siege was used from the local machine to generate concurrent traffic against the deployed application.

Traffic path:

```text
Siege
  |
  v
Internet
  |
  v
Application Load Balancer
  |
  v
ECS / Fargate
  |
  v
CloudWatch CPU Metric
  |
  v
Application Auto Scaling
```

A recorded load test produced approximately:

```text
Successful Transactions: 49,085
Availability:             99.95%
Transaction Rate:         408.77 transactions/sec
Concurrency:              ~243
```

An earlier test produced:

```text
Successful Transactions: 29,022
Availability:             99.27%
Transaction Rate:         240.29 transactions/sec
Concurrency:              175.10
```

Despite the high request volume, observed ECS CPU utilization remained far below the configured `50%` target, at approximately `2%` during the observed test.

As a result:

```text
Desired tasks: 1
Running tasks: 1
Scale-out events observed: 0
```

This is the expected behavior for the configured target-tracking policy because ECS auto scaling responds to the configured CPU metric, not directly to the number of HTTP requests or Siege concurrency.

The scaling threshold was not artificially lowered simply to force a scale-out demonstration.

---

# 11. Verification

The deployment was verified at each layer.

## Application

- Backend returned `SUCCESS <GUID>`.
- Frontend successfully displayed the backend response.
- Frontend was publicly accessible through the ALB.
- `/api/` returned backend JSON.

## Docker

- Backend Docker image built successfully.
- Frontend Docker image built successfully.
- Both containers ran locally.

## Terraform

- AWS infrastructure was successfully provisioned.
- Networking, ECR, ECS, IAM, ALB, CloudWatch, scaling, and Jenkins infrastructure were created.

## ECS

- Frontend service: `ACTIVE`
- Backend service: `ACTIVE`
- Desired tasks: `1`
- Running tasks: `1`
- Deployment reached steady state.

## ECR

- Separate frontend and backend repositories were created.
- Jenkins successfully pushed application images.

## Jenkins

- Jenkins checked out the private GitHub repository.
- Docker images were built.
- Images were pushed to ECR.
- ECS deployments were triggered.
- A later test build also completed successfully after AWS credential rotation.

## Load Testing

- Siege generated substantial concurrent traffic.
- Final recorded availability was `99.95%`.
- CPU remained below the auto scaling target, so no unnecessary scale-out occurred.

---

# 12. Security Considerations

The following security practices were applied or identified during the challenge:

- AWS access keys and secret access keys are not committed to Git.
- AWS console passwords are not committed to Git.
- GitHub Personal Access Tokens are not committed to Git.
- Jenkins credentials are not committed to Git.
- SSH private keys are not committed to Git.
- Terraform state files are excluded from Git.
- Jenkins credentials are stored using the Jenkins credential system.
- Exposed development credentials were rotated before final submission.
- The replacement AWS credential was tested locally and in Jenkins.
- The exposed legacy access key was deactivated and deleted.
- MFA should be enabled for interactive AWS identities.
- IAM roles and temporary credentials are preferable to long-lived access keys in production.
- Least-privilege IAM policies should be used in production.
- Jenkins network access should be restricted in production.
- HTTPS/TLS should be used for production-facing endpoints.
- Sensitive application values should be stored in an appropriate secrets-management system.

> Mounting `/var/run/docker.sock` into Jenkins grants the Jenkins container significant control over the Docker host. This was used for the challenge/lab environment but should be carefully evaluated before using the same design in production.

---

# 13. Troubleshooting Summary

This challenge included troubleshooting across multiple layers:

| Problem | Cause | Resolution |
| --- | --- | --- |
| React OpenSSL error | New Node version incompatible with old Webpack stack | Used Node.js 16 through NVM |
| Frontend did not display success response | Frontend read the wrong response property | Updated frontend to read `message` |
| Docker frontend could not reach backend | Browser-executed SPA was configured with `host.docker.internal` | Used `localhost:8080` for local browser testing |
| Terraform showed empty configuration | Terraform command run from wrong directory | Ran Terraform from `Terraform/` |
| Jenkins EC2 creation failed | Referenced EC2 key pair did not exist | Changed configuration to `terraform-key` |
| Jenkins Docker permission denied | Jenkins lacked correct Docker socket group access | Added Docker group access and container group ID |
| Jenkins home permission issue | Host directory ownership incompatible with Jenkins UID | Changed `/var/jenkins_home` ownership to UID 1000 |
| AWS `SignatureDoesNotMatch` | Credential was incorrectly transcribed from screenshot | Re-entered credential from authoritative machine-readable source |
| Frontend received HTML instead of JSON | `/api` did not route through `/api/*` backend rule | Changed request URL to `/api/` |
| ECS did not scale during Siege test | CPU stayed well below 50% target | Documented actual target-tracking behavior |

---

# 14. Cleanup

When the challenge environment is no longer needed, Terraform can be used to remove the managed infrastructure.

From the Terraform directory:

```bash
cd Terraform
terraform plan -destroy
terraform destroy
```

After Terraform completes, verify the AWS account for remaining resources that could continue generating charges, including:

- EC2 instances
- Elastic IP addresses
- NAT Gateways
- Application Load Balancers
- ECS services and clusters
- ECR repositories/images
- CloudWatch log groups

If Terraform cannot remove an ECR repository because it still contains images, remove the retained images as appropriate and run the destroy process again.

---

# Submission

## GitHub Repository

```text
https://github.com/jhjeremy27-web/JCloud
```

## Deployed Frontend

```text
http://devops-challenge-alb-1542206865.us-east-2.elb.amazonaws.com
```

## Backend Verification Endpoint

```text
http://devops-challenge-alb-1542206865.us-east-2.elb.amazonaws.com/api/
```

## Jenkins Server

```text
http://3.150.212.162:8080/
```

Jenkins credentials are **not** included in this repository. They should be provided only through the approved challenge submission form/channel.

The private GitHub repository should be shared with the reviewer specified in the challenge instructions.

---

# Final Result

This project demonstrates an end-to-end DevOps/cloud deployment workflow:

- Source code is version controlled in GitHub.
- Docker packages the frontend and backend.
- Terraform defines and provisions the AWS infrastructure.
- Amazon ECR stores application container images.
- Amazon ECS with AWS Fargate runs the frontend and backend.
- The application runs inside a VPC with public and private subnets.
- An Application Load Balancer provides public access and path-based routing.
- Jenkins automates Docker builds, ECR pushes, and ECS deployments.
- CloudWatch provides the CPU metric used by Application Auto Scaling.
- Siege validates application behavior under concurrent traffic.

The project also documents the troubleshooting required to move the application from local development through containerization, infrastructure provisioning, CI/CD automation, public routing, and load testing.

The result is a reproducible deployment process that can be followed by another engineer from source code to a running AWS environment.
