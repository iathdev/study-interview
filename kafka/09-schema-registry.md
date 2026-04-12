# Schema Registry

## 1. Vấn đề cần giải quyết

Kafka message chỉ là **bytes** — broker không hiểu nội dung. Producer và Consumer phải tự thỏa thuận format. Khi không có Schema Registry:

```
Producer (team A) gửi:  { "price": 100 }         ← int
Consumer (team B) đọc:  deserialize as float      ← crash

Producer đổi tên field: "amount" → "totalAmount"
→ Consumer cũ đọc được nhưng field trả về null
→ Không ai biết ai đã thay đổi gì, khi nào
```

**Hậu quả thực tế:**
- Schema được thỏa thuận qua Slack/email/wiki → dễ lệch
- Breaking change không bị phát hiện đến lúc production
- Không có lịch sử phiên bản schema
- Message payload phải đính kèm schema đầy đủ → lãng phí bandwidth

---

## 2. Schema Registry là gì?

Là một **service lưu trữ tập trung** cho schema (Avro, Protobuf, JSON Schema), cung cấp versioning và kiểm soát compatibility.

```
Producer                  Schema Registry              Consumer
   │                            │                          │
   ├── đăng ký schema ─────────▶│                          │
   │◀─ schema_id = 42 ──────────┤                          │
   │                            │                          │
   ├── gửi message: [0][42][data bytes] ─────────────────▶│
   │   (magic byte)(schema_id)(payload)                    │
   │                            │                          │
   │                            │◀─ tra schema_id=42 ──────┤
   │                            ├── trả schema ───────────▶│
   │                            │                          │
   │                            │              deserialize │
```

**Wire format**: Mỗi message chỉ cần **5 byte prefix** thay vì đính kèm schema đầy đủ:
- 1 byte: magic byte (`0x00`)
- 4 byte: schema ID (int)
- Phần còn lại: payload được serialize theo schema

---

## 3. Compatibility Modes

Quan trọng nhất — kiểm soát việc thay đổi schema có phá vỡ producer/consumer cũ không.

| Mode | Đảm bảo | Upgrade order |
|------|---------|---------------|
| `BACKWARD` (default) | Consumer **mới** đọc được data **cũ** | Upgrade consumer trước |
| `BACKWARD_TRANSITIVE` | Consumer mới đọc được **toàn bộ** history | Upgrade consumer trước |
| `FORWARD` | Consumer **cũ** đọc được data **mới** | Upgrade producer trước |
| `FORWARD_TRANSITIVE` | Tất cả consumer cũ đọc được data mới | Upgrade producer trước |
| `FULL` | Cả BACKWARD + FORWARD (adjacent) | Thứ tự tùy ý |
| `FULL_TRANSITIVE` | Cả BACKWARD + FORWARD (tất cả version) | Thứ tự tùy ý |
| `NONE` | Không kiểm tra | Tự quản lý |

**Quy tắc thay đổi schema (BACKWARD):**

| Thay đổi | Được phép? |
|---------|-----------|
| Thêm field có default value | Có |
| Xóa field có default value | Có |
| Thêm required field (không có default) | Không |
| Xóa required field | Không |
| Đổi tên field | Không (bị coi là xóa + thêm) |
| Đổi kiểu dữ liệu | Không |

---

## 4. Ví dụ thực tế (Avro)

```json
// Schema v1
{
  "type": "record",
  "name": "OrderEvent",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "amount",  "type": "double" }
  ]
}

// Schema v2 — BACKWARD compatible (thêm field có default)
{
  "type": "record",
  "name": "OrderEvent",
  "fields": [
    { "name": "orderId",  "type": "string" },
    { "name": "amount",   "type": "double" },
    { "name": "currency", "type": "string", "default": "VND" }  ← OK
  ]
}

// Schema v3 — KHÔNG compatible (thêm required field)
{
  "type": "record",
  "name": "OrderEvent",
  "fields": [
    { "name": "orderId",    "type": "string" },
    { "name": "amount",     "type": "double" },
    { "name": "customerId", "type": "string" }  ← KHÔNG có default → Registry từ chối
  ]
}
```

---

## 5. Ưu điểm

- **Schema tập trung**: Một nơi duy nhất làm source of truth, không cần thỏa thuận ngoài băng tần
- **Tiết kiệm bandwidth**: Message chỉ cần 5 byte overhead thay vì đính kèm schema đầy đủ mỗi message
- **Phát hiện breaking change sớm**: Registry từ chối đăng ký schema vi phạm compatibility — lỗi xảy ra ở build/deploy time, không phải production
- **Versioning**: Lưu lịch sử tất cả phiên bản schema, biết ai thay đổi gì
- **Hỗ trợ nhiều format**: Avro, Protobuf, JSON Schema
- **Tự động code generation**: Từ schema → generate class Java/Python/Go

---

## 6. Nhược điểm

- **Thêm dependency**: Một service nữa cần deploy, monitor, maintain. Registry down → producer/consumer có thể bị ảnh hưởng
- **Latency**: Lần đầu gặp schema ID → phải query Registry (sau đó cache local)
- **Phức tạp hơn**: Team cần học Avro/Protobuf, hiểu compatibility mode
- **Schema evolution khó**: Đổi tên field bị coi là breaking change → phải dùng alias hoặc tạo field mới song song
- **Single point of failure**: Nếu không deploy HA, Registry down → producer không đăng ký được schema mới

---

## 7. Bẫy thường gặp

**Bẫy 1: BACKWARD không phải BACKWARD_TRANSITIVE**

```
Schema v1 → v2: compatible (BACKWARD OK)
Schema v1 → v3: KHÔNG compatible (nhưng BACKWARD chỉ check với v2)
→ Consumer vẫn chạy v1 sẽ không đọc được data từ producer v3
→ Phải dùng BACKWARD_TRANSITIVE nếu cần hỗ trợ consumer trên version cũ
```

**Bẫy 2: Kafka Streams chỉ hỗ trợ BACKWARD**

Streams app phải được upgrade **trước** producer upstream vì phải đọc được cả schema input topic mới lẫn schema state store cũ.

**Bẫy 3: Registry down và không có cache**

Nếu producer không cache schema ID local và Registry down → mọi write đều fail. Cần cấu hình cache và fallback.

---

## 8. Khi nào cần Schema Registry?

| Tình huống | Cần? |
|-----------|------|
| Nhiều team cùng produce/consume 1 topic | Có |
| Topic tồn tại lâu dài, schema có thể thay đổi | Có |
| Cần audit lịch sử schema | Có |
| Hệ thống nhỏ, 1 team, schema ổn định | Không nhất thiết |
| Prototype, dev nhanh | Không |

---

## 8b. Flow Producer → Consumer khi thay đổi schema

### Flow 1: Producer đăng ký schema lần đầu

```
Producer                    Schema Registry              Kafka Broker
   │                               │                          │
   ├── POST /subjects/orders-value │                          │
   │   body: { schema: "..." }     │                          │
   │──────────────────────────────▶│                          │
   │◀── 200 OK { id: 42 } ─────────┤                          │
   │                               │                          │
   ├── serialize message:          │                          │
   │   [0x00][00 00 00 2A][bytes]  │                          │
   │   magic  schema_id=42  data   │                          │
   │──────────────────────────────────────────────────────────▶│
   │                               │                     lưu vào partition
```

### Flow 2: Consumer đọc lần đầu

```
Kafka Broker             Consumer                  Schema Registry
     │                      │                            │
     ├── message bytes ─────▶│                            │
     │   [0x00][42][data]    │                            │
     │                       ├── đọc magic byte 0x00      │
     │                       ├── đọc schema_id = 42       │
     │                       │                            │
     │                       ├── GET /schemas/ids/42 ─────▶│
     │                       │◀── { schema: "..." } ───────┤
     │                       │   (cache local: id=42→schema)│
     │                       │                            │
     │                       ├── deserialize bytes ──▶ object
```

Lần tiếp theo cùng schema_id → dùng cache, không cần gọi Registry.

### Flow 3: Producer thay đổi schema — COMPATIBLE (backward)

```
                   Schema v1: {orderId, amount}
                   Schema v2: {orderId, amount, currency="VND"}  ← thêm field có default

Producer v2                 Schema Registry              Consumer v1
   │                               │                          │
   ├── POST schema v2              │                          │
   │──────────────────────────────▶│                          │
   │   Registry kiểm tra           │                          │
   │   v2 backward-compatible v1?  │                          │
   │   → YES (field có default)    │                          │
   │◀── 200 OK { id: 43 } ─────────┤                          │
   │                               │                          │
   ├── gửi message schema_id=43 ──────────────────────────────▶│
   │                               │            đọc schema_id=43
   │                               │◀── GET /schemas/ids/43 ──┤
   │                               ├── trả schema v2 ─────────▶│
   │                               │                          │
   │                               │          deserialize v2 data
   │                               │          currency field → dùng default "VND"
   │                               │          → OK, không crash
```

### Flow 4: Producer thay đổi schema — BREAKING (không compatible)

```
                   Schema v1: {orderId, amount}
                   Schema v3: {orderId, amount, customerId}  ← required field, không có default

Producer                    Schema Registry
   │                               │
   ├── POST schema v3              │
   │──────────────────────────────▶│
   │   Registry kiểm tra           │
   │   v3 backward-compatible v2?  │
   │   → NO (required field mới)   │
   │◀── 409 Conflict ──────────────┤
   │   {"error": "Schema being     │
   │    registered is incompatible"}│
   │                               │
   ✗ Message KHÔNG được gửi        │
   ✗ Lỗi xảy ra ở deploy time,     │
     không phải production         │
```

### Flow 5: Tóm tắt toàn bộ

```
┌─────────────┐   (1) đăng ký schema   ┌──────────────────┐
│  Producer   │ ──────────────────────▶ │  Schema Registry │
│             │ ◀── schema_id (cache) ── │  (Avro/Protobuf) │
└─────────────┘                         └──────────────────┘
       │                                        ▲
       │ (2) gửi [magic][id][bytes]             │ (4) GET schema by id
       ▼                                        │     (cache sau lần đầu)
┌─────────────┐                         ┌──────┴───────┐
│ Kafka Topic │ ────────────────────────▶│   Consumer   │
└─────────────┘   (3) message bytes     └──────────────┘

Compatibility check xảy ra ở bước (1) — nếu fail, message không bao giờ được gửi.
```

---

## 9. Interview Q&A

**Q: Schema Registry giải quyết vấn đề gì?**

> Giải quyết vấn đề producer và consumer không đồng thuận về format message. Không có Registry, breaking change schema không bị phát hiện đến lúc production crash. Registry cung cấp: schema tập trung, versioning, compatibility enforcement (từ chối breaking change ở deploy time), và wire format nhỏ gọn (5 byte overhead thay vì embed schema vào mỗi message).

**Q: BACKWARD compatibility nghĩa là gì?**

> Consumer mới đọc được data cũ. Ví dụ: producer upgrade schema v2 (thêm field có default), consumer vẫn chạy v1 đọc được message từ v2 — field mới sẽ dùng default value. Upgrade consumer trước, sau đó mới upgrade producer.

**Q: Nhược điểm lớn nhất của Schema Registry?**

> Thêm một dependency cần quản lý. Nếu không deploy HA, Registry là single point of failure. Ngoài ra schema evolution bị giới hạn — đổi tên field bị coi là breaking change, phải workaround bằng alias hoặc deprecate field cũ + thêm field mới.
