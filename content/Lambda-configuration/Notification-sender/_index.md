---
title: "Creating NotificationSender Lambda Function"
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

In this section, we will create the **NotificationSender** Lambda function to send alert notifications about AWS resources that don't comply with regulations.

## Lambda Function: NotificationSender

#### Purpose

This function will perform important functions:

- Receive information from ResourceScanner about resources that need alerts
- Format notifications with detailed information about resources and violations
- Send email alerts via Amazon SNS
- Load notification configuration from SSM Parameter Store
- Provide detailed logging for tracking notifications

#### Workflow

NotificationSender is triggered when:

1. **ResourceScanner** detects resources missing required tags
2. **Step Functions** passes resource information to NotificationSender
3. Function loads configuration from SSM and formats notification
4. Sends email alert via configured SNS topic
5. Returns success/failure result

#### Event Input Format

The function receives events with the format:

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

#### Steps to Create Lambda Function

##### Step 1: Access Lambda Console

1. Log into AWS Console
2. Search for **"Lambda"** in the search bar
3. Select Lambda service

##### Step 2: Create New Function

1. Click the **"Create function"** button
2. Choose **"Author from scratch"**

##### Step 3: Configure Function

**Function configuration**:

- **Function name**: `NotificationSender`
- **Runtime**: `Python 3.10`

**Execution role**:

1. Find and click on **"Change default execution role"**
2. Select **"Use an existing role"**
3. In the dropdown, choose the **"ResourceManagerRole"** created earlier

##### Step 4: Deploy Code

1. Click **"Create function"** to create the Lambda function
2. Delete the existing sample code in the editor
3. Paste the following Python code into the editor:

```python
import json
import boto3
from datetime import datetime

def lambda_handler(event, context):
    print("[DEBUG] Notification Lambda triggered")
    print(f"[DEBUG] Event received: {json.dumps(event)}")

    ssm = boto3.client('ssm')
    sns = boto3.client('sns')

    # Load SNS config from SSM Parameter Store
    try:
        config_param = ssm.get_parameter(Name='/resource-management/notification-config')
        config = json.loads(config_param['Parameter']['Value'])
        print("[INFO] Loaded notification config")
    except Exception as e:
        print(f"[ERROR] Failed to load SSM config: {e}")
        return {'statusCode': 500, 'error': 'SSM config load failed'}

    # Extract input data
    resource_arn = event.get('resource_arn', 'Unknown')
    action = event.get('action', 'Unknown')
    details = event.get('details', '')
    tags = event.get('tags', {})

    print(f"[INFO] Sending alert for: {resource_arn} | action: {action}")

    # Format SNS message
    message = f"""
AWS Resource Management Alert

Resource: {resource_arn}
Action: {action}
Details: {details}
Tags: {json.dumps(tags, indent=2)}
Time: {datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S')} UTC
"""

    # Send SNS
    try:
        account_id = boto3.client('sts').get_caller_identity()['Account']
        region = boto3.Session().region_name
        topic_arn = f"arn:aws:sns:{region}:{account_id}:resource-management-alerts"

        print(f"[INFO] Publishing to SNS topic: {topic_arn}")
        sns.publish(
            TopicArn=topic_arn,
            Message=message,
            Subject=f"AWS Resource Alert: {action}"
        )
        print("[SUCCESS] SNS notification sent")

        return {
            'statusCode': 200,
            'message': 'SNS notification sent',
            'resource_arn': resource_arn
        }

    except Exception as e:
        print(f"[ERROR] SNS publish failed: {e}")
        return {
            'statusCode': 500,
            'error': str(e),
            'resource_arn': resource_arn
        }
```

4. Click the "Deploy" button to deploy the code

**Logging Levels**:

```
[DEBUG] - Detailed event and processing information
[INFO] - Important information about notifications
[ERROR] - Errors that occur during sending
[SUCCESS] - Confirmation when sending is successful
```

##### Timeout and Memory

Appropriate configuration for notification function:

- **Timeout**: 30 seconds (sufficient for SNS publish)
- **Memory**: 128MB (minimal for text processing)
- **Concurrent executions**: 10 (avoid spam notifications)

This Lambda function ensures the team is notified promptly about non-compliant resources, helping maintain compliance and optimize costs.
