# Application Hosting Patterns

This document covers common generic patterns used to host various types of applications (microservices, monoliths, open-source stacks) on the Proxmox infrastructure.

## Deployment Methodologies

Applications in this environment are typically deployed using one of three methodologies:

1. **Internal CI/CD (GitLab Runners)**
   - Used for internal proprietary apps with source code hosted on the internal GitLab instance.
   - Runners build images and push to the internal Container Registry.
   - SSH deployment script pulls the image to the `<APP_VM>` and updates the Docker Compose stack.
2. **External CI/CD (GitHub Actions)**
   - Used for applications where source code resides entirely on GitHub.
   - A dedicated deployment user (e.g., `github-deploy`) on the `<APP_VM>` receives the payload via SSH and restarts the `docker-compose` or bare-metal service.
3. **Manual / Direct Docker Compose**
   - Used for large third-party or open-source software stacks (e.g., knowledge bases, chat UIs).
   - A dedicated folder under a deployment directory (e.g., `~/apps/<SoftwareName>`) holds the `.env` and `docker-compose.yml` files.

---

## Pattern: AI Microservices Suite

When deploying a suite of AI microservices (e.g., utilizing FastAPI/Uvicorn, MCP servers, and LLM integrations):

### Architecture Structure
- **API Gateway**: A single microservice acts as the orchestrator/gateway, routing requests to downstream microservices.
- **Python ASGI Servers**: Microservices run via `uvicorn` (e.g., `uvicorn main:app --host 0.0.0.0 --port <PORT>`).
- **Model Context Protocol (MCP)**: Each core service may run an accompanying MCP server on a dedicated port to facilitate standardized AI context sharing.

### Nginx Proxy Requirements for AI/LLM Apps
AI/LLM apps often stream tokens and require long-lived connections. Their Nginx reverse proxy configurations must include:
- WebSocket support (`Upgrade` and `Connection` headers).
- Extended read timeouts (e.g., `proxy_read_timeout 300s;` or up to `900s`).
- Large body sizes if handling document uploads (`client_max_body_size 50M;`).

---

## Pattern: Large Open-Source Stacks

When hosting complex open-source applications requiring multiple embedded services (e.g., internal Vector DBs, Redis, Search engines):

### Data Segregation Strategy
- If an open-source tool requires a database *strictly for its own use* (e.g., a specific Meilisearch instance, a dedicated MongoDB), it is acceptable to deploy that database within the application's isolated Docker Compose stack on the `<APP_VM>`.
- Shared, core databases used by multiple internal applications must remain segregated on the `<DB_VM>`.

### Persistent Data Volumes
Ensure that data directories for these embedded databases are explicitly mapped to host directories (e.g., `./data-node`, `./redis-data`) to prevent data loss on container recreation.

---

## Pattern: Reverse Proxy Bypass

In rare cases, an application might require direct exposure without passing through the central `<PROXY_VM>` (e.g., a service requiring raw TCP throughput, or struggling with double-NAT proxying).

### Implementation
- **DNAT Direct**: Modify the Proxmox host's iptables DNAT rule to forward the specific port directly to the `<APP_VM>` instead of the `<PROXY_VM>`.
  ```bash
  iptables -t nat -A PREROUTING -p tcp --dport <APP_PORT> -j DNAT --to-destination <APP_VM_IP>:<APP_PORT>
  ```
- **Trade-offs**: Bypassing the proxy means losing centralized SSL termination. The application must either handle SSL itself, or remain unencrypted (HTTP only) if used strictly over secure VPNs or tunnels.

---

## Pattern: Simple API / UI Pairs

For standard full-stack applications (a frontend UI and a backend API):

### Port Mapping Strategy
- Map frontend and backend to distinct, adjacent ports on the `<APP_VM>` (e.g., `3001` for UI, `4001` for API).

### Routing Strategy
- Configure the `<PROXY_VM>` Nginx to map based on URI path.
  ```nginx
  location / {
      proxy_pass http://<APP_VM_IP>:3001;
  }
  location /api/ {
      proxy_pass http://<APP_VM_IP>:4001;
      # Disable buffering for streaming APIs if needed
      proxy_buffering off;
  }
  ```
