# 02 — Giao tiếp giữa các Microservice

Hai kiểu giao tiếp: **Đồng bộ (Sync)** và **Bất đồng bộ (Async)**.

---

## 1. Đồng bộ (Sync)

Caller gửi request và **chờ** response. Nếu downstream down → caller bị ảnh hưởng ngay.

### REST (HTTP/JSON)

```
Service A ──HTTP POST /orders──▶ Service B
          ◀────── 200 OK ────────
```

- Dễ debug, phổ biến, tooling tốt (Postman, Swagger)
- Không tối ưu hiệu năng (text-based, HTTP/1.1)
- Phù hợp: public API, giao tiếp external, team mới

### gRPC (Protobuf + HTTP/2)

```
Service A ──PlaceOrder(proto)──▶ Service B
          ◀──── OrderId ────────
```

| Tiêu chí       | REST (JSON + HTTP/1.1)        | gRPC (Protobuf + HTTP/2)         |
|---------------|-------------------------------|----------------------------------|
| Format        | Text (JSON)                   | Binary (Protobuf)                |
| Tốc độ        | Trung bình                    | Nhanh hơn ~5-10x                 |
| Payload size  | Lớn hơn                       | Nhỏ hơn (~3-10x)                 |
| Streaming     | Không hỗ trợ tốt              | Hỗ trợ bidirectional streaming   |
| Code gen      | Không tự động                 | Tự động từ `.proto`              |
| Debug         | Dễ (human-readable)           | Khó hơn, cần tooling             |
| Dùng khi      | Public API, external client   | Internal service-to-service      |

**Tại sao Protobuf nhanh hơn JSON?**
- Binary format → kích thước nhỏ hơn, không có whitespace/quotes
- Serialize/deserialize nhanh hơn vì schema cố định, không cần parse text
- HTTP/2 hỗ trợ multiplexing (nhiều request trên 1 TCP connection), header compression

---

## 2. Bất đồng bộ (Async) — Message Broker

Caller **không chờ** response. Gửi message vào broker, consumer xử lý khi có thể.

```
Order Service ──publish──▶ [Message Broker] ──deliver──▶ Payment Service
                                                    └──▶ Notification Service
```

**Khi nào dùng**: Khi downstream không cần trả lời ngay, khi cần fan-out đến nhiều service, khi cần resilience (downstream down thì message vẫn còn trong broker).

### Hai mô hình

**Point-to-Point (Queue)**
```
Producer ──▶ Queue ──▶ Consumer A  (chỉ 1 consumer nhận)
```
- Mỗi message chỉ xử lý bởi **một** consumer
- Dùng cho: task distribution, job queue

**Publish/Subscribe (Topic)**
```
Producer ──▶ Topic ──▶ Consumer A  (tất cả subscriber nhận)
                  └──▶ Consumer B
```
- Mỗi message deliver đến **tất cả** subscriber
- Dùng cho: event notification, fan-out

---

## 3. Khi nào dùng gì?

| Tình huống | Dùng |
|-----------|------|
| Cần response ngay (query data) | REST hoặc gRPC |
| Internal service, cần hiệu năng cao | gRPC |
| Loose coupling, downstream không cần reply ngay | Message Broker (Async) |
| Nhiều service cùng quan tâm 1 event | Message Broker (Pub/Sub) |
| Cần resilience khi service down | Message Broker |

---

## Interview Q&A

**Q: REST vs gRPC — chọn cái nào?**

> Internal service-to-service → gRPC: nhanh hơn, type-safe, streaming. Public API hoặc cần browser support → REST: tooling tốt hơn, dễ debug. Thực tế nhiều hệ thống dùng cả hai: REST cho external, gRPC cho internal.

**Q: Sync vs Async — trade-off là gì?**

> Sync: đơn giản, dễ trace, nhưng tăng coupling (A phụ thuộc B phải up). Async: loose coupling, resilient, nhưng eventual consistency, khó debug, cần xử lý idempotency và DLQ. Chọn Async khi hành động không cần kết quả ngay (gửi email, cộng điểm loyalty, update analytics).
