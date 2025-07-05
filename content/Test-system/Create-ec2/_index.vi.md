---
title: "Tạo EC2 Instance"
weight: 1
chapter: false
pre: " <b> 7.1 </b> "
---

## Mục tiêu

Test khả năng phát hiện và xử lý EC2 instance có và không có tag để đánh giá hiệu quả của compliance monitoring system.

## Các bước thực hiện

### 1. Mở EC2 Console

- Đăng nhập vào AWS Management Console
- Tìm kiếm và chọn service **EC2**
- Chọn region phù hợp (ví dụ: us-east-1, ap-southeast-1)
- Đảm bảo bạn đang ở đúng region mà compliance system đang monitor

### 2. Tạo EC2 Instance không có tag (Test case 1)

#### 2.1 Launch Instance

- Click nút **Launch Instance** màu cam
- Đặt tên instance: `test-instance-no-tag`
  ![Name](/images/7.Test/001-nameec2notag.png)

#### 2.2 Chọn AMI (Amazon Machine Image)

- Chọn **Amazon Linux 2023** (Free tier eligible)
- Hoặc chọn **Ubuntu Server 22.04 LTS** (Free tier eligible)

#### 2.3 Chọn Instance Type

- Chọn **t2.micro** (Free tier eligible)
- Instance này có 1 vCPU và 1 GiB Memory
  ![Choose system](/images/7.Test/002-choosesystem.png)

#### 2.4 Cấu hình Key Pair

- Chọn key pair có sẵn hoặc tạo mới
- Nếu tạo mới: đặt tên `test-keypair` và download file .pem
  ![Key pair](/images/7.Test/003-keypair.png)

#### 2.5 Network Settings

- Giữ mặc định VPC và subnet
- Tạo security group mới hoặc chọn có sẵn
- Cho phép SSH (port 22) từ IP của bạn

#### 2.6 **QUAN TRỌNG: Không thêm tag**

- Ở phần **Advanced details**, mở rộng **Tags**
- **KHÔNG thêm tag nào cả** - bỏ qua phần này hoàn toàn
- Mục đích: test việc phát hiện resource không tuân thủ tagging policy

#### 2.7 Launch Instance

- Review lại cấu hình
- Click **Launch instance**
- Chờ instance chuyển sang trạng thái **Running**

### 3. Tạo EC2 Instance có tag (Test case 2)

#### 3.1 Launch Instance thứ hai

- Click **Launch Instance** một lần nữa
- Đặt tên instance: `test-instance-with-tag`

#### 3.2 Cấu hình tương tự

- Chọn cùng AMI (Amazon Linux 2023 hoặc Ubuntu)
- Chọn instance type **t2.micro**
- Sử dụng cùng key pair đã tạo
- Cấu hình network settings tương tự

#### 3.3 **QUAN TRỌNG: Thêm tag bắt buộc**

- Ở phần **Advanced details**, mở rộng **Tags**
- Click **Add tag** và thêm các tag sau:

| Key           | Value | Description        |
| ------------- | ----- | ------------------ |
| `Environment` | `lab` | Môi trường sử dụng |

![name with tag](/images/7.Test/004-namewithtag.png)

## Lưu ý quan trọng

- Chỉ chạy test trong môi trường development/staging
- Đảm bảo terminate instances sau khi test để tránh phí phát sinh
- Kiểm tra IAM permissions trước khi thực hiện
- Ghi chú lại Instance ID và thời gian tạo để theo dõi kết quả
  ![All ec2](/images/7.Test/005-ec2-instance.png)
