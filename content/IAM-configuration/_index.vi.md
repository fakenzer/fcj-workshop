---
title: "Cấu hình IAM"
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

Chương này hướng dẫn chi tiết cách tạo và cấu hình IAM Policy và Role tùy chỉnh trong AWS để quản lý quyền truy cập, tối ưu hóa chi phí và đảm bảo tuân thủ quy tắc tagging.

## Mục tiêu

- Tạo IAM Policy với các quyền cụ thể cho việc quản lý tài nguyên AWS
- Tạo IAM Policy yêu cầu bắt buộc tag **"Environment"** cho các tài nguyên
- Tạo IAM Role có thể được sử dụng bởi các dịch vụ AWS
- Thiết lập hệ thống phân quyền an toàn và hiệu quả cho automation

## Kiến thức cần có

- Hiểu biết cơ bản về AWS IAM
- Kinh nghiệm làm việc với AWS Console
- Hiểu biết về JSON format
- Kiến thức về AWS tagging best practices

## Kết quả đạt được

Sau khi hoàn thành chương này, bạn sẽ có:

- **ManageResourcePolicy**: Chính sách IAM với quyền quản lý tài nguyên, tags và chi phí
- **RequireEnvironmentTagPolicy**: Chính sách yêu cầu bắt buộc tag environment
- **ResourceManagerRole**: Role toàn diện có thể được sử dụng cho automation và quản lý tài nguyên

## Lưu ý quan trọng

- Luôn kiểm tra kỹ các quyền trước khi tạo policy
- Tuân thủ nguyên tắc **"least privilege"** khi cấp quyền
- Thường xuyên review và cập nhật policy theo nhu cầu thực tế
- Backup cấu hình policy và role trước khi thay đổi
- Đảm bảo tất cả tài nguyên đều có tag **Environment** phù hợp
