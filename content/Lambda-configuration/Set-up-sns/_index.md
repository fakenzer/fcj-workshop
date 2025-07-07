---
title: "Resource Management Alerts Configuration"
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

This section provides detailed guidance on setting up SNS Topic and Subscription to receive email notifications for AWS resource management.

## Objectives

- Create SNS Topic for resource management alert system
- Configure Email Subscription to receive notifications
- Set up email endpoint for receiving alerts

## Expected Outcomes

After completing this chapter, you will have:

- **SNS Topic**: `resource-management-alerts` with Standard configuration
- **Email Subscription**: Automatic notifications via email your-email@gmail.com
- **Ready System**: For integration with other monitoring services

## Implementation Steps

### Step 1: Access Amazon SNS

![SNS](/images/3.Lambda/008-SNS.png)

1. Log into AWS Console
2. Search for and select **SNS** from the services list
3. In the left menu, select **Topics**

### Step 2: Create SNS Topic

1. Click **Create topic**
2. Select **Type**: Standard
3. Enter **Name**: `resource-management-alerts`
4. Leave other settings as default
5. Click **Create topic**
   ![Create SNS topic](/images/3.Lambda/009-createtopic.png)

### Step 3: Create Email Subscription

1. In the newly created SNS Topic, select the **Subscriptions** tab
2. Click **Create subscription**
3. Configure subscription:
   - **Topic ARN**: Will be automatically filled
   - **Protocol**: Select **Email**
   - **Endpoint**: Enter `your-email@gmail.com`
     {{%notice note%}}
     Replace `your-email@gmail.com` with your actual email address and replace `your-region` with the region you're using.
     {{%/notice%}}
4. Click **Create subscription**
   ![Create Subscriptions](/images/3.Lambda/010-subscriptions.png)

   {{%notice note%}}
   Replace `your-email@gmail.com` with your actual email address.
   {{%/notice%}}

### Step 4: Confirm Email Subscription

1. Check your email inbox
2. Find email from **AWS Notifications**
3. Click the **Confirm subscription** link in the email
4. Return to AWS Console to verify subscription status has changed to **Confirmed**
   ![email](/images/3.Lambda/011-email.png)

### Step 5: Verify Configuration

![Confirm email](/images/3.Lambda/012-confirmemail.png)

1. In the SNS Topic, **Subscriptions** tab
2. Confirm subscription status is **Confirmed**
3. You can test by clicking **Publish message** to send a test message

## Test Notifications

### Send Test Message

![Test mail](/images/3.Lambda/013-testmail.png)

1. In the SNS Topic, select **Publish message**
2. Enter **Subject**: `Test Resource Management Alert`
3. Enter **Message**:
   ```
   This is a test message for resource management alerts.
   System is working correctly.
   ```
4. Click **Publish message**
5. Check your email to confirm you received the notification
   ![Test message](/images/3.Lambda/014-testmessage.png)

## Important Notes

- **Email delivery**: Check spam folder if you don't receive emails
- **Subscription status**: Ensure status is "Confirmed" before using
- **Topic ARN**: Save the Topic ARN for use with other services
- **Cost optimization**: Standard topics have costs based on the number of messages sent
- **Security**: Only share Topic ARN with necessary services
