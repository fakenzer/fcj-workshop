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
        max_resources_per_message = config.get('max_resources_per_message', 10)  # Default to 10
        print(f"[INFO] Loaded notification config: max_resources_per_message={max_resources_per_message}")
    except Exception as e:
        print(f"[ERROR] Failed to load SSM config: {e}")
        return {'statusCode': 500, 'error': 'SSM config load failed'}

    # Extract resources from event
    resources = event.get('resources', [])
    if not resources:
        print("[ERROR] No resources provided in event")
        return {'statusCode': 400, 'error': 'No resources provided'}

    print(f"[INFO] Processing {len(resources)} resources for notification")

    # Initialize results
    results = []
    failed_resources = []
    account_id = boto3.client('sts').get_caller_identity()['Account']
    region = boto3.Session().region_name
    topic_arn = f"arn:aws:sns:{region}:{account_id}:resource-management-alerts"

    # Group resources into chunks to avoid SNS message size limit
    for i in range(0, len(resources), max_resources_per_message):
        chunk = resources[i:i + max_resources_per_message]
        print(f"[DEBUG] Processing resource chunk {i//max_resources_per_message + 1} with {len(chunk)} resources")

        # Format SNS message for the chunk
        message_lines = ["AWS Resource Management Alert\n"]
        for resource in chunk:
            resource_arn = resource.get('resource_arn', 'Unknown')
            action = resource.get('action', 'Unknown')
            details = resource.get('details', '')
            tags = resource.get('tags', {})
            service = resource.get('service', 'unknown')
            region = resource.get('region', 'unknown')
            extra_info = resource.get('extra_info', {})

            message_lines.append(f"Resource: {resource_arn}")
            message_lines.append(f"Service: {service}")
            message_lines.append(f"Region: {region}")
            message_lines.append(f"Action: {action}")
            message_lines.append(f"Details: {details}")
            if extra_info:
                message_lines.append(f"Extra Info: {json.dumps(extra_info, indent=2)}")
            message_lines.append(f"Tags: {json.dumps(tags, indent=2)}")
            message_lines.append(f"Time: {datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S')} UTC")
            message_lines.append("-" * 50)

        message = "\n".join(message_lines)
        subject = f"AWS Resource Alert: {len(chunk)} resources"

        # Send SNS notification
        try:
            print(f"[INFO] Publishing to SNS topic: {topic_arn}")
            sns.publish(
                TopicArn=topic_arn,
                Message=message,
                Subject=subject[:100]  # SNS subject limit is 100 characters
            )
            print(f"[SUCCESS] SNS notification sent for {len(chunk)} resources")
            results.extend([{
                'statusCode': 200,
                'message': 'SNS notification sent',
                'resource_arn': resource['resource_arn']
            } for resource in chunk])
        except Exception as e:
            print(f"[ERROR] SNS publish failed for chunk: {e}")
            failed_resources.extend([{
                'statusCode': 500,
                'error': str(e),
                'resource_arn': resource['resource_arn']
            } for resource in chunk])

    # Log summary
    print(f"[SUMMARY] Processed {len(resources)} resources")
    print(f"[SUMMARY] Successful notifications: {len(results)}")
    print(f"[SUMMARY] Failed notifications: {len(failed_resources)}")
    if failed_resources:
        print(f"[ERROR] Failed resources: {[r['resource_arn'] for r in failed_resources]}")

    # Return combined results
    return {
        'statusCode': 200 if not failed_resources else 500,
        'results': results + failed_resources,
        'summary': {
            'total_resources': len(resources),
            'successful': len(results),
            'failed': len(failed_resources)
        }
    }
```

4. Click the "Deploy" button to deploy the code

#### Timeout and Memory

Appropriate configuration for notification function:

- **Timeout**: 30 seconds (sufficient for SNS publish)
- **Memory**: 128MB (minimal for text processing)
- **Concurrent executions**: 10 (avoid spam notifications)

This Lambda function ensures the team is notified promptly about non-compliant resources, helping maintain compliance and optimize costs.
