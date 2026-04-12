# Hexagonal Architecture (Kiến trúc Lục giác)

## 1. Định nghĩa

**Hexagonal Architecture** là một mô hình thiết kế phần mềm được đề xuất bởi **Alistair Cockburn** nhằm tách biệt **core business logic** (logic nghiệp vụ cốt lõi) khỏi các phần phụ thuộc bên ngoài như:

- Database
- UI
- Message queues
- API / REST
- Frameworks, libraries...

Tên gọi "lục giác" chỉ là biểu tượng hình học, nhưng ý tưởng chính là: **mọi tương tác bên ngoài đều được kết nối với lõi qua các "cổng" (ports) và "bộ chuyển đổi" (adapters).**

![Hexagonal Architecture Diagram](./../images/hexagonal.webp)

---

## 2. Thành phần chính

### a. Core (Domain / Application logic)

- Là trung tâm hệ thống, chứa business logic thật sự.
- Không phụ thuộc bất kỳ công nghệ nào (framework, thư viện, cơ sở dữ liệu...).

### b. Ports

- Là các interface mô tả **các hành vi mà hệ thống mong đợi** (inbound và outbound).
- Ví dụ: `UserRepository`, `PaymentProcessor`, `OrderInputPort`

### c. Adapters

- Là các implementation cụ thể kết nối thế giới bên ngoài vào core thông qua ports.
- Ví dụ:
  - REST Controller → Input Adapter
  - MySQL Repository → Output Adapter
  - Kafka Consumer/Producer Adapter

---

## 3. Phân loại Port

| Loại Port        | Mô tả |
|------------------|-------|
| Inbound Ports    | Cổng đầu vào – tương tác từ bên ngoài vào core (UI, API, CLI) |
| Outbound Ports   | Cổng đầu ra – core yêu cầu thực hiện hành vi bên ngoài (DB, email, queue) |

---

## 4. Lợi ích và nhược điểm

### Lợi ích
- **Tách biệt rõ ràng** giữa logic nghiệp vụ và hạ tầng
- **Dễ test** core mà không cần phụ thuộc công nghệ
- **Thay đổi adapter không ảnh hưởng domain**
- **Dễ mở rộng**: thêm REST, gRPC, CLI mà không thay đổi core

### Nhược điểm
- **Nhiều interface/abstraction**: mỗi interaction cần Port + Adapter → boilerplate nhiều
- **Over-engineering** với CRUD app hoặc dự án nhỏ
- **Khó setup ban đầu**: cần thiết kế Port rõ ràng trước khi code

---

## 5. Sơ đồ tổng quan

```pgsql
     +--------------------------+
     |      Input Adapters      |  ← REST, CLI, Web
     +-----------+--------------+
                 ↓
          +-------------+
          |   Inbound   | ← Interface (port)
          |   Ports     |
          +------+------+
                 ↓
       +------------------+
       |     Application   |
       |    (Use Cases)    |
       +------------------+
                 ↓
          +-------------+
          |  Outbound   | ← Interface (port)
          |   Ports     |
          +------+------+
                 ↓
     +--------------------------+
     |     Output Adapters      | ← DB, Email, Kafka
     +--------------------------+
```


---

## 6. So sánh với kiến trúc truyền thống (Layered Architecture)

| Layered Architecture         | Hexagonal Architecture           |
|-----------------------------|----------------------------------|
| Tầng Controller → Service → Repository | Các adapter kết nối qua port |
| Gắn chặt với framework      | Tách core ra khỏi framework     |
| Phụ thuộc 1 chiều            | Phụ thuộc vào interface         |

---

## 7. Khi nào nên dùng?

- Dự án có logic nghiệp vụ phức tạp, cần test độc lập với hạ tầng
- Dự án cần dễ dàng thay đổi adapter (DB, API, UI...)
- Xây dựng ứng dụng lâu dài, dễ maintain, dễ mở rộng

---

## 8. Ví dụ code đầy đủ

Context: Service quản lý đơn hàng — nhận lệnh đặt hàng từ REST API hoặc Kafka, kiểm tra tồn kho qua HTTP, lưu order vào DB.

### Cấu trúc folder

```
src/
├── core/                              ← KHÔNG import framework
│   ├── domain/
│   │   └── Order.java                 ← Entity
│   ├── ports/
│   │   ├── inbound/
│   │   │   └── OrderInputPort.java    ← Interface (inbound)
│   │   └── outbound/
│   │       ├── InventoryPort.java     ← Interface (outbound)
│   │       └── OrderRepository.java  ← Interface (outbound)
│   └── usecases/
│       └── PlaceOrderUseCase.java     ← Business logic
│
└── adapters/
    ├── inbound/
    │   ├── rest/
    │   │   └── OrderController.java   ← REST → core
    │   └── messaging/
    │       └── OrderEventConsumer.java ← Kafka → core
    └── outbound/
        ├── persistence/
        │   └── OrderJpaAdapter.java   ← core → DB
        └── http/
            └── InventoryHttpAdapter.java ← core → HTTP
```

---

### `core/domain/Order.java` — Entity

```java
// Không import framework, không import Spring, không import JPA
public class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private OrderStatus status;

    private Order(OrderId id, CustomerId customerId, List<OrderItem> items) {
        this.id = id;
        this.customerId = customerId;
        this.items = List.copyOf(items);
        this.status = OrderStatus.PENDING;
    }

    public static Order create(CustomerId customerId, List<OrderItem> items) {
        if (items.isEmpty()) throw new DomainException("Order must have at least one item");
        return new Order(OrderId.generate(), customerId, items);
    }

    public Money totalAmount() {
        return items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::add);
    }

    public void confirm() {
        if (status != OrderStatus.PENDING) throw new DomainException("Only PENDING order can be confirmed");
        this.status = OrderStatus.CONFIRMED;
    }

    // getters...
}
```

---

### `core/ports/inbound/OrderInputPort.java` — Inbound Port

```java
// Interface — core định nghĩa, adapter bên ngoài gọi vào
public interface OrderInputPort {
    OrderId placeOrder(PlaceOrderCommand command);
}
```

---

### `core/ports/outbound/InventoryPort.java` — Outbound Port

```java
// Interface — core định nghĩa, adapter bên ngoài implement
public interface InventoryPort {
    void reserve(List<OrderItem> items);
}
```

---

### `core/ports/outbound/OrderRepository.java` — Outbound Port

```java
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}
```

---

### `core/usecases/PlaceOrderUseCase.java` — Business Logic

```java
// Implement inbound port, dùng outbound port — không biết gì về framework
public class PlaceOrderUseCase implements OrderInputPort {

    private final InventoryPort inventoryPort;
    private final OrderRepository orderRepository;

    public PlaceOrderUseCase(InventoryPort inventoryPort, OrderRepository orderRepository) {
        this.inventoryPort = inventoryPort;
        this.orderRepository = orderRepository;
    }

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        // 1. Tạo domain object — validate trong constructor
        Order order = Order.create(command.customerId(), command.items());

        // 2. Gọi outbound port (không biết đây là HTTP hay mock)
        inventoryPort.reserve(order.getItems());

        // 3. Confirm order
        order.confirm();

        // 4. Lưu qua outbound port (không biết đây là JPA hay in-memory)
        orderRepository.save(order);

        return order.getId();
    }
}
```

---

### `adapters/inbound/rest/OrderController.java` — Input Adapter (REST)

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderInputPort orderInputPort; // inject interface, không inject UseCase trực tiếp

    public OrderController(OrderInputPort orderInputPort) {
        this.orderInputPort = orderInputPort;
    }

    @PostMapping
    public ResponseEntity<Map<String, String>> placeOrder(@RequestBody PlaceOrderRequest request) {
        PlaceOrderCommand command = new PlaceOrderCommand(
            new CustomerId(request.customerId()),
            request.items().stream().map(PlaceOrderRequest.Item::toDomain).toList()
        );
        OrderId orderId = orderInputPort.placeOrder(command);
        return ResponseEntity.ok(Map.of("orderId", orderId.value()));
    }
}
```

---

### `adapters/inbound/messaging/OrderEventConsumer.java` — Input Adapter (Kafka)

```java
@Component
public class OrderEventConsumer {

    private final OrderInputPort orderInputPort; // cùng port, khác entry point

    @KafkaListener(topics = "order-requests")
    public void consume(PlaceOrderMessage message) {
        PlaceOrderCommand command = message.toCommand();
        orderInputPort.placeOrder(command); // gọi cùng use case như REST
    }
}
```

---

### `adapters/outbound/persistence/OrderJpaAdapter.java` — Output Adapter (DB)

```java
@Component
public class OrderJpaAdapter implements OrderRepository { // implement outbound port

    private final OrderJpaRepository jpaRepository; // Spring Data JPA

    @Override
    public void save(Order order) {
        OrderEntity entity = OrderEntity.fromDomain(order); // map domain → JPA entity
        jpaRepository.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.value())
            .map(OrderEntity::toDomain); // map JPA entity → domain
    }
}
```

---

### `adapters/outbound/http/InventoryHttpAdapter.java` — Output Adapter (HTTP)

```java
@Component
public class InventoryHttpAdapter implements InventoryPort { // implement outbound port

    private final RestTemplate restTemplate;

    @Override
    public void reserve(List<OrderItem> items) {
        ReserveRequest request = ReserveRequest.from(items);
        ResponseEntity<Void> response = restTemplate.postForEntity(
            "http://inventory-service/api/reservations",
            request,
            Void.class
        );
        if (!response.getStatusCode().is2xxSuccessful()) {
            throw new InfrastructureException("Inventory reservation failed");
        }
    }
}
```

---

### Unit Test — test core không cần HTTP, không cần DB

```java
class PlaceOrderUseCaseTest {

    // Mock outbound ports — không cần Spring context, không cần DB
    private final InventoryPort inventoryPort = Mockito.mock(InventoryPort.class);
    private final OrderRepository orderRepository = Mockito.mock(OrderRepository.class);
    private final PlaceOrderUseCase useCase = new PlaceOrderUseCase(inventoryPort, orderRepository);

    @Test
    void shouldPlaceOrderSuccessfully() {
        // Arrange
        PlaceOrderCommand command = new PlaceOrderCommand(
            new CustomerId("cust-1"),
            List.of(new OrderItem(new ProductId("prod-1"), 2, Money.of(50000)))
        );

        // Act
        OrderId orderId = useCase.placeOrder(command);

        // Assert
        assertNotNull(orderId);
        verify(inventoryPort).reserve(anyList());
        verify(orderRepository).save(any(Order.class));
    }

    @Test
    void shouldFailWhenItemsEmpty() {
        PlaceOrderCommand command = new PlaceOrderCommand(new CustomerId("cust-1"), List.of());
        assertThrows(DomainException.class, () -> useCase.placeOrder(command));
        verifyNoInteractions(inventoryPort, orderRepository);
    }
}
```

---

## 10. Các công nghệ phù hợp

- Java/Spring Boot (với interface, annotation @Component dễ define adapter)
- Node.js (với dependency injection bằng module)
- Python (sử dụng abstract class + DI frameworks)
- Go (interface + adapter pattern)

---

## 11. Interview Q&A

**Q: Hexagonal Architecture là gì? Tại sao cần nó?**

> Hexagonal (Ports & Adapters) tách biệt core business logic khỏi hạ tầng bằng cách định nghĩa Port (interface) ở giữa. Core không biết gì về DB hay framework — chỉ giao tiếp qua interface. Lợi ích chính: test độc lập core, thay đổi adapter (đổi DB, thêm gRPC) mà không chạm vào business logic.

---

**Q: Port và Adapter khác nhau thế nào?**

> - **Port** = interface, do core định nghĩa, mô tả hành vi mong đợi. Ví dụ: `OrderRepository`, `InventoryPort`.
> - **Adapter** = implementation cụ thể của port, thuộc tầng ngoài. Ví dụ: `OrderJpaAdapter` implement `OrderRepository`, `InventoryHttpAdapter` implement `InventoryPort`.
> - Một port có thể có nhiều adapter: test dùng mock adapter, production dùng JPA adapter.

---

**Q: Inbound Port và Outbound Port khác nhau thế nào?**

> - **Inbound Port**: core *expose* ra để bên ngoài gọi vào. Ví dụ: `OrderInputPort` — REST controller và Kafka consumer đều gọi vào cùng port này.
> - **Outbound Port**: core *yêu cầu* bên ngoài thực hiện. Ví dụ: `OrderRepository` — core cần lưu dữ liệu nhưng không biết lưu ở đâu.

---

**Q: Khác gì so với N-Layer truyền thống?**

> N-Layer: Controller → Service → Repository — Service biết Repository cụ thể, test phải mock từng layer.
> Hexagonal: Core chỉ biết interface (Port), không biết implementation. Adapter có thể swap tự do. Test core chỉ cần mock interface, không cần Spring context hay database.

---

**Q: Hexagonal và Clean Architecture giống/khác nhau thế nào?**

> Giống: cả hai đều cô lập core, mọi phụ thuộc hướng vào trong, giao tiếp qua interface.
> Khác: Clean Architecture phân tách rõ hơn giữa **Entity** (enterprise rules) và **Use Case** (application rules). Hexagonal không bắt buộc phân tách này — chỉ quan tâm port/adapter. Có thể nói Clean Architecture là Hexagonal với thêm lớp phân tầng bên trong core.

---

**Q: Khi nào KHÔNG nên dùng Hexagonal?**

> - CRUD app đơn giản: thêm port/adapter là over-engineering
> - Prototype, MVP cần ship nhanh
> - Team nhỏ, chưa quen với pattern — learning curve cao
> - Logic đơn giản, không cần swap adapter

---

**Q: Dependency Injection liên quan thế nào?**

> DI là cơ chế để "cắm" adapter vào core tại runtime mà core không biết cụ thể là adapter nào. Core chỉ khai báo cần `OrderRepository` (interface) trong constructor — Spring/DI container inject `OrderJpaAdapter` vào. Khi test, inject mock. → Core code không thay đổi, chỉ thay adapter được inject.

---

## 12. Kết luận

Hexagonal Architecture giúp xây dựng hệ thống:
- Dễ kiểm thử (testable)
- Dễ bảo trì (maintainable)
- Không phụ thuộc framework cụ thể
- Linh hoạt trong việc thay đổi adapter / công nghệ

