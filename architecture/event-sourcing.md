# Event Sourcing

Pattern lưu trữ state bằng cách **ghi lại toàn bộ chuỗi event đã xảy ra**, thay vì chỉ lưu trạng thái hiện tại. State hiện tại được tính bằng cách **replay** lại các event từ đầu.

> Thay vì lưu "Reservation đang CONFIRMED", lưu: "HELD lúc 10:00" → "CONFIRMED lúc 10:05". State = kết quả sau khi apply tất cả event theo thứ tự.

---

## 1. Truyền thống vs Event Sourcing

**Truyền thống — lưu current state:**

| id | status | updated_at |
|----|--------|------------|
| res-123 | CONFIRMED | 2025-06-15 10:05 |

Không biết: trước đó là gì, thay đổi khi nào, lý do gì.

**Event Sourcing — lưu chuỗi event:**

| sequence | event_type | occurred_on | data |
|----------|-----------|-------------|------|
| 1 | ReservationHeld | 10:00 | seatId: 12A, heldUntil: 10:15 |
| 2 | ReservationConfirmed | 10:05 | paymentId: pay-456 |

State hiện tại = apply event 1 rồi event 2.

---

## 2. Cách hoạt động

**Ghi (Write):**
1. Load toàn bộ event của aggregate từ Event Store
2. Replay để rebuild state hiện tại
3. Thực thi command → sinh event mới
4. Lưu event mới vào Event Store — **không lưu state**

**Đọc (Read):**
Load event → replay → ra state hiện tại.

---

## 3. Event Store

Database chuyên dụng để lưu event. Ba yêu cầu bắt buộc:

- **Append-only** — chỉ thêm, không sửa, không xóa
- **Ordered** — mỗi event có sequence number, đảm bảo thứ tự
- **Optimistic concurrency** — kiểm tra version trước khi append, tránh conflict khi nhiều request cùng ghi

Có thể dùng: **EventStoreDB** (chuyên dụng), PostgreSQL (append-only table), Kafka (log compaction).

---

## 4. Projection — Tạo Read Model

Event Store chỉ tối ưu cho việc **ghi** — mỗi lần đọc phải replay toàn bộ event, rất chậm khi cần query phức tạp. **Projection** giải quyết bằng cách lắng nghe event stream và liên tục cập nhật một **read model riêng** được tối ưu cho query.

```
Event Store (write-optimized)
    │
    │ event mới được append
    ▼
Projection Handler              ← lắng nghe event, cập nhật view
    │
    ▼
Read DB (read-optimized)        ← query trực tiếp vào đây, không replay
    ├── reservations_view
    ├── passenger_itinerary_view
    └── flight_manifest_view
```

### Một event stream, nhiều projection độc lập

Cùng một chuỗi event nhưng mỗi consumer cần nhìn data theo góc độ khác nhau:

```
ReservationConfirmed event
    │
    ├──▶ Projection A: ReservationListView (cho admin)
    │         id | passenger | flight | seat | status | price
    │
    ├──▶ Projection B: PassengerItineraryView (cho hành khách)
    │         flight | departure | seat | gate | baggage
    │
    └──▶ Projection C: FlightManifestView (cho phi công)
              seat | passenger_name | meal_preference | special_needs
```

Mỗi projection có schema riêng, tối ưu cho đúng use case của nó — không phải dùng chung một bảng cho tất cả.

### Projection là eventually consistent

Projection không cập nhật đồng bộ — có độ trễ nhỏ giữa lúc event được ghi vào Event Store và lúc read model phản ánh thay đổi. Thường tính bằng millisecond, chấp nhận được với hầu hết use case.

### Read DB là gì — cùng DB hay khác DB?

Tùy quy mô hệ thống, có 3 cách triển khai:

**Cách 1 — Cùng DB, khác bảng** (đơn giản, phổ biến nhất khi bắt đầu)
```
PostgreSQL
├── reservation_events     ← Event Store (write side)
├── reservations_view      ← Read model (read side)
└── flight_manifest_view   ← Read model khác
```
Dùng khi hệ thống chưa lớn, chưa cần tối ưu nhiều.

**Cách 2 — Khác DB instance, cùng loại**
```
PostgreSQL (write)          PostgreSQL replica (read)
reservation_events    ──▶   reservations_view
                            flight_manifest_view
```
Tách load read/write ra hai instance riêng.

**Cách 3 — Khác DB, khác công nghệ** (phổ biến trong hệ thống lớn)
```
Event Store (PostgreSQL / EventStoreDB)
    │
    ├──▶ Elasticsearch    ← full-text search, filter phức tạp
    ├──▶ Redis            ← cache read model, truy cập cực nhanh
    └──▶ MongoDB          ← document linh hoạt, nested data
```
Mỗi read model dùng công nghệ phù hợp với query pattern của nó.

### Read model được thiết kế thế nào

Read model **denormalized hoàn toàn** — không normalize, không join, tối ưu cho đúng một query:

```
Write side (normalized)            Read side (denormalized)
───────────────────────            ────────────────────────
reservations                       reservations_view
  id, flight_id, status      ──▶   id | passenger_name | flight_number
                                   | departure | seat | status | price
passengers                         (tất cả trong 1 bảng, query 1 lần là xong)
  id, name, email

flights
  id, number, departure
```

- Không cần JOIN khi đọc
- Schema thiết kế ngược từ query ra — cần hiển thị gì thì lưu y vậy
- Chấp nhận dư thừa data vì đọc nhanh quan trọng hơn

### Cơ chế sync giữa Event Store và Read Model

**Cách 1 — Đồng bộ trong cùng transaction**

Ghi event và cập nhật read model trong cùng một transaction DB. Đơn giản nhất, không có độ trễ.

```
BEGIN TRANSACTION
  INSERT INTO reservation_events (...)    ← ghi event
  UPDATE reservations_view SET status=... ← cập nhật read model ngay
COMMIT
```

- ✅ Read model luôn nhất quán với event store
- ❌ Chỉ dùng được khi cùng DB
- ❌ Transaction lớn hơn, chậm hơn

---

**Cách 2 — Qua Message Broker (phổ biến nhất)**

Sau khi ghi event vào Event Store, publish event lên Kafka. Projection Handler subscribe và cập nhật read model bất đồng bộ.

```
Application
    │ append event
    ▼
Event Store ──publish──▶ Kafka ──▶ Projection Handler ──▶ Read DB
```

- ✅ Read DB có thể là công nghệ khác (Elasticsearch, Redis...)
- ✅ Nhiều Projection Handler độc lập cùng subscribe một topic
- ❌ Có độ trễ — read model chậm hơn event store vài ms đến vài giây
- ❌ Cần xử lý idempotency (event có thể được deliver nhiều lần)

---

**Cách 3 — CDC (Change Data Capture)**

Thay vì application tự publish event, một tool như **Debezium** đọc trực tiếp **transaction log của DB** (WAL của PostgreSQL, binlog của MySQL) và phát hiện mọi thay đổi — rồi tự động sync sang nơi khác.

```
Application
    │ INSERT INTO reservation_events
    ▼
PostgreSQL
    │
    │ Debezium đọc WAL (transaction log)
    ▼
Kafka ──▶ Projection Handler ──▶ Read DB
```

Không cần application làm gì thêm — Debezium tự theo dõi DB và sync.

- ✅ Application không cần biết về sync — hoàn toàn tách biệt
- ✅ Không bao giờ miss event dù application crash (đọc từ log DB)
- ✅ Có thể sync sang nhiều nơi khác nhau từ một nguồn
- ❌ Cần cài thêm Debezium, phức tạp hơn để vận hành

---

**Cách 4 — Polling (đơn giản, ít dùng)**

Background job định kỳ chạy, đọc event mới từ Event Store và cập nhật read model.

```
Event Store ◀── SELECT events WHERE id > last_processed ── Background Job
                                                                  │
                                                                  ▼
                                                              Read DB
```

- ✅ Đơn giản, không cần Kafka
- ❌ Có độ trễ theo chu kỳ polling (5s, 10s...)
- ❌ Tốn query liên tục dù không có event mới

---

**Chọn cơ chế nào?**

| Tình huống | Cơ chế |
|-----------|--------|
| Read model cùng DB, cần nhất quán tuyệt đối | Transaction đồng bộ |
| Read model khác DB/công nghệ, chấp nhận delay nhỏ | Message Broker (Kafka) |
| Không muốn application biết về sync, cần reliability cao | CDC (Debezium) |
| Hệ thống nhỏ, đơn giản, không cần real-time | Polling |

### Rebuild projection dễ dàng

Đây là điểm mạnh lớn nhất. Nếu projection bị lỗi, cần thêm field, hoặc muốn tạo view mới cho use case mới:

```
Xóa read model cũ
       │
       ▼
Replay toàn bộ event từ Event Store
       │
       ▼
Read model mới được build lại hoàn toàn
```

Không mất data vì data thật nằm trong Event Store, projection chỉ là **derived data** — tính toán ra từ event.

---

## 5. Snapshot — Tránh replay quá nhiều event

Aggregate có hàng nghìn event → replay toàn bộ mỗi lần load rất chậm. **Snapshot** lưu lại state tại một điểm, lần sau chỉ replay từ đó trở đi.

```
event 1 ──▶ ... ──▶ event 100 ──[Snapshot: lưu state]──▶ event 101 ──▶ event 200
                                                                           ▲
                                                               Lần sau load từ đây
```

Thường snapshot mỗi 50-100 event tùy domain.

---

## 6. Event Sourcing + CQRS

Event Sourcing thường đi kèm **CQRS** (Command Query Responsibility Segregation) — tách biệt hoàn toàn write và read:

```
WRITE SIDE                              READ SIDE
──────────                              ─────────
Command ──▶ Aggregate                   Query ──▶ Read Model
               │                                      ▲
               │ Events                               │
               ▼                                      │
           Event Store ──▶ Projection Handler ─────────┘
```

- **Write side** — Event Store, đảm bảo consistency tuyệt đối
- **Read side** — DB tối ưu cho query, eventually consistent

---

## 7. Ưu và nhược điểm

**Ưu điểm:**
- **Audit log hoàn chỉnh** — biết toàn bộ lịch sử, ai làm gì, khi nào, tại sao
- **Temporal query** — xem state tại bất kỳ thời điểm nào trong quá khứ
- **Replay** — tạo read model mới bất cứ lúc nào từ event history
- **Debug** — reproduce bug bằng cách replay event đến thời điểm lỗi
- **Event sẵn có** — tự nhiên sinh ra event để publish cho các service khác

**Nhược điểm:**
- **Phức tạp** — khó hơn CRUD rất nhiều
- **Event schema evolution** — thay đổi cấu trúc event cũ rất khó, cần versioning cẩn thận
- **Eventual consistency** — read model không nhất quán ngay lập tức
- **Replay chậm** — nếu không có snapshot, aggregate nhiều event sẽ chậm

---

## 8. Khi nào dùng

| Nên dùng | Không nên dùng |
|----------|---------------|
| Cần audit trail đầy đủ (tài chính, y tế, pháp lý) | CRUD đơn giản — overkill |
| Cần xem state tại thời điểm trong quá khứ | Team chưa có kinh nghiệm |
| Domain phức tạp, nhiều state transition | Deadline gấp |
| Cần tích hợp nhiều service qua event | Không cần audit trail |

---

## 9. Event Sourcing vs Event-Driven Architecture

| | Event Sourcing | Event-Driven Architecture |
|--|---------------|--------------------------|
| **Mục đích** | Lưu trữ state | Giao tiếp giữa service |
| **Event lưu ở đâu** | Event Store (append-only DB) | Message Broker (Kafka, RabbitMQ) |
| **Event tồn tại** | Vĩnh viễn | Theo TTL của broker |
| **Ai consume** | Chính aggregate để rebuild state | Các service khác |

Hai pattern độc lập nhau — có thể dùng cái này mà không cần cái kia. Nhưng bổ trợ tốt cho nhau: Event Sourcing sinh ra event tự nhiên → dùng làm integration event cho EDA.
