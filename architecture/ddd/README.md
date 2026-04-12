`Domain > Subdomain > Bounded Context`

> Một Bounded Context nên tương ứng với một service trong microservice.

---

## Strategic Design (What & Why)

| File | Nội dung |
|------|----------|
| [1. Strategic & Tactical Design](1.%20Stategy%20-%20Tactical%20design.md) | Tổng quan hai lớp thiết kế trong DDD |
| [2. Domain](2.%20Domain.md) | Domain, Subdomain: Core / Supporting / Generic |
| [3. Bounded Context](3.%20Bounded%20context.md) | Ranh giới ngữ nghĩa, BC vs Subdomain, BC trong microservices |
| [4. Ubiquitous Language](4.%20Ubiquitous%20Language.md) | Ngôn ngữ chung giữa dev và domain expert |
| [5. Context Map](5.%20Context%20Map.md) | Quan hệ giữa các BC: ACL, OHS, Customer-Supplier, Shared Kernel... |

## Tactical Design (How)

| File | Nội dung |
|------|----------|
| [6. Entity](6.%20Entity.md) | Object có identity, equality theo ID, lifecycle, domain events |
| [7. Value Object](7.%20Value%20Object.md) | Object immutable, equality theo giá trị |
| [8. Aggregate](8.%20Aggregate.md) | Cụm Entity/VO, Aggregate Root, consistency boundary |
| [9. Domain Event](9.%20Domain%20Event.md) | Sự kiện domain, outbox pattern, event vs command |
| [11. Domain Service](11.%20Domain%20Service.md) | Business logic không thuộc Entity hay VO |
| [12. Repository](12.%20Repository.md) | Abstraction persistence, collection-like API |
| [13. Factory](13.%20Factory.md) | Tạo Aggregate phức tạp, encapsulate creation logic |

| [14. Event Storming](14.%20Event%20Storming.md) | Workshop khám phá domain bằng sticky notes |

## Ví dụ thực tế

| File | Nội dung |
|------|----------|
| [e-learning.md](e-learning.md) | DDD đầy đủ cho hệ thống E-Learning tiếng Anh |
