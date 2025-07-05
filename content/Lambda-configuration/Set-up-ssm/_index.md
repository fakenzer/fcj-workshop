---
title: "Setting up Resource Management System"
weight: 4
chapter: false
pre: " <b> 3.4. </b> "
---

## Configure Parameter Store

### Create Whitelist Parameter

![Systems Manager Console](/images/3.Lambda/004-ssm.png)

1. Access **"Systems Manager Console"**
2. Go to **"Parameter Store"**
3. Click **"Create parameter"**
4. Configure:
   - **Name**: `/resource-management/whitelist`
   - **Type**: String
   - **Value**:
   ```
   arn:aws:lambda:region:your-aws-id:function:ResourceScanner,arn:aws:lambda:region:your-aws-id:function:ResourceDeleter,arn:aws:lambda:region:your-aws-id:function:NotificationSender
   ```
   ![whitelist](/images/3.Lambda/005-whitelist.png)

### Create Notification Config

1. Create new parameter:
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
   ![Notification Config](/images/3.Lambda/006-notification-config.png)

### Create Policies Config

1. Create parameter:
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
