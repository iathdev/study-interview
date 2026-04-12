# 07 — Distributed Transaction

Trong Microservice, mỗi service có DB riêng. Không có ACID transaction nào bao phủ nhiều service → cần giải pháp riêng.

---

## 1. Two-Phase Commit (2PC)

### Cơ chế

Có 2 vai trò:
- **Coordinator (Điều phối viên)**: service trung tâm ra lệnh prepare/commit/rollback
- **Participant (Thành viên tham gia)**: các service/DB nhận lệnh và thực hiện

Coordinator điều phối tất cả participant thực hiện hoặc hủy đồng thời.

```
═══ Phase 1: Prepare (hỏi tất cả "sẵn sàng chưa?") ═══

Điều phối viên ──"Prepare?"──▶ Order DB (thành viên)     ──▶ lock row, ghi log ──▶ "Ready"
               ──"Prepare?"──▶ Payment DB (thành viên)   ──▶ lock row, ghi log ──▶ "Ready"
               ──"Prepare?"──▶ Inventory DB (thành viên) ──▶ lock row, ghi log ──▶ "Ready"

→ Tất cả trả lời "Ready" → sang Phase 2 Commit
→ Có 1 service trả "Abort" → sang Phase 2 Rollback

═══ Phase 2a: Commit (tất cả Ready) ═══

Điều phối viên ──"Commit"──▶ Order DB      ──▶ commit, release lock ──▶ "Done"
               ──"Commit"──▶ Payment DB    ──▶ commit, release lock ──▶ "Done"
               ──"Commit"──▶ Inventory DB  ──▶ commit, release lock ──▶ "Done"

═══ Phase 2b: Rollback (có ít nhất 1 Abort) ═══

Ví dụ: Payment DB trả "Abort" ở Phase 1

Điều phối viên ──"Rollback"──▶ Order DB      ──▶ undo, release lock
               ──"Rollback"──▶ Payment DB    ──▶ undo, release lock
               ──"Rollback"──▶ Inventory DB  ──▶ undo, release lock

→ Tất cả quay về trạng thái ban đầu, như chưa có gì xảy ra
```

### Ưu điểm
- Strong consistency (ACID)
- Đơn giản về mặt logic

### Nhược điểm
- **Blocking**: Điều phối viên crash trong Phase 2 → tất cả thành viên bị treo (giữ lock mãi)
- **Performance**: Resource bị lock trong suốt 2 phase → throughput thấp
- **Không phù hợp Microservice**: Service phải hỗ trợ XA protocol, nhiều DB NoSQL không hỗ trợ

### XA Transactions

XA là **giao thức chuẩn** để implement 2PC, định nghĩa interface giữa Transaction Manager và Resource Manager (`xa_prepare`, `xa_commit`, `xa_rollback`).

- Hỗ trợ: MySQL, PostgreSQL, Oracle
- Không hỗ trợ: MongoDB, Cassandra, Redis, DynamoDB
- Java: Spring JTA, Atomikos, Bitronix

→ **Thực tế**: Ít dùng trong Microservice hiện đại. Chỉ dùng khi tất cả service đều dùng SQL DB có hỗ trợ XA và yêu cầu strong consistency bắt buộc.

---

## 2. Saga Pattern

### Ý tưởng cốt lõi

2PC giống **1 người chỉ huy bảo tất cả đứng yên chờ** → ai cũng bị khóa.

Saga giống **dây chuyền sản xuất** — mỗi người làm phần của mình xong thì chuyền cho người tiếp theo. Nếu người nào đó làm hỏng → **quay ngược lại từng người để hoàn tác**.

```
2PC:   "Tất cả chuẩn bị... chờ... chờ... OK commit!"  → lock tất cả
Saga:  "A làm xong → B làm → C làm → Done"             → không ai bị lock
```

### Ví dụ — Đặt hàng

**Thành công — mỗi bước xong thì sang bước tiếp:**
```
Bước 1              Bước 2                Bước 3              Bước 4
Tạo đơn hàng ─OK─▶ Giữ hàng trong kho ─OK─▶ Thanh toán ─OK─▶ Xác nhận đơn
(Order DB)          (Inventory DB)            (Payment DB)     (Order DB)
```

**Thất bại — thanh toán lỗi → hoàn tác ngược lại từng bước:**
```
Tạo đơn hàng ─OK─▶ Giữ hàng trong kho ─OK─▶ Thanh toán ─FAIL!
                                                   │
                                          đi ngược lại:
                                                   │
                              Trả hàng về kho ◀────┘  (hoàn tác bước 2)
                    Hủy đơn hàng ◀─────────────────    (hoàn tác bước 1)
```

### Compensating Transaction (Giao dịch hoàn tác)

Mỗi bước **phải có sẵn** một hành động ngược lại — thiết kế từ đầu, không phải nghĩ khi lỗi xảy ra:

| Bước | Hành động | Hoàn tác |
|------|-----------|----------|
| 1 | Tạo đơn hàng | Hủy đơn hàng |
| 2 | Giữ hàng trong kho | Trả hàng về kho |
| 3 | Thanh toán | Hoàn tiền |

> **Hoàn tác ≠ Rollback**
>
> - **Rollback** (trong DB): quay về như chưa có gì xảy ra — dữ liệu biến mất
> - **Hoàn tác Saga**: tạo một **hành động mới** để đảo ngược — dữ liệu vẫn còn
>
> Ví dụ: thanh toán 500k đã charge thật vào thẻ → không thể "xóa" giao dịch → phải **tạo lệnh hoàn tiền 500k**. Cả 2 record đều tồn tại trong DB.

### Saga đảm bảo gì?

```
2PC:  Tất cả thành công HOẶC tất cả thất bại → không ai thấy trạng thái nửa vời
Saga: Trong lúc đang xử lý, service khác CÓ THỂ thấy trạng thái trung gian

Ví dụ: Đơn hàng đã tạo (bước 1 xong) nhưng chưa thanh toán (bước 3 chưa chạy)
       → Service khác query thấy đơn hàng "PENDING" → đây là eventual consistency
```

### Choreography vs Orchestration

→ Xem chi tiết tại [05-orchestration-vs-choreography.md](./05-orchestration-vs-choreography.md)

---

## 3. So sánh 2PC vs Saga

| Tiêu chí | 2PC | Saga |
|---------|-----|------|
| Consistency | Strong (ACID) | Eventual consistency |
| Performance | Thấp (lock resource) | Cao (local transaction) |
| Fault tolerance | Kém (coordinator SPOF) | Tốt (mỗi step độc lập) |
| Complexity | Logic đơn giản hơn | Phải thiết kế compensating tx |
| Phù hợp | Monolith, SQL DB, ít service | Microservice, nhiều DB type |
| NoSQL support | Không | Có |
| Blocking | Có | Không |
| Intermediate state | Không (atomic) | Có (thấy trạng thái trung gian) |

---

## 4. Khi nào dùng gì?

```
Hệ thống có nhiều service với DB riêng?
  Không → Single DB transaction là đủ

  Có → Cần distributed transaction

       Tất cả service dùng SQL DB hỗ trợ XA
       + Yêu cầu strong consistency bắt buộc?
         Có → 2PC / XA
         Không → Saga

       Microservice, NoSQL, cần scale?
         → Saga (Choreography hoặc Orchestration)
```

---

## Interview Q&A

**Q: Tại sao không dùng 2PC trong Microservice?**

> Ba lý do: (1) Blocking — điều phối viên crash khiến toàn bộ thành viên bị treo giữ lock; (2) Performance — resource bị lock trong suốt 2 phase, throughput thấp; (3) Compatibility — nhiều DB NoSQL và message broker không hỗ trợ XA protocol. Saga giải quyết cả ba vấn đề này.

**Q: Saga đảm bảo consistency không?**

> Saga chỉ đảm bảo **eventual consistency**, không phải strong consistency. Trong quá trình xử lý, service khác có thể thấy trạng thái trung gian (order CREATED nhưng payment chưa xong). Đây là trade-off chấp nhận được trong Microservice — đổi lấy availability và performance.

**Q: Compensating transaction có giống Rollback không?**

> Không giống. DB Rollback undo thay đổi về state trước đó (như chưa có gì xảy ra). Compensating transaction là một **business action mới** để đảo ngược kết quả — ví dụ: payment đã được charge thật → tạo refund record, không thể xóa payment record. Compensating action phải **idempotent** vì có thể bị gọi nhiều lần.
