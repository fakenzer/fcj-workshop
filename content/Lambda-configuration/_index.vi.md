---
title: "Thiết lập Lambda"
weight: 3
chapter: false
pre: " <b> 3. </b> "
---

Chương này hướng dẫn chi tiết cách tạo và cấu hình các Lambda Functions trong AWS để tự động hóa việc quản lý tài nguyên, giám sát chi phí và gửi thông báo khi phát hiện vi phạm quy tắc tagging.

## Mục tiêu

- Thiết lập hệ thống Resource Management System hoàn chỉnh
- Cấu hình các cảnh báo tự động cho việc quản lý tài nguyên
- Tạo Lambda Function ResourceScanner để quét và phát hiện tài nguyên không tuân thủ
- Tạo Lambda Function ResourceDeleter để xóa tài nguyên vi phạm quy tắc
- Tạo Lambda Function NotificationSender để gửi thông báo qua email và Slack

## Kiến thức cần có

- Hiểu biết cơ bản về AWS Lambda
- Kinh nghiệm làm việc với Python
- Kiến thức về AWS SDK (boto3 cho Python)
- Hiểu biết về AWS CloudWatch Events/EventBridge
- Kinh nghiệm với AWS SNS và SES
- Kiến thức về JSON format và cấu hình IAM

## Kết quả đạt được

Sau khi hoàn thành chương này, bạn sẽ có:

- **ResourceScanner Function**: Lambda quét tài nguyên và phát hiện vi phạm
- **ResourceDeleter Function**: Lambda xóa tài nguyên không tuân thủ quy tắc
- **NotificationSender Function**: Lambda gửi thông báo (email)

## Tổng quan kiến trúc

Hệ thống Lambda bao gồm 3 thành phần chính:

1. **ResourceScanner**: Quét tài nguyên AWS và kiểm tra compliance
2. **ResourceDeleter**: Xử lý việc xóa tài nguyên vi phạm quy tắc
3. **NotificationSender**: Gửi thông báo qua email

## Lưu ý quan trọng

- Luôn test Lambda functions trong môi trường development trước
- Cấu hình timeout và memory phù hợp cho mỗi function
- Đảm bảo IAM permissions được cấu hình chính xác
- Backup code và cấu hình Lambda functions thường xuyên
