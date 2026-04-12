# Kiến trúc phần mềm — Tổng quan

Kiến trúc phần mềm là cách tổ chức code và các thành phần hệ thống để đáp ứng yêu cầu kỹ thuật và nghiệp vụ. Mục tiêu chung: **tách biệt trách nhiệm**, dễ test, dễ thay đổi, dễ mở rộng.

---

## Các kiến trúc nội bộ (Internal Architecture)

Quyết định cách tổ chức code **bên trong** một service/application.

### So sánh nhanh

| Tiêu chí                   | N-Layer        | Hexagonal        | Onion            | Clean Architecture |
|---------------------------|----------------|------------------|------------------|--------------------|
| Phụ thuộc chiều            | Trên → dưới    | Ngoài → trong    | Ngoài → trong    | Ngoài → trong      |
| Domain là trung tâm?       | Không hẳn      | Có               | Có               | Có                 |
| Tách UseCase rõ ràng?      | Không          | Không            | Ít               | Có                 |
| Test domain độc lập?       | Khó            | Dễ               | Dễ               | Dễ                 |
| Phù hợp với                | CRUD đơn giản  | Dự án trung bình | Domain phức tạp  | DDD, hệ thống lớn  |
| Độ phức tạp setup          | Thấp           | Trung bình       | Trung bình       | Cao                |

### Quan hệ giữa các kiến trúc

```
N-Layer (đơn giản nhất)
    │
    └──▶ Hexagonal / Onion (cô lập domain, dùng port-adapter)
              │
              └──▶ Clean Architecture (phân tách rõ UseCase + Entity)
                        │
                        └──▶ DDD (tactical patterns: Aggregate, Entity, Domain Event...)
```

---

### N-Layer Architecture
→ [Chi tiết: layer.md](./layer.md)

- Chia thành 4 tầng: Presentation → Application → Domain → Data Access
- Tầng trên phụ thuộc tầng dưới
- Dễ hiểu, phù hợp team mới, CRUD app
- Nhược: dễ sinh "God Service", anemic domain model

---

### Hexagonal Architecture (Ports & Adapters)
→ [Chi tiết: hexagonal.md](./hexagonal.md)

- Core (domain + use case) ở giữa, mọi thứ bên ngoài kết nối qua **Port** (interface) và **Adapter** (implementation)
- Inbound: REST, CLI, Queue → Core
- Outbound: Core → DB, Email, External API
- Thay đổi adapter (đổi DB, thêm gRPC) không ảnh hưởng core

---

### Onion Architecture
→ [Chi tiết: onion.md](./onion.md)

- Tương tự Hexagonal, nhấn mạnh Domain Model là lõi trung tâm
- Từ trong ra: Domain Model → Domain Services / Interfaces → Application Services → External
- Được coi là tiền thân của Clean Architecture

---

### Clean Architecture
→ [Chi tiết: clean.md](./clean.md)

- Đề xuất bởi Robert C. Martin (Uncle Bob)
- 4 vòng: Entities → Use Cases → Interface Adapters → Frameworks & Drivers
- Tách biệt rõ **Entity** (enterprise rules) và **Use Case** (application rules)
- Mọi phụ thuộc hướng từ ngoài vào trong (Dependency Rule)

---

## Các kiến trúc và pattern bổ trợ

### Event-Driven Architecture (EDA)
→ [Chi tiết: event-driven-architecture.md](./event-driven-architecture.md)

- Các service giao tiếp qua **event** thay vì gọi trực tiếp
- Producer publish event → Message Broker (Kafka, RabbitMQ) → Consumer subscribe
- Loose coupling, resilient, scalable
- Hai pattern điều phối: **Choreography** (service tự biết) vs **Orchestration** (có orchestrator)

---

### Event Sourcing
→ [Chi tiết: event-sourcing.md](./event-sourcing.md)

- Lưu **chuỗi event** thay vì chỉ lưu state hiện tại
- State = replay toàn bộ event từ đầu
- Có đầy đủ audit trail, temporal query, dễ debug
- Thường kết hợp với **CQRS**: Write side dùng Event Store, Read side dùng Projection

---

### Event Storming
→ [Chi tiết: event_storming.md](./event_storming.md)

- Workshop khám phá nghiệp vụ, không phải kiến trúc kỹ thuật
- Dùng sticky notes để map Domain Event → Command → Actor → Aggregate → Policy
- Output: xác định Bounded Context, Aggregate cho DDD

---

### DDD (Domain-Driven Design)
→ [Chi tiết: ddd/README.md](./ddd/README.md)

- Phương pháp thiết kế phần mềm xoay quanh **domain model**
- Strategic: Bounded Context, Ubiquitous Language, Context Map
- Tactical: Entity, Value Object, Aggregate, Domain Event, Repository, Factory, Domain Service

---

## Khi nào dùng gì?

| Tình huống                                       | Kiến trúc phù hợp              |
|--------------------------------------------------|-------------------------------|
| CRUD app, team nhỏ, deadline gấp                 | N-Layer                       |
| Logic vừa, cần test tốt, dễ thay DB/API          | Hexagonal                     |
| Domain phức tạp, cần DDD                         | Clean + DDD                   |
| Hệ thống phân tán, nhiều service                 | EDA + Microservices           |
| Cần audit trail đầy đủ, temporal query           | Event Sourcing + CQRS         |
| Team mới cần align về nghiệp vụ                  | Event Storming (workshop)     |
