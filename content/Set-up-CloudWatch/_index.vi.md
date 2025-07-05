---
title: "Tạo CloudWatch Dashboard"
weight: 6
chapter: false
pre: " <b> 6. </b> "
---

## Giới thiệu

CloudWatch Dashboard là công cụ giám sát trực quan giúp theo dõi hiệu suất và trạng thái của các tài nguyên AWS trong thời gian thực. Hướng dẫn này sẽ giúp bạn tạo một dashboard hoàn chỉnh cho việc quản lý tài nguyên.

## Mục tiêu

Tạo một dashboard có tên `ResourceManagement` để giám sát:

- Lambda Functions
- Step Functions
- SNS Topics
- Các metrics quan trọng khác

## Bước 1: Truy cập CloudWatch Console

1. **Đăng nhập vào AWS Management Console**

   - Truy cập [AWS Console](https://console.aws.amazon.com/)
   - Đăng nhập bằng thông tin tài khoản AWS

2. **Chuyển đến CloudWatch**

   - Tìm kiếm "CloudWatch" trong thanh tìm kiếm
   - Hoặc chọn CloudWatch từ danh sách Services
     ![CloudWatch Console](/images/6.CloudWatch/001-cloudwatch.png)

3. **Truy cập Dashboards**
   - Trong menu bên trái, chọn **Dashboards**
   - Click **Create dashboard**

## Bước 2: Tạo Dashboard mới

1. **Đặt tên Dashboard**

   - **Dashboard name**: `ResourceManagement`
   - Click **Create dashboard**

2. **Chọn Widget type**
   - Hệ thống sẽ hiển thị hộp thoại chọn widget
   - Sẵn sàng thêm widget đầu tiên

## Bước 3: Thêm Widget 1 - Lambda Invocations

### 3.1 Cấu hình Widget

1. **Chọn widget type**: **Line chart**
2. **Click Add widget**

### 3.2 Cấu hình Metrics

1. **Chọn metric source**: **Lambda**
2. **Chọn metric category**: **By Function Name**
3. **Chọn metric**: **Invocations**
4. **Chọn Functions**:
   - Chọn tất cả functions liên quan đến resource management
   - Hoặc chọn specific functions cần monitor
     ![Create invocation metrics](/images/6.CloudWatch/002-createinvocationmetric.png)

### 3.3 Tùy chỉnh Widget

1. **Widget title**: "Lambda Invocations"
2. **Time range**: Last 1 hour (hoặc theo nhu cầu)
3. **Statistic**: Sum
4. **Period**: 5 minutes
5. **Click Create widget**

## Bước 4: Thêm Widget 2 - Lambda Errors

### 4.1 Cấu hình Widget

1. **Click Add widget**
2. **Chọn widget type**: **Number**
3. **Click Add widget**

### 4.2 Cấu hình Metrics

1. **Metric source**: **Lambda**
2. **Metric category**: **By Function Name**
3. **Metric**: **Errors**
4. **Functions**: Chọn các functions cần monitor
   ![Create error metrics](/images/6.CloudWatch/003-createerrormetric.png)

### 4.3 Tùy chỉnh Widget

1. **Widget title**: "Lambda Errors"
2. **Statistic**: Sum
3. **Period**: 5 minutes
4. **Click Create widget**

## Bước 5: Thêm Widget 3 - Step Functions Executions

### 5.1 Cấu hình Widget

1. **Click Add widget**
2. **Chọn widget type**: **Line chart**
3. **Click Add widget**

### 5.2 Cấu hình Metrics

1. **Metric source**: **Step Functions**
2. **Metric category**: **State Machine Metrics**
3. **Metric**: **ExecutionsStarted**
4. **State Machines**: Chọn các state machines cần monitor
   ![Create step function metric](/images/6.CloudWatch/004-createstepfunctionmetric.png)

### 5.3 Tùy chỉnh Widget

1. **Widget title**: "Step Functions Executions"
2. **Statistic**: Sum
3. **Period**: 5 minutes
4. **Click Create widget**

## Bước 6: Thêm Widget 4 - SNS Messages

### 6.1 Cấu hình Widget

1. **Click Add widget**
2. **Chọn widget type**: **Number**
3. **Click Add widget**

### 6.2 Cấu hình Metrics

1. **Metric source**: **SNS**
2. **Metric category**: **By Topic**
3. **Metric**: **NumberOfMessagesPublished**
4. **Topics**: Chọn các topics cần monitor
   ![Create SNS metric](/images/6.CloudWatch/005-createsnsmetric.png)

### 6.3 Tùy chỉnh Widget

1. **Widget title**: "SNS Messages Published"
2. **Statistic**: Sum
3. **Period**: 5 minutes
4. **Click Create widget**

## Bước 7: Hoàn thiện Dashboard

### 7.1 Sắp xếp Widgets

1. **Drag và drop** để sắp xếp widgets theo ý muốn
2. **Resize** widgets bằng cách kéo góc
3. **Align** widgets để tạo layout đẹp mắt

### 7.2 Lưu Dashboard

1. **Click Save dashboard**
2. Dashboard sẽ được lưu và có thể truy cập sau
