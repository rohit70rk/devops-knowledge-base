# SQS & Lambda Orchestration

This document details the configuration of the decoupling and orchestration layer that connects backend requests to the heavy GPU processing infrastructure.

## Architecture Concept

Instead of the backend invoking heavy GPU infrastructure directly via APIs, it pushes an asynchronous job to an Amazon SQS queue. An AWS Lambda function acts as a lightweight orchestrator, consuming the SQS message and launching the ECS GPU Task.

```text
Backend API → SQS Queue → Lambda Orchestrator → ECS Task
```

---

## SQS Configuration

### Primary Queue
- **Type**: Standard (FIFO not typically required if jobs are independent and idempotent).
- **Message Retention**: E.g., 4 days.
- **Visibility Timeout**: Ensure this matches or exceeds the Lambda function timeout (e.g., 30 seconds). Note: It does *not* need to match the GPU processing time, as Lambda only orchestrates the launch.
- **Payload Schema**: Keep messages small. Pass identifiers and URLs, not raw data.
  
  *Example:*
  ```json
  {
    "input": {
      "job_id": 1001,
      "source": "https://signed-url",
      "callback_url": "https://internal-api/webhook"
    }
  }
  ```

### Dead Letter Queue (DLQ)
Always configure a DLQ attached to the primary queue. 
- **Purpose**: Captures messages that repeatedly fail processing (e.g., malformed JSON that crashes the Lambda, or repeated ECS launch failures due to quota limits).
- **Review**: Monitor the DLQ. Unprocessed messages can be manually re-driven to the primary queue after the underlying bug is fixed.

---

## Lambda Orchestrator Configuration

### Lightweight Design
The Lambda function must remain extremely lightweight. Its execution time should be under a few seconds, incurring near-zero cost.
- **Memory**: 128 MB is sufficient.
- **Timeout**: 3 - 5 seconds.
- **Responsibilities**:
  1. Parse the SQS message.
  2. Invoke `ecs.run_task()`.
  3. Log the operation for observability.

**Antipattern**: *Never* download large files or perform AI inference within the Lambda. 

### ECS Orchestration (boto3 example)
The Lambda must translate the SQS payload into an ECS environment override.

```python
import boto3
import json
import os

ecs = boto3.client('ecs')

def lambda_handler(event, context):
    for record in event['Records']:
        payload = record['body']
        
        response = ecs.run_task(
            cluster=os.environ['ECS_CLUSTER'],
            taskDefinition=os.environ['ECS_TASK_DEFINITION'],
            capacityProviderStrategy=[
                {
                    'capacityProvider': os.environ['ECS_CAPACITY_PROVIDER'],
                    'weight': 1,
                    'base': 0
                }
            ],
            overrides={
                'containerOverrides': [
                    {
                        'name': os.environ['ECS_CONTAINER_NAME'],
                        'environment': [
                            {
                                'name': 'JOB_INPUT_JSON',
                                'value': payload
                            }
                        ]
                    }
                ]
            }
        )
        
        # Log the task ARN for traceability
        print(f"Launched task: {response['tasks'][0]['taskArn']}")
```

### Observability Callbacks
For advanced observability, the Lambda can make a quick HTTP POST to a backend status endpoint before exiting, registering the ECS Task ID against the Job ID. This is purely for operational tracking, not core processing state.

### Failure Behavior & At-Least-Once Delivery
If the Lambda crashes (e.g., `ecs.run_task()` throws an exception due to AWS quotas), the Lambda execution fails. The message is automatically returned to the SQS queue and retried. This provides robust at-least-once delivery guarantees.
