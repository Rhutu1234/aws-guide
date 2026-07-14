# AWS Compute Options: Lambda and ECS

*A practical guide to two of AWS's core compute services — Lambda for serverless, event-driven code execution, and ECS for orchestrating Docker containers — covering how each works, when to choose which, and how to deploy .NET workloads to them.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [AWS Lambda](#1-aws-lambda)
3. [Amazon ECS](#2-amazon-ecs)
4. [ECS Launch Types: Fargate vs. EC2](#3-ecs-launch-types-fargate-vs-ec2)
5. [Scaling Models Compared](#4-scaling-models-compared)
6. [Networking and Security](#5-networking-and-security)
7. [CI/CD and Deployment](#6-cicd-and-deployment)
8. [Cost Models](#7-cost-models)
9. [Choosing Between Them](#8-choosing-between-them)
10. [Combining Both](#9-combining-both)
11. [Quick Reference Table](#quick-reference-table)
12. [Conclusion](#conclusion)

---

## Introduction

AWS offers a wide spectrum of compute options, but two of the most commonly reached for — especially for backend and microservice workloads — sit at different points on the "how much do I manage" spectrum: **Lambda**, where you supply only a function and AWS handles everything else, and **ECS**, where you supply a container image and AWS orchestrates running it across a fleet of compute you either manage directly or hand off to Fargate.

```
Less operational control, more managed  ◄──────────────────►  More control, more responsibility
           AWS Lambda                    ECS on Fargate         ECS on EC2
```

This guide covers both in depth, then compares them directly so you can match the right tool to a given workload.

---

## 1. AWS Lambda

**AWS Lambda** is AWS's serverless compute service — you upload code (or a container image built to the Lambda runtime spec), Lambda runs it in response to a trigger, and you pay only for the compute time actually consumed, down to the millisecond.

### Anatomy of a Lambda function

```csharp
public class Function
{
    public async Task<APIGatewayProxyResponse> FunctionHandler(
        APIGatewayProxyRequest request, ILambdaContext context)
    {
        context.Logger.LogInformation($"Processing request: {request.Path}");

        var product = await GetProductAsync(request.PathParameters["id"]);

        return new APIGatewayProxyResponse
        {
            StatusCode = product is not null ? 200 : 404,
            Body = JsonSerializer.Serialize(product),
            Headers = new Dictionary<string, string> { ["Content-Type"] = "application/json" }
        };
    }
}
```

Every Lambda function has a **handler** (the entry point AWS calls) and runs inside a managed execution environment that AWS provisions, reuses across invocations when possible, and tears down after a period of inactivity.

### Event sources (triggers)

Lambda functions are invoked by **event sources** — AWS services or API calls that trigger execution:

| Event source | Typical use |
|---|---|
| **API Gateway** | HTTP APIs and REST APIs |
| **SQS** | Queue-based asynchronous processing |
| **S3** | React to object uploads/deletes |
| **DynamoDB Streams** | React to table changes |
| **EventBridge** | Scheduled (cron) jobs, event-driven architectures |
| **SNS** | Fan-out notifications |
| **Kinesis** | Streaming data processing |

```csharp
public class SqsFunction
{
    public async Task FunctionHandler(SQSEvent evnt, ILambdaContext context)
    {
        foreach (var record in evnt.Records)
        {
            var order = JsonSerializer.Deserialize<Order>(record.Body);
            await ProcessOrderAsync(order);
        }
    }
}
```

### Scheduled Lambdas (cron-style jobs)

```json
{
  "ScheduleExpression": "cron(0 2 * * ? *)",
  "Targets": [{ "Arn": "arn:aws:lambda:us-east-1:123456789:function:NightlyCleanup" }]
}
```

A lightweight alternative to running a continuously-hosted background worker for a job that only needs to run briefly, on a schedule.

### Cold starts and provisioned concurrency

The first invocation after a period of inactivity (or a scale-out event) pays the cost of initializing a fresh execution environment — loading the runtime, your code, and running any static initialization — before the handler itself runs. For .NET, this cold-start cost is typically more noticeable than for lighter runtimes like Node.js or Python, because the .NET runtime and JIT have more to initialize.

```csharp
// Static initialization runs once per environment, not per invocation —
// put expensive setup (DB connections, HTTP clients) here, not in the handler
private static readonly HttpClient _httpClient = new();

public async Task<string> FunctionHandler(string input, ILambdaContext context)
{
    // handler body runs on every invocation
}
```

For latency-sensitive APIs where cold starts are unacceptable, **Provisioned Concurrency** keeps a specified number of execution environments pre-initialized and ready:

```bash
aws lambda put-provisioned-concurrency-config \
    --function-name my-function \
    --qualifier prod \
    --provisioned-concurrency-config ProvisionedConcurrentExecutions=5
```

This trades away some of the "pay only when it runs" benefit (you pay for provisioned capacity whether or not it's actively invoked) in exchange for consistently low latency.

### .NET-specific performance options

- **Native AOT** — compiling a .NET Lambda function with Native AOT (available via the `Amazon.Lambda.RuntimeSupport` package's AOT support) eliminates JIT compilation from the cold-start path entirely, often cutting cold-start time dramatically for .NET functions specifically.
- **Lambda SnapStart** (currently for Java, with expanding runtime support) — takes a snapshot of an initialized execution environment and restores from it on subsequent cold starts, rather than reinitializing from scratch.

### Execution limits worth knowing

- **Timeout**: configurable up to 15 minutes maximum — Lambda is not designed for long-running processes.
- **Memory**: configurable from 128 MB up to 10,240 MB; CPU allocation scales proportionally with memory, so increasing memory can also speed up CPU-bound work.
- **Payload size**: 6 MB (synchronous invocations), 256 KB (asynchronous) — large payloads need to go through S3 or another intermediate store instead of being passed directly.

### Best fit

Event-driven processing (queue/S3/stream triggers), HTTP APIs with spiky or unpredictable traffic, scheduled jobs that run briefly, and workloads where "pay only for actual execution time" matters more than avoiding cold-start latency entirely.

---

## 2. Amazon ECS

**Amazon ECS (Elastic Container Service)** is AWS's native container orchestration service — you define your application as one or more **task definitions** (essentially a container spec, similar in spirit to a Kubernetes pod spec), and ECS schedules and runs them across a **cluster**.

### Core concepts

- **Task definition** — a blueprint describing one or more containers: image, CPU/memory, environment variables, port mappings, logging configuration.
- **Task** — a running instance of a task definition (comparable to a Kubernetes pod).
- **Service** — keeps a specified number of tasks running continuously, replacing any that fail, and integrates with load balancers.
- **Cluster** — the logical grouping of compute (EC2 instances or Fargate capacity) that tasks run on.

### A task definition

```json
{
  "family": "product-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "product-api",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/product-api:latest",
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/product-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### Creating a service

```bash
aws ecs create-service \
    --cluster my-cluster \
    --service-name product-api-service \
    --task-definition product-api \
    --desired-count 3 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[subnet-abc123],securityGroups=[sg-xyz789],assignPublicIp=ENABLED}" \
    --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=product-api,containerPort=8080"
```

ECS then keeps exactly `desired-count` tasks running, replacing any that crash or fail their health checks, and (if configured with a load balancer) automatically registers/deregisters tasks from the target group as they start and stop.

### Auto-scaling a service

```bash
aws application-autoscaling register-scalable-target \
    --service-namespace ecs \
    --resource-id service/my-cluster/product-api-service \
    --scalable-dimension ecs:service:DesiredCount \
    --min-capacity 2 --max-capacity 10

aws application-autoscaling put-scaling-policy \
    --service-namespace ecs \
    --resource-id service/my-cluster/product-api-service \
    --scalable-dimension ecs:service:DesiredCount \
    --policy-name cpu-scaling \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration '{
        "TargetValue": 70.0,
        "PredefinedMetricSpecification": { "PredefinedMetricType": "ECSServiceAverageCPUUtilization" }
    }'
```

### Best fit

Containerized applications and microservices already packaged as Docker images, teams wanting AWS-native orchestration without adopting Kubernetes, and workloads that need to run continuously rather than in short, event-triggered bursts.

---

## 3. ECS Launch Types: Fargate vs. EC2

ECS itself is the orchestration layer; you separately choose *where* the containers actually run.

### Fargate — serverless containers

```bash
--launch-type FARGATE
```

With Fargate, AWS manages the underlying compute entirely — you specify CPU/memory per task, and AWS provisions and manages the infrastructure transparently. There are no EC2 instances to patch, size, or manage capacity for.

- **Pros**: no server management at all, fine-grained per-task billing, simpler operational model.
- **Cons**: less control over the underlying instance type, typically a higher per-vCPU/GB cost than equivalent self-managed EC2 capacity, and some advanced networking/GPU scenarios aren't supported.

### EC2 launch type — self-managed instances

```bash
--launch-type EC2
```

With the EC2 launch type, you provision and manage an Auto Scaling Group of EC2 instances yourself (running the ECS agent), and ECS schedules tasks onto that existing fleet of capacity.

- **Pros**: full control over instance types (including GPU instances, Spot instances for cost savings, specific instance families), the ability to bin-pack many tasks tightly onto larger, more cost-efficient instances, and potentially lower cost at steady, high utilization.
- **Cons**: you're responsible for patching, scaling, and right-sizing the underlying instances yourself — real operational overhead ECS itself doesn't remove.

### Practical guidance

| | Fargate | EC2 launch type |
|---|---|---|
| Ops overhead | Lowest | Higher — you manage the instance fleet |
| Cost efficiency at scale | Lower per-unit efficiency | Can be more efficient with careful bin-packing/Spot usage |
| Startup/scale speed | Fast, no instance provisioning wait | Depends on Auto Scaling Group warm capacity |
| GPU/specialized instances | Limited support | Full EC2 instance type flexibility |
| Best for | Most services, especially with variable/moderate scale | High, steady-state scale where infrastructure cost efficiency matters most |

Most teams start with **Fargate** by default and only move specific, high-volume services to the EC2 launch type once the cost/control tradeoff becomes worth the added operational responsibility.

---

## 4. Scaling Models Compared

| | Lambda | ECS (Fargate) | ECS (EC2) |
|---|---|---|---|
| Scale unit | Individual function invocation | Task | Task (bin-packed onto existing instances) |
| Scale to zero | Yes | No (minimum running tasks, typically ≥1 for availability) | No |
| Cold start | Yes, more pronounced for .NET | No (task keeps running once started) | No |
| Scaling trigger | Automatic, per invocation/event | Application Auto Scaling (target tracking, step scaling) | Application Auto Scaling (tasks) + Auto Scaling Group (instances) |
| Max concurrency | Very high, near-instant (subject to account limits) | Bounded by configured max task count and cluster capacity | Bounded by both task count and underlying EC2 capacity |

---

## 5. Networking and Security

### Lambda

- Functions can optionally run inside a VPC to reach private resources (RDS, internal services), though this used to add cold-start latency — modern Hyperplane ENI improvements have largely mitigated this.
- IAM execution roles grant a function only the specific AWS permissions it needs (least-privilege), rather than broad account-wide access.
- Environment variables can be encrypted with KMS for secrets, though AWS Secrets Manager or Parameter Store are generally preferred for actual credentials.

### ECS

- The `awsvpc` network mode gives each task its own elastic network interface (ENI) and private IP, making security group rules apply per-task rather than per-host.
- Task IAM roles (distinct from the roles that let the ECS agent itself function) grant each task fine-grained AWS permissions, mirroring Lambda's per-function execution role model.
- Secrets integrate via AWS Secrets Manager or Systems Manager Parameter Store, injected directly into the container's environment at task startup rather than baked into the image.

### Shared building blocks

Both commonly sit behind an **Application Load Balancer (ALB)** for HTTP traffic (Lambda via API Gateway's integration, or directly for container-based Lambda; ECS services register directly with an ALB target group), and both can be fronted by **CloudFront** and protected by **AWS WAF** at the edge — the choice between Lambda and ECS is about compute hosting, not how traffic reaches the application.

---

## 6. CI/CD and Deployment

### Lambda

```yaml
# GitHub Actions
- run: dotnet lambda deploy-function my-function --function-role my-lambda-role
```

Or via AWS SAM/CDK for infrastructure-as-code deployment, including gradual traffic shifting between versions:

```yaml
# SAM template snippet
Type: AWS::Serverless::Function
Properties:
  AutoPublishAlias: live
  DeploymentPreference:
    Type: Canary10Percent5Minutes  # shift 10% of traffic, then the rest after 5 minutes if healthy
```

### ECS

```yaml
- run: |
    aws ecr get-login-password | docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com
    docker build -t product-api .
    docker tag product-api:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/product-api:${{ github.sha }}
    docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/product-api:${{ github.sha }}
    aws ecs update-service --cluster my-cluster --service product-api-service --force-new-deployment
```

ECS supports built-in **rolling updates** (replace old tasks with new ones gradually, respecting a minimum healthy percentage) and, via **AWS CodeDeploy** integration, **blue/green deployments** — provisioning an entirely new task set alongside the old one and shifting traffic over, with automated rollback if health checks fail.

---

## 7. Cost Models

| | Lambda | ECS Fargate | ECS EC2 |
|---|---|---|---|
| Billing basis | Per-invocation + GB-second of memory×duration, billed to the millisecond | Per-task, per-vCPU and GB-memory×second while running | Per-EC2-instance-hour, regardless of task packing efficiency |
| Idle cost | Near-zero at zero traffic | You pay for running tasks even at low traffic (no scale-to-zero) | You pay for provisioned instances regardless of utilization |
| Free tier | Generous monthly free invocations + compute time | None specific to Fargate | Standard EC2 pricing (Spot/Reserved discounts available) |
| Cost efficiency at high, steady volume | Can become more expensive than always-on alternatives at very high sustained invocation rates | Moderate — you pay a premium for the "no server management" convenience | Best, if instances are well bin-packed and rightsized (especially with Spot) |

A rough rule of thumb: **Lambda** is cheapest for spiky, low-to-moderate, or unpredictable workloads; **ECS Fargate** suits continuously running services without wanting to manage instances; **ECS on EC2** (especially with Spot Instances for fault-tolerant workloads) is the most cost-efficient at high, steady, predictable scale — at the cost of managing the instance fleet yourself.

---

## 8. Choosing Between Them

```
Is this event-driven, spiky, or scheduled work
(queue/S3/stream triggers, cron jobs, unpredictable-traffic APIs)?
        │
        ├── Yes → AWS Lambda
        │
        └── No — it's a continuously running service
                │
                ├── Is it already (or easily) containerized,
                │   and does it need to run continuously
                │   regardless of traffic?
                │       │
                │       └── Yes → ECS
                │               │
                │               ├── Want zero instance management? → Fargate
                │               └── Need max cost efficiency at scale,
                │                   GPU instances, or Spot savings? → EC2 launch type
                │
                └── (If you specifically need Kubernetes portability
                    or the broader K8s ecosystem, consider EKS instead
                    of ECS — outside this guide's scope.)
```

### Concrete scenarios

- **Resizing images uploaded to S3, a few times a minute** → Lambda with an S3 trigger.
- **A REST API with steady, predictable, moderate traffic all day** → ECS (Fargate to start).
- **A fleet of 30 interdependent microservices already packaged as Docker images** → ECS, likely EC2 launch type at scale for cost efficiency, or Fargate if operational simplicity is prioritized over squeezing out infrastructure cost.
- **A nightly report generation job that runs for 3 minutes** → Lambda on an EventBridge schedule — no reason to keep a service running 24/7 for a job that fires once a day.
- **A latency-critical API where even a rare cold start is unacceptable** → Either Lambda with Provisioned Concurrency, or ECS (which has no cold-start concept once a task is running).

---

## 9. Combining Both

Just as with Azure's compute options, real AWS architectures frequently use both together rather than picking one exclusively:

- **ECS** runs the main, continuously-available application services — the customer-facing API, core business logic microservices.
- **Lambda** handles the event-driven edges around that core system — processing uploaded files, sending notifications off an SQS queue, running scheduled maintenance jobs, responding to DynamoDB stream changes.

An ECS-hosted service can publish a message to SQS that a Lambda function consumes, and a Lambda function can call an internal ECS-hosted API over the VPC — the two compute models coexist naturally within the same VPC and account, each handling the part of the workload it's best suited for.

---

## Quick Reference Table

| Concept | Lambda | ECS |
|---|---|---|
| Model | Serverless function execution | Managed container orchestration |
| Unit of deployment | Individual function (code or container image) | Task definitions run as tasks/services |
| Scale to zero | Yes | No |
| Cold start | Yes (more pronounced for .NET without AOT) | No, once a task is running |
| Max execution time | 15 minutes | Unbounded — designed for continuous processes |
| Launch types | N/A (fully managed) | Fargate (serverless) or EC2 (self-managed instances) |
| Best for | Event-driven, spiky, scheduled workloads | Continuously running containerized services |
| Ops overhead | Lowest | Low (Fargate) to moderate/high (EC2 launch type) |
| Deployment strategy | Alias-based canary/traffic shifting | Rolling updates, or blue/green via CodeDeploy |

---

## Conclusion

Lambda and ECS answer different questions about how to run code on AWS. Lambda asks "how do I run small units of code only when something specific happens, without provisioning anything or paying for idle time?" ECS asks "how do I keep containerized services running continuously and reliably, with control over how and where they're scheduled?"

Neither is a universal default — the right starting point is whichever matches the actual shape of the workload: bursty and event-driven points toward Lambda, continuous and container-native points toward ECS (Fargate first, EC2 launch type once cost efficiency at scale genuinely outweighs the added operational responsibility). Most non-trivial systems end up using both, letting Lambda handle the event-driven edges around a core of continuously running ECS services.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the cold-start surprise that taught you to respect Provisioned Concurrency (or just switch to ECS).*
