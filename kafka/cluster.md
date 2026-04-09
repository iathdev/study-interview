# Các khái niệm cơ bản 2

**Apache Kafka**:
  - **Loại**: Nền tảng streaming phân tán (distributed streaming platform).
  - **Mô hình**: Hoạt động theo mô hình **pull**, nơi consumer chủ động yêu cầu tin nhắn từ broker.
  - **Cấu trúc**: Sử dụng **topics** chia thành các **partitions**, lưu trữ tin nhắn dưới dạng log bất biến (append-only log). Consumer đọc dữ liệu từ offset cụ thể trong topic.
  - **Giao thức**: Sử dụng giao thức TCP/IP tùy chỉnh, tối ưu cho hiệu suất cao nhưng ít tương thích hơn so với RabbitMQ.
  - **Lưu trữ tin nhắn**: Tin nhắn được lưu trữ lâu dài (cho đến khi đạt giới hạn thời gian hoặc kích thước), hỗ trợ replay (đọc lại) tin nhắn.

## 2.1 Broker và Cluster
Broker: đóng vai trò là máy chủ trung gian có trách nhiệm
nhận, lưu trữ, điều phối message giữa producers và consumers

Hiểu đơn giản:
- Producer (người gửi): gửi thư (message) tới bưu điện (Kafka Broker).
- Kafka Broker: nhận thư, lưu vào hòm thư (topic + partition).
- Consumer (người nhận): tới bưu điện lấy thư theo nhu cầu (đọc message từ topic).

- Mỗi broker lưu dữ liệu của nhiều partition
- 1 topic có nhiều partition và có thể mỗi partition này được lưu ở nhiều broker khác nhau

## 2.2 Topic replication
Topic Replication trong Apache Kafka là một cơ chế đảm bảo tính sẵn sàng (high availability) và khả năng chống mất dữ liệu bằng cách nhân bản (replicate) các Partition của một Topic sang nhiều Kafka Brokers.

Mục đích:
- Đảm bảo dữ liệu không bị mất nếu một broker gặp sự cố.
- Hỗ trợ khả năng failover: tự động chuyển sang replica khác khi leader broker mất kết nối.
- Cải thiện tính ổn định và độ tin cậy của hệ thống Kafka

Mỗi Partition có:
- 1 Leader replica (được ghi & đọc chính)
- N = Replication factor ≤ số lượng brokers
- N-1 Follower replicas (chỉ đồng bộ dữ liệu từ leader)
- Kafka sẽ tự động phân phối leader/follower để cân bằng tải giữa các broker

Ví dụ: Giả sử bạn có một Topic orders với:
- 3 Partitions
- Replication factor = 3
- Kafka cluster có 3 brokers

| Partition | Leader (ghi/đọc) | Follower Replicas  |
| --------- | ---------------- | ------------------ |
| 0         | Broker 1         | Broker 2, Broker 3 |
| 1         | Broker 2         | Broker 1, Broker 3 |
| 2         | Broker 3         | Broker 1, Broker 2 |

Các thuật ngữ:
| Thuật ngữ                   | Giải thích                                                                                                                        |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| **ISR (In-Sync Replicas)**  | Danh sách các replicas đang đồng bộ tốt với leader                                                                                |
| **Unclean Leader Election** | Nếu tất cả replicas mất kết nối, cho phép chọn leader từ các replica chưa đồng bộ hoàn toàn. **Cảnh báo: Có thể gây mất dữ liệu** |
| **min.insync.replicas**     | Số replica tối thiểu cần đồng bộ để producer tiếp tục ghi dữ liệu                                                                 |

### Ưu - Nhược điểm của Replication
Ưu điểm:
- Tăng độ tin cậy
- Hỗ trợ tự động failover
- Giảm downtime

Nhược điểm:
- Tốn tài nguyên (disk, network)
- Tăng độ trễ ghi dữ liệu nếu follower chậm
- Quản lý phức tạp hơn

Leader:
- Mỗi 1 partition trong topic thì chỉ có 1 leader

ISR broker (In-Sync Replicas)
- Là các partiton đồng bộ dữ liệu tốt từ leader 

## 2.3 Producer ACK

- Trong hệ thống xử lý message bất đồng bộ, khi một consumer (người nhận) xử lý xong một message, nó phải gửi ACK để báo với hệ thống (hoặc broker) rằng:
```
“Tôi đã nhận và xử lý message này thành công, không cần gửi lại.”
```

Nếu consumer không ACK, thì hệ thống sẽ:
- Giả định xử lý thất bại
- Và có thể gửi lại message đó cho consumer khác (retry / requeue)

Với Kafka:
- ACK xảy ra từ phía Producer → Kafka Broker

Nó giúp:
- Đảm bảo độ tin cậy trong truyền tải
- Tránh mất mát hoặc trùng lặp dữ liệu
- Điều phối lại message nếu consumer gặp lỗi


| Giá trị         | Mô tả                                                                                                                                                 |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `0`             | **Producer không cần xác nhận** từ broker. Gửi là xong, không quan tâm thành công hay thất bại. Hiệu suất cao nhưng dễ mất dữ liệu.                   |
| `1`             | Broker chỉ cần **leader partition** ghi thành công là đủ. An toàn hơn `0`, nhưng nếu leader chết trước khi follower kịp đồng bộ → có thể mất dữ liệu. |
| `all` hoặc `-1` | Broker chờ **tất cả các replicas trong ISR (in-sync replicas)** xác nhận. **An toàn nhất**, nhưng tốc độ chậm hơn.                                    |

So sánh các chế độ acks

| Thuộc tính           | `acks=0`        | `acks=1`      | `acks=all` / `acks=-1` |
| -------------------- | --------------- | ------------- | ---------------------- |
| Tốc độ ghi           | Nhanh nhất   	 | Nhanh         |  Chậm hơn              |
| An toàn dữ liệu      | Không an toàn   | Trung bình    | An toàn nhất           |
| Khả năng mất dữ liệu | Rất cao         | Có thể        | Gần như không          |
| Độ trễ (latency)     | Thấp            | Trung bình    | Cao hơn                |



