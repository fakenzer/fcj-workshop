---
title: "Creating CloudWatch Dashboard"
weight: 6
chapter: false
pre: " <b> 6. </b> "
---

## Introduction

CloudWatch Dashboard is a visual monitoring tool that helps track the performance and status of AWS resources in real-time. This guide will help you create a comprehensive dashboard for resource management.

## Objectives

Create a dashboard named `ResourceManagement` to monitor:

- Lambda Functions
- Step Functions
- SNS Topics
- Other important metrics

## Step 1: Access CloudWatch Console

1. **Log into AWS Management Console**

   - Access [AWS Console](https://console.aws.amazon.com/)
   - Log in with your AWS account credentials

2. **Navigate to CloudWatch**

   - Search for "CloudWatch" in the search bar
   - Or select CloudWatch from the Services list
     ![CloudWatch Console](/images/6.CloudWatch/001-cloudwatch.png)

3. **Access Dashboards**
   - In the left menu, select **Dashboards**
   - Click **Create dashboard**

## Step 2: Create New Dashboard

1. **Name the Dashboard**

   - **Dashboard name**: `ResourceManagement`
   - Click **Create dashboard**

2. **Choose Widget Type**
   - The system will display a widget selection dialog
   - Ready to add the first widget

## Step 3: Add Widget 1 - Lambda Invocations

### 3.1 Configure Widget

1. **Select widget type**: **Line chart**
2. **Click Add widget**

### 3.2 Configure Metrics

1. **Select metric source**: **Lambda**
2. **Select metric category**: **By Function Name**
3. **Select metric**: **Invocations**
4. **Select Functions**:
   - Select all functions related to resource management
   - Or select specific functions to monitor
     ![Create invocation metrics](/images/6.CloudWatch/002-createinvocationmetric.png)

### 3.3 Customize Widget

1. **Widget title**: "Lambda Invocations"
2. **Time range**: Last 1 hour (or as needed)
3. **Statistic**: Sum
4. **Period**: 5 minutes
5. **Click Create widget**

## Step 4: Add Widget 2 - Lambda Errors

### 4.1 Configure Widget

1. **Click Add widget**
2. **Select widget type**: **Number**
3. **Click Add widget**

### 4.2 Configure Metrics

1. **Metric source**: **Lambda**
2. **Metric category**: **By Function Name**
3. **Metric**: **Errors**
4. **Functions**: Select functions to monitor
   ![Create error metrics](/images/6.CloudWatch/003-createerrormetric.png)

### 4.3 Customize Widget

1. **Widget title**: "Lambda Errors"
2. **Statistic**: Sum
3. **Period**: 5 minutes
4. **Click Create widget**

## Step 5: Add Widget 3 - Step Functions Executions

### 5.1 Configure Widget

1. **Click Add widget**
2. **Select widget type**: **Line chart**
3. **Click Add widget**

### 5.2 Configure Metrics

1. **Metric source**: **Step Functions**
2. **Metric category**: **State Machine Metrics**
3. **Metric**: **ExecutionsStarted**
4. **State Machines**: Select state machines to monitor
   ![Create step function metric](/images/6.CloudWatch/004-createstepfunctionmetric.png)

### 5.3 Customize Widget

1. **Widget title**: "Step Functions Executions"
2. **Statistic**: Sum
3. **Period**: 5 minutes
4. **Click Create widget**

## Step 6: Add Widget 4 - SNS Messages

### 6.1 Configure Widget

1. **Click Add widget**
2. **Select widget type**: **Number**
3. **Click Add widget**

### 6.2 Configure Metrics

1. **Metric source**: **SNS**
2. **Metric category**: **By Topic**
3. **Metric**: **NumberOfMessagesPublished**
4. **Topics**: Select topics to monitor
   ![Create SNS metric](/images/6.CloudWatch/005-createsnsmetric.png)

### 6.3 Customize Widget

1. **Widget title**: "SNS Messages Published"
2. **Statistic**: Sum
3. **Period**: 5 minutes
4. **Click Create widget**

## Step 7: Finalize Dashboard

### 7.1 Arrange Widgets

1. **Drag and drop** to arrange widgets as desired
2. **Resize** widgets by dragging corners
3. **Align** widgets to create an attractive layout

### 7.2 Save Dashboard

1. **Click Save dashboard**
2. Dashboard will be saved and accessible later
