# Virtual Machine Provisioning Patterns

This document describes the standard VM roles, resource allocation patterns, and baseline configurations used in a typical microservices deployment on Proxmox.

## Common Baseline for All VMs

Regardless of their specific role, all VMs share a common baseline configuration:
- **OS**: Ubuntu Server LTS (e.g., 24.04 LTS).
- **Network Interface**: `virtio` connected to the internal bridge (e.g., `vmbr1`), with the firewall flag enabled (`firewall=1`).
- **Disk Configuration**: `virtio-scsi-single` controller with `discard=on` (TRIM support) and `iothread=1`.
- **System Users**: A primary unprivileged user matching the node's purpose (e.g., `<role>-node` or `deployer`).
- **Docker Runtime**: Most VMs run Docker CE and Docker Compose Plugin.

---

## Pattern: Application VM (`<APP_VM>`)

The Application VM hosts stateless and stateful microservices, AI models, frontend servers, and API backends.

| Resource | Typical Allocation | Rationale |
|----------|--------------------|-----------|
| **CPU** | High (e.g., 6+ cores) | Heavy processing for APIs and background tasks. |
| **RAM** | High (e.g., 24+ GB) | Required for multiple concurrent Docker containers and memory-intensive apps. |
| **Disk** | Modest (e.g., 100 GB) | Needs enough space for Docker images, volumes, and application logs. |

### Configuration Highlights
- **Deployment Paths**: Applications are grouped logically (e.g., `/opt/apps/` for CI/CD deployed apps, `/opt/manual-apps/` for manual drops).
- **Users**: Dedicated users for CI/CD deployments (e.g., `deployer` with SSH keys restricted to deployment servers).
- **Docker Networks**: Will naturally accrue many bridge networks (`br-*`) as Docker Compose stacks are spun up. Monitor Docker network overlaps if using many stacks.
- **Disk Monitoring**: Docker image layers and dangling volumes accumulate fast. High risk of disk exhaustion if cleanup isn't automated.

---

## Pattern: Database VM (`<DB_VM>`)

The Database VM isolates all database engines (PostgreSQL, MongoDB, Redis, etc.) from application compute.

| Resource | Typical Allocation | Rationale |
|----------|--------------------|-----------|
| **CPU** | Low/Modest (e.g., 2 cores) | Databases often rely more on memory and fast I/O than sheer compute. |
| **RAM** | Modest (e.g., 8 GB) | Tuned to provide adequate buffer caches. |
| **Disk** | Modest (e.g., 100 GB) | Data storage. Consider a separate mount for DB data vs OS to prevent DB filling the root partition. |

### Configuration Highlights
- **Deployment Strategy**: Databases run as isolated Docker Compose stacks (e.g., `~/databases/<project-db>/docker-compose.yml`).
- **Access**: External access routes through the `<PROXY_VM>` using HAProxy TCP forwarding, never directly exposed.
- **Backups**: Implement automated cron jobs to run `pg_dump` or `mongodump` via `docker exec`, and ship the artifacts off the VM.

---

## Pattern: Reverse Proxy VM (`<PROXY_VM>`)

The Proxy VM acts as the single entry point for all incoming traffic from the host's DNAT rules.

| Resource | Typical Allocation | Rationale |
|----------|--------------------|-----------|
| **CPU** | Low (e.g., 1 core) | Nginx and HAProxy are highly efficient and require minimal CPU. |
| **RAM** | Low (e.g., 2-3 GB) | Sufficient for SSL termination and connection tracking. |
| **Disk** | Low (e.g., 10-20 GB) | Only stores configuration files, certificates, and proxy logs. |

### Configuration Highlights
- **Software**: Bare-metal Nginx (HTTP/HTTPS) and HAProxy (TCP/Database routing). Docker is typically *not* installed to reduce overhead.
- **SSL Certificates**: Managed via Certbot.
- **Disk Monitoring**: Even a 10GB disk can fill up if Nginx access logs are not aggressively rotated.

---

## Pattern: CI/CD & Build VM (`<BUILD_VM>`)

The Build VM hosts source control, container registries, and runs CI/CD pipelines.

| Resource | Typical Allocation | Rationale |
|----------|--------------------|-----------|
| **CPU** | High (e.g., 6+ cores) | Needed for parallel Docker image builds and pipeline execution. |
| **RAM** | High (e.g., 16+ GB) | GitLab CE and Docker builds are extremely memory-intensive. |
| **Disk** | High (e.g., 250+ GB) | Must store the GitLab database, repositories, CI artifacts, and the entire Container Registry. |

### Configuration Highlights
- **Software**: GitLab CE Omnibus (bare-metal) alongside Docker for GitLab Runners.
- **Runners**: The GitLab Runner uses the `docker` executor with host socket binding (`/var/run/docker.sock`) to allow image building and pushing to the local registry.
- **Internal DNS**: Needs `/etc/hosts` entries to resolve its own public domain name back to the `<PROXY_VM>` for proper loopback routing.

---

## Pattern: Monitoring & Observability VM (`<MONITOR_VM>`)

A dedicated VM for metrics, logging, and tracing.

| Resource | Typical Allocation | Rationale |
|----------|--------------------|-----------|
| **CPU** | Modest (e.g., 3 cores) | Required for querying metrics and aggregating logs. |
| **RAM** | Modest (e.g., 8 GB) | Required for in-memory TSDB structures (Prometheus). |
| **Disk** | Modest/High (100+ GB) | Log aggregation (Loki/Elasticsearch) requires significant storage. |

### Configuration Highlights
- **Software**: Dockerized observability stack (Prometheus, Grafana, Loki, Alertmanager).
- **Network**: Needs access to scrape metrics from node-exporters and cAdvisor instances running on all other VMs.
