# 06 — RabbitMQ vs Kafka

## Kiến trúc cốt lõi khác nhau

**RabbitMQ — Push model (Message Broker)**
```
Producer ──▶ Exchange ──routing──▶ Queue ──push──▶ Consumer
```
- Broker **đẩy** message đến consumer
- Message bị **xóa** sau khi consumer xử lý xong (ack)
- Mạnh về **routing linh hoạt**: direct, fanout, topic, headers exchange

**Kafka — Pull model (Distributed Log)**
```
Producer ──▶ Topic (Partition 0) ──────────────────────▶ thời gian
                                  msg1 msg2 msg3 msg4
                                   │              │
                              Consumer A      Consumer B
                              (offset 1)      (offset 3)
```
- Consumer **tự kéo** message theo offset
- Message được **lưu lâu dài** (theo retention policy)
- Consumer có thể replay lại từ bất kỳ offset nào

---

## So sánh

| Tiêu chí | RabbitMQ | Kafka |
|---------|---------|-------|
| Mô hình | Push (broker → consumer) | Pull (consumer tự kéo) |
| Lưu message | Xóa sau khi ack | Lưu theo retention (ngày/tuần) |
| Replay | Không | Có — seek về offset cũ |
| Throughput | Hàng chục nghìn msg/s | Hàng triệu msg/s |
| Latency | Sub-millisecond | Milliseconds (có thể tune) |
| Routing | Rất linh hoạt (exchange/binding) | Đơn giản (hash by key → partition) |
| Ordering | Trong queue | Trong partition |
| Stream processing | Cần tool bên ngoài | Kafka Streams built-in |
| Multi-consumer | Mỗi queue — 1 consumer nhận | Nhiều consumer group, mỗi group nhận full |
| Complexity | Đơn giản hơn | Phức tạp hơn (partition, offset, ISR) |

---

## Khi nào dùng gì?

**Chọn RabbitMQ khi:**
- **Task queue**: phân phối job cho worker pool, mỗi job chỉ xử lý một lần
- **Complex routing**: route message theo nhiều tiêu chí (content, header)
- **Low latency**: cần sub-millisecond response
- **Request-reply**: hỗ trợ RPC pattern native
- Team chưa muốn quản lý Kafka cluster phức tạp

**Chọn Kafka khi:**
- **Event streaming**: lưu lịch sử event, cần replay
- **High throughput**: IoT, clickstream, log aggregation (triệu msg/s)
- **Nhiều consumer group** cùng consume 1 topic độc lập
- **Audit trail / Compliance**: event không bị xóa
- **Stream processing**: Kafka Streams, Flink integration

---

## Ví dụ minh họa

```
RabbitMQ: "Có 10 worker, phân phối email job đều cho các worker.
           Mỗi email chỉ gửi đúng một lần. Worker fail → worker
           khác lấy lại job."
→ Task queue, at-most-once processing

Kafka: "Mỗi khi user đặt hàng, lưu event OrderPlaced.
        Payment Service, Analytics Service, Notification Service
        đều cần consume toàn bộ event. Analytics cần replay 30
        ngày lịch sử."
→ Event streaming, multiple consumer groups, replay
```

---

## Interview Q&A

**Q: Kafka và RabbitMQ khác nhau điểm gì cốt lõi nhất?**

> Kafka là **distributed log** — message được lưu vĩnh viễn, consumer tự quản lý offset, có thể replay. RabbitMQ là **message broker** — message bị xóa sau khi ack, broker quản lý delivery. Kafka tối ưu cho throughput cao và event sourcing, RabbitMQ tối ưu cho routing phức tạp và task queue.

**Q: Có thể dùng cả hai không?**

> Có. Nhiều hệ thống dùng RabbitMQ cho task distribution (low-latency, complex routing) và Kafka cho event streaming (audit log, analytics). Ví dụ: RabbitMQ để gửi notification job, Kafka để lưu event log toàn hệ thống.
