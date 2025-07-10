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

4. Nhấp nút "Deploy" để triển khai code

#### Timeout và Memory

Cấu hình phù hợp cho notification function:

- **Timeout**: 30 seconds (đủ cho SNS publish)
- **Memory**: 128MB (minimal cho text processing)
- **Concurrent executions**: 10 (tránh spam notifications)

Lambda function này đảm bảo team được thông báo kịp thời về các tài nguyên không tuân thủ, giúp maintain compliance và optimize costs.
