# CQRS (Command Query Responsibility Segregation) và Ứng Dụng

## 1. Khái niệm cơ bản

CQRS là một mẫu kiến trúc phần mềm dùng để **tách biệt phần ghi (Command)** và **phần đọc (Query)** trong hệ thống.

- **Command (Ghi)**: Thực hiện các hành động thay đổi trạng thái hệ thống.
- **Query (Đọc)**: Truy vấn dữ liệu mà **không thay đổi** trạng thái.

### Ví dụ đơn giản:

| Hành động            | Phân loại | Tác động                  |
|----------------------|----------|---------------------------|
| `POST /orders`       | Command  | Thêm đơn hàng mới         |
| `GET /orders/{id}`   | Query    | Truy vấn chi tiết đơn hàng |
| `PUT /orders/{id}`   | Command  | Cập nhật đơn hàng         |
| `GET /orders`        | Query    | Lấy danh sách đơn hàng    |

---

## 2. Tại sao lại tách?

Đọc và ghi thường có **yêu cầu khác nhau** về hiệu năng, logic nghiệp vụ, và cấu trúc dữ liệu.

CQRS cho phép bạn tối ưu hóa riêng biệt:

- **Ghi**: Tập trung kiểm soát logic nghiệp vụ và tính nhất quán.
- **Đọc**: Tối ưu cho hiệu suất truy vấn, caching, scale.

---

## 3. Kiến trúc tổng thể CQRS

```pgsql

+----------+ +---------------------+ +----------------+
| Client | -----> | Command Handler | -----> | Write Database |
+----------+ +---------------------+ +----------------+

+----------+ +---------------------+ +----------------+
| Client | <----- | Query Handler | <----- | Read Database |
+----------+ +---------------------+ +----------------+
```


> Có thể dùng cùng một database hoặc tách riêng cho đọc và ghi.

---

## 4. Kết hợp với Event Sourcing (nâng cao)

- Khi ghi dữ liệu, bạn lưu **sự kiện (event)** thay vì trạng thái cuối.
- Trạng thái hiện tại có thể được dựng lại bằng cách "replay" các event.
- Phù hợp cho audit, undo/redo, lịch sử thay đổi...

---

## 5. Lợi ích của CQRS

| Lợi ích           | Mô tả                                                              |
|-------------------|---------------------------------------------------------------------|
| Tách biệt rõ ràng | Command và Query độc lập → dễ bảo trì                              |
| Tối ưu hiệu suất  | Dùng cấu trúc DB phù hợp cho từng phần                             |
| Dễ mở rộng        | Scale riêng phần đọc/ghi                                           |
| Bảo mật           | Phân quyền khác nhau cho đọc/ghi                                   |

---

## 6. Nhược điểm và cảnh báo

| Nhược điểm           | Mô tả                                                                 |
|----------------------|----------------------------------------------------------------------|
| Phức tạp hơn         | Quản lý thêm phần đồng bộ, consistency                              |
| Dễ sai nếu không rõ  | Phải xử lý **eventual consistency** nếu dùng DB tách biệt           |
| Không cần cho hệ nhỏ | CRUD đơn giản thì CQRS gây thừa phức tạp                            |

---

## 7. Ứng dụng thực tế

### Khi nên dùng:

- Ứng dụng lớn, nhiều nghiệp vụ
- Yêu cầu mở rộng hiệu năng
- Logic ghi và đọc khác biệt rõ rệt
- Cần audit/log/event tracking

### Ví dụ:

- Thương mại điện tử lớn
- Hệ thống ngân hàng, tài chính
- Business Intelligence (BI), analytics
- Hệ thống IoT, log tracking

---

## 8. Công nghệ hỗ trợ CQRS

### Backend:

- .NET, Java, Node.js, Go, Python

### Thư viện hỗ trợ:

- Axon Framework (Java)
- MediatR (.NET)
- NestJS CQRS Module (Node.js)
- Eventuate, EventStore

### Cơ sở dữ liệu:

- **Write DB**: PostgreSQL, MySQL, MongoDB
- **Read DB**: Redis, Elasticsearch, replicated DB

---

## 9. Tóm tắt

| Tiêu chí        | Query (Đọc)         | Command (Ghi)               |
|------------------|----------------------|------------------------------|
| Mục đích         | Truy vấn dữ liệu     | Thay đổi trạng thái          |
| Side effect      | Không                | Có                           |
| Scale độc lập    | Có thể               | Có thể                       |
| Triển khai       | REST API, GraphQL    | REST API, Message Queue      |

---

## 📌 Ghi chú thêm

- CQRS không phải lúc nào cũng cần thiết — hãy dùng khi có lý do rõ ràng.
- Có thể kết hợp CQRS một phần (chỉ tách Command/Query ở một số phần phức tạp).
