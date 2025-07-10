---
title: "Creating Lambda Function NotificationSender"
weight: 5
chapter: false
pre: " <b> 3.5. </b> "
---

In this section, we will create the Lambda function **NotificationSender** to send alert notifications about AWS resources that don't comply with regulations.

## Lambda Function: NotificationSender

### Purpose

This function will perform the following important functions:

- Receive information from ResourceScanner about resources that need alerts
- Format notifications with detailed information about resource and violation
- Send email alerts via Amazon SNS
- Load notification configuration from SSM Parameter Store
- Provide detailed logging for tracking notifications

### Operation Flow

NotificationSender is triggered when:

1. **ResourceScanner** detects resources missing required tags
2. **Step Functions** passes resource information to NotificationSender
3. Function loads configuration from SSM and formats notification
4. Sends email alert via configured SNS topic
5. Returns success/failure result

### Event Input Format

Function receives event with format:

```json
{
  "resource_arn": "arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0",
  "action": "SEND_WARNING",
  "details": "Missing required tags: ['Environment']",
  "tags": {
    "Name": "test"
  }
}
```

### Steps to Create Lambda Function

#### Step 1: Access Lambda Console

1. Log into AWS Console
2. Search for **"Lambda"** in the search bar
3. Select Lambda service

#### Step 2: Create New Function

1. Click the **"Create function"** button
2. Choose **"Author from scratch"**

#### Step 3: Configure Function

**Function configuration**:

- **Function name**: `NotificationSender`
- **Runtime**: `Python 3.10`

**Execution role**:

1. Find and click on **"Change default execution role"**
2. Choose **"Use an existing role"**
3. In the dropdown, select the **"ResourceManagerRole"** created earlier

#### Step 4: Deploy Code

1. Click **"Create function"** to create the Lambda function
2. Delete the existing sample code in the editor
3. Paste the following Python code into the editor:

```python
import json
import boto3
from datetime import datetime

def lambda_handler(event, context):
    print("[DEBUG] Notification Lambda triggered")
    print(f"[DEBUG] Event received: {json.dumps(event, indent=2)}")

    ssm = boto3.client('ssm')
    sns = boto3.client('sns')

    # Load SNS config from SSM Parameter Store
    try:
        config_param = ssm.get_parameter(Name='/resource-management/notification-config')
        config = json.loads(config_param['Parameter']['Value'])
        max_resources_per_message = config.get('max_resources_per_message', 10)
        print(f"[INFO] Loaded config: max_resources_per_message={max_resources_per_message}")
    except Exception as e:
        print(f"[ERROR] Failed to load SSM config: {e}")
        return {
            'statusCode': 500,
            'error': 'Failed to load config from SSM'
        }

    # Validate resources input
    resources = event.get('resources', [])
    if not isinstance(resources, list):
        print("[ERROR] 'resources' must be a list")
        return {
            'statusCode': 400,
            'error': "'resources' must be a list"
        }

    if not all(isinstance(r, dict) for r in resources):
        print("[ERROR] One or more resources are not valid dictionaries")
        return {
            'statusCode': 400,
            'error': "Invalid resource format (not a dict)",
            'invalid_resources': [r for r in resources if not isinstance(r, dict)]
        }

    if not resources:
        print("[ERROR] No resources provided in input")
        return {
            'statusCode': 400,
            'error': 'No resources provided'
        }

    print(f"[INFO] Sending notifications for {len(resources)} resources")

    # Prepare results
    results = []
    failed = []
    account_id = boto3.client('sts').get_caller_identity()['Account']
    region = boto3.session.Session().region_name
    topic_arn = f"arn:aws:sns:{region}:{account_id}:resource-management-alerts"

    # Send in chunks to avoid SNS size limits
    for i in range(0, len(resources), max_resources_per_message):
        chunk = resources[i:i + max_resources_per_message]
        print(f"[DEBUG] Sending chunk {i//max_resources_per_message + 1} of {len(chunk)} items")

        message_lines = [" AWS Resource Alert \n"]
        for res in chunk:
            message_lines.append(f"Resource: {res.get('resource_arn', 'N/A')}")
            message_lines.append(f"Service: {res.get('service', 'N/A')}")
            message_lines.append(f"Region: {res.get('region', 'N/A')}")
            message_lines.append(f"Action: {res.get('action', 'N/A')}")
            message_lines.append(f"Details: {res.get('details', '')}")
            message_lines.append(f"Tags: {json.dumps(res.get('tags', {}), indent=2)}")
            message_lines.append(f"Timestamp: {res.get('timestamp', datetime.utcnow().isoformat())}")
            message_lines.append("-" * 40)

        message = "\n".join(message_lines)
        subject = f"AWS Resource Alert: {len(chunk)} resource(s)"

        try:
            sns.publish(
                TopicArn=topic_arn,
                Message=message,
                Subject=subject[:100]  # SNS subject limit
            )
            print(f"[SUCCESS] Notification sent for {len(chunk)} resource(s)")
            results.extend([{
                'statusCode': 200,
                'message': 'Notification sent',
                'resource_arn': res['resource_arn']
            } for res in chunk])
        except Exception as e:
            print(f"[ERROR] Failed to send SNS message: {e}")
            failed.extend([{
                'statusCode': 500,
                'error': str(e),
                'resource_arn': res['resource_arn']
            } for res in chunk])

    # Return summary
    summary = {
        'total_resources': len(resources),
        'successful': len(results),
        'failed': len(failed)
    }

    return {
        'statusCode': 200 if not failed else 500,
        'results': results + failed,
        'summary': summary
    }

```

4. Click the "Deploy" button to deploy the code

#### Timeout and Memory

Appropriate configuration for notification function:

- **Timeout**: 30 seconds (sufficient for SNS publish)
- **Memory**: 128MB (minimal for text processing)
- **Concurrent executions**: 10 (avoid spam notifications)

This Lambda function ensures the team is notified promptly about non-compliant resources, helping maintain compliance and optimize costs.
