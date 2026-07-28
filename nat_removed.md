# NAT Gateway Removal & VPC Endpoints
### Kallcon Backend Infrastructure Documentation

> **Project:** Kallcon Backend
>
> **Cloud Provider:** AWS
>
> **Compute:** Amazon ECS Fargate
>
> **Container Registry:** Amazon ECR
>
> **Database:** Amazon RDS PostgreSQL
>
> **Region:** ap-south-1 (Mumbai)
>
> **Architecture:** Private ECS Tasks (No NAT Gateway)

---

# 📖 Background

Initially, the ECS Fargate tasks were deployed inside **private subnets** with a **NAT Gateway**.

```
Private ECS
      │
      ▼
 NAT Gateway
      │
      ▼
 Internet
      │
      ▼
 AWS Services
```

The infrastructure worked correctly.

Later, to reduce infrastructure cost, the NAT Gateway was removed.

Immediately after removing it, ECS deployments started failing.

---

# 🎯 Goal

Run ECS Fargate tasks **inside private subnets without a NAT Gateway** while maintaining access to all required AWS services through **VPC Endpoints**.

---

# 🏗 Final Architecture

```
                           Internet Gateway
                                   │
                     ┌─────────────┴─────────────┐
                     │                           │
               Public Subnets              Private Subnets
                     │                           │
                     │                     ECS Fargate Tasks
                     │                           │
                     │          ┌────────────────┼────────────────┐
                     │          │                │                │
                     ▼          ▼                ▼                ▼
                Application    ECR API       CloudWatch       Parameter Store
                Load Balancer  Endpoint      Logs Endpoint    (SSM)
                                │                │                │
                                ▼                ▼                ▼
                             Interface       Interface       Interface
                              Endpoint        Endpoint        Endpoint

                                     │
                                     ▼
                              Amazon ECR DKR
                              Interface Endpoint

                                     │
                                     ▼
                               Amazon S3
                             Gateway Endpoint

                                     │
                                     ▼
                             Amazon KMS
                          Interface Endpoint
```

---

# ❌ Initial Failure

After deleting the NAT Gateway:

```
ResourceInitializationError

unable to pull registry auth from Amazon ECR

GetAuthorizationToken

dial tcp xx.xx.xx.xx:443

i/o timeout
```

This indicated that ECS could no longer reach Amazon ECR.

---

# 🔍 Root Cause

Removing the NAT Gateway removed outbound internet access from the private subnets.

Although ECS tasks were inside the VPC, they still needed network connectivity to AWS managed services.

Without a NAT Gateway, this connectivity must be provided using **VPC Endpoints**.

---

# ✅ Required VPC Endpoints

## 1. Amazon ECR API

```
com.amazonaws.ap-south-1.ecr.api
```

Purpose:

- Authenticate with Amazon ECR
- Retrieve authorization token
- Download image metadata

---

## 2. Amazon ECR Docker

```
com.amazonaws.ap-south-1.ecr.dkr
```

Purpose:

- Pull Docker image manifests
- Download container image metadata

---

## 3. Amazon S3 Gateway Endpoint

```
com.amazonaws.ap-south-1.s3
```

Purpose:

Although images are stored in Amazon ECR, the **image layers themselves are stored in Amazon S3**.

Without this endpoint, ECS cannot download image layers.

Endpoint Type:

```
Gateway Endpoint
```

---

## 4. AWS Systems Manager (SSM)

```
com.amazonaws.ap-south-1.ssm
```

Purpose:

Retrieve application configuration stored inside Parameter Store.

Examples:

- JWT_SECRET
- Database Password
- Database Username

---

## 5. AWS KMS

```
com.amazonaws.ap-south-1.kms
```

Purpose:

Decrypt SecureString parameters stored inside Parameter Store.

---

## 6. Amazon CloudWatch Logs

```
com.amazonaws.ap-south-1.logs
```

Purpose:

- Create log streams
- Send application logs
- Validate awslogs configuration

Without this endpoint, ECS produced:

```
failed to validate logger args

The task cannot find the Amazon CloudWatch log group

There is a connection issue between the task and Amazon CloudWatch.
```

---

# 🔐 Endpoint Security Group

A dedicated security group was created.

```
kallcon-vpc-endpoint-sg
```

Inbound

```
HTTPS (443)

Source:

kallcon-ecs-auth-sg
```

Outbound

```
HTTPS (443)

Destination:

kallcon-ecs-auth-sg
```

---

# ⚠ Important Discovery

Initially, all endpoints were created successfully.

However, the endpoint ENIs were attached to the **default security group**, not the intended endpoint security group.

This caused:

```
dial tcp xx.xx.xx.xx:443

i/o timeout
```

Lesson:

> Never assume the endpoint uses the correct Security Group.
Always verify the ENI attachment.

---

# 📋 Troubleshooting Timeline

## Phase 1

Error:

```
unable to retrieve secrets from SSM
```

Investigation:

- Parameter Store
- IAM
- KMS

Result:

Missing Interface Endpoints.

---

## Phase 2

Error:

```
unable to pull registry auth from Amazon ECR

dial tcp 10.x.x.x:443

i/o timeout
```

Investigation:

- Route Tables
- Security Groups
- Network ACLs
- Endpoint Security Groups
- Endpoint ENIs

Result:

Endpoint ENIs attached to incorrect Security Group.

---

## Phase 3

Error:

```
failed to validate logger args

The task cannot find the Amazon CloudWatch log group
```

Investigation:

CloudWatch Logs connectivity.

Result:

Missing CloudWatch Logs Interface Endpoint.

---

## Phase 4

Task Status

```
RUNNING ✅
```

Deployment completed successfully.

---

# 📚 Lessons Learned

## Never remove a NAT Gateway blindly.

Understand every AWS service the workload depends on.

---

## Interface Endpoints are service-specific.

Each AWS service requires its own Interface Endpoint.

Examples:

- ECR API
- ECR Docker
- CloudWatch Logs
- KMS
- SSM

---

## S3 uses a Gateway Endpoint.

Unlike most AWS services, Amazon S3 uses a Gateway Endpoint instead of an Interface Endpoint.

---

## Verify the Endpoint ENIs.

Do not stop after creating an endpoint.

Always verify:

- Correct Subnets
- Correct Security Group
- Private DNS Enabled

---

## Read the Exact ECS Error

Every error represented a different startup phase.

Examples:

```
Unable to retrieve secrets
```

↓

Parameter Store

---

```
Unable to pull registry auth
```

↓

Amazon ECR

---

```
Failed to validate logger args
```

↓

CloudWatch Logs

Understanding the error helped identify exactly which AWS dependency failed.

---

# 🏁 Final Result

Infrastructure now runs successfully with:

- ✅ No NAT Gateway
- ✅ ECS Fargate
- ✅ Private Subnets
- ✅ Amazon ECR
- ✅ Amazon RDS
- ✅ Parameter Store
- ✅ CloudWatch Logs
- ✅ GitHub Actions
- ✅ GitHub OIDC
- ✅ VPC Interface Endpoints
- ✅ S3 Gateway Endpoint

---

# 🎯 Key Takeaway

> Running ECS Fargate inside private subnets without a NAT Gateway is fully supported by AWS.

However, every AWS dependency must be reachable through the appropriate **VPC Endpoint**.

When troubleshooting ECS startup failures:

1. Read the exact `ResourceInitializationError`.
2. Identify which AWS service is failing.
3. Verify the corresponding VPC Endpoint.
4. Check the endpoint ENI, security group, subnet, and Private DNS.
5. Fix one dependency at a time until the task reaches the `RUNNING` state.

---

**Author:** Kallcon DevOps Documentation

**Purpose:** Internal infrastructure runbook for debugging NAT-less ECS Fargate deployments.