---
title: "Tạo RDS Instance"
weight: 3
chapter: false
pre: " <b> 7.3 </b> "
---

## Mục tiêu

Test khả năng phát hiện và xử lý RDS instance không có tag

## Các bước thực hiện

### 1. Mở RDS Console

- Truy cập AWS Console
- Chọn service **RDS**

### 2. Create Database

- Click **Create database**
- Chọn **Standard create**
- Chọn engine (MySQL hoặc PostgreSQL)

### 3. Cấu hình Database

- Chọn template **Free tier**
- Đặt DB instance identifier (ví dụ: `test-db-instance`)
- Cấu hình credentials:
  - Master username: `admin`
  - Master password: Đặt password mạnh

### 4. Quan trọng: Không thêm tag

- Ở phần **Additional configuration**
- Tại section **Tags**, **không thêm tag nào**
- Bỏ qua phần này để test việc phát hiện resource không có tag

### 5. Create Database

- Review và click **Create database**
- Chờ database được tạo (có thể mất 5-10 phút)
