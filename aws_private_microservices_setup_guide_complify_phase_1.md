# AWS Complify Microservices – Step‑by‑Step Setup Guide (Phase 1)

Architecture:
- 1 VPC
- 2 Public Subnets
- 2 Private App Subnets
- Internet Gateway
- NAT Gateway
- ALB Public
- ECS Fargate (2 services)
- RDS PostgreSQL (Private)
- ElastiCache Redis (Private)
- S3 + VPC Endpoint
- ECR (2 repositories)
- CloudWatch Logs
- Route53

---

# Step 1: Create VPC

Open:
AWS Console → VPC → Create VPC

Settings:

Name: complify-vpc
CIDR: 10.0.0.0/16

Create

---

# Step 2: Create Subnets

Create 4 subnets:

Public Subnet A
CIDR: 10.0.1.0/24
AZ: me-central-1a

Public Subnet B
CIDR: 10.0.2.0/24
AZ: me-central-1b

Private App Subnet A
CIDR: 10.0.10.0/24
AZ: me-central-1a

Private App Subnet B
CIDR: 10.0.11.0/24
AZ: me-central-1b

Enable auto assign public IP only on public subnets.

---

# Step 3: Create Internet Gateway

VPC → Internet Gateway

Create:
complify-igw

Attach to:
complify-vpc

---

# Step 4: NAT Gateway Count

For production:

Create 2 NAT Gateways

NAT-A → Public Subnet A
NAT-B → Public Subnet B

Allocate Elastic IP for each.

Reason:
If one AZ fails, private services still access internet.

Budget option:
1 NAT gateway works but creates single point of failure.

---

# Step 5: Create Route Tables

Create Public Route Table

Routes:
0.0.0.0/0 → IGW

Associate:
Public Subnet A
Public Subnet B


Create Private Route Table A

Routes:
0.0.0.0/0 → NAT-A

Associate:
Private App Subnet A


Create Private Route Table B

Routes:
0.0.0.0/0 → NAT-B

Associate:
Private App Subnet B

---

# Step 6: Security Groups

ALB Security Group

Inbound:
443 → 0.0.0.0/0
80 → 0.0.0.0/0

Outbound:
4001 → ECS SG
4002 → ECS SG


ECS Security Group

Inbound:
4001 from ALB SG
4002 from ALB SG

Outbound:
5432 → RDS
6379 → Redis
443 → S3 endpoint
443 → internet via NAT


RDS Security Group

Inbound:
5432 from ECS SG

No public access


Redis Security Group

Inbound:
6379 from ECS SG

No public access

---

# Step 7: Create S3 Bucket

Open:
S3 → Create bucket

Bucket name:
complify-documents

Disable public access:
ON

Enable versioning:
ON

Create bucket

---

# Step 8: Create S3 Gateway VPC Endpoint

VPC → Endpoints

Create endpoint

Service:
com.amazonaws.me-central-1.s3

Type:
Gateway

Choose:
complify-vpc

Attach:
Private route tables only

Create

Result:
Private ECS tasks access S3 without NAT.

---

# Step 9: Create RDS PostgreSQL

Open:
RDS → Create database

Engine:
PostgreSQL

Template:
Production

DB instance:
db.t4g.micro

Multi‑AZ:
Enable

Storage:
20–50 GB gp3

Public Access:
NO

VPC:
complify-vpc

Subnet Group:
Create DB subnet group

Add:
Private Subnet A
Private Subnet B

Security Group:
RDS-SG

Create

After deployment copy endpoint:

DB_HOST=<endpoint>
DB_PORT=5432

---

# Step 10: Create Redis (ElastiCache)

Open:
ElastiCache

Redis OSS

Cluster mode:
Disabled

Node:
cache.t4g.micro

Subnet group:
Private subnets

Public access:
Disabled

Security Group:
Redis-SG

Create

Copy endpoint:

REDIS_HOST=<endpoint>
REDIS_PORT=6379

---

# Step 11: Create ECR Repositories

ECR → Create repository

Repository 1:
complify-ingestion

Repository 2:
complify-retrieval

Mutable tags:
Enabled

Push images:

aws ecr get-login-password --region me-central-1 | docker login --username AWS --password-stdin account-id.dkr.ecr.me-central-1.amazonaws.com

Tag:

docker tag ingestion:latest repo-url/complify-ingestion:latest

docker tag retrieval:latest repo-url/complify-retrieval:latest

Push:

docker push repo-url/complify-ingestion:latest

docker push repo-url/complify-retrieval:latest

---

# Step 12: Create ECS Cluster

ECS → Cluster → Create

Type:
Networking only

Name:
complify-cluster

Create

---

# Step 13: Create CloudWatch Log Groups

CloudWatch → Log Groups

Create:

/ecs/complify-ingestion
/ecs/complify-retrieval

---

# Step 14: Create Task Definition (Service 1)

Task:
complify-ingestion-task

Launch:
Fargate

CPU:
1 vCPU

Memory:
2GB

Port:
4001

Environment:

DB_HOST=<rds endpoint>
DB_PORT=5432
REDIS_HOST=<redis endpoint>
REDIS_PORT=6379
S3_BUCKET=complify-documents
AWS_REGION=me-central-1

Logs:
CloudWatch

Image:
ECR ingestion image

Create

---

# Step 15: Create Task Definition (Service 2)

Task:
complify-retrieval-task

Port:
4002

Image:
ECR retrieval image

Same env variables

Create

---

# Step 16: Create Application Load Balancer

EC2 → Load Balancer

Type:
Application Load Balancer

Name:
complify-alb

Scheme:
Internet facing

Subnets:
Public A
Public B

Security group:
ALB-SG

---

# Step 17: Create Target Groups

Target Group 1

Name:
ingestion-tg

Port:
4001
Target type:
IP


Target Group 2

Name:
retrieval-tg

Port:
4002
Target type:
IP

---

# Step 18: Listener Rules

HTTPS Listener:
443

Rule 1:
/api/upload/*
→ ingestion-tg

Rule 2:
/api/search/*
→ retrieval-tg

---

# Step 19: Create ECS Services

Service 1:
complify-ingestion-service

Task:
complify-ingestion-task

Subnets:
Private A + B

Assign Public IP:
Disabled

Attach:
ingestion-tg

Desired tasks:
2


Service 2:
complify-retrieval-service

Task:
retrieval-task

Attach:
retrieval-tg

Desired:
2

---

# Step 20: Route53

Open Route53

Create hosted zone

Domain:
complify.com

Create A record:

api.complify.com

Alias → ALB DNS

---

# Final Flow

User
↓
Route53
↓
HTTPS
↓
ALB
↓
Listener Rules
↓
ECS Fargate Services
↓
RDS PostgreSQL
↓
Redis
↓
S3 via VPC Endpoint
↓
CloudWatch Logs

Production Result:

Public: ALB only
Private: ECS + RDS + Redis + S3 access
Multi-AZ deployment
No public DB access

