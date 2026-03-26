# 🚀 AWS Cheat Sheet (Quick Revision Guide)

A comprehensive AWS reference guide with real-world scenarios and clean structure.

---

## 📌 Table of Contents

1. [Load Balancers](#load-balancers)
2. [CloudFront](#cloudfront)
3. [Lambda](#lambda)
4. [DynamoDB](#dynamodb)
5. [API Gateway](#api-gateway)
6. [SAM & CDK](#sam--cdk)
7. [Cognito](#cognito)
8. [Step Functions](#step-functions)
9. [AppSync & Amplify](#appsync--amplify)
10. [STS & IAM](#sts--iam)
11. [Security (KMS, Secrets Manager)](#security)
12. [Containers (ECS, Fargate)](#containers)
13. [Elastic Beanstalk](#elastic-beanstalk)
14. [CI/CD Pipeline](#cicd-pipeline)
15. [CloudFormation](#cloudformation)
16. [Monitoring (CloudWatch, X-Ray)](#monitoring)
17. [Messaging (SQS, SNS, Kinesis)](#messaging-services)

---

## 🔄 Load Balancers

### Types of Load Balancers

| Type | Layer | Speed | Use Case |
|------|-------|-------|----------|
| **Classic LB** | Layer 4 & 7 | Moderate | Legacy applications |
| **Application LB (ALB)** | Layer 7 (HTTP/HTTPS) | 400ms | URL/hostname/params routing |
| **Network LB (NLB)** | Layer 4 (TCP/UDP) | 100ms ⚡ | Millions of requests |
| **Gateway LB** | Layer 3-7 | Ultra-fast | 3rd-party compliance, single entry/exit |

### Key Features

**ALB (Application Load Balancer)**
- Routes based on:
  - URL path
  - Hostname
  - Query parameters
- Client IP available in `X-FORWARDED-FOR` header

**NLB (Network Load Balancer)**
- Handles millions of requests/second
- 100ms latency
- Extreme performance

**Gateway LB**
- Uses GENEVE protocol (Port 6081)
- Single point of entry/exit
- 3rd-party compliance checking

---

## 🌐 CloudFront

**CDN (Content Delivery Network) Service**

### Key Features

```yaml
Geo Restrictions:       # Allow/block by country
Cross-Region Replication: # Replicate content across regions
Caching:              # Cache facility for faster delivery
Invalidation:         # Update content & purge cache
```

### Access Control

**OAC (Origin Access Control)**
- Restrict access via CloudFront (not S3 directly)
- Provides signed URLs (per file)
- Provides signed cookies (multiple files)

```bash
# Scenario: Serve private S3 content via CloudFront
1. Create OAC
2. Attach to CloudFront distribution
3. Update S3 bucket policy to allow CloudFront only
4. Generate signed URLs/cookies for users
```

---

## ⚡ Lambda

**Serverless Compute Service**

### Core Specifications

| Aspect | Details |
|--------|---------|
| **Max Execution Time** | 15 minutes |
| **Max RAM** | 10 GB |
| **Temp Storage** | 10 GB (`/tmp` directory) |
| **CPU** | Proportional to RAM |
| **Concurrent Limit** | 1000 (default) |
| **Package Size (Zipped)** | 50 MB |
| **Package Size (Unzipped)** | 250 MB |

### Invocation Types

**Synchronous**
- EC2, API Gateway, Application Load Balancer
- Request → Wait → Response

**Asynchronous**
- SNS, EventBridge, DynamoDB, SQS
- Request → Acknowledge → Process

### Advanced Features

| Feature | Purpose |
|---------|---------|
| **Event Source Mapping** | Listen to SQS/Kinesis and auto-trigger |
| **Lambda@Edge** | Manipulate requests at CloudFront edge |
| **Destinations** | Send results to SQS, SNS, Lambda, EventBridge |
| **Concurrency** | Reserved or Provisioned (warm start) |
| **Layers** | Reusable dependencies for multiple functions |
| **Aliases** | Version management (DEV, PROD, etc.) |
| **Environment Variables** | 4 KB limit for key-value pairs |

### Important Considerations

✅ **Connection Best Practices**
```python
# DO: Create DB connection at top level (reused across invocations)
import psycopg2

conn = psycopg2.connect(  # Connection created once, reused
    host="rds-instance",
    user="admin"
)

def lambda_handler(event, context):
    cursor = conn.cursor()
    # Query using existing connection
```

⚠️ **Cold Start vs Warm Start**
- **Provisioned Concurrency**: Keep instances warm (no cold start) ✨
- **Reserved Concurrency**: Some cold starts expected ❄️
- **On-Demand**: Most cold starts

⚠️ **VPC Behavior**
- By default: Lambda runs **outside VPC** → Can access internet but NOT EC2/RDS
- To access VPC resources: Attach proper IAM role + NAT gateway

### IAM Requirements

```json
{
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents",
    "xray:PutTraceSegments",
    "sqs:ReceiveMessage",
    "dynamodb:GetItem"
  ],
  "Resource": "*"
}
```

### 🧠 Real-World Scenario

```bash
# Scenario: Process user uploads from S3
1. S3 upload → triggers Lambda (event source mapping)
2. Lambda processes image (resize, convert)
3. Lambda writes result to DynamoDB (destination)
4. CloudWatch logs track execution
5. X-Ray traces performance issues

# Scenario: API with 10 million requests/day
Problem: Cold starts causing 3s latency
Solution: Use Provisioned Concurrency for consistent 100ms response
```

---

## 🗄️ DynamoDB

**NoSQL Serverless Database**

### Data Model

| Concept | Details |
|---------|---------|
| **Item** | Row in table (up to 400 KB) |
| **Attribute** | Column (can be NULL) |
| **Partition Key** | Primary key (hash key) - REQUIRED |
| **Sort Key** | Optional secondary key for range queries |

### Pricing Model

| Operation | Capacity Unit |
|-----------|---------------|
| 1 item write (≤ 1 KB) | 1 WCU |
| 1 strongly consistent read (≤ 4 KB) | 1 RCU |
| 2 eventually consistent reads (≤ 4 KB) | 1 RCU |

**RCU Calculation**: 1 RCU = 1 strongly consistent read OR 2 eventually consistent reads for ≤ 4 KB item

### Capacity Modes

```yaml
Provisioned:  # Fixed read/write capacity (cheaper if predictable)
On-Demand:    # Auto-scale (use for unpredictable workloads)
```

### Core Operations

| Operation | Limit | Notes |
|-----------|-------|-------|
| `PutItem` | 1 item | Create/overwrite |
| `UpdateItem` | 1 item | Conditional writes supported |
| `GetItem` | 1 item | | |
| `DeleteItem` | 1 item | | |
| `Scan` | 1 MB | Scans entire table (expensive) |
| `Query` | 1 MB | Scans by partition key (efficient) |
| `BatchGetItem` | 16 MB / 100 items | | |
| `BatchWriteItem` | N/A | UPDATE not supported in batch |

### Indexes

| Index Type | Scope | When Created | Use Case |
|-----------|-------|-------------|----------|
| **Local Secondary Index (LSI)** | Single partition | At table creation | Alternative sort key (≤ 10 GB) |
| **Global Secondary Index (GSI)** | Entire table | Anytime | Query by different partition key |

### Advanced Features

| Feature | Purpose |
|---------|---------|
| **PartiQL** | SQL-like query language |
| **Optimistic Locking** | Prevent concurrent conflicts |
| **DAX (DynamoDB Accelerator)** | In-memory cache for sub-millisecond latency |
| **Streams** | Audit modifications (24-hour retention) |
| **TTL** | Auto-delete items after expiration (no WCU cost) |
| **Transactions** | ACID properties (2× RCU/WCU cost) |

### Security

```bash
IAM:           # Fine-grained access control
VPC Endpoint:  # Access without internet
Encryption:    # At-rest via KMS
```

### 🧠 Real-World Scenario

```bash
# Scenario: User profile lookup by ID and email
Table: Users
- PartitionKey: UserID (unique)
- SortKey: CreatedDate (range queries)
- GSI: EmailIndex (query by email)

# Query:
1. Get user by ID → Query PartitionKey + SortKey
2. Find user by email → Query on GSI
3. Cache with DAX → Sub-millisecond response (100× faster)
4. Enable Streams → Audit all changes
```

---

## 🔌 API Gateway

**API Management & Routing Service**

### Endpoint Types

| Type | Features | Use Case |
|------|----------|----------|
| **Edge-Optimized (Default)** | Uses CloudFront | Global APIs |
| **Regional** | No CloudFront | Regional APIs (lower latency for regional users) |
| **Private** | VPC-only access | Internal APIs |

### API Types

| Type | Protocol | Auth Support | Use Case |
|------|----------|--------------|----------|
| **REST API** | HTTP/HTTPS | IAM, Cognito, Lambda Authorizer | Traditional REST services |
| **HTTP API** | HTTP/HTTPS | OAuth 2.0, OpenID Connect | Modern, lightweight APIs |
| **WebSocket API** | WebSocket | Custom auth | Real-time (chat, notifications) |

### Integration Types

```yaml
Mock:           # Return hardcoded response
HTTP:           # Forward to external HTTP endpoint
Lambda:         # Invoke Lambda function
AWS_PROXY:      # Lambda proxy integration (minimal mapping)
```

### Request/Response Transformation

**Mapping Templates**
- Transform requests before backend receives
- Transform responses before client receives
- Uses VTL (Velocity Template Language)

```vlt
#set($inputRoot = $input.path('$'))
{
  "transformed_body": $inputRoot.body,
  "timestamp": $context.requestTimeOverride.timestamp
}
```

### Features

| Feature | Details |
|---------|---------|
| **Stages** | dev, test, prod (stage variables like environment variables) |
| **Caching** | Up to 237 GB, TTL up to 5 minutes |
| **Usage Plans** | Rate limiting with API keys |
| **CORS** | Handle cross-origin requests |
| **Throttling** | Prevent abuse (429 Too Many Requests) |

### Authentication & Authorization

| Method | Type | Best For |
|--------|------|----------|
| **IAM** | Native AWS | AWS account users |
| **Cognito User Pools** | Managed auth | User registration/login |
| **Lambda Authorizer** | Custom logic | 3rd-party tokens (JWT, OAuth) |
| **Resource Policy** | Account-level | Cross-account access |

### Troubleshooting

```bash
# Issue: 429 Too Many Requests
# Solution: Check throttling settings & usage plan

# Issue: High latency
# Solution: Check CloudWatch "IntegrationLatency" metric
```

### 🧠 Real-World Scenario

```bash
# Scenario: Public API with role-based access
API Gateway (Regional)
├─ /public (no auth) → Lambda execution role: ReadOnly
├─ /users/{id} (Cognito) → Lambda execution role: ReadWrite
└─ /admin (Lambda Authorizer) → Lambda execution role: FullAccess

Features:
- Stage variables: API_URL env var per stage
- Caching: 5 min cache for GET /users
- Usage plan: 10k req/day per API key
```

---

## 🛠️ SAM & CDK

### SAM (Serverless Application Model)

**Infrastructure-as-Code for Serverless Apps**

```bash
# Commands
sam build          # Fetch dependencies, create deployment artifacts
sam package        # Package & upload to S3, generate CloudFormation template
sam deploy         # Deploy to CloudFormation

# Local Development
sam local start-api    # Run API Gateway locally
sam local invoke       # Invoke Lambda locally
```

**Template Version**
- Version `2016-10-31` = SAM template

**Features**
- Simplified CloudFormation syntax for serverless
- Supports Lambda, API Gateway, DynamoDB
- Deploy in 2 commands
- Integration with SAR (Serverless Application Repository)

### CDK (Cloud Development Kit)

**Infrastructure-as-Code in Programming Languages**

```typescript
// Write infrastructure in TypeScript, JavaScript, Python, etc.
import * as cdk from 'aws-cdk-lib';

const app = new cdk.App();
const stack = new cdk.Stack(app, 'MyStack');

new s3.Bucket(stack, 'MyBucket', {
  versioned: true
});

// CDK compiles to CloudFormation JSON/YAML
```

### SAM vs CDK

| Aspect | SAM | CDK |
|--------|-----|-----|
| **Scope** | Serverless only | All AWS services |
| **Languages** | YAML templates | TypeScript, Python, Java, etc. |
| **Abstraction** | High-level | Low-level & high-level constructs |
| **Local Testing** | Built-in (`sam local`) | Requires setup |

---

## 🔐 Cognito

**User Authentication & Authorization**

### User Pools

```yaml
Purpose:        Create Serverless user database
Use Case:       User registration, login, profile management
Integration:    ALB, API Gateway
Features:       
  - Hosted UI (customizable logo, CSS)
  - Email verification
  - MFA
  - Social login (Facebook, Google, etc.)
```

### Identity Pools

```yaml
Purpose:        Provide temporary AWS credentials to users
Use Case:       Access AWS services (S3, DynamoDB, etc.)
Users:
  - Authenticated (via Cognito User Pools, Social, SAML, OIDC)
  - Unauthenticated (guests)
Mapping:        Users → IAM roles & policies (policy variables supported)
```

### AppSync & Sync

| Service | Purpose |
|---------|---------|
| **AppSync** | GraphQL managed service with real-time WebSocket |
| **Amplify DataStore** | Sync data from device to cloud (offline-first) |

---

## 🔀 Step Functions

**Workflow Orchestration as State Machines**

### State Types

```json
{
  "Comment": "Order processing workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Pass",
      "Result": { "valid": true },
      "Next": "CheckInventory"
    },
    "CheckInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...",
      "Retry": [{"ErrorEquals": ["InventoryError"], "MaxAttempts": 2}],
      "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "OrderFailed"}],
      "Next": "ProcessPayment"
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "End": true
    },
    "OrderFailed": {
      "Type": "Fail",
      "Error": "OrderProcessingFailed",
      "Cause": "Failed to process order"
    }
  }
}
```

### State Types Reference

| State | Purpose |
|-------|---------|
| **Task** | Execute Lambda, DynamoDB, etc. |
| **Pass** | Pass input to output (inject data) |
| **Choice** | Conditional branching |
| **Wait** | Delay execution |
| **Parallel** | Execute multiple branches |
| **Map** | Iterate over array items |
| **Succeed** | Success endpoint |
| **Fail** | Failure endpoint |

### Error Handling

```yaml
Retry:          # Retry on specific errors (exponential backoff)
Catch:          # Handle errors and transition to another state
ResultPath:     # Contains error information ($, null, or path)
```

### Step Functions Types

| Type | Execution | Throughput | Cost | Use Case |
|------|-----------|-----------|------|----------|
| **Standard** | Exactly-once | 4,000/s | Standard | Business workflows |
| **Express** | At-least-once | 100,000/s | Higher | Event processing, streaming |

### Invocation Methods

```bash
API Gateway    # Synchronous HTTP request
SDK           # Programmatic invocation
EventBridge   # Event-driven trigger
Console       # Manual trigger
```

---

## 📱 AppSync & Amplify

### AppSync

**GraphQL Managed Service**

```yaml
Features:
  - Real-time data via WebSocket
  - Resolvers: Query, Mutation, Subscription
  - Data sources: Lambda, RDS, DynamoDB, ElasticSearch, HTTP
  - Authentication: API Key, IAM, Cognito, OIDC, Lambda Authorizer
```

### Amplify

**Full-Stack Web & Mobile Framework**

| Component | Purpose |
|-----------|---------|
| **Amplify Studio** | IDE for building web/mobile apps (drag-drop) |
| **Amplify CLI** | Command-line tool for setup & deployment |
| **Amplify Hosting** | Host web apps (like Netlify) |
| **Amplify UI Library** | Pre-built, AWS-integrated components |

---

## 🔑 STS & IAM

### STS (Security Token Service)

**Temporary AWS Credentials (up to 1 hour)**

| API | Purpose |
|-----|---------|
| **AssumeRole** | Assume role within same or different account |
| **AssumeRoleWithSAML** | Credentials for SAML-logged users |
| **AssumeRoleWithWebIdentity** | Credentials for IdP users (Facebook, Google, OIDC) |
| **GetSessionToken** | MFA session token for extra security |
| **GetFederationToken** | Temporary credentials for federated users |
| **GetCallerIdentity** | Get info about current IAM user/role |
| **DecodeAuthorizationMessage** | Decode denied access error messages |

### IAM Concepts

**Policy Evaluation Logic**
```
1. By default: DENY
2. Explicit ALLOW: Can access
3. Explicit DENY: Cannot access (wins over ALLOW)
   ↳ Even if S3 bucket policy says ALLOW, S3 Bucket Policy explicit DENY wins
```

**Dynamic Policy Variables**
```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::my-bucket/${aws:username}/*"
}
// User 'john' can only access my-bucket/john/ folder
```

**Policy Types**

| Type | Lifecycle | Use Case |
|------|-----------|----------|
| **Identity-based** | Until manually deleted | User, role, group permissions |
| **Resource-based** | Until manually deleted | S3 bucket, Lambda, Role trust |
| **Inline Policy** | Deleted with principal | 1:1 user-policy relationships |

**Trust Policy**
```json
{
  "Effect": "Allow",
  "Principal": {"Service": "lambda.amazonaws.com"},
  "Action": "sts:AssumeRole"
}
// Allows Lambda to assume this role (PassRole)
```

### Directory Services

| Service | Features | Use Case |
|---------|----------|----------|
| **Microsoft AD** | Full AD with forests/trees | Enterprise on-premises |
| **AWS Managed AD** | Managed AD in AWS with trust | Hybrid AD setup |
| **AD Connector** | Proxy to on-prem AD | Use existing on-prem AD |
| **Simple AD** | AWS-only AD (no on-prem link) | Small deployments |

---

## 🔒 Security

### KMS (Key Management Service)

**Encryption Key Management**

| Type | Algorithm | Use Case |
|------|-----------|----------|
| **Symmetric** | AES-256 | All data types (most common) |
| **Asymmetric** | RSA | Public/private key encryption |

**Data Encryption Strategy**
```
Data < 4 KB:          Use KMS Encrypt API directly
Data > 4 KB:          Use Envelope Encryption (GenerateDataKey API)
  ├─ KMS encrypts data key
  ├─ Encrypted data key travels with data
  └─ Recipient uses KMS to decrypt data key
```

**Volume Migration with Encryption**
```bash
1. Create snapshot of encrypted volume
2. Copy snapshot (can re-encrypt with different key during copy)
3. Create new volume from copied snapshot
```

### Secrets Manager vs Parameter Store

| Aspect | Secrets Manager ($$$) | Parameter Store ($) |
|--------|----------------------|------------------|
| **Use Case** | Database credentials, API keys | Configuration values |
| **Auto Rotation** | ✅ (Lambda-based) | ❌ (manual or Lambda via EventBridge) |
| **KMS Encryption** | Mandatory | Optional |
| **CloudFormation** | ✅ | ✅ |
| **Cost** | Higher | Lower |

### S3 SSL/TLS Enforcement

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::my-bucket/*"],
  "Condition": {
    "Bool": {"aws:SecureTransport": "false"}
  }
}
```

### CloudWatch Logs Encryption

```bash
# Encryption configured at Log Group level
# Enable KMS encryption for sensitive logs
```

---

## 🐳 Containers

### Docker Fundamentals

**Image Registries**
- **Docker Hub**: Public repository
- **Amazon ECR**: Public + private repositories (AWS-native)

**Architecture**
- VMs use Hypervisor
- Containers use Docker Daemon

### Container Management Services

| Service | Infrastructure | Serverless | When to Use |
|---------|-----------------|-----------|------------|
| **ECS (Elastic Container Service)** | EC2 managed by you | Fargate available | Simple containerized apps |
| **EKS (Elastic Kubernetes Service)** | Kubernetes clusters | Fargate available | Complex microservices |
| **Fargate** | Fully managed serverless | Yes | No infrastructure management |

### ECS (Elastic Container Service)

**Architecture**
```
ECS Cluster
├─ EC2 Instances (with ECS Agent)
│  └─ Registered to cluster
├─ ECS Tasks (running containers)
│  └─ Defined in ECS Task Definition
└─ ECS Services (orchestration)
   └─ Auto-scaling, load balancing
```

**Key Concepts**

| Concept | Details |
|---------|---------|
| **Task Definition** | Blueprint for containers (JSON format, up to 10 containers per definition) |
| **Task** | Instance of task definition (running container) |
| **Service** | Long-running tasks with auto-scaling & load balancing |
| **Cluster** | Logical grouping of EC2 instances or Fargate capacity |

**IAM Roles**
```yaml
ecsInstanceRole:     # EC2 instance to talk to ECS agent
ecsTaskRoleExecution: # Task to pull images, access CloudWatch
ecsTaskRole:         # Task permissions (S3, DynamoDB, etc.)
```

### Fargate

**Serverless Container Compute**

```yaml
Features:
  - No EC2 management
  - Each task gets private IP
  - No host port mapping (only container port)
  - Shared volume support (EFS)
  - Storage: up to 200 GB
  - Auto-scaling: Adjust task count dynamically
```

### Load Balancing & Networking

**Dynamic Port Mapping**
```
EC2 host:8000 → Container:3000 (dynamic port assignment)
↳ Allows multiple containers per host
```

**Storage**
- **EFS (Elastic File System)**: Shared across containers
- **Local storage**: Container `/tmp` (ephemeral)
- ❌ S3 NOT suitable as file system

### Task Placement Strategies

| Strategy | Behavior | Use Case |
|----------|----------|----------|
| **Binpack** | Pack on fewest instances (save money) | Cost optimization |
| **Spread** | Spread across AZs (high availability) | Production |
| **Distinct** | Each instance different OS/type | Compliance |
| **Random** | Random placement | Testing |
| **Memberof** | Custom cluster query language | Specific placement |

### Auto-Scaling

```yaml
Target Scaling:     # Scale based on CloudWatch metric
Step Scaling:       # Scale based on alarm breaches
Schedule Scaling:   # Scale based on time of day
```

### Rolling Updates (Zero-Downtime Deployment)

```yaml
minimumHealthyPercent: 100  # Minimum tasks running
desiredCount: 100           # Total desired tasks
maximumPercent: 200         # Max tasks during update (200 = 100 + 100 new)
```

### 🧠 Real-World Scenario

```bash
# Scenario: Node.js API in containers with auto-scaling

ECS Service Configuration:
- Task Definition: node:16, 512 MB RAM, Port 3000
- Service: Desired 10 tasks
- ALB: Route HTTP traffic to tasks
- Auto-scaling: Scale from 5-20 tasks based on CPU > 70%
- EFS: Shared logs volume (survives container restart)
- Update: Rolling deployment (min 50%, max 200%)

Invocation:
- EventBridge rule → Invoke ECS task (scheduled job)
- SQS message → ECS task (process item from queue)
```

---

## 🎯 Elastic Beanstalk

**Developer-Centric Deployment Service**

```yaml
Philosophy: Deploy code, Beanstalk handles infrastructure
What Beanstalk Does:
  - Auto-scaling
  - Load balancing
  - Environment management
  - Deployment orchestration
```

### Tiers

| Tier | Components | Use Case |
|------|-----------|----------|
| **Web Server** | ALB + EC2 (auto-scaling) | Web applications |
| **Worker** | SQS + EC2 (auto-scaling) | Background jobs |

### Deployment Strategies

| Strategy | Downtime | Cost | Speed | Use Case |
|----------|----------|------|-------|----------|
| **All at Once** | ❌ Yes | $ | Fast | Dev/test only |
| **Rolling** | ✅ No | $ | Moderate | Production (≤ capacity) |
| **Rolling with Batches** | ✅ No | $$ | Moderate | Full capacity during deploy |
| **Immutable** | ✅ No | $$$ | Slow | Zero downtime (high cost) |
| **Blue/Green** | ✅ No | $$$ | Moderate | Route53 weighted switching |
| **Traffic Splitting** | ✅ No | $$ | Moderate | Canary deployment (5-50% new) |

### Configuration

```bash
# Deployment
eb create my-env          # Create environment
eb deploy                 # Deploy new version
eb scale 10               # Scale to 10 instances

# Extensions (.ebextensions)
.ebextensions/
├─ logging.config        # YAML/JSON config files
├─ custom.config
└─ https.config
```

**Extensions Requirements**
- Location: `.ebextensions/` directory in root
- Format: YAML/JSON with `.config` extension
- Syntax: Use `option_settings` to modify defaults
- Capability: Add RDS, ElastiCache, DynamoDB, etc.

⚠️ **Note**: Extensions deleted when environment deleted

### Application Management

```bash
beanstalk:
  - Application Versions: Up to 1,000 stored
  - Lifecycle Policy: Auto-delete old versions
  - Cloning: Clone deployment & config (not ALB settings)
```

### Docker Support

| Docker Type | ECS Used | Auto-scaling |
|-------------|----------|--------------|
| Single container | ❌ No | ALB direct |
| Multi-container | ✅ Yes | ECS auto-scaling |

### 🧠 Real-World Scenario

```bash
# Scenario: Django app with rolling updates

Beanstalk Config:
- Platform: Python 3.9
- Tier: Web Server with ALB
- Environment: prod, dev
- Instances: 3-10 (auto-scaling on CPU)

Deployment:
eb deploy            # Old: Rolling strategy activated
                     # Servers: 1 serving, 1 new code, 1 old code
                     # → Eventually all running new code
                     # Zero downtime!
                     
Update Strategy:
minimumHealthyPercent: 67%   # At least 2/3 instances up
maximumPercent: 150%         # Max 5 instances (3 + 2 new)
```

---

## 🚀 CI/CD Pipeline

### AWS CodeCommit

**Git Repository Hosting**

```yaml
Features:
  - Private Git repositories (unlimited size)
  - Fully managed, highly available
  - AWS account security
  - Integrated with Jenkins, CodeBuild
Authentication:
  - SSH keys
  - HTTPS
Authorization:
  - IAM users/roles
```

### CodePipeline

**Visual CICD Orchestration**

```
Source (CodeCommit/GitHub/S3)
    ↓
Build (CodeBuild/Jenkins)
    ↓
Test (CodeBuild/Jenkins)
    ↓
Deploy (EC2/Beanstalk/CloudFormation)
    ↓
Success!

Artifacts: Stored in S3 between stages
Logs: CloudWatch Events visibility
```

### CodeBuild

**Build & Compile Service**

```yaml
Config File:      buildspec.yml (in root or custom path)
Build Phases:
  - install      # Install dependencies
  - pre_build    # Pre-build tasks
  - build        # Compile code
  - post_build   # Post-build tasks
Cache:            # Cache dependencies between builds
Artifacts:        # Output (build results)
Environment:      # Runner type, variables
```

```yaml
# buildspec.yml example
version: 0.2

phases:
  install:
    commands:
      - echo "Installing dependencies..."
      - npm install
  build:
    commands:
      - echo "Building app..."
      - npm run build
  post_build:
    commands:
      - echo "Build complete"

artifacts:
  files:
    - '**/*'
  name: BuildArtifact
```

### CodeDeploy

**Deployment Automation**

```yaml
Config File:      appspec.yml
Content:
  - Files section: Which files to deploy
  - Hooks: Pre-deploy, post-deploy scripts
Deployment Stages:
  - OneAtATime    # Slow, safe
  - HalfAtATime   # Moderate
  - AllAtOnce     # Fast, risky
  - Custom        # Custom percentages
```

### Related Services

| Service | Purpose |
|---------|---------|
| **CodeStar** | Integrated solution (GitHub, CodeCommit, CodeBuild, Deploy, CloudFormation, CodePipeline, CloudWatch) |
| **CodeArtifact** | Artifact repository (Maven, npm, PyPI, NuGet) |
| **CodeGuru Reviewer** | ML-powered code reviews (static analysis) |
| **CodeGuru Profiler** | ML-powered performance analysis (production) |

### 🧠 Real-World Scenario

```bash
# Scenario: Node.js app CICD pipeline

CodeCommit
    ↓ (on push to main)
CodeBuild
    ├─ npm install
    ├─ npm test
    └─ npm run build
    ↓ (success)
CodeDeploy
    ├─ Server 1: Deploy new version (in-service)
    ├─ Server 2: Deploy new version (in-service)
    └─ Server 3: Deploy new version (in-service)
    ↓ (all passing)
Beanstalk
    └─ Update environment with new version

# Rollback: CodeDeploy can redeploy previous version
```

---

## 📋 CloudFormation

**Infrastructure-as-Code Service**

### Core Concepts

```yaml
Resources:          # AWS components (EC2, S3, Lambda, etc.)
Parameters:         # Input values (user-configurable)
Mappings:          # Static lookup tables
Conditions:        # Conditional resource creation
Outputs:           # Export values for other stacks
```

### Properties Reference

| Property | Purpose | Example |
|----------|---------|---------|
| **Fn::Ref** | Reference resource or parameter | `!Ref MyBucket` |
| **Fn::GetAtt** | Get resource attributes | `!GetAtt MyBucket.Arn` |
| **Fn::Sub** | String substitution | `!Sub "arn:aws:s3:::${BucketName}"` |
| **Fn::ImportValue** | Import from other stack | `!ImportValue ExportedValue` |
| **Fn::FindInMap** | Lookup from mapping | `!FindInMap [RegionMap, !Ref AWS::Region, AMI]` |

### Pseudo Parameters (Auto-populated)

```yaml
AWS::AccountId        # Your AWS account ID
AWS::Region          # Region where stack is created
AWS::StackName       # Stack name
AWS::StackId         # Stack ID
AWS::NotificationARNs # SNS notifications
```

### Parameters with Constraints

```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - t2.small
      - t2.medium
```

### Stack Management

| Feature | Purpose |
|---------|---------|
| **StackSets** | Create/update/delete stacks across multiple accounts & regions |
| **Drift Detection** | Detect manual config changes over time |
| **Change Sets** | Preview changes before applying |
| **Stack Policies** | Prevent accidental updates/deletes |

### Template Example

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'S3 bucket with versioning'

Parameters:
  BucketName:
    Type: String
    Default: my-bucket
    Description: Name of S3 bucket

Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref BucketName
      VersioningConfiguration:
        Status: Enabled

Outputs:
  BucketArn:
    Value: !GetAtt MyBucket.Arn
    Export:
      Name: !Sub "${AWS::StackName}-BucketArn"
```

---

## 📊 Monitoring

### CloudWatch

**Metrics, Logs, Alarms**

```yaml
Custom Metrics:      # PutMetricData API
Log Retention:       # Automatic rotation
Log Insights:        # SQL-like log queries
Composite Alarms:    # Combine 2-3 alarms (AND/OR)
Events:             # Trigger actions based on metrics
Export:             # CreateExportTask (logs → S3)
```

### X-Ray

**Distributed Tracing & Performance**

```yaml
Purpose:            # Debug production issues
Architecture:       # Graph of service calls
Instrumentation:    # X-Ray daemon (for EC2)
Integration:        # Lambda, ECS, EC2, ALB, API Gateway
```

### CloudTrail

**Audit & Compliance Logging**

```yaml
Logs:              # All API calls to AWS resources
Output:            # S3 bucket for long-term storage
Compliance:        # Immutable audit trail
Insights:          # Detect unusual activities (CloudWatch Insights)
```

### 🧠 Real-World Scenario

```bash
# Scenario: Monitor microservices

CloudWatch:
- Metric: Lambda invocations per second
- Alarm: If > 1000/s, scale up Fargate
- Dashboard: Visual overview of all metrics

X-Ray:
- Trace API → Lambda → DynamoDB → S3
- Identify bottleneck (DynamoDB slow query)
- Optimize: Add DAX cache layer

CloudTrail:
- Log all IAM changes for compliance
- Detect: User deleted production Lambda function
- Alert: Send SNS notification to admins
```

---

## 📨 Messaging Services

### SQS (Simple Queue Service)

**Message Queue Service**

```yaml
Model:              Queue-based (producer-consumer)
Message Retention:  4-14 days (configurable)
Message Size:       Up to 256 KB (Extended Client for larger)
Throughput:         Unlimited
Latency:            ~10 ms
```

### Core Concepts

| Concept | Details |
|---------|---------|
| **Visibility Timeout** | Default 30s; message hidden from other consumers during processing |
| **MessageGroupId** | FIFO only; guarantees processing order |
| **ChangeMessageVisibility** | Extend timeout if processing takes longer |
| **PurgeQueue** | Delete all messages (2-60 min delay) |

### Message Processing Patterns

```bash
# Standard SQS
Producer → Queue → Consumer 1 & 2 & 3 (compete)
↳ At-least-once delivery, can have duplicates

# FIFO SQS
Producer → Queue → Consumer (ordered)
↳ Exactly-once delivery, no duplicates
↳ Throughput: 300 msg/s (batching: 3000 msg/s)
↳ MessageGroupId: Required for ordering
```

### Advanced Features

| Feature | Purpose |
|---------|---------|
| **Dead Letter Queue (DLQ)** | Store failed messages (after retries) |
| **Redrive Policy** | Route messages to DLQ after N retries |
| **Long Polling** | Wait up to 20s for messages (reduces API calls) |
| **Delay** | Postpone message delivery (0-900 seconds) |
| **Message Deduplication** | Prevent duplicate messages (5-min window) |

### Deduplication Strategies

```yaml
Method 1: MessageDeduplicationId
         Each message has unique ID
         
Method 2: ContentBasedDeduplication
         Hash message content to detect duplicates
```

### SQS Extended Client

```bash
# Problem: Message > 256 KB

Solution:
1. Store message in S3
2. Send S3 reference in SQS (< 256 KB)
3. Consumer retrieves from S3
↳ Works with messages up to 2 GB
```

### 🧠 Scenario: Order Processing

```bash
# E-commerce order workflow

Web App
  └─ Order received
     └─ POST to /orders
        └─ SQS: {"orderId": "123", "items": [...]}
           ├─ Consumer 1: Process payment
           ├─ Consumer 2: Reserve inventory
           └─ Consumer 3: Send notification
           
If Consumer fails:
  → Message DLQ after 3 retries
  → Admin reviews & redrive to main queue
```

---

### SNS (Simple Notification Service)

**Pub/Sub Messaging Service**

```yaml
Model:             Pub/Sub (one-to-many)
Topics:            Up to 100,000
Subscriptions:     Up to 12,500,000
Protocols:         Email, SMS, HTTP, Lambda, SQS, etc.
Message Size:      Up to 256 KB
```

### Comparison: SNS vs SQS

| Aspect | SNS | SQS |
|--------|-----|-----|
| **Pattern** | Pub/Sub | Queue |
| **Subscribers** | Many | One at-a-time |
| **Message Delivery** | Push | Pull |
| **Retention** | Real-time only | 4-14 days |
| **Ordering** | No FIFO | FIFO available |

---

### Kinesis

**Real-Time Data Streaming**

#### Kinesis Data Streams

```yaml
Purpose:           Collect, process, analyze streaming data
Retention:         Up to 365 days
Immutability:      Data never deleted from stream
Ordering:          By partition ID (shard)
Throughput:        1 MB/s in, 2 MB/s out per shard
```

**Capacity Modes**

```yaml
Provisioned:       Fixed shards (cheaper if predictable)
On-Demand:         Auto-scale based on traffic
```

**Data Flow**
```
Producer
  └─ PutRecord/PutRecords API
     └─ Shard (partition by ID)
        ├─ Consumer 1 (KCL - Kinesis Client Library)
        ├─ Consumer 2 (Lambda with event source mapping)
        └─ Consumer 3 (Kinesis Data Analytics)
```

**Consumer Types**

```yaml
Classic Fan-Out:   Shared throughput (not recommended)
Enhanced Fan-Out:  Dedicated throughput (recommended)
```

**Shard Management**

```bash
# Hot shard (too much traffic)
→ Use shard splitting to divide into 2 shards

# Cold shard (low traffic)
→ Use merge shard to combine with another

# Error: ThrottleProvisionedException
→ Contact AWS support to increase capacity
```

**KCL (Kinesis Client Library)**
- Java library for consuming from Kinesis
- Handles shard balancing, checkpointing, retry logic
- Can be run on EC2, Fargate, on-premises

#### Kinesis Data Firehose

```yaml
Purpose:           Load streaming data into data stores
Destinations:      S3, Redshift, Splunk, HTTP endpoint
Latency:           ~60 seconds (near real-time)
Auto-scaling:      Automatic
```

**Firehose → S3 → Analytics Workflow**
```
Stream → Firehose (60s batching) → S3 → Athena queries
↳ Perfect for log analysis, clickstream data
```

#### Kinesis Data Analytics

```yaml
Purpose:           Analyze streaming data with SQL/Flink
Input:             Kinesis Streams, Firehose
Output:            Lambda, SNS, Kinesis Streams
Limitation:        Cannot read from Firehose ❌
```

#### Kinesis Video Streams

```yaml
Purpose:           Capture, process, store video/audio
Consumers:         ML, analytics, archival
```

### 🧠 Real-World Scenario

```bash
# Scenario: Real-time clickstream analytics

Mobile App
  └─ Clicks: (userId, pageId, timestamp)
     └─ Kinesis Data Streams (provisioned: 100 shards)
        ├─ Enhanced Fan-Out Consumer 1
        │   └─ Lambda → User activity tracking
        ├─ Enhanced Fan-Out Consumer 2
        │   └─ Kinesis Data Analytics
        │       └─ Real-time dashboard
        └─ Firehose
            └─ S3 → Nightly analysis

Features:
- Partition by userId → Ensure user data goes to same shard
- KCL handles auto-scaling & checkpointing
- Analytics: Running 10-minute aggregations
- If throttling → AWS support for shard increase
```

---

## ⚡ Quick Reference

### Service Selection Matrix

| Need | Service |
|------|---------|
| API endpoint | API Gateway |
| Background jobs | Lambda + SQS or Step Functions |
| Real-time streaming | Kinesis Data Streams |
| User auth | Cognito |
| Database queries | DynamoDB or RDS |
| Container deployment | ECS + Fargate or Beanstalk |
| Workflow orchestration | Step Functions or SFn Express |
| Data warehouse | Redshift |
| Analytics | QuickSight or Athena |
| Monitoring | CloudWatch + X-Ray |

---

## 🎯 Important Reminders

✅ **Best Practices**
- Always use VPC endpoints for security
- Enable encryption at rest and in transit
- Use IAM roles, never hardcode credentials
- Enable CloudTrail for audit trails
- Monitor with CloudWatch + X-Ray
- Test deployments with blue/green or canary

⚠️ **Avoid**
- Direct access to databases from Lambda (use connection pooling)
- Using AWS root account for daily work
- Storing secrets in environment variables
- Unencrypted data in transit
- Manual infrastructure changes (use IaC)

---

## 🚀 Mental Model

```
Production Workload
├─ Load → API Gateway
├─ Process → Lambda (async via SQS)
├─ Store → DynamoDB
├─ Monitor → CloudWatch + X-Ray
├─ Secure → KMS + IAM
└─ Deploy → CloudFormation + CodePipeline
```

---

## 🧩 Your AWS Journey

You now have a comprehensive reference guide covering:
- ✅ Compute (Lambda, ECS, Beanstalk)
- ✅ Storage (S3, DynamoDB)
- ✅ Networking (API Gateway, CloudFront, ALB)
- ✅ Integration (SQS, SNS, Kinesis, Step Functions)
- ✅ Security (IAM, KMS, Cognito)
- ✅ Monitoring (CloudWatch, X-Ray, CloudTrail)
- ✅ Deployment (CloudFormation, CodePipeline, SAM, CDK)

Good luck! 🚀
