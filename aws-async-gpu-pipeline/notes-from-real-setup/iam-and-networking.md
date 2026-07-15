# IAM & Networking Configuration

This document outlines the networking architecture and security posture for a private, highly-isolated GPU processing environment on AWS.

## Network Architecture & VPC Endpoints

To maintain strict security boundaries and optimize data transfer costs, ECS GPU workers should run in private subnets without direct internet access via NAT Gateways wherever possible. 
Instead, rely on **AWS PrivateLink VPC Endpoints** for AWS service communication.

### Required VPC Endpoints
- **Amazon S3**: Gateway endpoint for artifact download/upload (no data transfer costs).
- **Amazon CloudWatch Logs**: Interface endpoint (`com.amazonaws.<REGION>.logs`) for container log streaming.
- **Amazon SQS**: Interface endpoint for queue interaction (if ECS needs to read queues directly, though in our architecture Lambda handles SQS).
- **AWS STS**: Interface endpoint for temporary credential assumption.
- **AWS Secrets Manager**: Interface endpoint to fetch runtime secrets without internet access.
- **Amazon ECR (API & DKR)**: Interface endpoints if you plan to pull container images dynamically rather than baking them into the AMI.

### Security Group Constraints
VPC Interface Endpoints rely on Security Groups.
- **ECS Worker SG**: Controls the GPU instance outbound access.
- **VPC Endpoint SG**: Must explicitly allow inbound `HTTPS (TCP/443)` from the **ECS Worker SG**. 

> [!CAUTION]
> **CloudWatch Connectivity Blackholes**: A common failure point in private subnets is forgetting to allow the ECS Worker SG to communicate with the CloudWatch Logs VPC Endpoint SG over port 443. This results in the ECS Task failing to launch, or hanging indefinitely while trying to initialize the `awslogs` driver.

---

## IAM Roles Matrix

Strict least-privilege IAM roles are required across the orchestration stack.

### 1. Lambda Execution Role
Assumed by the orchestrator function.
**Required Permissions:**
- `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes`
- `ecs:RunTask` (restricted to the specific Task Definition ARN and Cluster).
- `iam:PassRole` (restricted to the ECS Task Execution Role and Task Role).
- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`.

### 2. ECS Instance Role (EC2 Profile)
Assumed by the underlying `g4dn` EC2 instance.
**Required Permissions:**
- `ecs:RegisterContainerInstance`, `ecs:Submit*` (typically via the `AmazonEC2ContainerServiceforEC2Role` managed policy).
- `ssm:UpdateInstanceInformation` (to allow AWS Systems Manager Session Manager access for debugging without SSH keys).
- Permissions to read specific S3 buckets if the AMI bootstrapping scripts require it.

### 3. ECS Task Execution Role
Assumed by the ECS Agent on the host to prepare the container environment.
**Required Permissions:**
- `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:GetDownloadUrlForLayer` (if pulling images).
- `logs:CreateLogStream`, `logs:PutLogEvents` (crucial for container stdout streaming).
- `secretsmanager:GetSecretValue` (if injecting secrets into the container environment).

### 4. ECS Task Role
Assumed by the application container itself at runtime.
**Required Permissions:**
- `s3:GetObject`, `s3:PutObject` (scoped exactly to the input/output bucket ARNs).
- Any other specific AWS APIs the AI application calls during processing.
