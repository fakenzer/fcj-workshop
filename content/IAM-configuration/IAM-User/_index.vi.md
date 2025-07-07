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

### Bước 3: Cấu hình User

1. **User name**: `TagEnforcementUser`
2. Chọn **"Provide user access to the AWS Management Console"**
3. Chọn **"I want to create an IAM user"**
4. **Console password**: Chọn **"Custom password"** và nhập mật khẩu mạnh
5. Bỏ tick **"Users must create a new password at next sign-in"** (tùy chọn)
6. Nhấp **"Next"**

### Bước 4: Gán Permissions

1. Chọn **"Attach policies directly"**
2. Tìm kiếm **"RequireEnvironmentTagPolicy"**
3. Tick chọn policy **RequireEnvironmentTagPolicy**
4. Thêm các policy cơ bản khác nếu cần:
   - **IAMReadOnlyAccess** (để user có thể xem thông tin IAM)
   - **ReadOnlyAccess** (để user có thể xem tài nguyên hiện có)

### Bước 5: Review và Create

1. Review lại tất cả thông tin
2. Nhấp **"Create user"**

## Kết quả

Sau khi hoàn thành phần này, bạn sẽ có:

1. **TagEnforcementUser**: User với quyền bị giới hạn bởi tag policy
2. **Tag Enforcement**: Tất cả tài nguyên mới phải có tag Environment
3. **Compliance**: Đảm bảo tuân thủ quy định về tag
4. **Testing Environment**: Môi trường để test policy hoạt động

User này sẽ không thể tạo tài nguyên EC2, S3, RDS nếu không có tag Environment với giá trị hợp lệ, đảm bảo governance và cost management hiệu quả.
