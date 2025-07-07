---
title: "Thiết lập Resource Management System"
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

## Cấu hình Parameter Store

### Tạo Whitelist Parameter

![Systems Manager Console](/images/3.Lambda/004-ssm.png)

1. Truy cập **"Systems Manager"**
2. Vào **"Parameter Store"**
3. Nhấp **"Create parameter"**
4. Cấu hình:

   - **Name**: `/resource-management/whitelist`
   - **Type**: String
   - **Value**:

   ```
   arn:aws:lambda:region:your-aws-id:function:ResourceScanner,arn:aws:lambda:region:your-aws-id:function:ResourceDeleter,arn:aws:lambda:region:your-aws-id:function:NotificationSender
   ```

   {{%notice note%}}
   Thay thế `your-aws-id` bằng Account ID thực tế của bạn và thay thế `region` bằng region bạn dùng.
   {{%/notice%}}

   ![whitelist](/images/3.Lambda/005-whitelist.png)

### Tạo Notification Config

1. Tạo parameter mới:

   - **Name**: `/resource-management/notification-config`
   - **Type**: String
   - **Value**:

   ```json
   {
     "email": "your-email@gmail.com",
     "allowed_environments": ["prod", "dev", "test", "lab"],
     "deletion_delay_hours": 48
   }
   ```

   {{%notice note%}}
   Thay thế `your-email@gmail.com` bằng mail thực tế của bạn.
   {{%/notice%}}
   ![Notification Config](/images/3.Lambda/006-notification-config.png)

### Tạo Policies Config

1. Tạo parameter:
   - **Name**: `/resource-management/policies`
   - **Type**: String
   - **Value**:
   ```json
   {
     "required_tags": ["Environment"],
     "allowed_environments": ["prod", "dev", "test", "lab"],
     "auto_delete_environments": ["lab"],
     "max_resource_age_days": 30
   }
   ```
   ![Policies](/images/3.Lambda/007-policies.png)
