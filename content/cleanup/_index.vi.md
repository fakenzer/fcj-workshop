---
title: "Dọn dẹp Tài nguyên AWS"
weight: 8
chapter: false
pre: " <b> 8. </b> "
---

## Xóa Tài nguyên Giám sát

### CloudWatch

- Vào **Dashboards** / Chọn và xóa tất cả các dashboard

### SNS

- Vào **SNS Console** / **Subscriptions** / Xóa tất cả các subscription trước
- Vào **Topics** / Chọn và xóa tất cả các topic

## Xóa Tài nguyên Serverless

### Lambda Functions

- Vào **Lambda Console** / **Functions** / Chọn và xóa tất cả các function

### Step Functions

- Vào **Step Functions Console** / **State machines** / Xóa tất cả các state machine

## Xóa Tài nguyên Event

### EventBridge

- Vào **EventBridge Console** / **Rules** / Xóa các target khỏi rule trước
- Xóa tất cả các rule tùy chỉnh

### Systems Manager

- Vào **Systems Manager Console** / **Parameter Store** / Xóa tất cả các parameter

## Xóa Tài nguyên Bảo mật

### IAM

- Vào **IAM Console** / **Roles** / Tách các policy khỏi role trước
- Xóa các role tùy chỉnh
- Vào **Policies** / Xóa các policy tùy chỉnh

## Xác minh Cuối cùng

### Kiểm tra từng Dịch vụ

- CloudWatch: Không có alarm, dashboard, log group
- SNS: Không có topic hay subscription
- Lambda: Không có function hay layer
- Step Functions: Không có state machine
- EventBridge: Không có rule
- Systems Manager: Không có parameter
- IAM: Không có role hay policy tùy chỉnh

### Theo dõi Hóa đơn

- Vào **Billing Console** / Kiểm tra các khoản phí bất thường
- Thiết lập cảnh báo hóa đơn cho việc giám sát trong tương lai

## Lưu ý Quan trọng

- Luôn sao lưu dữ liệu quan trọng trước khi xóa
- Một số tài nguyên có thể cần thời gian để xóa hoàn toàn
- Xóa các tài nguyên phụ thuộc trước để tránh lỗi
- Kiểm tra dashboard hóa đơn sau khi dọn dẹp
