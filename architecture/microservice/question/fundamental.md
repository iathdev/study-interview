Các nguyên tắc cơ bản (fundamentals) của thiết kế **Microservices Architecture** bao gồm:

- **Độc lập (Decentralized Data Management)**: Mỗi microservice quản lý cơ sở dữ liệu riêng, tránh phụ thuộc vào cơ sở dữ liệu tập trung để đảm bảo tính tự trị.
- **Phân chia theo chức năng (Domain-Driven Design)**: Thiết kế dựa trên các miền nghiệp vụ (business domains), chia nhỏ ứng dụng thành các dịch vụ có chức năng rõ ràng.
- **Tính độc lập triển khai (Independent Deployment)**: Mỗi microservice có thể được phát triển, thử nghiệm và triển khai mà không ảnh hưởng đến các dịch vụ khác.
- **Giao tiếp thông qua API (Inter-Service Communication)**: Sử dụng giao thức nhẹ như REST, gRPC hoặc message queues để các microservice tương tác với nhau.
- **Tính phục hồi (Resilience)**: Thiết kế để xử lý lỗi cục bộ mà không làm sụp đổ toàn hệ thống, thường thông qua cơ chế như circuit breakers.
- **Khả năng mở rộng (Scalability)**: Cho phép mở rộng từng microservice riêng lẻ dựa trên nhu cầu, thay vì toàn bộ ứng dụng.
- **Tự động hóa (Automation)**: Sử dụng CI/CD và các công cụ quản lý container (như Docker, Kubernetes) để tự động hóa triển khai và quản lý.
- **Combine loose coupling with high cohesion**: Đảm bảo các dịch vụ độc lập nhưng gắn kết chặt chẽ trong nội bộ.

---

## Circuit breakers
- Circuit breakers là một cơ chế trong kiến trúc phần mềm, đặc biệt trong microservices, dùng để bảo vệ hệ thống khỏi lỗi lan truyền khi một dịch vụ hoặc thành phần gặp sự cố. Khi một dịch vụ thất bại liên tục (ví dụ: không phản hồi), circuit breaker sẽ "ngắt" kết nối tạm thời, ngăn các yêu cầu tiếp tục gửi đến dịch vụ đó. Sau một thời gian, nó sẽ thử khôi phục lại kết nối để kiểm tra xem dịch vụ đã hoạt động trở lại chưa. 
- Ý tưởng này tương tự như cầu dao điện: khi phát hiện sự cố, nó ngắt dòng để tránh hư hại thêm. Trong microservices, circuit breakers thường được tích hợp với thư viện như Hystrix hoặc Resilience4j để tăng tính phục hồi (resilience).-