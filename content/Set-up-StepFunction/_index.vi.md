---
title: "Tạo Step Functions"
weight: 4
chapter: false
pre: " <b> 4. </b> "
---

Trong phần này, chúng ta sẽ tạo Step Functions **ResourceManagementWorkflow** để orchestrate toàn bộ quy trình quản lý tài nguyên AWS tự động.

## Step Functions: ResourceManagementWorkflow

#### Mục đích

Step Functions này sẽ điều phối toàn bộ workflow quản lý tài nguyên:

- Khởi động quá trình scan tài nguyên qua ResourceScanner
- Kiểm tra và xử lý danh sách tài nguyên được phát hiện
- Phân loại và thực hiện các hành động phù hợp cho từng tài nguyên
- Gửi thông báo cảnh báo hoặc xóa tài nguyên theo chính sách
- Xử lý song song nhiều tài nguyên để tối ưu hiệu suất

#### Luồng hoạt động

Workflow thực hiện theo các bước:

1. **ScanResources**: Trigger ResourceScanner để tìm tài nguyên vi phạm
2. **CheckResourcesExist**: Kiểm tra có tài nguyên cần xử lý không
3. **CheckResourcesNotEmpty**: Xác nhận danh sách tài nguyên không rỗng
4. **ProcessEachResource**: Xử lý song song từng tài nguyên
5. **DecideAction**: Phân loại hành động cần thực hiện
6. **Execute Actions**: Thực hiện delete, warning, hoặc schedule delete

#### Các loại Action

- **DELETE_IMMEDIATE**: Xóa ngay lập tức (tài nguyên không có tag Environment)
- **SEND_WARNING**: Gửi cảnh báo (tài nguyên thiếu một số tag)
- **NO_ACTION**: Chờ 48h rồi xóa (tài nguyên có tag Environment nhưng vi phạm khác)

#### Các bước tạo Step Functions

##### Bước 1: Truy cập Step Functions Console

1. Đăng nhập vào AWS Console
2. Tìm kiếm **"Step Functions"** trong thanh tìm kiếm
3. Chọn dịch vụ AWS Step Functions
   ![Step Functions](/images/4.StepFunctions/001-stepfunction.png)

##### Bước 2: Tạo State Machine mới

1. Nhấp vào nút **"Create state machine"**
2. Chọn **"Write your workflow in code"**
3. Chọn **"Standard"** state machine type

##### Bước 3: Cấu hình State Machine

**State machine configuration**:

- **Name**: `ResourceManagementWorkflow`
- **Type**: `Standard`
  ![Create Step Functions](/images/4.StepFunctions/002-createstepfunction.png)

##### Bước 4: Definition Code

Paste đoạn JSON definition sau vào editor:

```json
{
  "Comment": "Resource Management Workflow",
  "StartAt": "ScanResources",
  "States": {
    "ScanResources": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:your-region:your-aws-id:function:ResourceScanner",
      "Next": "CheckResourcesExist"
    },
    "CheckResourcesExist": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.resources",
          "IsPresent": true,
          "Next": "CheckResourcesNotEmpty"
        }
      ],
      "Default": "NoResourcesFound"
    },
    "CheckResourcesNotEmpty": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.resources[0]",
          "IsPresent": true,
          "Next": "ProcessEachResource"
        }
      ],
      "Default": "NoResourcesFound"
    },
    "NoResourcesFound": {
      "Type": "Pass",
      "Result": {
        "message": "No resources found to process",
        "status": "completed"
      },
      "End": true
    },
    "ProcessEachResource": {
      "Type": "Map",
      "ItemsPath": "$.resources",
      "MaxConcurrency": 10,
      "Parameters": {
        "resource_arn.$": "$$.Map.Item.Value.resource_arn",
        "action.$": "$$.Map.Item.Value.action",
        "tags.$": "$$.Map.Item.Value.tags",
        "details.$": "$$.Map.Item.Value.details",
        "timestamp.$": "$$.Map.Item.Value.timestamp"
      },
      "Iterator": {
        "StartAt": "DecideAction",
        "States": {
          "DecideAction": {
            "Type": "Choice",
            "Choices": [
              {
                "Variable": "$.action",
                "StringEquals": "DELETE_IMMEDIATE",
                "Next": "DeleteNow"
              },
              {
                "Variable": "$.action",
                "StringEquals": "SEND_WARNING",
                "Next": "SendWarning"
              },
              {
                "Variable": "$.action",
                "StringEquals": "NO_ACTION",
                "Next": "WaitThenDelete"
              }
            ],
            "Default": "NoAction"
          },
          "DeleteNow": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:your-region:your-aws-id:function:ResourceDeleter",
            "End": true
          },
          "SendWarning": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:your-region:your-aws-id:function:NotificationSender",
            "End": true
          },
          "WaitThenDelete": {
            "Type": "Wait",
            "Seconds": 172800,
            "Next": "DeleteScheduled"
          },
          "DeleteScheduled": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:your-region:your-aws-id:function:ResourceDeleter",
            "End": true
          },
          "NoAction": {
            "Type": "Pass",
            "End": true
          }
        }
      },
      "End": true
    }
  }
}
```

![Workflow](/images/4.StepFunctions/003-workflow.png)
{{%notice note%}}
Thay thế `your-aws-id` bằng Account ID thực tế của bạn và thay thế `your-region` bằng region bạn dùng.
{{%/notice%}}

##### Bước 7: Review và Deploy

1. Review toàn bộ configuration
2. Nhấp **"Create state machine"**
3. Kiểm tra state machine được tạo thành công

#### Performance Configuration

- **MaxConcurrency**: 10 (xử lý song song tối đa 10 tài nguyên)
- **Timeout**: 1 hour (đủ cho cả wait 48h của scheduled delete)
- **Retry Policy**: 3 lần với exponential backoff

Workflow này đảm bảo quản lý tài nguyên AWS hiệu quả, tự động và có thể scale theo nhu cầu của tổ chức.
