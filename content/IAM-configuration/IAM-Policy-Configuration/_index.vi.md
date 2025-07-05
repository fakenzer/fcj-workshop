---
title: "Tạo IAM Policies"
weight: 1
chapter: false
pre: " <b> 2.1 </b> "
---

Trong phần này, chúng ta sẽ tạo hai custom policies quan trọng:

1. **ManageResourcePolicy**: Quản lý tài nguyên và chi phí
2. **RequireEnvironmentTagPolicy**: Yêu cầu bắt buộc tag Environment

## Policy 1: ManageResourcePolicy

#### Mục đích

Policy này cung cấp quyền để:

- Quản lý tags cho tài nguyên AWS
- Điều khiển Step Functions workflows

#### Các bước tạo policy

##### Bước 1: Truy cập IAM Console

1. Đăng nhập vào AWS Console
2. Tìm kiếm **"IAM"** trong thanh tìm kiếm
3. Chọn dịch vụ IAM

![Choose IAM Service](/images/2.IAM/001-Chooseiam.png)

##### Bước 2: Tạo Policy mới

1. Trong giao diện IAM, chọn **"Policies"** từ menu bên trái
2. Nhấp **"Create policy"**
   ![Choose create policy](/images/2.IAM/002-choosecreatepolicy.png)
3. Chọn tab **"JSON"** thay vì Visual editor

##### Bước 3: Cấu hình Policy JSON

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "tag:GetResources",
        "tag:TagResources",
        "tag:UntagResources",
        "states:StartExecution",
        "states:DescribeExecution"
      ],
      "Resource": "*"
    }
  ]
}
```

##### Bước 4: Hoàn tất Policy

1. Nhấp **"Next"** để review
2. Điền thông tin:
   - **Policy name**: `ManageResourcePolicy`
   - **Description**: `Policy for managing resources, tags and Step Functions`
3. Nhấp **"Create policy"**
   ![Review policy](/images/2.IAM/003-reviewpolicy.png)

#### Chi tiết quyền trong ManageResourcePolicy

##### Quyền quản lý Tags

```json
"tag:GetResources"     // Liệt kê tài nguyên và tags
"tag:TagResources"     // Thêm tags vào tài nguyên
"tag:UntagResources"   // Xóa tags khỏi tài nguyên
```

**Sử dụng**:

- Tự động tag tài nguyên theo quy chuẩn
- Phân loại tài nguyên theo department/project
- Tracking cost allocation

##### Quyền Step Functions

```json
"states:StartExecution"    // Khởi chạy workflow
"states:DescribeExecution" // Theo dõi workflow status
```

**Sử dụng**:

- Orchestrate complex workflows
- Automate cost optimization tasks
- Coordinate multiple AWS services

## Policy 2: RequireEnvironmentTagPolicy

#### Mục đích

Policy này đảm bảo:

- Tất cả tài nguyên mới phải có tag **"Environment"**
- Chỉ cho phép các giá trị: prod, dev, test, lab
- Ngăn chặn tạo tài nguyên không có tag

#### Cấu hình Policy JSON

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:RunInstances", "s3:CreateBucket", "rds:CreateDBInstance"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "*",
          "aws:RequestTag/Environment": ["prod", "dev", "test", "lab"]
        }
      }
    },
    {
      "Effect": "Deny",
      "Action": ["ec2:RunInstances", "s3:CreateBucket", "rds:CreateDBInstance"],
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringNotLike": {
          "aws:RequestTag/Environment": "*"
        }
      }
    }
  ]
}
```

#### Các bước tạo RequireEnvironmentTagPolicy

##### Bước 1: Tạo policy mới

1. Trong IAM Console, chọn **"Policies"**
2. Nhấp **"Create policy"**
3. Chọn tab **"JSON"**

##### Bước 2: Paste JSON policy

Copy và paste đoạn JSON RequireEnvironmentTagPolicy ở trên

##### Bước 3: Hoàn tất

1. **Policy name**: `RequireEnvironmentTagPolicy`
2. **Description**: `Enforce environment tag requirement for resource creation`
3. Nhấp **"Create policy"**

   ![Review policy](/images/2.IAM/004-reviewpolicy2.png)

## Kết quả

Sau khi hoàn thành phần này, bạn sẽ có:

1. **ManageResourcePolicy**:

   - Quản lý tags
   - Điều khiển Step Functions workflows

2. **RequireEnvironmentTagPolicy**:
   - Enforce environment tag requirement
   - Ngăn chặn việc tạo ra các tài nguyên không có tag
   - Support governance và compliance

Hai policies này sẽ được sử dụng trong các phần tiếp theo để tạo role và user với quyền phù hợp.
