Cam kết Hai Giai đoạn (Two-Phase Commit - 2PC) là một thuật toán phân tán dùng để đảm bảo tính **nguyên tử** và **nhất quán** trong các hệ thống cơ sở dữ liệu phân tán. Nó điều phối các giao dịch để đảm bảo tất cả các nút tham gia hoặc cùng cam kết (commit) hoặc cùng hủy (abort), tránh cập nhật không đầy đủ. Dưới đây là tóm tắt ngắn gọn:

### Các thành phần
- Coordinator (Điều phối viên): Điều khiển quá trình commit/rollback.
- Participants (Người tham gia): Các node chứa dữ liệu cần commit.

### Các giai đoạn của 2PC
1. **Giai đoạn Chuẩn bị (Prepare Phase)**:
   - Điều phối viên (transaction manager) gửi thông điệp "chuẩn bị" đến tất cả các nút tham gia.
   - Mỗi nút thực hiện giao dịch cục bộ (đến bước trước khi cam kết) và trả lời:
     - **"Sẵn sàng" (Ready)**: Nút có thể commit giao dịch.
     - **"Hủy" (Abort)**: Nút không thể commit (do lỗi hoặc xung đột).
   - Các nút ghi quyết định vào nhật ký (log) để đảm bảo tính bền vững.

2. **Giai đoạn Cam kết (Commit Phase)**:
   - Nếu **tất cả nút trả lời "Sẵn sàng"**, điều phối viên gửi thông điệp "cam kết", và các nút hoàn tất giao dịch.
   - Nếu **bất kỳ nút nào trả lời "Hủy"** hoặc không phản hồi, điều phối viên gửi thông điệp "hủy", và tất cả nút hoàn tác (rollback) các thay đổi.
   - Các nút xác nhận việc cam kết hoặc hủy cho điều phối viên.

### Đặc điểm chính
- **Tính nguyên tử**: Đảm bảo tất cả nút cam kết hoặc hủy, tránh cam kết một phần.
- **Tính bền vững**: Nhật ký đảm bảo các nút có thể khôi phục và hoàn thành giao dịch sau lỗi.
- **Tính chặn (Blocking)**: Nếu điều phối viên hoặc nút gặp sự cố, giao thức có thể bị chặn cho đến khi khôi phục.
- **Đồng thuận**: Yêu cầu tất cả nút đồng ý về kết quả.

### Ưu điểm
- Đảm bảo tính nhất quán trong các hệ thống phân tán.
- Đơn giản, được sử dụng rộng rãi trong cơ sở dữ liệu phân tán.

### Nhược điểm
- **Chặn**: Lỗi của nút hoặc điều phối viên có thể gây chậm trễ.
- **Khả năng mở rộng**: Điều phối viên có thể trở thành nút thắt cổ chai trong hệ thống lớn.
- **Chi phí**: Yêu cầu nhiều vòng giao tiếp, làm tăng độ trễ.

### Ví dụ
Hãy tưởng tượng một hệ thống ngân hàng chuyển tiền giữa hai tài khoản trên hai máy chủ khác nhau:
1. Điều phối viên yêu cầu cả hai máy chủ chuẩn bị thao tác ghi nợ và ghi có.
2. Nếu cả hai máy chủ xác nhận sẵn sàng, điều phối viên ra lệnh cam kết.
3. Nếu một máy chủ thất bại hoặc không thể thực hiện, điều phối viên ra lệnh hủy, đảm bảo không mất hoặc tạo ra tiền.

### Các phương pháp thay thế
- **Cam kết Ba Giai đoạn (3PC)**: Giảm tình trạng chặn bằng cách thêm giai đoạn tiền cam kết, nhưng phức tạp hơn.
- **Paxos hoặc Raft**: Các thuật toán đồng thuận cho hệ thống phân tán chịu lỗi, thường dùng trong cơ sở dữ liệu hiện đại.
