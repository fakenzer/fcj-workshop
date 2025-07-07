---
title: "Kiểm tra EventBridge Rule"
weight: 7
chapter: false
pre: " <b> 7.4. </b> "
---

Trong phần này, chúng ta sẽ thay đổi lịch trình EventBridge Rule từ daily sang chạy theo phút để kiểm tra hoạt động của hệ thống Resource Management.

## Bước 1: Thay đổi Schedule EventBridge Rule

### Truy cập và chỉnh sửa Rule

1. Vào **EventBridge Console**
2. Chọn **Rules** → Click vào `ResourceManagementSchedule`
3. Nhấp **Edit**
4. Thay đổi **Schedule pattern**

### Thay đổi Schedule Expression

**Từ:**

```
0 17 * * ? *  (Daily at 5:00 PM UTC)
```

**Thành:**

```
rate(2 minutes)
```

Hoặc sử dụng Cron expression:

```
*/2 * * * ? *  (Every 2 minutes)
```

5. Nhấp **Update rule**
6. Xác nhận rule đã được cập nhật

## Bước 2: Kiểm tra hoạt động của hệ thống

### 2.1 Kiểm tra Step Functions Executions

1. Vào **Step Functions Console**
2. Chọn `ResourceManagementWorkflow`
3. Xem tab **Executions**
4. Sau 2-3 phút, sẽ thấy executions mới xuất hiện

**Các trạng thái cần kiểm tra:**

- **SUCCEEDED**: Workflow chạy thành công
- **FAILED**: Có lỗi xảy ra
- **RUNNING**: Đang chạy
- **TIMED_OUT**: Timeout

![Step Functions Executions](/images/7.Test/012-execute-stepfunction.png)

### 2.2 Kiểm tra Chi tiết Execution

1. Click vào execution gần nhất
2. Xem **Graph inspector** để theo dõi flow
3. Kiểm tra **Input** và **Output** của từng step
4. Xem **Execution event history**

**Các step quan trọng:**

- `ScanResources`: Quét tài nguyên EC2, S3, RDS
- `CheckResourcesExist`: Kiểm tra có tài nguyên vi phạm không
- `ProcessResources`: Xử lý từng tài nguyên
- `DecideAction`: Quyết định hành động (DELETE/WARNING/NO_ACTION)

### 2.3 Kiểm tra CloudWatch Logs

#### ResourceScanner Lambda Logs

1. Vào **CloudWatch Console**
2. Chọn **Logs** → **Log groups**
3. Tìm `/aws/lambda/ResourceScanner`
4. Xem log streams gần nhất

**Log cần tìm:**

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

1. Tìm `/aws/lambda/ResourceDeleter`
2. Xem logs khi có tài nguyên bị xóa

**Log cần tìm:**

```
[INFO] Processing resource deletion...
[INFO] Deleting EC2 instance: i-1234567890abcdef0
[INFO] EC2 instance deleted successfully
[WARNING] Cannot delete RDS instance: protection enabled
```

#### NotificationSender Lambda Logs

1. Tìm `/aws/lambda/NotificationSender`
2. Xem logs khi gửi email

**Log cần tìm:**

```
[INFO] Sending notification for 2 resources
[INFO] Email sent successfully to admin@company.com
[INFO] Notification sent via SES
```

### 2.4 Kiểm tra Email Notifications

**Nếu có tài nguyên vi phạm, sẽ nhận email với nội dung:**

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

### 2.5 Kiểm tra Tài nguyên bị Xóa

#### Kiểm tra EC2 Instances

1. Vào **EC2 Console**
2. Kiểm tra **Instances** → **Terminated**
3. Xem instances có tag Environment = "lab" đã bị xóa chưa

#### Kiểm tra S3 Buckets

1. Vào **S3 Console**
2. Kiểm tra buckets không có tag Environment có bị xóa không

#### Kiểm tra RDS Instances

1. Vào **RDS Console**
2. Kiểm tra RDS instances có tag Environment = "lab" đã bị xóa chưa

## Bước 3: Xác minh kết quả

### Kết quả mong đợi sau 2-5 phút:

**EventBridge Rule trigger thành công**

- Executions mới xuất hiện trong Step Functions

  **ResourceScanner chạy thành công**

- Log shows scanning EC2, S3, RDS
- Phát hiện được tài nguyên vi phạm

  **Hành động được thực hiện đúng:**

- **DELETE_IMMEDIATE**: Tài nguyên có tag Environment = "lab" bị xóa ngay
- **SEND_WARNING**: Email cảnh báo được gửi cho tài nguyên thiếu tag
- **NO_ACTION**: Tài nguyên khác chờ 48h

**Notifications hoạt động:**

- Email được gửi
- Log confirmation trong CloudWatch

### Troubleshooting nếu không hoạt động:

**Không có executions mới:**

- Kiểm tra EventBridge Rule có ENABLED không
- Kiểm tra IAM permissions của ResourceManagerRole

**Execution FAILED:**

- Xem execution history để tìm step bị lỗi
- Kiểm tra Lambda function permissions
- Xem CloudWatch logs của Lambda functions

**Không nhận được email:**

- Kiểm tra SES configuration
- Kiểm tra email address đã verify chưa
- Xem logs của NotificationSender Lambda

## Bước 4: Rollback về Production Schedule

**Sau khi kiểm tra xong, rollback ngay:**

1. Vào **EventBridge Console**
2. Edit rule `ResourceManagementSchedule`
3. Thay đổi lại schedule:

```
0 17 * * ? *
```

4. Nhấp **Update rule**

{{%notice warning%}}
**Quan trọng:** Nhớ rollback về lịch trình daily để tránh chạy quá thường xuyên trong production!
{{%/notice%}}

## Tóm tắt kiểm tra

Hệ thống hoạt động đúng khi:

- Step Functions executions chạy mỗi 2 phút
- ResourceScanner quét được EC2, S3, RDS
- Tài nguyên vi phạm được phát hiện và xử lý
- Email notifications được gửi
- Tài nguyên có tag Environment = "lab" bị xóa tự động
- Logs ghi lại đầy đủ hoạt động
