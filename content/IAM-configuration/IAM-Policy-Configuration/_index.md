---
title: "Create IAM Policies"
weight: 1
chapter: false
pre: " <b> 2.1 </b> "
---

In this section, we will create two important custom policies:

1. **ManageResourcePolicy**: Manage resources and costs
2. **RequireEnvironmentTagPolicy**: Require mandatory Environment tag

## Policy 1: ManageResourcePolicy

### Purpose

This policy provides permissions to:

- Manage tags for AWS resources
- Control Step Functions workflows

### Steps to create policy

#### Step 1: Access IAM Console

1. Log in to AWS Console
2. Search for **"IAM"** in the search bar
3. Select IAM service

![Choose IAM Service](/images/2.IAM/001-Chooseiam.png)

#### Step 2: Create new Policy

1. In the IAM interface, select **"Policies"** from the left menu
2. Click **"Create policy"**
   ![Choose create policy](/images/2.IAM/002-choosecreatepolicy.png)
3. Select **"JSON"** tab instead of Visual editor

#### Step 3: Configure Policy JSON

Copy and paste this JSON snippet.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "tag:GetResources",
        "tag:TagResources",
        "tag:UntagResources",
        "states:StartExecution",
        "states:DescribeExecution"
      ],
      "Resource": "*"
    }
  ]
}
```

#### Step 4: Complete Policy

1. Click **"Next"** to review
2. Fill in information:
   - **Policy name**: `ManageResourcePolicy`
   - **Description**: `Policy for managing resources, tags and Step Functions`
3. Click **"Create policy"**
   ![Review policy](/images/2.IAM/003-reviewpolicy.png)

## Policy 2: RequireEnvironmentTagPolicy

### Purpose

This policy ensures:

- All new resources must have **"Environment"** tag
- Only allows values: prod, dev, test, lab
- Prevents creation of resources without tags

### Policy JSON Configuration

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:RunInstances", "s3:CreateBucket", "rds:CreateDBInstance"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "*",
          "aws:RequestTag/Environment": ["prod", "dev", "test", "lab"]
        }
      }
    },
    {
      "Effect": "Deny",
      "Action": ["ec2:RunInstances", "s3:CreateBucket", "rds:CreateDBInstance"],
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringNotLike": {
          "aws:RequestTag/Environment": "*"
        }
      }
    }
  ]
}
```

### Steps to create RequireEnvironmentTagPolicy

#### Step 1: Create new policy

1. In IAM Console, select **"Policies"**
2. Click **"Create policy"**
3. Select **"JSON"** tab

#### Step 2: Paste JSON policy

Copy and paste the RequireEnvironmentTagPolicy JSON above

#### Step 3: Complete

1. **Policy name**: `RequireEnvironmentTagPolicy`
2. **Description**: `Enforce environment tag requirement for resource creation`
3. Click **"Create policy"**

   ![Review policy](/images/2.IAM/004-reviewpolicy2.png)

## Results

After completing this section, you will have:

1. **ManageResourcePolicy**:

   - Manage tags
   - Control Step Functions workflows

2. **RequireEnvironmentTagPolicy**:
   - Enforce environment tag requirement
   - Prevent creation of resources without tags
   - Support governance and compliance

These two policies will be used in the following sections to create roles and users with appropriate permissions.
