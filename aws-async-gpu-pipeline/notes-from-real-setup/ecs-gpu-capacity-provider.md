# ECS GPU & Capacity Provider Configuration

This document covers the configuration of the ECS cluster, Capacity Provider, and Custom AMIs required to achieve zero-idle-cost GPU processing.

## Auto Scaling Group & Capacity Provider Setup

To achieve scale-to-zero, you must attach an Auto Scaling Group (ASG) to an ECS Capacity Provider.

### ASG Configuration
- **Min Capacity**: `0`
- **Desired Capacity**: `0`
- **Max Capacity**: `>= 1` (depending on concurrency needs)

### Capacity Provider Behavior
The Capacity Provider bridges ECS and the ASG. When an ECS task is scheduled (e.g., by Lambda) and there is no EC2 instance available, the Capacity Provider signals the ASG to scale up. 
Once the task finishes and moves to `STOPPED`, the Capacity Provider signals a scale-down.

> [!TIP]
> **Auto Scaling Cooldown**: Configure a cooldown period (e.g., 600 seconds / 10 minutes) on the ASG. This prevents rapid launch/termination cycles when multiple jobs arrive slightly staggered, allowing the existing instance to process subsequent tasks.

---

## ECS Task Definition

### Resource Allocation Strategy
For heavy GPU workloads (e.g., transcription, diarization):
- Allocate **1 full GPU** to the task.
- Allocate the majority of the host's CPU and Memory, but leave adequate headroom for the OS, Docker daemon, and ECS Agent.
- *Example for a `g4dn.xlarge` (4 vCPU, 16GB RAM):* Allocate `4096` CPU units and `12288 MB` Memory.

### Environment Injection
Orchestration payloads should be injected dynamically. Use a unified environment variable (e.g., `JOB_INPUT_JSON`) to pass job configurations, source URLs, and webhook destinations into the container.

### Secrets Management
Avoid hardcoding API keys or database passwords in the task definition. 
Use **AWS Secrets Manager** and integrate it with ECS by configuring `secrets` in the task definition. The `ECS Task Execution Role` will fetch and securely inject them at runtime.

---

## The Pre-Baked GPU AMI

Relying on standard AMIs introduces massive cold-start latency due to gigabytes of Docker image pulls and AI model downloads. 

### Custom AMI Contents
Use an EC2 Image Builder pipeline (or Packer) to pre-bake a custom AMI containing:
1. Amazon Linux (or standard OS base).
2. NVIDIA Drivers & CUDA Toolkit.
3. ECS Agent & Docker.
4. **The Application Container Image** (`docker pull <APP_IMAGE>`).
5. **Pre-downloaded AI Models** (baked directly into the image or volume).

### ECS Agent Optimization (`/etc/ecs/ecs.config`)
Configure the ECS agent on the AMI to utilize the baked-in image rather than pulling from Docker Hub/ECR.

```ini
ECS_CLUSTER=<CLUSTER_NAME>
ECS_ENABLE_GPU_SUPPORT=true
ECS_IMAGE_PULL_BEHAVIOR=prefer-cached
```
Setting `prefer-cached` forces ECS to use the local container image, drastically reducing startup time and bandwidth costs.

---

## ECS Logging Strategy

Rely on the `awslogs` driver to stream container logs to CloudWatch.
- **Log Driver**: `awslogs`
- **Log Group**: e.g., `/ecs/gpu-pipeline`
- **Stream Prefix**: `ecs`

Ensure the `ECS Task Execution Role` has permissions (`logs:CreateLogStream`, `logs:PutLogEvents`) to write to CloudWatch.

---

## Execution Roles Overview

1. **ECS Instance Role**: Attached to the EC2 worker instances. Allows the host to register with the ECS Cluster, pull from ECR, and interface with SSM Session Manager.
2. **ECS Task Execution Role**: Assumed by the ECS agent to prepare the task (pulling images, creating CloudWatch streams, fetching Secrets Manager secrets).
3. **ECS Task Role**: Assumed by the container itself at runtime (e.g., granting the container permission to upload results directly to S3).
