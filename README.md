# 🚀 DevOps Knowledge Base

Welcome to my personal DevOps and Cloud Engineering knowledge base. This repository is a centralized hub of production-tested infrastructure architectures, deployment patterns, and operational runbooks. 

Many of the guides and configurations documented here are abstracted from real-world systems I designed and built from scratch-including self-hosted CI/CD platforms on Proxmox, and scalable, serverless GPU processing pipelines on AWS for AI workloads.

## 📂 Repository Index

### 🐙 GitHub Version Control
* [Git Workflow: Infrastructure Segregation](github/branch_workflow_guide.md) — Strategy for isolating Docker and CI/CD configurations using `.gitattributes` merge strategies.

### 🦊 GitLab CI/CD & Self-Hosting
* [1. GitLab CE Setup Guide](gitlab/1-gitlab-ce-setup-guide.md) — Installation, Nginx reverse proxy routing, SSL configuration, and GitLab Runner initialization.
* [2. SSH Hardening for CI/CD](gitlab/2-ssh-config-for-ci-cd.md) — Creating deployment users and securing passwordless SSH authentication for pipeline deployments.
* [3. Protect GitLab CI/CD Variables](gitlab/3-gitlab-variables-for-ci-cd.md) — Best practices for scoping, masking, and protecting deployment secrets and SSH keys.
* [4. Secure Authentication & Registry Lifecycle](gitlab/4-gitlab-auth-and-registry-lifecycle-basics.md) — Managing deploy tokens and establishing automated container registry cleanup policies.
* [5. First App Deploy with CI/CD](gitlab/5-gitlab-first-app-deploy-with-ci-cd.md) — End-to-end CI/CD pipeline creation encompassing Build, Test, Push, Deploy, and automated Rollback stages.
* [6. Centralized Email Notification](gitlab/6-gitlab-ce-email-notification.md) — Developing shared CI templates for consistent, cross-project SMTP deployment notifications.
* **Real-World Setup Notes:**
  * [Architecture & Design](gitlab/notes-from-real-setup/architecture-and-design.md)
  * [Installation & Setup](gitlab/notes-from-real-setup/installation-and-setup.md)
  * [Application Deployment Pattern](gitlab/notes-from-real-setup/application-deployment-pattern.md)
  * [CI/CD Pipelines](gitlab/notes-from-real-setup/ci-cd-pipelines.md)
  * [Operations & Maintenance](gitlab/notes-from-real-setup/operations-and-maintenance.md)

### ☸️ Kubernetes
* [Kubernetes Cluster Setup Guide](kubernetes/k8s-setup-guide.md) — Bare-metal cluster initialization, Containerd runtime configuration, and Calico CNI networking setup.

### 🖥️ Proxmox Infrastructure
* [Proxmox VM Creation Guide](proxmox-infrastructure/proxmox-vm-creation-guide.md) — Step-by-step bare-metal VM provisioning without templates, including day-one OS and UFW configurations.
* [System Information Collection Scripts](proxmox-infrastructure/collect-system-information.md) — Bash scripting for comprehensive auditing of Proxmox host health and individual VM prerequisites.
* **Real-World Setup Notes:**
  * [Architecture & Networking](proxmox-infrastructure/notes-from-real-setup/architecture-and-networking.md)
  * [Proxmox Host Configuration](proxmox-infrastructure/notes-from-real-setup/proxmox-host-configuration.md)
  * [VM Provisioning Patterns](proxmox-infrastructure/notes-from-real-setup/vm-provisioning-patterns.md)
  * [Application Hosting Patterns](proxmox-infrastructure/notes-from-real-setup/application-hosting-patterns.md)
  * [Operations & Maintenance](proxmox-infrastructure/notes-from-real-setup/operations-and-maintenance.md)

### ☁️ AWS Async GPU Pipeline (Event-Driven)
* **Real-World Setup Notes:**
  * [Architecture & Design](aws-async-gpu-pipeline/notes-from-real-setup/architecture-and-design.md)
  * [ECS GPU Capacity Provider](aws-async-gpu-pipeline/notes-from-real-setup/ecs-gpu-capacity-provider.md)
  * [SQS & Lambda Orchestration](aws-async-gpu-pipeline/notes-from-real-setup/sqs-lambda-orchestration.md)
  * [IAM & Networking](aws-async-gpu-pipeline/notes-from-real-setup/iam-and-networking.md)
  * [Troubleshooting & Operations](aws-async-gpu-pipeline/notes-from-real-setup/troubleshooting-and-operations.md)

---

## 👨‍💻 About the Author

**Rohit Kumar** *Python Developer | Cloud & DevOps Engineer*

I am a Python Developer with hands-on experience in backend engineering, cloud infrastructure, and DevOps practices. I specialize in building scalable, cloud-native applications and architecting self-hosted infrastructure environments from the ground up.

Previously, I worked as a Python Developer & DevOps Intern at Banao Technologies (ATGWorld Pvt Ltd), where I contributed to Vidya (an AI-powered personalized learning platform), built serverless GPU processing pipelines on AWS, and managed self-hosted GitLab CI/CD and Proxmox infrastructure. 

Currently, I am working as a Freelance Python Developer building scientific dataset management platforms, AI workflow orchestration, and knowledge platforms. **I am actively open to full-time opportunities** in Python Development, Backend Engineering, DevOps, Cloud Engineering, and Full-Stack Development roles.

**Tech Stack:** Python • FastAPI • Django • REST APIs • PostgreSQL • AWS • Docker • GitLab CI/CD • Linux • Proxmox VE

* **GitHub:** [@rohit70rk](https://github.com/rohit70rk)
* **LinkedIn:** [rohit70rk](https://www.linkedin.com/in/rohit70rk/)
* **Email:** [rohit70r.bth@gmail.com](mailto:rohit70r.bth@gmail.com)
