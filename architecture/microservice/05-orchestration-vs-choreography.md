# 05 — Orchestration vs Choreography

Hai cách điều phối luồng xử lý trong Saga và Event-Driven Architecture.

---

## Choreography — Mỗi service tự biết mình làm gì

Không có trung tâm điều phối. Mỗi service lắng nghe event và tự quyết định hành động, sau đó phát event tiếp theo.

```
Order Service ──OrderPlaced──▶ Broker
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
              Payment         Inventory    Notification
              Service         Service      Service
              (charge)        (reserve)    (send email)
                │
         PaymentCompleted
                │
                ▼
             Broker → Order Service (confirm)
```

**Trường hợp thất bại:**
```
Payment Service thất bại
    └──▶ publish PaymentFailed
              └──▶ Order Service nhận → cancel order
              └──▶ Inventory Service nhận → release reservation
```

**Ưu điểm:**
- Loose coupling — service không biết nhau, chỉ biết event
- Dễ thêm service mới (chỉ cần subscribe thêm event)
- Không có single point of failure

**Nhược điểm:**
- Khó trace flow — phải query nhiều service/log để hiểu trạng thái saga
- Compensating transaction phức tạp — mỗi service tự xử lý
- Dễ ra circular dependency nếu thiết kế event không cẩn thận

---

## Orchestration — Có một orchestrator điều phối

Một service trung tâm (orchestrator) ra **command** cho từng service theo thứ tự và xử lý kết quả.

```
                  Order Orchestrator
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
    gọi Payment   gọi Inventory  gọi Notification
    Service       Service        Service
          │             │
    PaymentFailed        │
          │             │
    gọi CancelOrder ◀───┘ (rollback)
```

**Trường hợp thất bại:**
```
Orchestrator gọi Payment → fail
    └──▶ Orchestrator gọi CancelOrder (compensating)
    └──▶ Orchestrator gọi ReleaseInventory (compensating)
    └──▶ Cập nhật saga state = FAILED
```

**Lưu trạng thái saga:**
```sql
-- Orchestrator lưu từng bước vào DB
CREATE TABLE saga_state (
    saga_id    VARCHAR(36) PRIMARY KEY,
    step       VARCHAR(50),   -- order_created, payment_done, ...
    status     VARCHAR(20),   -- in_progress, completed, failed
    data       JSON,
    updated_at TIMESTAMP
);
```

**Ưu điểm:**
- Flow rõ ràng, dễ trace và monitor
- Xử lý compensating transaction tập trung tại orchestrator
- Dễ debug — chỉ cần xem state của orchestrator

**Nhược điểm:**
- Orchestrator là single point of failure (cần deploy HA)
- Coupling cao hơn — service phụ thuộc orchestrator
- Orchestrator có thể trở thành God service nếu không giới hạn scope

---

## So sánh

| Tiêu chí | Choreography | Orchestration |
|---------|-------------|--------------|
| Điều phối | Phân tán qua event | Tập trung tại orchestrator |
| Coupling | Loose | Tighter |
| Trace/Debug | Khó (phân tán) | Dễ (tập trung) |
| Thêm service mới | Dễ (subscribe thêm) | Cần update orchestrator |
| Compensating tx | Mỗi service tự xử lý | Orchestrator xử lý |
| SPOF | Không | Có (orchestrator) |
| Dùng khi | Flow đơn giản, team độc lập | Flow phức tạp, cần visibility |

---

## Interview Q&A

**Q: Choreography hay Orchestration — chọn gì?**

> Không có câu trả lời tuyệt đối. Flow đơn giản, ít bước, team độc lập cao → Choreography. Flow phức tạp, nhiều bước, cần compensating transaction rõ ràng, cần audit trail → Orchestration. Thực tế nhiều hệ thống dùng cả hai: dùng Orchestration cho saga phức tạp (checkout flow), Choreography cho notification/analytics (fan-out đơn giản).

**Q: Orchestrator lưu state ở đâu?**

> Thường lưu trong database (PostgreSQL, MongoDB) với bảng saga_state. Mỗi bước cập nhật trạng thái trước khi gọi service tiếp theo — đảm bảo idempotency khi orchestrator restart. Có thể dùng Redis cho tốc độ nếu kết hợp với DB backup.
