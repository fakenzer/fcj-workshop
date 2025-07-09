---
title: "Workshop AWS Tìm và diệt các chi phí tự động bằng tag"
weight: 1
chapter: false
pre: " <b> 1. </b> "
---

# Giới thiệu về workshop

Trong hành trình khám phá và làm việc với AWS, chắc hẳn ai trong chúng ta cũng từng gặp những khoảnh khắc "giật mình" khi nhận hóa đơn dịch vụ vượt xa dự tính, chỉ vì lỡ quên tắt các tài nguyên không còn dùng đến. Có thể là sau một buổi lab vội vàng, bạn quên dọn dẹp EC2 instance hay khi chuyển vùng, bạn không để ý đến những EC2, RDS, S3 bucket vẫn âm thầm "ngốn" chi phí hoặc chỉ đơn giản là bật dịch vụ để dùng vài ngày nhưng rồi để đó cả tháng. Những sai sót nhỏ này có thể dẫn đến hậu quả lớn về tài chính. Để giải quyết vấn đề, mình muốn chia sẻ ý tưởng bài lab này về một hệ thống tự động giúp tìm kiếm và xóa bỏ các tài nguyên AWS không sử dụng dựa vào tag, giúp tiết kiệm chi phí và yên tâm hơn khi bắt đầu học cloud.

### Lược đồ kiến trúc

Sơ đồ sau đây cung cấp một hình ảnh trực quan về các dịch vụ được sử dụng trong lab đơn giản này và cách chúng được kết nối. Ứng dụng này sử dụng **IAM, Lambda, SNS, AWS Step Functions, Amazon EventBridge, AWS Systems Manager.** để xóa các tài nguyên trong bài lab như **VPC, EC2, S3, RDS.**

Khi bạn đi qua workshop, bạn sẽ tìm hiểu chi tiết về các dịch vụ này và tìm thấy các tài nguyên giúp bạn nắm bắt chúng nhanh chóng.

![Architecture Application Schema](/images/1.Introduce/001-architectdiagram.png)

### Những gì bạn sẽ đạt được

Một hệ thống giúp bạn tìm và tối ưu chi phí với giá rất rẻ hoặc miễn phí nếu bạn cấu hình đúng cách.

Hiểu hơn về các dịch vụ của AWS như:

- **IAM**: Quản lý người dùng và phân quyền truy cập. Cấu hình đúng giúp hạn chế rủi ro và kiểm soát tài nguyên hiệu quả.
- **Lambda**: Chạy mã không cần máy chủ (serverless), chỉ tính phí khi thực thi – rất tiết kiệm.
- **AWS EventBridge**: Tự động xử lý các sự kiện AWS để kích hoạt workflow tối ưu chi phí.
- **CloudWatch**: Theo dõi hiệu suất, log và cảnh báo – giúp giám sát chi phí và hoạt động bất thường.
- **...**

### Yêu cầu

- **Tài khoản AWS:** với quyền quản trị viên.

- **Nếu bạn sử dụng tài khoản AWS FreeTier thì thật tuyệt vời**

{{% notice note%}}
Các tài khoản được tạo trong vòng 24 giờ qua có thể chưa có quyền truy cập vào các dịch vụ cần thiết cho lab này
{{% /notice%}}
