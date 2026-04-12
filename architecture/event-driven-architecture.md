# Event-Driven Architecture (EDA)

Kiến trúc trong đó các service giao tiếp với nhau **thông qua event** thay vì gọi trực tiếp. Producer phát event, consumer lắng nghe và phản ứng — hai bên không biết về nhau.

> Thay vì A gọi thẳng B và C, A chỉ nói "tôi vừa làm xong việc này" — B và C tự quyết định có quan tâm không.

---

## 1. So sánh với kiến trúc truyền thống

**Request-Driven (truyền thống):**
```
Order Service ──gọi thẳng──▶ Payment Service
             ──gọi thẳng──▶ Notification Service
             ──gọi thẳng──▶ Inventory Service
```
- Order Service phải biết địa chỉ từng service
- Payment down → Order lỗi theo
- Thêm service mới → phải sửa Order Service

**Event-Driven:**
```
Order Service ──[OrderPlaced]──▶ Message Broker
                                        │
                             ┌──────────┼──────────┐
                             ▼          ▼          ▼
                         Payment    Notification  Inventory
                         Service    Service       Service
```
- Order Service không biết ai consume
- Payment down → event nằm trong broker, xử lý khi lên lại
- Thêm service mới → chỉ cần subscribe, không đụng Order Service

---

## 2. Các khái niệm cốt lõi

| Khái niệm | Mô tả |
|-----------|-------|
| **Event** | Thông báo về điều đã xảy ra. Immutable, đặt tên thì quá khứ: `OrderPlaced`, `PaymentFailed` |
| **Producer** | Service tạo và publish event. Không quan tâm ai consume |
| **Consumer** | Service lắng nghe và xử lý event theo logic của mình |
| **Message Broker** | Trung gian nhận và deliver event: Kafka, RabbitMQ, AWS SQS |

---

## 3. Choreography vs Orchestration

Hãy tưởng tượng một đám cưới cần 3 việc: nấu ăn, trang trí, mời khách.

### Choreography — Ai việc nấy, không ai chỉ huy

Chủ nhà chỉ thông báo vào group chat: *"Cưới ngày 15!"*

```
Chủ nhà ──"Cưới ngày 15!"──▶ Group chat
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
               Đầu bếp        Thợ trang trí   Người mời khách
             (tự biết nấu)  (tự biết trang trí) (tự biết mời)
```

Không ai gọi điện chỉ đạo. Mỗi người tự biết việc mình khi nghe tin.

Trong hệ thống — Order Service publish `OrderPlaced`, các service tự subscribe và tự xử lý. Nếu Payment xong thì publish tiếp `PaymentCompleted`, các service khác lại tự lắng nghe.

- ✅ Thêm service mới chỉ cần subscribe, không đụng code cũ
- ✅ Service chết không kéo nhau chết theo
- ❌ Muốn biết flow đang ở đâu → phải trace qua log nhiều service
- ❌ Rollback khi lỗi giữa chừng rất khó

### Orchestration — Có một người chỉ huy

Chủ nhà gọi điện trực tiếp từng người, chờ xong mới gọi tiếp:

```
Chủ nhà (chỉ huy)
    │
    ├──"Mày nấu đi"──▶ Đầu bếp ──"xong"──▶ Chủ nhà
    ├──"Mày trang trí đi"──▶ Thợ ──"xong"──▶ Chủ nhà
    └──"Mày mời khách đi"──▶ Người mời ──"xong"──▶ Chủ nhà
```

Chủ nhà biết toàn bộ flow, biết ai đang làm gì, quyết định bước tiếp theo. Nếu Inventory báo hết hàng → Orchestrator tự động gọi Payment hoàn tiền.

- ✅ Nhìn vào Orchestrator là biết flow đang ở đâu
- ✅ Rollback dễ — orchestrator điều phối từng bước
- ❌ Orchestrator biết quá nhiều, coupling cao hơn
- ❌ Orchestrator là single point of failure

### Khi nào dùng gì

| | Choreography | Orchestration |
|--|-------------|---------------|
| **Flow** | Đơn giản, ít bước | Phức tạp, nhiều bước phụ thuộc nhau |
| **Rollback** | Khó | Dễ |
| **Visibility** | Khó trace | Dễ trace |
| **Coupling** | Thấp | Cao hơn |
| **Thêm service** | Chỉ cần subscribe | Phải sửa orchestrator |
| **Dùng khi** | Notify, fan-out | Payment flow, booking flow, cần Saga |

---

## 4. Các pattern phổ biến

### Pub/Sub
Một event, nhiều consumer nhận đồng thời. Consumer độc lập với nhau.
```
Producer ──▶ Topic ──▶ Consumer A
                  ──▶ Consumer B
                  ──▶ Consumer C
```
Dùng khi: notification, fan-out, broadcast.

### Event Queue (Point-to-Point)
Một event chỉ được xử lý bởi **một** consumer — load balancing giữa các instance.
```
Producer ──▶ Queue ──▶ Consumer instance 1
                  hoặc Consumer instance 2
                  hoặc Consumer instance 3
```
Dùng khi: task queue, đảm bảo chỉ xử lý một lần.

### Event Streaming
Event được lưu lại vĩnh viễn, consumer đọc từ offset bất kỳ — kể cả event từ hôm qua.
```
Kafka Topic ─────────────────────────────────▶ thời gian
             event1  event2  event3  event4
               │                      │
          Consumer A             Consumer B
         (offset 1)             (offset 4)
```
Dùng khi: audit log, replay, consumer mới cần đọc lại lịch sử.

---

## 5. Ưu và nhược điểm

**Ưu điểm:**
- **Loose coupling** — producer không biết consumer, dễ thêm/bỏ service
- **Resilience** — consumer down không ảnh hưởng producer
- **Scalability** — scale từng consumer group độc lập
- **Audit trail** — event log là lịch sử đầy đủ

**Nhược điểm:**
- **Eventual consistency** — data không nhất quán ngay lập tức
- **Khó debug** — flow không tuyến tính, phải trace qua nhiều service
- **Complexity** — cần broker, monitoring, dead letter queue
- **Ordering** — đảm bảo thứ tự event khó trong hệ thống phân tán

---

## 6. Lưu ý khi triển khai

### Idempotency — Xử lý trùng lặp
Broker có thể deliver event nhiều lần (at-least-once). Consumer phải xử lý được trùng lặp bằng cách kiểm tra `eventId` đã xử lý chưa trước khi làm.

### Dead Letter Queue (DLQ)
Event xử lý thất bại nhiều lần → đẩy vào DLQ để xem xét thủ công, không block queue chính.
```
Main Queue ──fail 3 lần──▶ Dead Letter Queue ──▶ alert / manual review
```

### Ordering
Kafka đảm bảo thứ tự trong cùng một partition. Dùng cùng partition key (ví dụ `orderId`) để các event của cùng một order được xử lý đúng thứ tự.

### Outbox Pattern — Đảm bảo event không bị mất

**Vấn đề:** Service commit DB thành công nhưng crash trước khi publish event → event mất, các service khác không biết gì.

```
commit DB ✓  →  crash  →  publish event ✗
```

**Giải pháp:** Lưu event vào bảng `outbox` trong cùng transaction với data. Background job đọc outbox rồi publish lên Kafka — nếu crash thì job chạy lại, event không bao giờ mất.

```
┌─────────────────────────────────────┐
│          Same Transaction           │
│  UPDATE reservations ...            │
│  INSERT INTO outbox (event_data)    │
└──────────────────┬──────────────────┘
                   │
         Outbox Processor (background job)
                   │
                   ▼
           PUBLISH to Kafka
                   │
                   ▼
           DELETE FROM outbox
```

Kết hợp Outbox (at-least-once delivery) + Idempotency ở consumer = đảm bảo event được xử lý đúng một lần.

---

## 7. Ví dụ — Hệ thống đặt vé máy bay

```
Reservation Service
    ├── ReservationConfirmed ──▶ Kafka: reservation-events
    │                                   ├──▶ Notification (email xác nhận)
    │                                   ├──▶ Loyalty BC (cộng điểm)
    │                                   └──▶ Analytics BC
    │
    └── ReservationCancelled ──▶ Kafka: reservation-events
                                        ├──▶ Notification (email hủy)
                                        ├──▶ Inventory (giải phóng ghế)
                                        └──▶ Analytics BC

Flight Service
    └── FlightDelayed ──▶ Kafka: flight-events
                                  ├──▶ Notification (báo hành khách)
                                  ├──▶ Reservation (cập nhật thông tin)
                                  └──▶ Check-in Service
```
