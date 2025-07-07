---
title: "Tạo RDS Database"
weight: 3
chapter: false
pre: " <b> 7.3 </b> "
---

## Mục tiêu

Test khả năng phát hiện và xử lý RDS database có và không có tag để đánh giá hiệu quả của compliance monitoring system đối với database resources.

## Các bước thực hiện

### 1. Mở RDS Console

- Đăng nhập vào AWS Management Console
- Tìm kiếm và chọn service **RDS**
- Đảm bảo bạn có quyền tạo và quản lý RDS database

### 2. Tạo RDS Database KHÔNG có tag (Test case 1)

#### 2.1 Create Database

- Click nút **Create database**
- Chọn **Standard create** để có nhiều option cấu hình

#### 2.2 Engine Options

- Chọn **MySQL** hoặc **PostgreSQL** (miễn phí tier)
- Chọn **Version** mới nhất
- Chọn **Templates**: **Free tier** để tiết kiệm chi phí
  ![Choose free tie](/images/7.Test/009-choosefreetie.png)

#### 2.3 Settings

- **DB instance identifier**: `test-db-no-tag`
- **Master username**: `admin`
- **Master password**: Tạo password mạnh (ít nhất 8 ký tự)
- **Confirm password**: Nhập lại password

![Setting](/images/7.Test/010-settingaccount.png)

#### 2.4 Instance Configuration

- **DB instance class**: Chọn **db.t3.micro** (Free tier eligible)
- **Storage type**: **General Purpose SSD (gp2)**
- **Allocated storage**: 20 GB (minimum)
  ![Setting](/images/7.Test/012-setting.png)

#### 2.5 Availability & Durability

- **Multi-AZ deployment**: Chọn **No** để tiết kiệm chi phí
- Dành cho môi trường testing

#### 2.6 Connectivity

- **Virtual private cloud (VPC)**: Chọn Default VPC
- **Subnet group**: Chọn default
- **Public access**: **Yes** (để có thể test kết nối)
- **VPC security group**: Chọn default hoặc tạo security group mới

#### 2.7 Database Authentication

- Giữ mặc định **Password authentication**

#### 2.8 **QUAN TRỌNG: Không thêm tag**

- Ở phần **Additional configuration**, mở rộng **Tags**
- **KHÔNG thêm tag nào**
- Bỏ qua phần này hoàn toàn
- Mục đích: test việc phát hiện RDS database không tuân thủ tagging policy

![No Tag](/images/7.Test/013-notag.png)

#### 2.9 Create Database

- Review lại tất cả settings
- Click **Create database**
- Chờ database được tạo (khoảng 10-15 phút)

### 3. Tạo RDS Database CÓ tag (Test case 2)

#### 3.1 Create Database thứ hai

- Click **Create database** một lần nữa
- Chọn **Standard create**

#### 3.2 Engine Options

- Chọn cùng engine (MySQL/PostgreSQL)
- Chọn cùng version
- Chọn **Templates**: **Free tier**

#### 3.3 Settings

- **DB instance identifier**: `test-db-with-tag`
- **Master username**: `admin`
- **Master password**: Sử dụng cùng password
- **Confirm password**: Nhập lại password

#### 3.4 Cấu hình tương tự

- Cùng Instance Configuration
- Cùng Availability & Durability settings
- Cùng Connectivity settings
- Cùng Database Authentication

#### 3.5 **QUAN TRỌNG: Thêm tag bắt buộc**

- Ở phần **Additional configuration**, mở rộng **Tags**
- Click **Add tag**
- Thêm các tag sau:

| Key           | Value | Description        |
| ------------- | ----- | ------------------ |
| `Environment` | `lab` | Môi trường sử dụng |

![Tag](/images/7.Test/011-tag.png)

#### 3.6 Create Database

- Review tất cả settings
- Click **Create database**
- Chờ database được tạo thành công

### 4. Lưu ý quan trọng

#### 4.1 Chi phí

- RDS Free tier có giới hạn 750 giờ/tháng
- Theo dõi usage để tránh phát sinh chi phí
- Có thể delete database sau khi test xong

#### 4.2 Security

- Chỉ mở public access trong môi trường test
- Sử dụng strong password
- Cấu hình security group cẩn thận

#### 4.3 Monitoring

- Database sẽ được compliance monitoring system kiểm tra
- Kỳ vọng:
  - `test-db-no-tag`: Trigger compliance violation
  - `test-db-with-tag`: Pass compliance check
