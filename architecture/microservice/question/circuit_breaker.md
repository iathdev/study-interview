Trong kiến trúc microservice, **Circuit Breaker** là một cơ chế bảo vệ hệ thống, giúp xử lý lỗi khi một dịch vụ bị lỗi hoặc phản hồi chậm, tránh làm sập cả hệ thống. Nó hoạt động như một "cầu chì" trong hệ thống điện, ngắt kết nối tạm thời đến dịch vụ gặp vấn đề để bảo vệ các thành phần khác.

### **Circuit Breaker làm gì?**
1. **Ngăn chặn lỗi lan truyền (Cascading Failures)**:
   - Khi một microservice (ví dụ: Payment Service) bị lỗi hoặc quá tải, Circuit Breaker ngăn các yêu cầu tiếp tục gửi đến dịch vụ đó, tránh làm ảnh hưởng đến các dịch vụ khác hoặc toàn bộ hệ thống.

2. **Giảm thời gian chờ không cần thiết**:
   - Nếu một dịch vụ phản hồi chậm (timeout), Circuit Breaker sẽ "ngắt mạch", trả về lỗi ngay lập tức thay vì để client hoặc các dịch vụ khác chờ lâu.

3. **Cho phép phục hồi (Recovery)**:
   - Circuit Breaker theo dõi trạng thái của dịch vụ. Khi dịch vụ có dấu hiệu hoạt động lại bình thường, nó sẽ cho phép thử gửi yêu cầu lại.

4. **Cải thiện trải nghiệm người dùng**:
   - Thay vì client nhận lỗi không rõ ràng hoặc chờ lâu, Circuit Breaker có thể trả về phản hồi mặc định (fallback) hoặc thông báo lỗi thân thiện (ví dụ: "Hệ thống thanh toán tạm thời không khả dụng, vui lòng thử lại sau").

### **Cách Circuit Breaker hoạt động**
Circuit Breaker có 3 trạng thái chính:
1. **Closed (Đóng)**:
   - Các yêu cầu được gửi bình thường đến dịch vụ.
   - Nếu tỷ lệ lỗi hoặc timeout vượt ngưỡng (ví dụ: 50% yêu cầu thất bại), Circuit Breaker chuyển sang trạng thái **Open**.

2. **Open (Mở)**:
   - Ngăn toàn bộ yêu cầu đến dịch vụ bị lỗi, trả về lỗi hoặc fallback ngay lập tức.
   - Sau một khoảng thời gian (timeout), Circuit Breaker chuyển sang trạng thái **Half-Open**.

3. **Half-Open (Nửa mở)**:
   - Cho phép thử một số yêu cầu để kiểm tra xem dịch vụ đã phục hồi chưa.
   - Nếu yêu cầu thành công, chuyển về **Closed**; nếu thất bại, quay lại **Open**.

### **Ví dụ trong flow đặt hàng**
- Trong flow đặt hàng (như đã mô tả trước), nếu **Payment Service** bị lỗi (timeout hoặc trả về HTTP 500):
  - Circuit Breaker (trong Order Service) phát hiện lỗi, chuyển sang trạng thái **Open**, ngừng gửi yêu cầu đến Payment Service.
  - Order Service trả về fallback (ví dụ: thông báo "Thanh toán không khả dụng, vui lòng thử lại").
  - Sau 30 giây, Circuit Breaker thử gửi lại một yêu cầu (**Half-Open**). Nếu Payment Service hoạt động, trạng thái trở về **Closed**.

### **Công cụ hỗ trợ Circuit Breaker**
- **Hystrix** (Netflix): Thư viện Java phổ biến, cung cấp Circuit Breaker, fallback, và monitoring.
- **Resilience4j**: Thư viện nhẹ, thay thế Hystrix, hỗ trợ Java hiện đại.
- **Spring Cloud Circuit Breaker**: Tích hợp với các framework như Spring Boot.
- **Istio/Envoy**: Cung cấp Circuit Breaker ở tầng mạng trong Kubernetes.

### **Lợi ích**
- Tăng độ tin cậy của hệ thống.
- Giảm tải cho dịch vụ bị lỗi, tạo cơ hội để nó phục hồi.
- Cải thiện hiệu suất bằng cách tránh chờ đợi không cần thiết.

### **Thách thức**
- Cần cấu hình ngưỡng lỗi và thời gian timeout hợp lý.
- Xử lý fallback đúng cách để không ảnh hưởng trải nghiệm người dùng.
- Tăng độ phức tạp trong việc giám sát và debug.
