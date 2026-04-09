# Giải thích về ProxySQL

## ProxySQL là gì?

ProxySQL là một lớp proxy trung gian cho MySQL/MariaDB. Nó nằm giữa ứng dụng và database, giúp thực hiện các tác vụ như route truy vấn thông minh, cân bằng tải (load balancing), connection pooling, failover, và thống kê truy vấn thời gian thực mà không cần thay đổi code ứng dụng.

## Vị trí trong hệ thống

Ứng dụng -> ProxySQL -> Các database MySQL/MariaDB

ProxySQL hoạt động như cổng trung gian: nhận truy vấn từ client, phân tích và định tuyến đến server backend phù hợp.

## Tính năng nổi bật

| Tính năng              | Mô tả |
|------------------------|------|
| Load balancing         | Cân bằng truy vấn đọc giữa nhiều replica và ghi về master |
| Connection pooling     | Tái sử dụng kết nối, giảm overhead mở/đóng kết nối liên tục |
| Query routing          | Định tuyến truy vấn SELECT về replica, truy vấn ghi về master |
| Query rewrite          | Cho phép thay đổi nội dung truy vấn theo rule trước khi gửi đến DB |
| Failover detection     | Tự động phát hiện và chuyển đổi master/slave khi có lỗi |
| Monitoring             | Thống kê QPS, latency, trạng thái query, kết nối backend |
| Security và ACL        | Kiểm soát truy cập, lọc query độc hại hoặc chặn truy vấn cụ thể |

## Cách hoạt động

1. Ứng dụng kết nối đến ProxySQL
2. ProxySQL nhận truy vấn và phân tích nội dung
3. ProxySQL định tuyến truy vấn đến master hoặc replica phù hợp
4. Trả kết quả lại cho ứng dụng

Mọi thao tác đều diễn ra trong suốt, ứng dụng không cần biết các node backend.

## Ví dụ về query routing

| Truy vấn                    | Server đích |
|----------------------------|-------------|
| SELECT * FROM users        | Replica     |
| UPDATE users SET ...       | Master      |

Quy tắc định tuyến được cấu hình bằng các query rule trong ProxySQL.

## So sánh với các công cụ khác

| Tính năng         | ProxySQL | MySQL Router | HAProxy |
|-------------------|----------|--------------|---------|
| Query Routing     | Có       | Không        | Không   |
| Connection Pool   | Có       | Không        | Không   |
| Load Balancing    | Có       | Có           | Có      |
| Query Rewrite     | Có       | Không        | Không   |
| Monitor DB        | Có       | Có           | Không   |
| Failover Handling | Có       | Cơ bản       | Không   |

## Khi nào nên dùng ProxySQL?

| Nhu cầu                              | Có nên dùng |
|-------------------------------------|-------------|
| Cần scale đọc/ghi MySQL             | Nên dùng    |
| Microservices kết nối nhiều DB      | Nên dùng    |
| Tự động failover không chỉnh sửa app| Nên dùng    |
| Theo dõi truy vấn thời gian thực    | Nên dùng    |
| Hệ thống nhỏ, 1 database             | Không cần   |

## Tổng kết

ProxySQL là công cụ rất mạnh để tối ưu hóa, quản lý và bảo vệ hệ thống MySQL/MariaDB. Nó giúp tách biệt logic routing, connection, failover khỏi ứng dụng, từ đó tăng khả năng mở rộng và bảo trì hệ thống hiệu quả hơn.
