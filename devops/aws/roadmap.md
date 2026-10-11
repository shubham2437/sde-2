[⬅ Back to AWS VPC](./netwroking/aws-vpc.md)

# AWS Learning Roadmap (SDE-2 / DevOps)

> Goal: deploy a production-style stack on AWS: **PostgreSQL, Redis and 3 services**, built with code, deployed automatically and monitored.
>
> Tip: on GitHub, click the **☰ outline button** at the top right of this file to jump to any phase. Tick the checkboxes as you go.

---

## Table of Contents

0. [Phase 0: VPC Networking (done)](#phase-0-vpc-networking-done)
1. [Phase 1: Core Foundations](#phase-1-core-foundations-1-2-weeks)
2. [Phase 2: Data Layer](#phase-2-data-layer-1-week)
3. [Phase 3: Traffic and Scaling](#phase-3-traffic-and-scaling-1-2-weeks)
4. [Phase 4: Containers](#phase-4-containers-2-3-weeks)
5. [Phase 5: Automation (IaC + CI/CD)](#phase-5-automation-iac--cicd-2-3-weeks)
6. [Phase 6: Operations and Security](#phase-6-operations-and-security-ongoing)
7. [Phase 7: Serverless and Messaging](#phase-7-serverless-and-messaging-as-needed)
8. [Final Project](#final-project)
9. [Certification](#certification-optional)
10. [Cost Safety Rules](#cost-safety-rules)
11. [Study Habits](#study-habits)

---

## Roadmap at a Glance

```
Phase 0  VPC networking          ✅ done
   │
Phase 1  IAM · EC2 · CLI · billing
   │
Phase 2  RDS PostgreSQL · ElastiCache Redis · S3 · Secrets Manager
   │
Phase 3  ALB · Auto Scaling · Route 53 · ACM · CloudFront
   │
Phase 4  Docker · ECR · ECS Fargate · (EKS later)
   │
Phase 5  Terraform · GitHub Actions CI/CD · environments
   │
Phase 6  CloudWatch · CloudTrail · KMS · WAF · Well-Architected   (ongoing)
   │
Phase 7  Lambda · API Gateway · SQS · SNS · EventBridge · DynamoDB (as needed)
   │
Final    Full production stack, built with Terraform, deployed by CI/CD
```

| Phase | Topic | Time | Main AWS cost while learning |
|---|---|---|---|
| 0 | VPC networking | Done | NAT gateway (~$32/month) if left running |
| 1 | Core foundations | 1–2 weeks | Free tier |
| 2 | Data layer | 1 week | Free tier (micro DB / cache) |
| 3 | Traffic and scaling | 1–2 weeks | ALB (free tier 12 months, if eligible), domain (~$12/year) |
| 4 | Containers | 2–3 weeks | Fargate (small hourly cost) |
| 5 | Automation | 2–3 weeks | Same as resources created |
| 6 | Operations and security | Ongoing | Mostly free / low |
| 7 | Serverless and messaging | As needed | Mostly free tier |

---

## Phase 0: VPC Networking (done)

What I already know:

- [x] VPC belongs to a **Region**, subnet belongs to an **AZ**, EC2 belongs to an **AZ**
- [x] CIDR math (`/16`, `/24`, `/27`), AWS reserves 5 IPs per subnet
- [x] Public vs private subnets (decided by the **route table**, not a setting)
- [x] Internet Gateway, route tables, `0.0.0.0/0` default route, `local` route
- [x] NAT gateway (outbound only, SNAT + PAT, connection tracking, cost)
- [x] Public vs private NAT, VPC peering, Transit Gateway, VPN, Direct Connect
- [x] Security groups (stateful) vs NACLs (stateless), inbound vs outbound
- [x] ENI (virtual network card, holds IPs and security groups)
- [x] VPC DNS (Route 53 Resolver at VPC+2, DHCP option sets, private hosted zones)

My VPC design:

| Subnet | AZ | CIDR | Holds |
|---|---|---|---|
| public-1 | us-east-1a | `10.0.0.0/27` | ALB, NAT gateway, bastion |
| public-2 | us-east-1b | `10.0.0.32/27` | ALB |
| app-1 | us-east-1a | `10.0.0.64/27` | 3 services |
| app-2 | us-east-1b | `10.0.0.96/27` | 3 services (copies) |
| data-1 | us-east-1a | `10.0.0.128/27` | PostgreSQL, Redis |
| data-2 | us-east-1b | `10.0.0.160/27` | PostgreSQL standby, Redis replica |

Remaining VPC labs:

- [ ] Enable **DNS hostnames** on my VPC
- [ ] Lab: launch public EC2 in public-1, SSH in, install nginx
- [ ] Lab: launch private EC2 in app-1, reach it through the bastion, test NAT with `curl ifconfig.me`
- [ ] Lab: break things on purpose (remove routes, SG rules, NACL deny) and fix them
- [ ] Lab: S3 gateway endpoint (free), test `aws s3 ls` without NAT
- [ ] Lab: VPC Flow Logs to CloudWatch, read ACCEPT / REJECT records

---

## Phase 1: Core Foundations (1–2 weeks)

| Topic | What to learn |
|---|---|
| **IAM** | Users, groups, **roles**, policies (JSON), least privilege, MFA, never use root daily |
| **EC2** | Instance types, AMIs, key pairs, user data, EBS volumes and snapshots, instance metadata (IMDSv2), stop vs terminate |
| **AWS CLI** | `aws configure`, profiles, running common commands |
| **Billing safety** | Budgets and alerts, Free Tier usage, Cost Explorer |

Checklist:

- [ ] Enable MFA on the root account, create an admin IAM user, stop using root
- [ ] Write a custom IAM policy (e.g. read-only access to one S3 bucket)
- [ ] Attach an **IAM role** to EC2 (instead of putting access keys on the server)
- [ ] Launch EC2 with a **user data** script that installs nginx automatically
- [ ] Create an EBS snapshot and restore it to a new volume
- [ ] Install and configure the AWS CLI; list EC2 instances and S3 buckets from the terminal
- [ ] Create a budget alert (e.g. $5/month)

**Build:** an EC2 web server that configures itself on boot, with a role (no keys on disk).

---

## Phase 2: Data Layer (1 week)

| Topic | What to learn |
|---|---|
| **RDS PostgreSQL** | DB subnet groups, parameter groups, automated backups and snapshots, Multi-AZ vs read replicas, connecting from EC2 |
| **ElastiCache Redis** | Cluster mode vs single node, replicas, encryption in transit, connecting from EC2 |
| **S3** | Buckets, objects, storage classes, bucket policies, block public access, presigned URLs, lifecycle rules, versioning |
| **Secrets Manager / Parameter Store** | Storing DB passwords and API keys instead of hard-coding them |

Checklist:

- [ ] Create DB subnet group (data-1, data-2) and launch RDS PostgreSQL (free tier micro, Single-AZ)
- [ ] Connect from a private EC2 with `psql`
- [ ] Create cache subnet group and launch ElastiCache Redis (micro, 1 node); test with `redis-cli ping`
- [ ] Take a manual RDS snapshot and restore it
- [ ] Create an S3 bucket, upload a file, generate a presigned URL
- [ ] Store the DB password in Secrets Manager and read it from EC2 using the instance role

**Build:** private EC2 → RDS PostgreSQL + Redis in data subnets, password from Secrets Manager.

---

## Phase 3: Traffic and Scaling (1–2 weeks)

| Topic | What to learn |
|---|---|
| **ALB** | Target groups, listeners, health checks, path-based routing (`/users` → service 1, `/orders` → service 2) |
| **Auto Scaling Groups** | Launch templates, desired/min/max, scaling policies, self-healing |
| **Route 53** | Hosted zones, A and alias records, health checks, routing policies |
| **ACM** | Free SSL/TLS certificates for HTTPS on the ALB |
| **CloudFront** | CDN in front of S3 or the ALB, caching |

Checklist:

- [ ] ALB in public-1 + public-2, targets in app-1 + app-2
- [ ] Path-based routing to 3 target groups (one per service)
- [ ] Auto Scaling Group with a launch template; terminate an instance and watch it come back
- [ ] Point a domain at the ALB with a Route 53 alias record
- [ ] HTTPS with an ACM certificate, redirect HTTP → HTTPS
- [ ] Host a static site on S3 behind CloudFront

**Build:** `https://api.mydomain.com` → ALB → 3 services in private subnets.

---

## Phase 4: Containers (2–3 weeks)

| Topic | What to learn |
|---|---|
| **Docker** | Dockerfile, images, layers, docker-compose (learn locally first) |
| **ECR** | Private image registry, push/pull, lifecycle policies |
| **ECS with Fargate** | Clusters, task definitions, services, task roles, service discovery, rolling deploys. **Best fit for 3 services** |
| **EKS** (later) | Kubernetes on AWS. Learn after ECS; control plane alone costs ~$73/month |

Checklist:

- [ ] Dockerize the 3 services and run them locally with docker-compose (+ Postgres + Redis)
- [ ] Push images to ECR
- [ ] Run each service as an ECS Fargate service in private subnets behind the ALB
- [ ] Use a task role for AWS access and Secrets Manager for env secrets
- [ ] Deploy a new image version with a rolling update
- [ ] Add S3 gateway endpoint + ECR interface endpoints to reduce NAT costs

**Build:** 3 containerized services on ECS Fargate, connected to RDS and Redis.

---

## Phase 5: Automation (IaC + CI/CD) (2–3 weeks)

| Topic | What to learn |
|---|---|
| **Terraform** | Providers, resources, variables, outputs, modules, remote state (S3 + locking), `plan` / `apply` / `destroy` |
| **CI/CD** | GitHub Actions (or CodePipeline): build → test → push image to ECR → deploy to ECS |
| **Environments** | Separate dev and prod (workspaces or separate state / accounts) |

Checklist:

- [ ] Rebuild the whole VPC (subnets, IGW, NAT, route tables, SGs) in Terraform
- [ ] Add RDS, Redis, ALB and ECS to Terraform as modules
- [ ] Store Terraform state in S3 with locking
- [ ] `terraform destroy` everything and recreate it in one command
- [ ] GitHub Actions pipeline: push to `main` → build image → push to ECR → deploy to ECS
- [ ] Use GitHub OIDC with an IAM role (no long-lived AWS keys in GitHub)

**Build:** the entire stack as code, deployed automatically on every push. **Most valuable skill on this list.**

---

## Phase 6: Operations and Security (ongoing)

| Topic | What to learn |
|---|---|
| **CloudWatch** | Logs, metrics, alarms, dashboards, Container Insights |
| **CloudTrail** | Audit log of who did what in the account |
| **KMS** | Encryption keys for RDS, S3, EBS, Secrets Manager |
| **WAF** | Protect the ALB from SQL injection, XSS, bad bots, rate limiting |
| **GuardDuty** | Threat detection |
| **Well-Architected Framework** | 6 pillars: operational excellence, security, reliability, performance, cost, sustainability |

Checklist:

- [ ] Ship service logs to CloudWatch Logs
- [ ] Create alarms (high CPU, 5xx errors on ALB, RDS free storage low) with SNS email alerts
- [ ] Build a CloudWatch dashboard for the stack
- [ ] Turn on CloudTrail and find a recent API call
- [ ] Attach AWS WAF with managed rules to the ALB
- [ ] Review the stack against the Well-Architected pillars

---

## Phase 7: Serverless and Messaging (as needed)

| Topic | When it's useful |
|---|---|
| **Lambda + API Gateway** | Small APIs, scheduled jobs, event handlers |
| **SQS** | Queue between services (async jobs, retries, decoupling) |
| **SNS** | Fan-out notifications |
| **EventBridge** | Event bus, scheduled rules, cross-service events |
| **DynamoDB** | NoSQL key-value workloads |

Checklist:

- [ ] Lambda triggered by an S3 upload
- [ ] Service 1 sends jobs to SQS, service 2 consumes them (with a dead-letter queue)
- [ ] Scheduled EventBridge rule running a Lambda nightly

---

## Final Project

```
Users
  │
Route 53 (api.mydomain.com)
  │
CloudFront + ACM (HTTPS) + WAF
  │
ALB (public-1, public-2)
  │
ECS Fargate: service-1 · service-2 · service-3   (app-1, app-2)
  │        └──── SQS between services
  ├── RDS PostgreSQL (Multi-AZ)                  (data-1, data-2)
  ├── ElastiCache Redis                          (data-1, data-2)
  └── Secrets Manager · S3

Built with Terraform · deployed by GitHub Actions · monitored by CloudWatch
```

- [ ] Whole stack in Terraform
- [ ] CI/CD deploys on push
- [ ] HTTPS on a custom domain
- [ ] Alarms and dashboard
- [ ] Architecture diagram + README in this repo
- [ ] Cost estimate written down (monthly)

---

## Certification (optional)

**AWS Certified Solutions Architect – Associate (SAA-C03)** matches Phases 0–3 and 6–7 closely. Good next step after the foundations.

- [ ] Finish Phases 0–3
- [ ] Practice exams
- [ ] Book the exam

---

## Cost Safety Rules

- [ ] Budget alert set before starting any phase
- [ ] **Delete NAT gateways when not in use**, then **release the Elastic IP**
- [ ] Stop or terminate EC2, RDS and ElastiCache after each session
- [ ] Delete unused ALBs, EBS volumes, snapshots and Elastic IPs
- [ ] Prefer `terraform destroy` once Phase 5 is done
- [ ] Check **Billing → Free Tier** weekly (rules differ for accounts created after July 2025)

Clean-up order: **EC2 → ALB → RDS → ElastiCache → NAT gateway → Elastic IP**. VPC, subnets, route tables, IGW and security groups are free.

---

## Study Habits

- **Build, then break.** Troubleshooting teaches more than setup.
- **One note file per topic** in this repo, with a "What I built" section.
- **Console first, then CLI, then Terraform** for every service.
- **Draw the architecture** before building anything.
- **Write down costs** for every resource you create.

> AWS prices, quotas and free tier rules change. Check the official AWS pricing pages and documentation before relying on any number in this file.

[⬅ Back to AWS VPC](./netwroking/aws-vpc.md)
