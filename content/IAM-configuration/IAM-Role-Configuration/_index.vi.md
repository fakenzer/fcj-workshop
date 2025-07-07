---
title: "Tạo IAM Role"
weight: 2
chapter: false
pre: " <b> 2.2 </b> "
---

Trong phần này, chúng ta sẽ tạo IAM Role có tên **"ResourceManagerRole"** với đầy đủ quyền để:

- Quản lý tài nguyên AWS (EC2, S3, RDS)
- Thực hiện automation workflows
- Giám sát và gửi notifications
- Integrate với các AWS services

## Mục đích Role

ResourceManagerRole được thiết kế để:

- **Service Integration**: Được sử dụng bởi Lambda, Step Functions, Systems Manager
- **Resource Management**: Quản lý EC2, S3, RDS với full permissions
- **Automation**: Chạy automation workflows và scripts
- **Monitoring**: Gửi alerts và ghi logs

## Điều kiện tiên quyết

- Đã tạo thành công các policies:
  - `ManageResourcePolicy` (từ phần 2.1)
- Có quyền truy cập AWS Console với permissions tạo IAM Role
- Hiểu về trusted entities và assume role
  ![Manage Resource Policy](/images/2.IAM/005-manageresourcepolicy.png)

## Các bước tạo Role

### Bước 1: Truy cập IAM Roles

1. Trong AWS Console, vào service IAM
2. Tìm thanh sidebar bên trái
3. Nhấp vào mục **"Roles"**
4. Trang quản lý IAM Roles sẽ được hiển thị

### Bước 2: Tạo Role mới

1. Nhấp vào nút **"Create role"**
2. Chuyển đến trang "Select trusted entity"

![Manage Resource Policy](/images/2.IAM/006-createrole.png)

### Bước 3: Cấu hình Trusted Entity

1. Trong phần **"Trusted entity type"**, chọn **"Custom trust policy"**
2. Paste đoạn JSON trust policy sau vào ô văn bản:

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

3. Nhấp "Next" để tiếp tục

### Bước 4: Gán AWS Managed Policies

Tại trang **"Add permissions"**, tìm và chọn các AWS managed policies sau:

#### 4.1 Lambda và Execution

```
AWSLambdaBasicExecutionRole
```

- Quyền ghi logs vào CloudWatch
- Cần thiết cho Lambda functions

#### 4.2 Notification Services

```
AmazonSNSFullAccess
```

- Gửi notifications qua SNS
- Quản lý topics và subscriptions

#### 4.3 Systems Management

```
AmazonSSMFullAccess
```

- Chạy automation documents
- Quản lý parameters và patches
- Execute run commands

#### 4.4 Monitoring và Logging

```
CloudWatchFullAccess
```

- Tạo và quản lý CloudWatch metrics
- Quản lý alarms và dashboards
- Log management

#### 4.5 Workflow Orchestration

```
AWSStepFunctionsFullAccess
```

- Tạo và quản lý Step Functions
- Execute state machines

#### 4.6 Compute Services

```
AmazonEC2FullAccess
```

- Quản lý EC2 instances, volumes, snapshots
- Network và security groups

#### 4.7 Storage Services

```
AmazonS3FullAccess
```

- Quản lý S3 buckets và objects
- Lifecycle policies

#### 4.8 Database Services

```
AmazonRDSFullAccess
```

- Quản lý RDS instances
- Snapshots và backups

### Bước 5: Gán Custom Policies

1. Trong cùng trang **"Add permissions"**, tìm kiếm **"ManageResourcePolicy"**
2. Đánh dấu chọn policy này
3. Nhấp "Next" để chuyển sang bước cuối

### Bước 6: Hoàn tất Role

1. Tại trang **"Name, review and create"**, điền thông tin:

   - **Role name**: `ResourceManagerRole`
   - **Description**: `Comprehensive role for resource management and automation`

2. Kiểm tra lại toàn bộ cấu hình:

   - Tên role chính xác
   - Trust policy đã được cấu hình đúng
   - Tất cả AWS managed policies đã được gán
   - Custom policy "ManageResourcePolicy" đã được gán
     ![Review Role](/images/2.IAM/007-reviewrole.png)

3. Nhấp "Create role"
