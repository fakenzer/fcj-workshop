---
title: "Tạo S3 Bucket"
weight: 2
chapter: false
pre: " <b> 7.2 </b> "
---

## Mục tiêu

Test khả năng phát hiện và xử lý S3 bucket có và không có tag để đánh giá hiệu quả của compliance monitoring system đối với storage resources.

## Các bước thực hiện

### 1. Mở S3 Console

- Đăng nhập vào AWS Management Console
- Tìm kiếm và chọn service **S3**
- Đảm bảo bạn có quyền tạo và quản lý S3 bucket

### 2. Tạo S3 Bucket KHÔNG có tag (Test case 1)

#### 2.1 Create Bucket

- Click nút **Create bucket** màu cam
- Nhập bucket name: `test-bucket-no-tag`

  ![Name s3](/images/7.Test/006-nameofs3.png)

#### 2.2 Chọn AWS Region

- Chọn region phù hợp (ví dụ: us-east-1, ap-southeast-1)
- Đảm bảo cùng region với compliance monitoring system

#### 2.3 Object Ownership

- Chọn **ACLs disabled (recommended)**
- Bucket owner sẽ own tất cả objects

#### 2.4 Block Public Access settings

- Giữ mặc định: **Block all public access** được tick
- Đảm bảo bucket được bảo mật

#### 2.5 Bucket Versioning

- Chọn **Disable** để đơn giản hóa test
- Có thể enable nếu cần test versioning compliance

#### 2.6 Default encryption

- Chọn **Amazon S3 managed keys (SSE-S3)**
- Hoặc giữ mặc định

#### 2.7 **QUAN TRỌNG: Không thêm tag**

- Ở phần **Tags**, **KHÔNG thêm tag nào**
- Bỏ qua phần này hoàn toàn
- Mục đích: test việc phát hiện S3 bucket không tuân thủ tagging policy

#### 2.8 Create Bucket

- Review lại tất cả settings
- Click **Create bucket**
- Chờ bucket được tạo thành công

### 3. Tạo S3 Bucket CÓ tag (Test case 2)

#### 3.1 Create Bucket thứ hai

- Click **Create bucket** một lần nữa
- Nhập bucket name: `test-bucket-with-tag`
  ![Name s3](/images/7.Test/007-nameofs3withtag.png)

#### 3.2 Cấu hình tương tự

- Chọn cùng AWS Region
- Cùng Object Ownership settings
- Cùng Block Public Access settings
- Cùng Versioning và encryption settings

#### 3.3 **QUAN TRỌNG: Thêm tag bắt buộc**

- Ở phần **Tags**, click **Add tag**
- Thêm các tag sau:

| Key           | Value | Description        |
| ------------- | ----- | ------------------ |
| `Environment` | `lab` | Môi trường sử dụng |

![Tag s3](/images/7.Test/008-tags3.png)

#### 3.4 Create Bucket

- Review tất cả settings
- Click **Create bucket**
- Chờ bucket được tạo thành công

#### 4.4 Create Bucket

- Review và create bucket

### 5. Upload test file (Optional)

#### 5.1 Upload file vào bucket không có tag

- Click vào bucket `test-bucket-no-tag`
- Click **Upload**
- Chọn file test hoặc tạo file text đơn giản
- Upload file

#### 5.2 Upload file vào bucket có tag

- Làm tương tự với bucket `test-bucket-with-tag`
- Mục đích: test compliance check cho object bên trong bucket

### 6. Kiểm tra kết quả

#### 6.1 Xem danh sách buckets

- Trong S3 Console, xem danh sách buckets
- Ghi chú tên và creation time của 2 buckets

#### 6.2 Kiểm tra tags

- Click vào từng bucket
- Vào tab **Properties**
- Scroll xuống phần **Tags** để xem chi tiết
- So sánh sự khác biệt giữa các bucket
