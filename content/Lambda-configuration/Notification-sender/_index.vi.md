---
title: "Tạo Lambda Function NotificationSender"
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

Trong phần này, chúng ta sẽ tạo Lambda function **NotificationSender** để gửi thông báo cảnh báo về các tài nguyên AWS không tuân thủ quy định.

## Lambda Function: NotificationSender

#### Mục đích

Function này sẽ thực hiện các chức năng quan trọng:

- Nhận thông tin từ ResourceScanner về tài nguyên cần cảnh báo
- Format thông báo với thông tin chi tiết về resource và violation
- Gửi email alerts qua Amazon SNS
- Load cấu hình notification từ SSM Parameter Store
- Cung cấp logging chi tiết để tracking notifications

#### Luồng hoạt động

NotificationSender được trigger khi:

1. **ResourceScanner** phát hiện tài nguyên thiếu required tags
2. **Step Functions** chuyển thông tin tài nguyên tới NotificationSender
3. Function load cấu hình từ SSM và format notification
4. Gửi email alert qua SNS topic đã cấu hình
5. Trả về kết quả thành công/thất bại

#### Event Input Format

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

#### Các bước tạo Lambda Function

##### Bước 1: Truy cập Lambda Console

1. Đăng nhập vào AWS Console
2. Tìm kiếm **"Lambda"** trong thanh tìm kiếm
3. Chọn dịch vụ Lambda

##### Bước 2: Tạo Function mới

1. Nhấp vào nút **"Create function"**
2. Chọn **"Author from scratch"**

##### Bước 3: Cấu hình Function

**Function configuration**:

- **Function name**: `NotificationSender`
- **Runtime**: `Python 3.10`

**Execution role**:

1. Tìm và nhấp vào mục **"Change default execution role"**
2. Chọn **"Use an existing role"**
3. Trong dropdown, chọn role **"ResourceManagerRole"** đã tạo trước đó

##### Bước 4: Deploy Code

1. Nhấp **"Create function"** để tạo Lambda function
2. Xóa code mẫu hiện có trong editor
3. Paste đoạn code Python sau vào editor:

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

4. Nhấp nút "Deploy" để triển khai code

**Logging Levels**:

```
[DEBUG] - Chi tiết event và processing
[INFO] - Thông tin quan trọng về notification
[ERROR] - Lỗi xảy ra trong quá trình gửi
[SUCCESS] - Confirmation khi gửi thành công
```

##### Timeout và Memory

Cấu hình phù hợp cho notification function:

- **Timeout**: 30 seconds (đủ cho SNS publish)
- **Memory**: 128MB (minimal cho text processing)
- **Concurrent executions**: 10 (tránh spam notifications)

Lambda function này đảm bảo team được thông báo kịp thời về các tài nguyên không tuân thủ, giúp maintain compliance và optimize costs.
