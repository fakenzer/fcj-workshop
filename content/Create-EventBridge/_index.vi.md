---
title: "Tạo EventBridge Rule"
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

Trong phần này, chúng ta sẽ tạo EventBridge Rule để tự động trigger Step Functions **ResourceManagementWorkflow** theo lịch trình định kỳ.

## EventBridge Rule: ResourceManagementSchedule

#### Mục đích

EventBridge Rule này sẽ thực hiện:

- Tự động trigger ResourceManagementWorkflow theo lịch trình
- Chạy daily compliance check vào 5:00 PM UTC
- Gửi input parameters phù hợp cho Step Functions
- Sử dụng IAM role tự động được AWS tạo để invoke Step Functions
- Đảm bảo workflow chạy ổn định và có thể monitor

#### Lịch trình hoạt động

Rule sẽ trigger workflow:

1. **Hàng ngày lúc 5:00 PM UTC** (0:00 AM giờ Việt Nam)
2. **Có thể customize** theo nhu cầu organization

#### Các bước tạo EventBridge Rule

##### Bước 1: Truy cập EventBridge Console

1. Đăng nhập vào AWS Console
2. Tìm kiếm **"EventBridge"** trong thanh tìm kiếm
3. Chọn dịch vụ Amazon EventBridge
   ![Event Bridge](/images/5.EventBridge/001-eventbridge.png)

##### Bước 2: Tạo Rule mới

1. Trong sidebar, chọn **"Rules"**
2. Nhấp vào nút **"Create rule"**
3. Đảm bảo đang ở Event bus **"default"**

##### Bước 3: Cấu hình Rule Definition

**Rule detail**:

- **Name**: `ResourceManagementSchedule`
- **Description**: `Daily trigger for resource management compliance check`
- **Event bus**: `default`
- **Rule type**: `Schedule`
  ![Create rule](/images/5.EventBridge/002-createrule.png)

##### Bước 4: Cấu hình Schedule Pattern

**Schedule pattern**:

1. Chọn "A schedule that runs at a regular rate, such as every 10 minutes"
2. Hoặc chọn "A fine-grained schedule that runs at a specific time"

**Option 1: Cron Expression (Recommended)**

```
0 17 * * ? *
```

- Chạy hàng ngày lúc 5:00 PM UTC
  ![Configuration cron](/images/5.EventBridge/003-configurationcron.png)
  **Option 2: Rate Expression**

```
rate(1 day)
```

- Chạy mỗi 24 giờ từ lần đầu tiên được tạo

##### Bước 5: Cấu hình Target

**Target configuration**:

1. Nhấp **"Add target"**
2. Trong dropdown **"Select a target"**, chọn **"Step Functions state machine"**

**Step Functions configuration**:

- **State machine**: Chọn `ResourceManagementWorkflow`
- **Execution role**: Chọn **"Use existing role"**

**AWS sẽ tự động tạo role với format**: `Amazon_EventBridge_Invoke_Step_Functions_Role_[random_id]`
![Select target](/images/5.EventBridge/004-selecttarget.png)

##### Bước 6: Cấu hình Additional Settings

**Retry policy**:

- **Maximum age of event**: 24 hours
- **Retry attempts**: 3

**Dead letter queue**:

- Chọn "None" hoặc tạo SQS queue để handle failed events

##### Bước 7: Review và Create

1. Review tất cả configuration
2. Nhấp **"Create rule"**
3. Verify rule được tạo thành công

#### Monitoring và Troubleshooting

##### CloudWatch Metrics

EventBridge cung cấp metrics:

- **InvocationsCount**: Số lần rule được trigger
- **SuccessfulInvocations**: Số lần thành công
- **FailedInvocations**: Số lần thất bại
- **TargetErrors**: Lỗi khi invoke target

##### CloudWatch Logs

**EventBridge không tự động log**, nhưng có thể enable:

##### Step Functions Execution History

Sau khi EventBridge trigger:

1. Vào Step Functions Console
2. Chọn `ResourceManagementWorkflow`
3. Xem "Executions" tab để track status

#### Customization Options

##### Multiple Schedules

Có thể tạo nhiều rules cho different use cases:

```json
{
  "WeeklyDeepScan": "0 17 ? * MON *",
  "DailyLightScan": "0 17 * * ? *",
  "HourlyTestScan": "0 * * * ? *"
}
```

##### Dynamic Input

Sử dụng input transformer để dynamic values:

```json
{
  "scan_type": "daily_compliance_check",
  "timestamp": "<aws.events.event.ingestion-time>",
  "region": "<aws.events.event.region>"
}
```

##### Conditional Execution

Thêm input conditions:

```json
{
  "scan_type": "daily_compliance_check",
  "conditions": {
    "only_business_hours": true,
    "skip_weekends": false,
    "environment_filter": ["prod", "staging"]
  }
}
```

EventBridge Rule này đảm bảo resource management workflow chạy tự động, consistent và có thể monitor hiệu quả.
