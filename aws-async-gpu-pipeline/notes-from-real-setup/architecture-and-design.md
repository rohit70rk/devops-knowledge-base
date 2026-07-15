# AWS Async GPU Pipeline Architecture

This document provides a high-level overview of an AWS-native, event-driven, scale-to-zero architecture designed for asynchronous GPU processing workloads (e.g., video transcription, AI model inference, deep learning tasks).

## Objective

- Eliminate always-running GPU infrastructure and third-party vendor dependencies.
- Scale GPU workers from zero only when processing jobs are queued.
- Return to zero infrastructure cost automatically upon completion.
- Provide reliable, observable, and failure-tolerant execution.

---

## High-Level Component Diagram

```text
┌─────────────────────┐
│   Backend API       │
│      (EC2)          │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│       SQS           │
│  processing-queue   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│      Lambda         │
│   Orchestrator      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│      Amazon ECS     │
│      RunTask        │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Capacity Provider   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Auto Scaling Group  │
│   Min = 0           │
│   Desired = 0       │
│   Max = N           │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ GPU EC2 Worker      │
│  (e.g., g4dn.*)     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ AI Processing       │
│ Docker Container    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Webhook Callback    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Backend Database    │
└─────────────────────┘
```

---

## Core Processing Flow

### Step 1 – Job Creation
A client uploads data to cloud storage (e.g., Amazon S3). The backend creates a database record marking the job state as `processing` and publishes a message to an Amazon SQS queue.

**Payload Example:**
```json
{
  "input": {
    "job_id": 12345,
    "source_url": "https://signed-url-for-s3-object",
    "callback_url": "https://backend.api.internal/webhook"
  }
}
```

### Step 2 – Lambda Orchestration
The SQS message triggers an AWS Lambda function. 
The Lambda's sole responsibility is orchestration:
1. Parse the SQS message.
2. Call `ecs.run_task()` to schedule an ECS task.
3. Pass the SQS payload directly into the ECS container via an environment variable (e.g., `JOB_INPUT_JSON`).
4. Log the ECS Task ID back to the backend's internal observability API.

> [!NOTE]
> Lambda does **not** process data or download large files. It executes in milliseconds, keeping orchestration costs negligible.

### Step 3 – Capacity Provider Scaling
When Lambda schedules the ECS task, the cluster's ASG Desired Capacity is currently `0`.
The **ECS Capacity Provider** detects the pending task and automatically requests capacity, updating the ASG Desired Capacity from `0 → 1`.

### Step 4 – GPU Worker Startup
An EC2 instance launches using a highly-optimized, pre-baked AMI.
The AMI contains:
- Operating System & NVIDIA Drivers
- Docker & ECS Agent
- The AI Processing Container Image
- Large pre-downloaded AI Models (e.g., HuggingFace weights)

This eliminates multi-gigabyte downloads during cold starts.

### Step 5 – AI Processing
The GPU task executes its logic:
1. Downloads the source artifact.
2. Runs the heavy AI inference on the GPU.
3. Generates the structured output.
4. Uploads results to S3 (if applicable) and executes the `POST callback_url` to notify the backend.

### Step 6 – Automatic Scale Down
Upon completion, the container exits with code `0`.
The ECS task moves to `STOPPED`.
The Capacity Provider detects no further pending tasks and initiates a scale-in event, terminating the GPU instance.
Infrastructure returns to `0` capacity, incurring no idle costs.

---

## Failure Handling

- **SQS Durability**: Jobs remain in the queue until the Lambda successfully orchestrates the ECS task.
- **ECS Launch Failures**: If ECS fails to accept the task (e.g., quota limits), Lambda fails, and the message returns to SQS for automatic retry.
- **Processing Failures**: If the AI container crashes, it reports failure via the webhook callback, or the backend timeouts the job using an internal reconciliation loop.
- **Dead Letter Queue (DLQ)**: Poison pill payloads that repeatedly crash the Lambda are sent to an SQS DLQ for manual inspection.
