---
title: "Cấu hình Resource Management Alerts"
weight: 5
chapter: false
pre: " <b> 3.5. </b> "
---

Chương này hướng dẫn chi tiết cách thiết lập SNS Topic và Subscription để nhận thông báo email cho việc quản lý tài nguyên AWS.

## Mục tiêu

- Tạo SNS Topic cho hệ thống cảnh báo quản lý tài nguyên
- Cấu hình Email Subscription để nhận thông báo
- Thiết lập endpoint email cho việc nhận alerts

## Kết quả đạt được

Sau khi hoàn thành chương này, bạn sẽ có:

- **SNS Topic**: `resource-management-alerts` với cấu hình Standard
- **Email Subscription**: Thông báo tự động qua email your-email@gmail.com
- **Hệ thống sẵn sàng**: Để tích hợp với các dịch vụ monitoring khác

## Các bước thực hiện

### Bước 1: Truy cập Amazon SNS

![SNS](/images/3.Lambda/008-SNS.png)

1. Đăng nhập vào AWS Console
2. Tìm kiếm và chọn **SNS** trong danh sách dịch vụ
3. Trong menu bên trái, chọn **Topics**

### Bước 2: Tạo SNS Topic

1. Nhấp vào **Create topic**
2. Chọn **Type**: Standard
3. Nhập **Name**: `resource-management-alerts`
4. Để các cài đặt khác mặc định
5. Nhấp **Create topic**
   ![Create SNS topic](/images/3.Lambda/009-createtopic.png)

### Bước 3: Tạo Email Subscription

1. Trong SNS Topic vừa tạo, chọn tab **Subscriptions**
2. Nhấp **Create subscription**
3. Cấu hình subscription:
   - **Topic ARN**: Sẽ được tự động điền
   - **Protocol**: Chọn **Email**
   - **Endpoint**: Nhập `your-email@gmail.com`
4. Nhấp **Create subscription**
   ![Create Subscriptions](/images/3.Lambda/010-subscriptions.png)

### Bước 4: Xác nhận Email Subscription

1. Kiểm tra hộp thư email your-email@gmail.com
2. Tìm email từ **AWS Notifications**
3. Nhấp vào link **Confirm subscription** trong email
4. Quay lại AWS Console để kiểm tra status subscription đã chuyển thành **Confirmed**
   ![email](/images/3.Lambda/011-email.png)

### Bước 5: Kiểm tra cấu hình

![Confirm email](/images/3.Lambda/012-confirmemail.png)

1. Trong SNS Topic, tab **Subscriptions**
2. Xác nhận subscription status là **Confirmed**
3. Có thể test bằng cách nhấp **Publish message** để gửi thử

## Test thông báo

### Gửi test message

![Test mail](/images/3.Lambda/013-testmail.png)

1. Trong SNS Topic, chọn **Publish message**
2. Nhập **Subject**: `Test Resource Management Alert`
3. Nhập **Message**:
   ```
   This is a test message for resource management alerts.
   System is working correctly.
   ```
4. Nhấp **Publish message**
5. Kiểm tra email để xác nhận nhận được thông báo
   ![Test message](/images/3.Lambda/014-testmessage.png)

## Lưu ý quan trọng

- **Email delivery**: Kiểm tra spam folder nếu không nhận được email
- **Subscription status**: Đảm bảo status là "Confirmed" trước khi sử dụng
- **Topic ARN**: Lưu lại Topic ARN để sử dụng cho các dịch vụ khác
- **Cost optimization**: Standard topics có chi phí theo số lượng message gửi
- **Security**: Chỉ chia sẻ Topic ARN với những dịch vụ cần thiết
