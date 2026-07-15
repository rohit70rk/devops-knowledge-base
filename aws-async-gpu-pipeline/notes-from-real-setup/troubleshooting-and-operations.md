# Troubleshooting & Operations Runbook

When debugging the asynchronous GPU processing pipeline, the most efficient strategy is to trace the request sequentially across the architectural boundaries.

```text
Backend → SQS → Lambda → ECS → GPU Worker → Webhook
```

---

## 1. SQS Queue Issues
**Symptom:** Queue depth increases steadily, messages remain visible, and no ECS tasks appear.
- **Verify Lambda Trigger**: Check if the SQS Event Source Mapping on the Orchestrator Lambda is enabled.
- **Check DLQ**: If messages are moving to the Dead Letter Queue, the Lambda is failing repeatedly (e.g., payload parsing exceptions, or AWS quota limits hit during `ecs.run_task`).

## 2. Lambda Execution Issues
**Symptom:** Lambda executes, but no ECS task is created.
- **Check Lambda Logs**: Trace the execution in CloudWatch (`/aws/lambda/<ORCHESTRATOR_NAME>`).
- **Common Failure 1 (IAM)**: Lambda lacks `ecs:RunTask` or `iam:PassRole`.
- **Common Failure 2 (Payload parsing)**: A mismatch between the JSON structure emitted by the backend and the structure expected by Lambda. Ensure the orchestrator injects the raw payload into the ECS environment cleanly without over-parsing.

## 3. ECS Task Stuck in PENDING
**Symptom:** Task remains `PENDING` for several minutes instead of transitioning to `RUNNING`.
- **Check Auto Scaling Group**: Ensure the ASG Desired Capacity successfully scaled from 0 → 1. 
- **Check Capacity Provider**: Ensure the Capacity Provider is correctly attached to the ECS Cluster.
- **AWS Quotas**: If the ASG fails to launch an instance, check the EC2 Activity History. You may have hit a vCPU quota for On-Demand G-series instances.

## 4. ECS `RESOURCE:GPU` Error
**Symptom:** Task fails instantly with a `RESOURCE:GPU` exception.
- **Root Cause**: The ECS Agent running on the EC2 instance failed to detect the NVIDIA GPU hardware.
- **Resolution**: Ensure you are using an ECS-Optimized GPU AMI (or a custom AMI built upon one). Verify that the NVIDIA drivers are functioning (`nvidia-smi` works on the host) and that `ECS_ENABLE_GPU_SUPPORT=true` is set in `/etc/ecs/ecs.config`.

## 5. CloudWatch Log Initialization Timeout
**Symptom:** ECS task attempts to start, but the ECS console shows errors retrieving logs, or the task fails to transition to `RUNNING`.
- **Root Cause**: If the instance is in a private subnet, the ECS agent cannot reach the CloudWatch Logs API.
- **Resolution**: Ensure the VPC Interface Endpoint for CloudWatch Logs (`com.amazonaws.<REGION>.logs`) is active, and its Security Group permits inbound TCP/443 traffic from the ECS Worker Security Group.

## 6. S3/Artifact Download Failures
**Symptom:** Application logs inside the container show `403 Forbidden` when attempting to fetch the source artifact.
- **Root Cause**: Passing raw S3 object URIs requires the ECS Task Role to have `s3:GetObject` permissions. If passing HTTP URLs, they must be pre-signed.
- **Resolution**: Generate and pass a Presigned URL from the backend to the SQS queue, allowing the container to download the file without complex IAM credential handshakes.

---

## Validating GPU Utilization

If processing is succeeding but you suspect it's falling back to CPU execution, validate GPU utilization:
1. Connect to the running `g4dn` EC2 instance (e.g., via AWS Systems Manager Session Manager).
2. Execute:
   ```bash
   watch -n 2 nvidia-smi
   ```
3. **Expected Output**: GPU Utilization should spike (e.g., 90-100%), and GPU Memory Usage should show active allocations (e.g., several GBs used by Python processes). If utilization remains at 0%, the AI framework within the container failed to bind to CUDA.
