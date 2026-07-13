# 🚀 DevOps Knowledge Base

Welcome to my personal DevOps documentation repository. This project serves as a centralized hub for enterprise-grade infrastructure guides, configuration snippets, and CI/CD workflows. 

These documents also act as the technical backbone for the articles featured on my personal portfolio blog.

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

### ☸️ Kubernetes
* [Kubernetes Cluster Setup Guide](kubernetes/k8s-setup-guide.md) — Bare-metal cluster initialization, Containerd runtime configuration, and Calico CNI networking setup.

### 🖥️ Proxmox Infrastructure
* [Proxmox VM Creation Guide](proxmox-infrastructure/proxmox-vm-creation-guide.md) — Step-by-step bare-metal VM provisioning without templates, including day-one OS and UFW configurations.
* [System Information Collection Scripts](proxmox-infrastructure/collect-system-information.md) — Bash scripting for comprehensive auditing of Proxmox host health and individual VM prerequisites.

---

## 👨‍💻 About the Author

**Rohit Kumar** *Computer Science Graduate | Python Developer | Cloud & DevOps Engineer*

I am passionate about building robust applications and scalable infrastructure systems. I specialize in backend development (Python), containerization (Docker/Kubernetes), and architecting self-hosted infrastructure environments from the ground up.

* **GitHub:** [@rohit70rk](https://github.com/rohit70rk)
* **LinkedIn:** [@rohit70rk](https://www.linkedin.com/in/rohit70rk/)
* **Email:** [rohitkumar81953@gmail.com](mailto:rohitkumar81953@gmail.com)
