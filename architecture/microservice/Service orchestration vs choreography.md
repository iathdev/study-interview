Dưới đây là so sánh chi tiết giữa **Choreography Saga** và **Orchestration Saga**, bao gồm ưu điểm, nhược điểm, ứng dụng và cách triển khai từng loại một cách ngắn gọn, súc tích bằng tiếng Việt.

---

### **1. Choreography Saga (Saga phối hợp)**

#### **Khái niệm**
- Mỗi dịch vụ tự quyết định hành động dựa trên các **sự kiện** (events) phát ra từ các dịch vụ khác. Không có trung tâm điều phối, các dịch vụ giao tiếp qua hệ thống sự kiện hoặc hàng đợi tin nhắn (message queue).
- Ví dụ: Dịch vụ A phát sự kiện "OrderCreated" → Dịch vụ B nhận và xử lý → Phát sự kiện "PaymentProcessed" → Dịch vụ C tiếp tục.

#### **Ưu điểm**
- **Linh hoạt**: Các dịch vụ độc lập, không phụ thuộc vào một điểm điều khiển trung tâm.
- **Khả năng mở rộng**: Dễ thêm dịch vụ mới mà không cần thay đổi logic điều phối.
- **Phân tán**: Không có điểm tập trung, giảm nguy cơ tắc nghẽn.

#### **Nhược điểm**
- **Khó theo dõi**: Luồng xử lý phân tán, khó giám sát toàn bộ saga.
- **Phức tạp khi lỗi**: Xử lý giao dịch bù trừ đòi hỏi các dịch vụ phải tự nhận biết và xử lý lỗi.
- **Khó gỡ lỗi**: Khi hệ thống lớn, việc xác định lỗi trong chuỗi sự kiện trở nên phức tạp.

#### **Ứng dụng**
- **Hệ thống sự kiện-driven**: Thương mại điện tử (đặt hàng, thanh toán, giao hàng).
- **Hệ thống không cần điều phối chặt chẽ**: Quản lý quy trình phi tập trung như IoT, hệ thống phân tích dữ liệu.
- **Hệ thống yêu cầu mở rộng nhanh**: Các ứng dụng cần thêm dịch vụ mà không muốn thay đổi logic hiện có.

#### **Triển khai**
1. **Công nghệ sử dụng**: Sử dụng message broker (như Kafka, RabbitMQ) hoặc hệ thống publish/subscribe (như AWS SNS/SQS).
2. **Luồng thực thi**:
   - Mỗi dịch vụ đăng ký (subscribe) các sự kiện liên quan từ message broker.
   - Khi một dịch vụ hoàn thành giao dịch cục bộ, nó phát (publish) một sự kiện mới.
   - Dịch vụ khác nhận sự kiện, thực hiện giao dịch và tiếp tục phát sự kiện.
3. **Xử lý lỗi**: Mỗi dịch vụ phải tự thực hiện giao dịch bù trừ khi nhận sự kiện lỗi (ví dụ: "PaymentFailed").
4. **Ví dụ code (giả mã)**:
   ```javascript
   // Dịch vụ Order
   async function handleOrderCreation(order) {
       await db.saveOrder(order);
       publishEvent('OrderCreated', order);
   }

   // Dịch vụ Payment
   subscribe('OrderCreated', async (order) => {
       try {
           await processPayment(order);
           publishEvent('PaymentProcessed', order);
       } catch (error) {
           publishEvent('PaymentFailed', order);
       }
   });
   ```

---

### **2. Orchestration Saga (Saga điều phối)**

#### **Khái niệm**
- Một dịch vụ trung tâm (orchestrator) điều phối toàn bộ saga, ra lệnh (commands) cho các dịch vụ khác thực hiện giao dịch cục bộ.
- Ví dụ: Orchestrator ra lệnh "CreateOrder" cho dịch vụ Order → "ProcessPayment" cho dịch vụ Payment → "ScheduleDelivery" cho dịch vụ Delivery.

#### **Ưu điểm**
- **Dễ theo dõi**: Orchestrator quản lý toàn bộ luồng, dễ giám sát và gỡ lỗi.
- **Kiểm soát chặt chẽ**: Logic xử lý tập trung, dễ thực hiện giao dịch bù trừ khi có lỗi.
- **Đơn giản hóa dịch vụ**: Các dịch vụ chỉ cần thực hiện lệnh, không cần hiểu toàn bộ luồng saga.

#### **Nhược điểm**
- **Điểm tập trung**: Orchestrator có thể trở thành điểm lỗi duy nhất (single point of failure).
- **Phụ thuộc**: Các dịch vụ phụ thuộc vào orchestrator, làm giảm tính độc lập.
- **Tắc nghẽn**: Nếu orchestrator xử lý quá nhiều saga, có thể gây chậm trễ.

#### **Ứng dụng**
- **Quy trình kinh doanh phức tạp**: Cần kiểm soát chặt chẽ các bước như xử lý đơn hàng trong thương mại điện tử, chuyển tiền ngân hàng.
- **Hệ thống cần minh bạch**: Dễ giám sát trạng thái saga, như trong hệ thống tài chính.
- **Hệ thống có luồng cố định**: Các bước thực hiện theo trình tự rõ ràng.

#### **Triển khai**
1. **Công nghệ sử dụng**: Sử dụng hệ thống lệnh (command-based) qua REST API, gRPC, hoặc message queue.
2. **Luồng thực thi**:
   - Orchestrator lưu trữ trạng thái saga (ví dụ: trong cơ sở dữ liệu hoặc bộ nhớ tạm).
   - Gửi lệnh đến từng dịch vụ theo trình tự.
   - Nhận phản hồi từ dịch vụ để quyết định bước tiếp theo hoặc kích hoạt giao dịch bù trừ.
3. **Xử lý lỗi**: Orchestrator quản lý lỗi, ra lệnh bù trừ nếu cần.
4. **Ví dụ code (giả mã)**:
   ```javascript
   // Orchestrator
   async function runOrderSaga(order) {
       try {
           await orderService.createOrder(order);
           await paymentService.processPayment(order);
           await deliveryService.scheduleDelivery(order);
       } catch (error) {
           // Giao dịch bù trừ
           await orderService.cancelOrder(order);
           throw new Error('Saga failed');
       }
   }

   // Dịch vụ Order
   async function createOrder(order) {
       await db.saveOrder(order);
       return { status: 'success' };
   }
   ```

---

### **So sánh ngắn gọn**

| Tiêu chí             | Choreography Saga                          | Orchestration Saga                         |
|----------------------|--------------------------------------------|--------------------------------------------|
| **Điều phối**        | Phân tán, dựa trên sự kiện                 | Tập trung, dựa trên orchestrator           |
| **Ưu điểm**          | Linh hoạt, mở rộng tốt, không tập trung    | Dễ theo dõi, kiểm soát chặt chẽ            |
| **Nhược điểm**       | Khó theo dõi, gỡ lỗi phức tạp              | Điểm tập trung, phụ thuộc vào orchestrator |
| **Ứng dụng**         | Hệ thống phi tập trung, sự kiện-driven     | Quy trình phức tạp, cần kiểm soát chặt     |
| **Công nghệ**        | Message broker (Kafka, RabbitMQ)           | API (REST, gRPC), message queue            |

---

### **Tóm tắt triển khai**
- **Choreography**: Phù hợp khi cần tính độc lập và mở rộng cao, nhưng đòi hỏi thiết kế sự kiện cẩn thận và hệ thống giám sát mạnh mẽ.
- **Orchestration**: Phù hợp khi cần kiểm soát chặt chẽ và dễ dàng theo dõi, nhưng phải đảm bảo orchestrator có khả năng xử lý tải lớn và đáng tin cậy.

---

Dưới đây là ví dụ về **luồng xử lý** cho **Choreography Saga** và **Orchestration Saga** trong một kịch bản thương mại điện tử: **Đặt hàng → Thanh toán → Giao hàng**. Tôi sẽ mô tả luồng xử lý cụ thể, bao gồm cả trường hợp thành công và thất bại, một cách ngắn gọn và rõ ràng bằng tiếng Việt.

---

### **Kịch bản**
Một khách hàng đặt hàng trên website, hệ thống cần:
1. Tạo đơn hàng (Order Service).
2. Xử lý thanh toán (Payment Service).
3. Lên lịch giao hàng (Delivery Service).

---

### **1. Choreography Saga (Saga phối hợp)**

#### **Luồng xử lý**
- **Cơ chế**: Các dịch vụ giao tiếp qua **sự kiện** (events) sử dụng message broker (như Kafka).
- **Trạng thái**: Không có trung tâm điều phối, mỗi dịch vụ tự phản ứng với sự kiện.

#### **Trường hợp thành công**
1. **Khách hàng đặt hàng**:
   - Order Service tạo đơn hàng trong cơ sở dữ liệu.
   - Phát sự kiện: `OrderCreated` (gửi đến message broker).
2. **Thanh toán**:
   - Payment Service nhận sự kiện `OrderCreated`.
   - Xử lý thanh toán (ví dụ: trừ tiền từ thẻ tín dụng).
   - Phát sự kiện: `PaymentProcessed`.
3. **Giao hàng**:
   - Delivery Service nhận sự kiện `PaymentProcessed`.
   - Lên lịch giao hàng.
   - Phát sự kiện: `DeliveryScheduled`.
4. **Kết thúc**:
   - Hệ thống thông báo khách hàng: "Đơn hàng đã được xử lý và lên lịch giao."

#### **Trường hợp thất bại**
- **Tình huống**: Thanh toán thất bại (ví dụ: thẻ không đủ tiền).
1. **Khách hàng đặt hàng**:
   - Order Service tạo đơn hàng, phát sự kiện `OrderCreated`.
2. **Thanh toán thất bại**:
   - Payment Service nhận `OrderCreated`, cố gắng xử lý thanh toán nhưng thất bại.
   - Phát sự kiện: `PaymentFailed`.
3. **Hủy đơn hàng**:
   - Order Service nhận sự kiện `PaymentFailed`.
   - Thực hiện giao dịch bù trừ: hủy đơn hàng trong cơ sở dữ liệu.
   - Phát sự kiện: `OrderCancelled`.
4. **Kết thúc**:
   - Khách hàng nhận thông báo: "Đơn hàng bị hủy do thanh toán thất bại."

#### **Sơ đồ luồng (giả định)**:
```
Khách hàng → Order Service (OrderCreated) → Payment Service (PaymentProcessed) → Delivery Service (DeliveryScheduled)
          ↳ (PaymentFailed) → Order Service (OrderCancelled)
```

---

### **2. Orchestration Saga (Saga điều phối)**

#### **Luồng xử lý**
- **Cơ chế**: Một **Orchestrator** (dịch vụ điều phối) quản lý toàn bộ luồng, gửi **lệnh** (commands) đến các dịch vụ qua REST API hoặc message queue.
- **Trạng thái**: Orchestrator lưu trạng thái saga (ví dụ: "đang tạo đơn", "đang thanh toán").

#### **Trường hợp thành công**
1. **Khách hàng đặt hàng**:
   - Orchestrator nhận yêu cầu đặt hàng.
   - Gửi lệnh `CreateOrder` đến Order Service.
   - Order Service tạo đơn hàng, trả về trạng thái "success".
2. **Thanh toán**:
   - Orchestrator gửi lệnh `ProcessPayment` đến Payment Service.
   - Payment Service xử lý thanh toán, trả về trạng thái "success".
3. **Giao hàng**:
   - Orchestrator gửi lệnh `ScheduleDelivery` đến Delivery Service.
   - Delivery Service lên lịch giao hàng, trả về trạng thái "success".
4. **Kết thúc**:
   - Orchestrator cập nhật trạng thái saga thành "completed".
   - Thông báo khách hàng: "Đơn hàng đã được xử lý và lên lịch giao."

#### **Trường hợp thất bại**
- **Tình huống**: Thanh toán thất bại.
1. **Khách hàng đặt hàng**:
   - Orchestrator gửi `CreateOrder` đến Order Service, đơn hàng được tạo.
2. **Thanh toán thất bại**:
   - Orchestrator gửi `ProcessPayment` đến Payment Service, nhận phản hồi "failed".
3. **Hủy đơn hàng**:
   - Orchestrator gửi lệnh `CancelOrder` đến Order Service (giao dịch bù trừ).
   - Order Service hủy đơn hàng.
4. **Kết thúc**:
   - Orchestrator cập nhật trạng thái saga thành "failed".
   - Thông báo khách hàng: "Đơn hàng bị hủy do thanh toán thất bại."

#### **Sơ đồ luồng (giả định)**:
```
Khách hàng → Orchestrator → Order Service (CreateOrder)
                    ↳ Payment Service (ProcessPayment)
                    ↳ Delivery Service (ScheduleDelivery)
                    ↳ (nếu lỗi) Order Service (CancelOrder)
```

---

### **So sánh luồng xử lý**

| Tiêu chí             | Choreography Saga                          | Orchestration Saga                         |
|----------------------|--------------------------------------------|--------------------------------------------|
| **Luồng thành công** | Sự kiện tuần tự: OrderCreated → PaymentProcessed → DeliveryScheduled | Orchestrator ra lệnh: CreateOrder → ProcessPayment → ScheduleDelivery |
| **Luồng thất bại**   | Sự kiện lỗi: PaymentFailed → OrderCancelled | Orchestrator phát hiện lỗi, ra lệnh CancelOrder |
| **Quản lý trạng thái** | Phân tán, mỗi dịch vụ tự quản lý          | Tập trung tại Orchestrator                 |
| **Giao tiếp**        | Qua message broker (Kafka, RabbitMQ)       | Qua API hoặc message queue                 |

---

### **Ghi chú triển khai**
- **Choreography**:
  - Cần thiết kế sự kiện rõ ràng (tên sự kiện, cấu trúc dữ liệu).
  - Sử dụng message broker để đảm bảo tin cậy (at-least-once delivery).
  - Mỗi dịch vụ phải xử lý giao dịch bù trừ dựa trên sự kiện lỗi.
- **Orchestration**:
  - Orchestrator cần lưu trạng thái saga (ví dụ: trong Redis, MongoDB).
  - Đảm bảo Orchestrator có khả năng chịu lỗi (retry, timeout).
  - Dịch vụ chỉ cần xử lý lệnh từ Orchestrator, không cần hiểu toàn bộ luồng.

----

Dưới đây là phần giải thích chi tiết hơn về ví dụ **Orchestration Saga** trong kịch bản thương mại điện tử (Đặt hàng → Thanh toán → Giao hàng), tập trung vào cách **Order Service tạo đơn hàng, trả về trạng thái "success"**, và đặc biệt là cách **lưu trạng thái** của saga, bao gồm **lưu vào đâu** và **lưu như thế nào**.

---

### **Bối cảnh ví dụ**
- **Kịch bản**: Khách hàng đặt một đơn hàng trên website. Hệ thống cần:
  1. Tạo đơn hàng (Order Service).
  2. Xử lý thanh toán (Payment Service).
  3. Lên lịch giao hàng (Delivery Service).
- **Orchestration Saga**: Một **Orchestrator** (dịch vụ điều phối) quản lý toàn bộ luồng, gửi lệnh (commands) đến các dịch vụ liên quan và theo dõi trạng thái của saga.
- **Trọng tâm**: Order Service tạo đơn hàng, trả về trạng thái "success", và cách Orchestrator lưu trữ trạng thái saga.

---

### **Luồng xử lý chi tiết (Orchestration Saga)**

#### **1. Tổng quan vai trò Orchestrator**
- **Orchestrator** là một dịch vụ trung tâm chịu trách nhiệm:
  - Ra lệnh cho các dịch vụ (Order, Payment, Delivery) thực hiện các giao dịch cục bộ.
  - Lưu trạng thái của saga để biết tiến trình (ví dụ: "đang tạo đơn", "thanh toán hoàn tất").
  - Xử lý lỗi bằng cách ra lệnh thực hiện giao dịch bù trừ (compensating transactions).
- Orchestrator giao tiếp với các dịch vụ qua **REST API**, **gRPC**, hoặc **message queue** (như RabbitMQ, Kafka).

#### **2. Order Service tạo đơn hàng và trả về trạng thái "success"**
- **Bước thực thi**:
  1. Orchestrator nhận yêu cầu đặt hàng từ khách hàng (ví dụ: qua API `/create-order`).
  2. Orchestrator gửi lệnh `CreateOrder` đến Order Service (qua HTTP POST hoặc message queue).
  3. Order Service:
     - Nhận lệnh, kiểm tra dữ liệu đơn hàng (ví dụ: sản phẩm, số lượng, thông tin khách hàng).
     - Lưu đơn hàng vào cơ sở dữ liệu của Order Service (thường là cơ sở dữ liệu quan hệ như MySQL hoặc NoSQL như MongoDB).
     - Trả về phản hồi `{ status: "success", orderId: "12345" }` cho Orchestrator.
  4. Orchestrator cập nhật trạng thái saga (chi tiết ở phần sau).

- **Ví dụ giả mã (Node.js, REST API)**:
  ```javascript
  // Orchestrator gửi lệnh
  async function startSaga(orderData) {
      const response = await fetch('http://order-service/create-order', {
          method: 'POST',
          body: JSON.stringify(orderData),
      });
      const result = await response.json();
      if (result.status === 'success') {
          // Cập nhật trạng thái saga
          await updateSagaState({ sagaId: orderData.sagaId, step: 'order_created', orderId: result.orderId });
      } else {
          throw new Error('Order creation failed');
      }
  }

  // Order Service xử lý lệnh
  app.post('/create-order', async (req, res) => {
      const order = req.body;
      try {
          const orderId = await db.saveOrder(order); // Lưu vào MongoDB/MySQL
          res.json({ status: 'success', orderId });
      } catch (error) {
          res.json({ status: 'failed', error });
      }
  });
  ```

#### **3. Lưu trạng thái saga: Lưu vào đâu và lưu như thế nào?**

##### **Lưu vào đâu?**
Trạng thái saga được lưu trữ bởi **Orchestrator** trong một **cơ sở dữ liệu** hoặc **bộ nhớ tạm** để theo dõi tiến trình và phục hồi khi có lỗi. Các tùy chọn phổ biến:
1. **Cơ sở dữ liệu quan hệ (Relational Database)**:
   - Ví dụ: MySQL, PostgreSQL.
   - Lưu trữ bảng `SagaState` với các cột như `sagaId`, `step`, `status`, `orderId`, `data`.
   - Ưu điểm: Đáng tin cậy, hỗ trợ giao dịch, dễ tra cứu.
2. **Cơ sở dữ liệu NoSQL**:
   - Ví dụ: MongoDB, DynamoDB.
   - Lưu dưới dạng tài liệu JSON với các trường như `{ sagaId, currentStep, orderId, payload }`.
   - Ưu điểm: Linh hoạt, dễ mở rộng.
3. **Bộ nhớ tạm (Cache)**:
   - Ví dụ: Redis, Memcached.
   - Lưu trạng thái tạm thời, phù hợp với saga ngắn hạn.
   - Ưu điểm: Nhanh, nhưng cần cơ chế đồng bộ hóa với cơ sở dữ liệu chính để tránh mất dữ liệu.
4. **Message Broker (trong một số trường hợp)**:
   - Ví dụ: Kafka, RabbitMQ.
   - Lưu trạng thái dưới dạng nhật ký sự kiện (event log).
   - Ưu điểm: Phù hợp với hệ thống event-sourcing, nhưng phức tạp hơn.

##### **Lưu như thế nào?**
- **Cấu trúc dữ liệu lưu trữ**:
  - Mỗi saga có một định danh duy nhất (`sagaId`) để theo dõi.
  - Trạng thái bao gồm:
    - `sagaId`: Mã định danh saga (ví dụ: UUID).
    - `currentStep`: Bước hiện tại (ví dụ: "order_created", "payment_processed").
    - `status`: Trạng thái tổng thể (ví dụ: "in_progress", "completed", "failed").
    - `data`: Dữ liệu liên quan (ví dụ: `orderId`, chi tiết đơn hàng).
    - `compensatingActions`: Danh sách các hành động bù trừ cần thực hiện nếu lỗi.
- **Quy trình lưu trữ**:
  1. Trước khi gửi lệnh, Orchestrator lưu trạng thái ban đầu (ví dụ: `{ sagaId: "123", step: "init", status: "in_progress" }`).
  2. Sau khi nhận phản hồi từ Order Service (`status: "success"`), cập nhật trạng thái (ví dụ: `{ step: "order_created", orderId: "12345" }`).
  3. Tiếp tục cập nhật trạng thái sau mỗi bước (thanh toán, giao hàng).
  4. Nếu lỗi, lưu trạng thái lỗi và danh sách hành động bù trừ.

- **Ví dụ cấu trúc bảng (MySQL)**:
  ```sql
  CREATE TABLE SagaState (
      sagaId VARCHAR(36) PRIMARY KEY,
      currentStep VARCHAR(50),
      status VARCHAR(20),
      orderId VARCHAR(36),
      data JSON,
      createdAt TIMESTAMP,
      updatedAt TIMESTAMP
  );

  -- Lưu trạng thái sau khi Order Service trả về success
  INSERT INTO SagaState (sagaId, currentStep, status, orderId, data, createdAt, updatedAt)
  VALUES ('123', 'order_created', 'in_progress', '12345', '{"customerId": "cust1"}', NOW(), NOW());
  ```

- **Ví dụ tài liệu (MongoDB)**:
  ```javascript
  {
      sagaId: "123",
      currentStep: "order_created",
      status: "in_progress",
      orderId: "12345",
      data: { customerId: "cust1", items: [{ id: "item1", quantity: 2 }] },
      createdAt: ISODate("2025-07-24T11:41:00Z"),
      updatedAt: ISODate("2025-07-24T11:41:00Z")
  }
  ```

#### **4. Trường hợp thất bại (Thanh toán thất bại)**
- **Luồng xử lý**:
  1. Orchestrator gửi `CreateOrder` → Order Service trả về `{ status: "success", orderId: "12345" }`.
  2. Orchestrator cập nhật trạng thái: `{ sagaId: "123", step: "order_created", orderId: "12345" }`.
  3. Orchestrator gửi `ProcessPayment` → Payment Service trả về `{ status: "failed", error: "insufficient_funds" }`.
  4. Orchestrator:
     - Cập nhật trạng thái: `{ step: "payment_failed", status: "failed" }`.
     - Gửi lệnh `CancelOrder` đến Order Service (giao dịch bù trừ).
     - Order Service hủy đơn hàng, trả về `{ status: "success" }`.
  5. Orchestrator cập nhật trạng thái cuối: `{ step: "order_cancelled", status: "failed" }`.
  6. Thông báo khách hàng: "Đơn hàng bị hủy do thanh toán thất bại."

- **Lưu trạng thái khi lỗi**:
  ```sql
  UPDATE SagaState
  SET currentStep = 'payment_failed', status = 'failed', updatedAt = NOW()
  WHERE sagaId = '123';

  UPDATE SagaState
  SET currentStep = 'order_cancelled', status = 'failed', updatedAt = NOW()
  WHERE sagaId = '123';
  ```

#### **5. Một số lưu ý triển khai**
- **Tính chịu lỗi**:
  - Orchestrator cần cơ chế **retry** (thử lại) nếu một dịch vụ tạm thời không phản hồi.
  - Sử dụng **timeout** để tránh chờ vô hạn.
- **Tính nhất quán**:
  - Dùng cơ sở dữ liệu giao dịch (transactional database) để đảm bảo trạng thái saga được cập nhật chính xác.
  - Nếu dùng Redis, cần đồng bộ với cơ sở dữ liệu chính để tránh mất dữ liệu.
- **Hiệu suất**:
  - Lưu trạng thái trong Redis cho tốc độ cao, nhưng định kỳ sao lưu vào MySQL/MongoDB.
  - Sử dụng message queue (Kafka, RabbitMQ) để gửi lệnh bất đồng bộ, giảm tải cho Orchestrator.
- **Gỡ lỗi**:
  - Lưu nhật ký (logs) cho mỗi bước saga để dễ dàng tra cứu khi có lỗi.
  - Dùng công cụ như Jaeger hoặc Zipkin để theo dõi luồng.

---

### **Tóm tắt**
- **Order Service tạo đơn hàng**:
  - Nhận lệnh `CreateOrder` từ Orchestrator, lưu đơn hàng vào cơ sở dữ liệu (MySQL/MongoDB), trả về `{ status: "success", orderId }`.
- **Lưu trạng thái saga**:
  - **Ở đâu**: MySQL, MongoDB, Redis, hoặc Kafka (tùy thiết kế).
  - **Như thế nào**: Lưu dưới dạng bảng/tài liệu với các trường `sagaId`, `currentStep`, `status`, `orderId`, `data`.
  - Cập nhật trạng thái sau mỗi bước, bao gồm cả khi lỗi để thực hiện giao dịch bù trừ.
- **Ví dụ thực tế**:
  - Trạng thái saga được lưu trong MongoDB với cấu trúc JSON, cập nhật sau mỗi lệnh (`order_created`, `payment_processed`, v.v.).
  - Nếu thanh toán thất bại, Orchestrator ra lệnh `CancelOrder` và cập nhật trạng thái thành `order_cancelled`.
