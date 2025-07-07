---
title: "Testing EventBridge Rule"
weight: 7
chapter: false
pre: " <b> 7.4. </b> "
---

In this section, we will change the EventBridge Rule schedule from daily to run every minute to test the Resource Management system operation.

## Step 1: Modify EventBridge Rule Schedule

### Access and Edit Rule

1. Go to **EventBridge Console**
2. Select **Rules** → Click on `ResourceManagementSchedule`
3. Click **Edit**
4. Change **Schedule pattern**

### Change Schedule Expression

**From:**

```
0 17 * * ? *  (Daily at 5:00 PM UTC)
```

**To:**

```
rate(2 minutes)
```

Or use Cron expression:

```
*/2 * * * ? *  (Every 2 minutes)
```

5. Click **Update rule**
6. Confirm the rule has been updated

## Step 2: Test System Operation

### 2.1 Check Step Functions Executions

1. Go to **Step Functions Console**
2. Select `ResourceManagementWorkflow`
3. View the **Executions** tab
4. After 2-3 minutes, new executions should appear

**States to check:**

- **SUCCEEDED**: Workflow ran successfully
- **FAILED**: An error occurred
- **RUNNING**: Currently running
- **TIMED_OUT**: Timeout occurred

![Step Functions Executions](/images/7.Test/012-execute-stepfunction.png)

### 2.2 Check Execution Details

1. Click on the most recent execution
2. View **Graph inspector** to track the flow
3. Check **Input** and **Output** of each step
4. View **Execution event history**

**Important steps:**

- `ScanResources`: Scan EC2, S3, RDS resources
- `CheckResourcesExist`: Check if there are violating resources
- `ProcessResources`: Process each resource
- `DecideAction`: Decide action (DELETE/WARNING/NO_ACTION)

### 2.3 Check CloudWatch Logs

#### ResourceScanner Lambda Logs

1. Go to **CloudWatch Console**
2. Select **Logs** → **Log groups**
3. Find `/aws/lambda/ResourceScanner`
4. View the most recent log streams

**Logs to look for:**

```
[INFO] Starting resource scan...
[INFO] Scanning EC2 instances...
[INFO] Found 3 EC2 instances
[INFO] Scanning S3 buckets...
[INFO] Found 2 S3 buckets
[INFO] Scanning RDS instances...
[INFO] Found 1 RDS instance
[INFO] Resource scan completed. Total violations: 2
```

#### ResourceDeleter Lambda Logs

1. Find `/aws/lambda/ResourceDeleter`
2. View logs when resources are deleted

**Logs to look for:**

```
[INFO] Processing resource deletion...
[INFO] Deleting EC2 instance: i-1234567890abcdef0
[INFO] EC2 instance deleted successfully
[WARNING] Cannot delete RDS instance: protection enabled
```

#### NotificationSender Lambda Logs

1. Find `/aws/lambda/NotificationSender`
2. View logs when emails are sent

**Logs to look for:**

```
[INFO] Sending notification for 2 resources
[INFO] Email sent successfully to admin@company.com
[INFO] Notification sent via SES
```

### 2.4 Check Email Notifications

**If there are violating resources, you will receive an email with content:**

```
Subject: AWS Resource Compliance Alert

Dear AWS Administrator,

The following resources have been found in violation of tagging policies:

EC2 Instances:
- i-1234567890abcdef0 (us-east-1)
  Missing tags: Environment
  Action: Will be deleted in 48 hours

S3 Buckets:
- my-test-bucket-2024
  Missing tags: Environment
  Action: Warning sent

Please add the required tags or the resources will be automatically deleted.

Best regards,
AWS Resource Management System
```

### 2.5 Check Deleted Resources

#### Check EC2 Instances

1. Go to **EC2 Console**
2. Check **Instances** → **Terminated**
3. See if instances with Environment = "lab" tag have been deleted

#### Check S3 Buckets

1. Go to **S3 Console**
2. Check if buckets without Environment tag have been deleted

#### Check RDS Instances

1. Go to **RDS Console**
2. Check if RDS instances with Environment = "lab" tag have been deleted

## Step 3: Verify Results

### Expected results after 2-5 minutes:

**EventBridge Rule triggers successfully**

- New executions appear in Step Functions

**ResourceScanner runs successfully**

- Logs show scanning EC2, S3, RDS
- Violating resources are detected

**Actions are performed correctly:**

- **DELETE_IMMEDIATE**: Resources with Environment = "lab" tag are deleted immediately
- **SEND_WARNING**: Warning emails are sent for resources missing tags
- **NO_ACTION**: Other resources wait 48h

**Notifications work:**

- Emails are sent
- Log confirmation in CloudWatch

### Troubleshooting if not working:

**No new executions:**

- Check if EventBridge Rule is ENABLED
- Check IAM permissions of ResourceManagerRole

**Execution FAILED:**

- View execution history to find the failing step
- Check Lambda function permissions
- View CloudWatch logs of Lambda functions

**No email received:**

- Check SES configuration
- Check if email address is verified
- View NotificationSender Lambda logs

## Step 4: Rollback to Production Schedule

**After testing, rollback immediately:**

1. Go to **EventBridge Console**
2. Edit rule `ResourceManagementSchedule`
3. Change schedule back to:

```
0 17 * * ? *
```

4. Click **Update rule**

{{%notice warning%}}
**Important:** Remember to rollback to daily schedule to avoid running too frequently in production!
{{%/notice%}}

## Testing Summary

The system works correctly when:

- Step Functions executions run every 2 minutes
- ResourceScanner scans EC2, S3, RDS successfully
- Violating resources are detected and processed
- Email notifications are sent
- Resources with Environment = "lab" tag are automatically deleted
- Logs record all activities completely
