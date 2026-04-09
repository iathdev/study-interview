# Replication

Cơ chế:
+ Bất đồng bộ
+ Tất cả các thay đổi (command) của master sẽ được lưu vào file binary log 
+ Slave đọc file binary log và thực hiện những thao tác trong file
+ I/O thread: đọc các sự kiện từ binary log trên master

| Vai trò    | Mô tả                                                                   |
| ---------- | ----------------------------------------------------------------------- |
| **Master** | Server chính, nơi xử lý **ghi dữ liệu (INSERT, UPDATE, DELETE)**        |
| **Slave**  | Một hoặc nhiều server phụ, dùng để **đọc dữ liệu (SELECT)** hoặc backup |

Mọi thao tác ghi trên master sẽ được log lại và gửi sang slave để slave thực hiện lại giống hệt.

## Cơ chế sao lưu & đồng bộ hoạt động như thế nào?

- Bước 1: Master ghi dữ liệu
Khi có INSERT, UPDATE, DELETE, master sẽ ghi lại thao tác này vào binary log (binlog)

- Bước 2: Slave đọc binlog từ master
Mỗi slave sẽ chạy một replication thread để kết nối đến master

Slave hỏi: “Tôi đã đọc đến binlog position X, có gì mới không?”
Master gửi phần log mới → Slave copy về

- Bước 3: Slave replay lại log
Slave diễn giải binlog (gọi là SQL thread) và thực hiện thao tác y hệt như master

Kết quả:
Dữ liệu trên slave giống hệt master (gần như theo thời gian thực)

## Lợi ích

| Lợi ích            | Ghi chú                                          |
| ------------------ | ------------------------------------------------ |
| Tăng hiệu suất đọc | Cho phép các app đọc từ slave                    |
| Chống mất dữ liệu  | Nếu master hỏng, có thể phục hồi từ slave        |
| Hỗ trợ backup      | Có thể backup từ slave mà không ảnh hưởng master |
| Phân tải           | Tách read/write giúp server nhẹ hơn              |

## Lưu ý và rủi ro

| Vấn đề                           | Chi tiết                                                        |
| -------------------------------- | --------------------------------------------------------------- |
| **Replication lag**              | Slave có thể bị chậm hơn vài giây nếu nhiều ghi                 |
| **Không đồng bộ 2 chiều**        | Slave **chỉ đọc** – nếu bạn ghi vào slave → lỗi dữ liệu         |
| **Failover phức tạp**            | Nếu master chết, cần cấu hình lại để slave thành master         |
| **Không thay thế backup đầy đủ** | Replication ≠ backup (vì xóa nhầm trên master → slave cũng mất) |

## Khi master bị down thì làm cách nào để tự động chuyển slave thành master
+ Đối với AWS có RDS Multi-AZ (Availability Zone)
+ Nên sử dụng kết hợp Read replicas & Multi-AZ để đảm bảo vừa scale read & đảm bảo failover

Cơ chế:
+ Primary (Master): chạy trong một AZ cụ thể
+ Standby (Slave):
- AWS tự động tạo standby trên AZ khác, sao lưu dữ liệu của Primary
- Sao chép đồng bộ từ master > standby
+ Failover: Khi master lỗi sẽ chuyển standby thành primary
+ Tự động cập nhất DNS để trỏ tới stanby mới

Nhược điểm:
+ Không cải thiện đọc (phải dùng kết hợp read replicas)
+ Chi phí cao

# Binlog là gì?
Binlog (Binary Log) là file nhị phân trên máy chủ MySQL, ghi lại toàn bộ các thao tác ghi (write) ảnh hưởng đến dữ liệu.

Dùng cho:
- Replication (đồng bộ từ master → slave)
- Point-in-time recovery (khôi phục dữ liệu tới một thời điểm cụ thể)
- Audit (kiểm tra thay đổi dữ liệu)

## Binlog lưu cái gì?

| Loại ghi                          | Mô tả                               |
| --------------------------------- | ----------------------------------- |
| `INSERT`, `UPDATE`, `DELETE`      | Các thao tác làm thay đổi dữ liệu   |
| `CREATE TABLE`, `DROP`, `ALTER`   | Các thay đổi về schema (DDL)        |
| `BEGIN`, `COMMIT` (transaction)   | Ghi nhận phạm vi giao dịch          |
| `SET AUTOCOMMIT=1`, `USE db_name` | Thông tin môi trường                |
| Thông tin replication             | Position, server ID, GTID (nếu bật) |




