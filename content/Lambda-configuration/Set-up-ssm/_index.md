---
title: "Resource Management System Setup"
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

## Parameter Store Configuration

### Create Whitelist Parameter

![Systems Manager Console](/images/3.Lambda/004-ssm.png)

1. Access **"Systems Manager"**
2. Go to **"Parameter Store"**
3. Click **"Create parameter"**
4. Configure:

   - **Name**: `/resource-management/whitelist`
   - **Type**: String
   - **Value**:

   ```
   arn:aws:lambda:region:your-aws-id:function:ResourceScanner,arn:aws:lambda:region:your-aws-id:function:ResourceDeleter,arn:aws:lambda:region:your-aws-id:function:NotificationSender
   ```

   {{%notice note%}}
   Replace `your-aws-id` with your actual Account ID and replace `region` with the region you're using.
   {{%/notice%}}

   ![whitelist](/images/3.Lambda/005-whitelist.png)

### Create Notification Config

1. Create a new parameter:

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
   Replace `your-email@gmail.com` with your actual email address.
   {{%/notice%}}
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
