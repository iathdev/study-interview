gRPC nhanh hơn REST và việc truyền dữ liệu dưới dạng binary góp phần quan trọng vào hiệu suất này. Dưới đây là lý do chi tiết:

### 1. **Tại sao gRPC nhanh hơn REST?**

- **Giao thức HTTP/2 so với HTTP/1.1**:
  - gRPC sử dụng **HTTP/2**, hỗ trợ các tính năng như **multiplexing** (nhiều yêu cầu trên một kết nối TCP), **nén header**, và **kết nối liên tục**, giúp giảm độ trễ và tăng hiệu quả so với HTTP/1.1 thường được dùng trong REST.
  - REST thường dựa trên HTTP/1.1, yêu cầu mở nhiều kết nối cho các yêu cầu đồng thời, dẫn đến chi phí cao hơn về tài nguyên và thời gian.

- **Sử dụng Protocol Buffers (Protobuf)**:
  - gRPC dùng **Protocol Buffers**, một định dạng nhị phân (binary) nhỏ gọn và hiệu quả, thay vì JSON hoặc XML như REST. Protobuf được thiết kế để tuần tự hóa (serialize) và giải tuần tự hóa (deserialize) dữ liệu nhanh hơn, giảm kích thước payload.
  - JSON/XML của REST là dạng văn bản (text-based), cồng kềnh hơn và yêu cầu xử lý phức tạp hơn khi phân tích cú pháp.

- **Hiệu suất truyền tải**:
  - gRPC hỗ trợ **streaming** (unary, server streaming, client streaming, bidirectional streaming), cho phép truyền dữ liệu theo thời gian thực mà không cần tạo nhiều yêu cầu như REST.
  - REST thường chỉ hỗ trợ mô hình request-response đơn giản, không tối ưu cho các ứng dụng cần truyền dữ liệu liên tục.

- **Định nghĩa giao diện chặt chẽ**:
  - gRPC sử dụng **.proto files** để định nghĩa schema rõ ràng, giúp tự động tạo mã client/server, giảm lỗi và tối ưu hóa giao tiếp. REST dựa vào tài liệu API (như OpenAPI), dễ gây sai lệch và không tự động hóa tốt bằng.

- **Tối ưu hóa mạng**:
  - gRPC giảm thiểu số lượng gói tin và sử dụng **nén dữ liệu** hiệu quả hơn, đặc biệt với các yêu cầu lớn hoặc lặp lại.
  - REST với JSON/XML thường tạo ra payload lớn hơn, dẫn đến thời gian truyền lâu hơn.

### 2. **Tại sao truyền binary lại nhanh?**

- **Kích thước dữ liệu nhỏ hơn**:
  - Dữ liệu binary (như Protobuf) được mã hóa thành chuỗi byte compact, loại bỏ các thành phần dư thừa (như dấu ngoặc, khoảng trắng trong JSON). Ví dụ, một đối tượng JSON `{"name": "John", "age": 30}` có thể lớn hơn gấp nhiều lần so với bản mã hóa binary tương ứng.
  - Kích thước nhỏ hơn đồng nghĩa với ít dữ liệu cần truyền qua mạng, giảm thời gian truyền và băng thông.

- **Tốc độ tuần tự hóa/giải tuần tự hóa**:
  - Protobuf sử dụng các quy tắc mã hóa cố định, giúp việc tuần tự hóa (serialize) và giải tuần tự hóa (deserialize) nhanh hơn nhiều so với phân tích cú pháp JSON/XML, vốn đòi hỏi xử lý văn bản phức tạp.
  - Binary không cần chuyển đổi giữa các định dạng văn bản, giảm chi phí CPU.

- **Tối ưu cho máy tính**:
  - Dữ liệu binary gần với cách máy tính lưu trữ và xử lý dữ liệu (byte), giảm bước trung gian so với văn bản. Điều này đặc biệt quan trọng trong các hệ thống yêu cầu hiệu suất cao, như microservices hoặc ứng dụng thời gian thực.

- **Nén hiệu quả**:
  - Binary dễ nén hơn và kết hợp tốt với các cơ chế nén của HTTP/2, giảm thêm kích thước dữ liệu khi truyền.

### Tóm lại:
| Tiêu chí        | REST (JSON + HTTP/1.1) | gRPC (Protobuf + HTTP/2)            |
| --------------- | ---------------------- | ----------------------------------- |
| Data Format     | Text (JSON)            | Binary (Protobuf)                   |
| Performance     | Trung bình             | Rất nhanh                           |
| Payload Size    | Lớn hơn                | Nhỏ hơn                             |
| Streaming       | Không hỗ trợ tốt       | Hỗ trợ streaming (1 chiều, 2 chiều) |
| Code Generation | Không tích hợp sẵn     | Có, từ `.proto`                     |
| Dễ debug        | Dễ hơn (vì là text)    | Khó hơn nếu không dùng tooling      |
