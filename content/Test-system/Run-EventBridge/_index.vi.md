---
title: "Chạy EventBridge"
weight: 7
chapter: false
pre: " <b> 7.4 </b> "
---

Test thủ công Lambda function ResourceManager để verify hoạt động

## Các bước thực hiện

### 1. Mở Lambda Console

- Truy cập AWS Console
- Chọn service **Lambda**

### 2. Chọn Function

- Tìm và click vào function `ResourceManager`
- Kiểm tra function đã được deploy đúng

### 3. Tạo Test Event

- Click **Test**
- Chọn **Create new test event**
- Đặt tên event: `ScheduledEventTest`

### 4. Cấu hình Test Event

```json
{
  "source": "aws.events",
  "detail-type": "Scheduled Event"
}
```

### 5. Chạy Test

- Click **Save**
- Click **Test** để chạy function
- Observe execution results

## Kết quả mong đợi

- Function chạy thành công
- Return status 200
- Execution time hiển thị
- Không có error messages

## Kiểm tra Chi tiết

### 1. CloudWatch Logs

- Vào CloudWatch console
- Tìm log group: `/aws/lambda/ResourceManager`
- Kiểm tra latest log stream
- Verify các log messages:
  - Function start
  - Resource scanning
  - Processing results
  - Function completion

### 2. DynamoDB Records

- Vào DynamoDB console
- Mở table resource compliance
- Kiểm tra có record mới không
- Verify timestamp và resource info

### 3. SNS Notifications

- Check email inbox
- Verify notification nếu có resource không comply
- Kiểm tra format và content của notification

## Advanced Testing

### Test với Event khác

```json
{
  "source": "aws.ec2",
  "detail-type": "EC2 Instance State-change Notification",
  "detail": {
    "state": "running",
    "instance-id": "i-1234567890abcdef0"
  }
}
```

### Test Error Handling

```json
{
  "source": "invalid.source",
  "detail-type": "Invalid Event"
}
```

## Monitoring và Debug

### 1. Function Metrics

- Vào Lambda console → Monitoring tab
- Kiểm tra:
  - Invocations
  - Duration
  - Error rate
  - Success rate

### 2. CloudWatch Alarms

- Kiểm tra có alarm nào triggered không
- Verify threshold settings

### 3. X-Ray Tracing (nếu enable)

- Vào X-Ray console
- Kiểm tra trace details
- Analyze performance bottlenecks

## Troubleshooting

### Function Timeout

- Kiểm tra timeout setting (default 3 seconds)
- Increase nếu function cần thời gian lâu hơn

### Permission Errors

- Verify IAM role của Lambda function
- Kiểm tra policies attached
- Test từng service permission

### Memory Issues

- Kiểm tra memory usage trong logs
- Increase memory allocation nếu cần

### Environment Variables

- Verify tất cả environment variables
- Kiểm tra SNS topic ARN
- Kiểm tra DynamoDB table name

## Cleanup

- Không cần cleanup cho manual test
- Có thể delete test events nếu không cần
- Monitor CloudWatch logs size để tránh chi phí
