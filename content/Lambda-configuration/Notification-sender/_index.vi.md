---
title: "Tạo Lambda Function NotificationSender"
weight: 5
chapter: false
pre: " <b> 3.5. </b> "
---

Trong phần này, chúng ta sẽ tạo Lambda function **NotificationSender** để gửi thông báo cảnh báo về các tài nguyên AWS không tuân thủ quy định.

## Lambda Function: NotificationSender

### Mục đích

Function này sẽ thực hiện các chức năng quan trọng:

- Nhận thông tin từ ResourceScanner về tài nguyên cần cảnh báo
- Format thông báo với thông tin chi tiết về resource và violation
- Gửi email alerts qua Amazon SNS
- Load cấu hình notification từ SSM Parameter Store
- Cung cấp logging chi tiết để tracking notifications

### Luồng hoạt động

NotificationSender được trigger khi:

1. **ResourceScanner** phát hiện tài nguyên thiếu required tags
2. **Step Functions** chuyển thông tin tài nguyên tới NotificationSender
3. Function load cấu hình từ SSM và format notification
4. Gửi email alert qua SNS topic đã cấu hình
5. Trả về kết quả thành công/thất bại

### Event Input Format

Function nhận event với format:

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

### Các bước tạo Lambda Function

#### Bước 1: Truy cập Lambda Console

1. Đăng nhập vào AWS Console
2. Tìm kiếm **"Lambda"** trong thanh tìm kiếm
3. Chọn dịch vụ Lambda

#### Bước 2: Tạo Function mới

1. Nhấp vào nút **"Create function"**
2. Chọn **"Author from scratch"**

#### Bước 3: Cấu hình Function

**Function configuration**:

- **Function name**: `NotificationSender`
- **Runtime**: `Python 3.10`

**Execution role**:

1. Tìm và nhấp vào mục **"Change default execution role"**
2. Chọn **"Use an existing role"**
3. Trong dropdown, chọn role **"ResourceManagerRole"** đã tạo trước đó

#### Bước 4: Deploy Code

1. Nhấp **"Create function"** để tạo Lambda function
2. Xóa code mẫu hiện có trong editor
3. Paste đoạn code Python sau vào editor:

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

4. Nhấp nút "Deploy" để triển khai code

#### Timeout và Memory

Cấu hình phù hợp cho notification function:

- **Timeout**: 30 seconds (đủ cho SNS publish)
- **Memory**: 128MB (minimal cho text processing)
- **Concurrent executions**: 10 (tránh spam notifications)

Lambda function này đảm bảo team được thông báo kịp thời về các tài nguyên không tuân thủ, giúp maintain compliance và optimize costs.
