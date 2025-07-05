---
title: "Creating IAM Role"
weight: 2
chapter: false
pre: " <b> 2.2 </b> "
---

In this section, we will create an IAM Role named **"ResourceManagerRole"** with full permissions to:

- Manage AWS resources (EC2, S3, RDS)
- Execute automation workflows
- Monitor and send notifications
- Integrate with AWS services

## Role Purpose

ResourceManagerRole is designed to:

- **Service Integration**: Used by Lambda, Step Functions, Systems Manager
- **Resource Management**: Manage EC2, S3, RDS with full permissions
- **Automation**: Run automation workflows and scripts
- **Monitoring**: Send alerts and write logs

## Prerequisites

- Successfully created policies:
  - `ManageResourcePolicy` (from section 2.1)
- Have AWS Console access with permissions to create IAM Role
- Understanding of trusted entities and assume role
  ![Manage Resource Policy](/images/2.IAM/005-manageresourcepolicy.png)

## Steps to Create Role

### Step 1: Access IAM Roles

1. In AWS Console, go to IAM service
2. Find the left sidebar
3. Click on **"Roles"**
4. The IAM Roles management page will be displayed

### Step 2: Create New Role

1. Click the **"Create role"** button
2. Navigate to the "Select trusted entity" page

![Manage Resource Policy](/images/2.IAM/006-createrole.png)

### Step 3: Configure Trusted Entity

1. In the "Trusted entity type" section, select **"Custom trust policy"**
2. Paste the following JSON trust policy into the text box:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "lambda.amazonaws.com",
          "states.amazonaws.com",
          "ssm.amazonaws.com",
          "ec2.amazonaws.com",
          "events.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

3. Click "Next" to continue

### Step 4: Assign AWS Managed Policies

On the "Add permissions" page, find and select the following AWS managed policies:

#### 4.1 Lambda and Execution

```
AWSLambdaBasicExecutionRole
```

- Permission to write logs to CloudWatch
- Required for Lambda functions

#### 4.2 Notification Services

```
AmazonSNSFullAccess
```

- Send notifications via SNS
- Manage topics and subscriptions

#### 4.3 Systems Management

```
AmazonSSMFullAccess
```

- Run automation documents
- Manage parameters and patches
- Execute run commands

#### 4.4 Monitoring and Logging

```
CloudWatchFullAccess
```

- Create and manage CloudWatch metrics
- Manage alarms and dashboards
- Log management

#### 4.5 Workflow Orchestration

```
AWSStepFunctionsFullAccess
```

- Create and manage Step Functions
- Execute state machines

#### 4.6 Compute Services

```
AmazonEC2FullAccess
```

- Manage EC2 instances, volumes, snapshots
- Network and security groups

#### 4.7 Storage Services

```
AmazonS3FullAccess
```

- Manage S3 buckets and objects
- Lifecycle policies

#### 4.8 Database Services

```
AmazonRDSFullAccess
```

- Manage RDS instances
- Snapshots and backups

### Step 5: Assign Custom Policies

1. On the same "Add permissions" page, search for **"ManageResourcePolicy"**
2. Check this policy
3. Click "Next" to move to the final step

### Step 6: Complete Role

1. On the "Name, review and create" page, fill in the information:

   - **Role name**: `ResourceManagerRole`
   - **Description**: `Comprehensive role for resource management and automation`

2. Review the entire configuration:

   - Role name is correct
   - Trust policy is properly configured
   - All AWS managed policies are assigned
   - Custom policy "ManageResourcePolicy" is assigned
     ![Review Role](/images/2.IAM/007-reviewrole.png)

3. Click "Create role"
