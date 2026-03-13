# DevOps Roadmap 2026

This roadmap outlines a **structured 16-week journey** covering Linux, Cloud, Containers, Infrastructure as Code, CI/CD, Python automation, Kubernetes, and SRE practices.

The goal is to build **real production-ready skills through projects**, not just theoretical knowledge.

---

# Module 1 — Linux Foundations

## Week 1: Linux for DevOps + Basic Shell Scripting

### Topics

* Linux command line fundamentals
* File permissions and ownership
* Process management
* System monitoring
* Bash scripting basics:

  * Variables
  * Loops
  * Conditionals
  * Functions
  * I/O redirection

### Projects

* System monitoring script
* Automated backup and cleanup tool
* Real-time log monitoring with alerts

---

# Module 2 — Cloud (AWS) + Containers

## Week 2: AWS Fundamentals + Networking + EC2

### Topics

* Cloud computing basics
* Networking fundamentals
* VPC architecture
* EC2 instances
* Load Balancers
* Auto Scaling
* IAM
* Route53

### Projects

* Deploy web application on EC2
* Configure custom domain with SSL and Nginx
* Auto-scaling infrastructure with load testing

---

## Week 3: AWS Storage, Databases & Networking

### Topics

* S3
* EBS
* EFS
* CloudFront
* RDS
* Disaster recovery strategies

### Projects

* Static website hosting using S3 + CloudFront
* Cross-account S3 replication with lifecycle policies
* RDS disaster recovery simulation

---

## Week 4: Docker & Docker Compose

### Topics

* Docker architecture
* Images and containers
* Container networking
* Volumes
* Multi-stage builds
* Docker security best practices
* Docker Compose for multi-container applications

### Projects

* Containerize a full-stack application
* Build a real-time container monitoring dashboard with alerts

---

# Module 3 — Production Containers, Infrastructure as Code & CI/CD

## Week 5: AWS ECS + Terraform Basics

### Topics

* ECS clusters, services, tasks
* Application Load Balancers
* Kubernetes vs ECS decision-making
* Terraform fundamentals

  * Resources
  * State management
  * Modules
  * Workspaces

### Projects

* Deploy a two-tier application on ECS using Terraform
* Configure domain, SSL, autoscaling and load testing

---

## Week 6: Terraform Advanced + Git + SDLC

### Topics

* Terraform modules
* Import existing infrastructure
* Drift detection
* Multi-environment infrastructure
* Git workflows
* Software development lifecycle
* Jira-based workflow simulation

### Projects

* Convert manually created infrastructure into Terraform
* Build reusable Terraform modules
* Simulate a real team DevOps workflow

---

## Week 7: CI/CD with GitHub Actions

### Topics

* GitHub Actions fundamentals
* Workflow design
* Reusable actions
* Multi-environment deployments
* Branching strategies

### Projects

* Fully automated Terraform infrastructure deployment
* CI/CD pipeline for ECS two-tier application

---

## Week 8: Microservices on ECS + DevSecOps Introduction

### Topics

* Microservices architecture on ECS
* Dynamic Terraform environments (dev vs prod)
* OIDC keyless authentication
* Docker vulnerability scanning
* Infrastructure security scanning (Checkov, tfsec)

### Projects

* Microservices deployment pipeline
* Automated vulnerability scanning in CI pipeline
* Security linting and testing in CI

---

# Module 4 — Python for DevOps

## Week 9: Python Basics for DevOps

### Topics

* Python data structures
* CLI tools in Python
* boto3 for AWS automation
* REST API calls

### Projects

* AWS resource creation tool
* Cloud usage reporting tool using boto3

---

## Week 10: Serverless Automation with AWS Lambda

### Topics

* AWS Lambda
* EventBridge
* SQS
* SNS
* Lambda layers

### Projects

* Cloud usage report email automation
* IAM key rotation automation
* Image processing pipeline deployed via Terraform

---

## Week 11: FinOps + Security Automation

### Topics

* RDS cost analysis
* File scanning with ClamAV
* Large-scale database migration strategies

### Projects

* Automated RDS migration pipeline
* Secure inbound file processing pipeline

---

# Module 5 — Kubernetes

## Week 12: Kubernetes Basics (Minikube / Kind)

### Topics

* Pods
* Deployments
* Services
* StatefulSets
* ConfigMaps
* Network Policies

### Projects

* Deploy two-tier and three-tier applications
* Prometheus and Grafana monitoring setup

---

## Week 13: Kubernetes on AWS EKS

### Topics

* EKS cluster provisioning using Terraform
* IAM Roles for Service Accounts (IRSA)
* AWS Fargate for serverless pods
* cert-manager
* Kubernetes ingress controllers

### Projects

* Deploy production-grade three-tier application
* Configure domain, SSL, and rolling upgrades

---

## Week 14: GitOps, Helm & Kustomize

### Topics

* Helm package manager
* Kustomize configuration management
* ArgoCD GitOps workflows
* Init containers
* Kubernetes CronJobs

### Projects

* Full GitOps deployment pipeline using ArgoCD

---

## Week 15: Stateful Applications & Advanced Troubleshooting

### Topics

* StatefulSets
* Multi-AZ high availability
* Istio service mesh
* Network policies
* Advanced Kubernetes debugging techniques

### Projects

* Stateful application deployment
* Real-world debugging scenarios

---

## Week 16: Kubernetes Monitoring + SRE Practices

### Topics

* Prometheus
* Grafana
* Loki logging
* OpenTelemetry
* AWS CloudWatch
* Service Level Indicators (SLI)
* Service Level Objectives (SLO)
* Error budgets
* Incident management
* Root cause analysis (RCA)

### Projects

* Full SRE incident simulation on Kubernetes

---

# Final Outcome

After completing this roadmap you will have hands-on experience with:

* Linux system administration
* Bash automation
* AWS infrastructure
* Docker containers
* Infrastructure as Code (Terraform)
* CI/CD pipelines
* Python automation
* Kubernetes production deployments
* Observability and monitoring
* Site Reliability Engineering practices

---