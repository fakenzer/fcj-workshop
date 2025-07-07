---
title: "Tạo IAM User "
weight: 3
chapter: false
pre: " <b> 2.3 </b> "
---

Trong phần này, chúng ta sẽ tạo một IAM User và gán policy **RequireEnvironmentTagPolicy** để đảm bảo user này chỉ có thể tạo tài nguyên khi có tag Environment bắt buộc.

## Mục đích

Tạo một IAM User có những đặc điểm sau:

- Bắt buộc phải có tag **"Environment"** khi tạo tài nguyên
- Chỉ cho phép các giá trị tag: `prod`, `dev`, `test`, `lab`
- Ngăn chặn tạo tài nguyên EC2, S3, RDS nếu không có tag
- Đảm bảo tuân thủ governance và compliance

## Các bước tạo IAM User

### Bước 1: Truy cập IAM Console

1. Đăng nhập vào AWS Console
2. Tìm kiếm **"IAM"** trong thanh tìm kiếm
3. Chọn dịch vụ IAM

### Bước 2: Tạo User mới

1. Trong giao diện IAM, chọn **"Users"** từ menu bên trái
2. Nhấp **"Create user"**

![Choose create user](/images/2.IAM/005-choosecreateuser.png)

### Bước 3: Cấu hình User

1. **User name**: `TagEnforcementUser`
2. Chọn **"Provide user access to the AWS Management Console"**
3. Chọn **"I want to create an IAM user"**
4. **Console password**: Chọn **"Custom password"** và nhập mật khẩu mạnh
5. Bỏ tick **"Users must create a new password at next sign-in"** (tùy chọn)
6. Nhấp **"Next"**

![Configure user](/images/2.IAM/006-configureuser.png)

### Bước 4: Gán Permissions

1. Chọn **"Attach policies directly"**
2. Tìm kiếm **"RequireEnvironmentTagPolicy"**
3. Tick chọn policy **RequireEnvironmentTagPolicy**
4. Thêm các policy cơ bản khác nếu cần:
   - **IAMReadOnlyAccess** (để user có thể xem thông tin IAM)
   - **ReadOnlyAccess** (để user có thể xem tài nguyên hiện có)

![Attach policies](/images/2.IAM/007-attachpolicies.png)

### Bước 5: Thêm Tags (tùy chọn)

1. Nhấp **"Next"** để đến bước Tags
2. Thêm các tags cho user:
   - **Key**: `Department`, **Value**: `DevOps`
   - **Key**: `Purpose`, **Value**: `TagEnforcement`
   - **Key**: `Environment`, **Value**: `management`

### Bước 6: Review và Create

1. Review lại tất cả thông tin
2. Nhấp **"Create user"**

![Review user](/images/2.IAM/008-reviewuser.png)

## Tạo Access Key (tùy chọn)

Nếu user cần truy cập qua CLI hoặc SDK:

### Bước 1: Tạo Access Key

1. Vào user **TagEnforcementUser** vừa tạo
2. Chọn tab **"Security credentials"**
3. Nhấp **"Create access key"**
4. Chọn **"Command Line Interface (CLI)"**
5. Tick **"I understand the above recommendation"**
6. Nhấp **"Next"**

### Bước 2: Cấu hình Access Key

1. **Description tag**: `CLI access for tag enforcement testing`
2. Nhấp **"Create access key"**
3. **Lưu lại Access Key ID và Secret Access Key**

![Create access key](/images/2.IAM/009-createaccesskey.png)

## Kiểm tra Policy hoạt động

### Test Case 1: Tạo EC2 Instance không có tag (Sẽ bị từ chối)

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t2.micro \
  --region us-east-1
```

**Kết quả mong đợi**: Lỗi Access Denied

### Test Case 2: Tạo EC2 Instance với tag hợp lệ (Sẽ thành công)

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t2.micro \
  --region us-east-1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Environment,Value=dev}]'
```

**Kết quả mong đợi**: Tạo thành công

### Test Case 3: Tạo S3 Bucket với tag không hợp lệ (Sẽ bị từ chối)

```bash
aws s3api create-bucket \
  --bucket my-test-bucket-123 \
  --region us-east-1 \
  --create-bucket-configuration LocationConstraint=us-east-1 \
  --tagging 'TagSet=[{Key=Environment,Value=invalid}]'
```

**Kết quả mong đợi**: Lỗi Access Denied

### Test Case 4: Tạo S3 Bucket với tag hợp lệ (Sẽ thành công)

```bash
aws s3api create-bucket \
  --bucket my-test-bucket-123 \
  --region us-east-1 \
  --create-bucket-configuration LocationConstraint=us-east-1 \
  --tagging 'TagSet=[{Key=Environment,Value=test}]'
```

**Kết quả mong đợi**: Tạo thành công

## Thông tin đăng nhập User

Sau khi tạo xong, bạn sẽ có thông tin:

- **User name**: `TagEnforcementUser`
- **Console sign-in URL**: `https://your-account-id.signin.aws.amazon.com/console`
- **Password**: Mật khẩu đã thiết lập
- **Access Key ID**: (nếu đã tạo)
- **Secret Access Key**: (nếu đã tạo)

## Lưu ý quan trọng

1. **Bảo mật**: Không chia sẻ thông tin đăng nhập
2. **Giá trị tag hợp lệ**: Chỉ `prod`, `dev`, `test`, `lab`
3. **Tài nguyên được kiểm soát**: EC2, S3, RDS
4. **Monitoring**: Theo dõi CloudTrail để xem các hành động của user

## Kết quả

Sau khi hoàn thành phần này, bạn sẽ có:

1. **TagEnforcementUser**: User với quyền bị giới hạn bởi tag policy
2. **Tag Enforcement**: Tất cả tài nguyên mới phải có tag Environment
3. **Compliance**: Đảm bảo tuân thủ quy định về tag
4. **Testing Environment**: Môi trường để test policy hoạt động

User này sẽ không thể tạo tài nguyên EC2, S3, RDS nếu không có tag Environment với giá trị hợp lệ, đảm bảo governance và cost management hiệu quả.
