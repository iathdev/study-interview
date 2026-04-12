# 01 — Monolithic vs Microservice

## Monolithic

Toàn bộ hệ thống là một project duy nhất. Tất cả module chạy chung một process.

```
+------------------------------------------+
|              Monolithic App              |
|  [UI] → [Business Logic] → [Database]   |
+------------------------------------------+
```

**Dùng khi**: Dự án nhỏ, team nhỏ, cần ship nhanh, logic chưa phức tạp.

---

## Microservice

Chia thành nhiều service nhỏ, mỗi service đảm nhận một domain/chức năng cụ thể. Giao tiếp qua HTTP/REST, gRPC, hoặc Message Broker.

```
[API Gateway]
     │
     ├──▶ Order Service   ──▶ Order DB
     ├──▶ Payment Service ──▶ Payment DB
     └──▶ User Service    ──▶ User DB
```

**Dùng khi**: Hệ thống lớn, team lớn, cần scale từng phần độc lập, logic phức tạp theo domain.

---

## So sánh

| Tiêu chí          | Monolithic                       | Microservice                         |
|-------------------|----------------------------------|--------------------------------------|
| Triển khai        | Đơn giản, 1 lần deploy           | Phức tạp, deploy từng service        |
| Scale             | Scale toàn bộ app                | Scale từng service độc lập           |
| Fault isolation   | 1 lỗi → cả hệ thống ảnh hưởng   | 1 service lỗi → service khác vẫn OK  |
| Tech stack        | Thống nhất                       | Tự do chọn per-service               |
| Testing           | Dễ (in-process)                  | Khó hơn (network, contract testing)  |
| Consistency       | Dễ (1 DB, 1 transaction)         | Khó (distributed transaction)        |
| Team              | Phù hợp team nhỏ                 | Cần team lớn, DevOps mạnh            |
| Phù hợp           | Startup, MVP, CRUD đơn giản      | Hệ thống lớn, nhiều domain           |

---

## Thách thức của Microservice

- **Distributed transaction**: Không có ACID transaction qua nhiều service → phải dùng Saga
- **Data consistency**: Mỗi service có DB riêng → eventual consistency
- **Observability**: Cần distributed tracing (Jaeger, Zipkin), centralized logging
- **Network latency**: Gọi qua mạng chậm hơn in-process
- **Service discovery**: Tìm địa chỉ service động → cần Service Registry
- **Complexity**: Nhiều moving parts hơn Monolithic rất nhiều

---

## Interview Q&A

**Q: Khi nào nên chuyển từ Monolithic sang Microservice?**

> Khi team lớn hơn và cần deploy độc lập, khi một phần hệ thống cần scale khác phần còn lại, khi domain đã đủ rõ ràng để tách biệt (dùng DDD để xác định Bounded Context). Đừng bắt đầu với Microservice — start with Monolith, tách ra khi thực sự cần.

**Q: Nhược điểm lớn nhất của Microservice?**

> Distributed transaction và data consistency. Không có single DB transaction nên phải dùng Saga (eventual consistency). Cộng thêm complexity về infra: service discovery, load balancing, distributed tracing, circuit breaker.
