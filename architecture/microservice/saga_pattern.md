**Mô hình Saga** là một thiết kế trong hệ thống phân tán, dùng để quản lý các giao dịch phức tạp, kéo dài qua nhiều dịch vụ trong kiến trúc microservices, đảm bảo tính nhất quán dữ liệu mà không cần giao dịch phân tán truyền thống.

### Khái niệm chính
- **Saga**: Chuỗi giao dịch cục bộ, mỗi dịch vụ thực hiện một bước và kích hoạt bước tiếp theo qua thông điệp không đồng bộ (sự kiện hoặc lệnh).
- **Mục đích**: Đảm bảo tính nhất quán, giảm phụ thuộc, tăng khả năng mở rộng và phục hồi lỗi.

### Loại Saga
1. **Choreography**: Dịch vụ tự phối hợp qua sự kiện, linh hoạt nhưng khó theo dõi.
2. **Orchestration**: Một dịch vụ điều phối ra lệnh, dễ quản lý nhưng có nguy cơ tập trung.

### Cách hoạt động
- Mỗi dịch vụ thực hiện giao dịch cục bộ, phát sự kiện/lệnh để tiếp tục.
- Nếu lỗi, thực hiện **giao dịch bù trừ** để hoàn tác các bước trước.

### Ví dụ
Đặt hàng → Thanh toán → Giao hàng. Nếu thanh toán thất bại, hủy đơn hàng.

### Ưu điểm
- Linh hoạt, mở rộng tốt, xử lý lỗi hiệu quả.

### Nhược điểm
- Phức tạp trong quản lý, không đảm bảo nguyên tử hoàn toàn.
