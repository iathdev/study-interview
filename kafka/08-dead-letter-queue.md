# Dead Letter Queue (DLQ)

> Tổng hợp từ tài liệu chính thức, Confluent engineering blog, Spring Kafka docs, và các nguồn phỏng vấn senior.

---

## 1. DLQ là gì? Tại sao cần?

**DLQ (Dead Letter Queue)** là một queue/topic đặc biệt chứa các message mà consumer **không thể xử lý thành công** sau nhiều lần retry.

**Vấn đề nếu không có DLQ:**

```
Consumer ──fail──▶ retry 1 ──fail──▶ retry 2 ──fail──▶ retry 3 ──fail──▶ ???
                                                                           │
                               Stuck ◀───────────────────────────────────┘
                               Partition bị block, không xử lý message tiếp theo
```

Kafka không tự động skip message lỗi — consumer phải commit offset để advance. Nếu không xử lý được mà không commit, consumer sẽ loop mãi một message (gọi là **poison pill**), block toàn bộ partition.

**Với DLQ:**
```
Consumer ──fail──▶ retry ──fail──▶ publish to DLQ ──▶ commit offset
                                                              │
                                        Tiếp tục xử lý message tiếp theo
                                        Message lỗi được lưu lại để review/reprocess
```

---

## 2. Kafka không có DLQ native

Khác với **RabbitMQ** (có `x-dead-letter-exchange` built-in), **Kafka không có cơ chế DLQ native**. DLQ trong Kafka được implement theo convention: tạo thêm topic riêng cho message lỗi.

**Hai cách implement phổ biến:**

| Cách | Tool | Phù hợp |
|------|------|---------|
| Retry Topics + DLT | Spring Kafka `@RetryableTopic` | Spring Boot app |
| Custom retry logic | Tự implement | Non-Spring, kiểm soát hoàn toàn |

---

## 3. Retry Topics Pattern

Thay vì retry ngay lập tức (có thể gây thundering herd), pattern này dùng **nhiều retry topic** với delay tăng dần, sau đó mới đến DLT (Dead Letter Topic).

```
orders ──fail──▶ orders-retry-1 (delay 1s)
                     ──fail──▶ orders-retry-2 (delay 10s)
                                   ──fail──▶ orders-retry-3 (delay 60s)
                                                 ──fail──▶ orders-dlt
```

**Flow chi tiết:**

```
1. Consumer xử lý từ topic "orders" → fail
2. Publish message vào "orders-retry-1" → commit offset "orders"
   (message mang header: retryCount=1, originalTopic=orders, failReason=...)
3. Retry consumer đọc "orders-retry-1" sau delay → fail
4. Publish vào "orders-retry-2"
5. Retry lần 3 từ "orders-retry-2" → fail
6. Publish vào "orders-dlt" → alert team
7. Message nằm trong DLT đến khi được review và reprocess thủ công
```

---

## 4. Implement với Spring Kafka

### 4.1 `@RetryableTopic` — cách đơn giản nhất

```java
@Component
public class OrderConsumer {

    @RetryableTopic(
        attempts = "4",                          // 1 lần chính + 3 retry
        backoff = @Backoff(
            delay = 1_000,                       // retry 1: sau 1s
            multiplier = 10,                     // retry 2: 10s, retry 3: 100s
            maxDelay = 60_000                    // cap ở 60s
        ),
        dltTopicSuffix = "-dlt",                 // topic DLT: orders-dlt
        retryTopicSuffix = "-retry",             // retry topics: orders-retry-0, orders-retry-1...
        autoCreateTopics = "false",              // tạo topic thủ công, kiểm soát partition/replication
        include = {RetryableException.class},    // chỉ retry với exception này
        exclude = {NonRetryableException.class}  // không retry, đẩy thẳng vào DLT
    )
    @KafkaListener(topics = "orders")
    public void consume(OrderEvent event) {
        orderService.process(event);
    }

    @DltHandler                                  // xử lý message đến DLT
    public void handleDlt(OrderEvent event,
                          @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                          @Header(KafkaHeaders.EXCEPTION_MESSAGE) String errorMessage) {
        log.error("Message moved to DLT. topic={}, error={}, event={}", topic, errorMessage, event);
        // alert, lưu vào DB để review, gửi Slack notification...
        deadLetterService.record(event, errorMessage);
    }
}
```

### 4.2 DLQ Header — thông tin đính kèm message

Spring Kafka tự động thêm headers vào DLT message:

| Header | Nội dung |
|--------|---------|
| `kafka_dlt-original-topic` | Topic gốc (orders) |
| `kafka_dlt-original-partition` | Partition gốc |
| `kafka_dlt-original-offset` | Offset gốc |
| `kafka_dlt-original-timestamp` | Timestamp gốc |
| `kafka_dlt-exception-fqcn` | Tên exception |
| `kafka_dlt-exception-message` | Message lỗi |
| `kafka_dlt-exception-stacktrace` | Stack trace |

→ Đủ thông tin để debug và reprocess.

### 4.3 Custom DLQ — không dùng Spring annotation

```java
@KafkaListener(topics = "orders")
public void consume(ConsumerRecord<String, OrderEvent> record) {
    try {
        orderService.process(record.value());
    } catch (RetryableException e) {
        // Sẽ được retry theo cơ chế khác (ErrorHandler)
        throw e;
    } catch (Exception e) {
        // Non-retryable: đẩy thẳng vào DLQ
        publishToDlq(record, e);
    }
}

private void publishToDlq(ConsumerRecord<String, OrderEvent> original, Exception cause) {
    ProducerRecord<String, OrderEvent> dlqRecord = new ProducerRecord<>(
        "orders-dlt",
        original.partition(),       // giữ partition để trace
        original.key(),
        original.value()
    );

    // Đính kèm thông tin lỗi vào header
    dlqRecord.headers()
        .add("original-topic", original.topic().getBytes())
        .add("original-offset", longToBytes(original.offset()))
        .add("error-message", cause.getMessage().getBytes())
        .add("error-class", cause.getClass().getName().getBytes())
        .add("failed-at", Instant.now().toString().getBytes());

    kafkaTemplate.send(dlqRecord).get(); // sync để đảm bảo ghi xong trước khi commit
}
```

---

## 5. Các bài toán thực tế

### Bài toán 1: Poison Pill — message không bao giờ deserialize được

```
Scenario: Producer gửi message với schema mới (thêm required field),
          consumer cũ không deserialize được → crash liên tục

orders partition 0:
  offset 0 → OK
  offset 1 → OK
  offset 2 → DeserializationException (mãi mãi) ← STUCK ở đây
  offset 3 → chưa xử lý
  offset 4 → chưa xử lý
```

**Giải pháp**: Dùng `ErrorHandlingDeserializer` của Spring Kafka — wrap deserializer để catch exception, tạo `DeserializationException` record với payload là raw bytes → consumer vẫn nhận được record, có thể tự quyết định đẩy vào DLQ.

```java
@Bean
public ConsumerFactory<String, OrderEvent> consumerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
    props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);
    // ...
    return new DefaultKafkaConsumerFactory<>(props);
}

@KafkaListener(topics = "orders")
public void consume(OrderEvent event,
                    @Header(value = SerializationUtils.DESERIALIZATION_EXCEPTION_HEADER,
                            required = false) byte[] deserException) {
    if (deserException != null) {
        // Không deserialize được → đẩy thẳng vào DLQ, không retry
        deadLetterService.record(event, "Deserialization failed");
        return;
    }
    orderService.process(event);
}
```

---

### Bài toán 2: Downstream service tạm thời down

```
Scenario: Payment Service down 5 phút, Order Consumer cần gọi Payment Service
          → mọi message trong 5 phút này đều fail
```

**Sai**: Không có retry → tất cả 5 phút message bị mất vào DLQ.

**Đúng**: Exponential backoff retry + DLQ chỉ sau nhiều lần fail.

```java
@RetryableTopic(
    attempts = "5",
    backoff = @Backoff(delay = 2_000, multiplier = 2, maxDelay = 30_000)
    // Retry schedule: 2s → 4s → 8s → 16s → DLT
    // Tổng thời gian chờ: ~30s — đủ để Payment recover nhanh
)
@KafkaListener(topics = "orders")
public void consume(OrderEvent event) {
    paymentService.charge(event); // có thể throw RetryableException nếu HTTP 5xx
}
```

**Phân biệt retryable vs non-retryable:**

| Exception | Retryable? | Lý do |
|-----------|------------|-------|
| `HttpServerErrorException` (5xx) | Có | Service tạm thời lỗi |
| `HttpClientErrorException` (4xx) | Không | Bad request — retry cũng vô ích |
| `JsonParseException` | Không | Data lỗi, retry không fix được |
| `OptimisticLockException` | Có | Conflict tạm thời, retry giải quyết được |
| `DataIntegrityViolationException` | Không | Duplicate, business rule violation |

---

### Bài toán 3: DLQ tích lũy quá nhiều — phải reprocess

```
Scenario: Bug trong OrderConsumer gây 50,000 message vào DLT.
          Bug đã fix, cần replay toàn bộ DLT.
```

**Reprocess từ DLT:**

```java
@Service
public class DlqReprocessor {

    @KafkaListener(topics = "orders-dlt", groupId = "orders-dlt-reprocessor")
    public void reprocess(ConsumerRecord<String, OrderEvent> record) {
        try {
            // Publish lại vào topic gốc để consumer chính xử lý lại
            kafkaTemplate.send("orders", record.key(), record.value());
            log.info("Reprocessed DLT message. offset={}", record.offset());
        } catch (Exception e) {
            log.error("Failed to reprocess DLT message. offset={}", record.offset(), e);
            // Có thể publish vào DLT thứ 2 (orders-dlt-v2) hoặc alert manual
        }
    }
}
```

**Lưu ý khi reprocess:**
- Đảm bảo **idempotency** ở consumer chính — message có thể đến 2 lần (từ DLT và original nếu có delay)
- Reprocess từng batch nhỏ trước, kiểm tra kết quả rồi mới toàn bộ
- Cần idempotency key (thường là `eventId` trong message) để tránh double-process

---

### Bài toán 4: Message trong DLQ cần xử lý theo nghiệp vụ khác nhau

```
Scenario: DLT của payment-events chứa nhiều loại lỗi:
  - Lỗi kỹ thuật (timeout) → nên reprocess tự động
  - Lỗi nghiệp vụ (insufficient balance) → cần notify user, không reprocess
  - Lỗi dữ liệu (bad format) → cần dev fix trước khi reprocess
```

**Giải pháp: Phân loại trong DLT handler**

```java
@DltHandler
public void handleDlt(PaymentEvent event,
                      @Header(KafkaHeaders.EXCEPTION_FQCN) String exceptionClass,
                      @Header(KafkaHeaders.EXCEPTION_MESSAGE) String errorMessage) {

    DlqCategory category = classify(exceptionClass, errorMessage);

    switch (category) {
        case TECHNICAL_ERROR -> {
            // Lên lịch reprocess tự động sau 30 phút
            scheduler.schedule(() -> kafkaTemplate.send("payment-events", event), 30, MINUTES);
        }
        case BUSINESS_ERROR -> {
            // Notify user, không reprocess
            notificationService.notifyUser(event.getUserId(), "Payment failed: " + errorMessage);
            auditRepo.save(AuditRecord.businessFailure(event, errorMessage));
        }
        case DATA_ERROR -> {
            // Alert dev team, lưu vào DB để fix thủ công
            alertService.alertDev("Data error in DLT", event, errorMessage);
            deadLetterRepo.save(DeadLetterRecord.requiresManualFix(event, errorMessage));
        }
    }
}
```

---

## 6. Cơ chế Reprocess / Resend từ DLT

Message vào DLT không có nghĩa là mất — có thể xử lý lại sau khi fix bug hoặc recover service. Có 3 cách:

---

### Cách 1: Consumer tự động resend về topic gốc

```java
@KafkaListener(topics = "orders-dlt", groupId = "orders-dlt-reprocessor")
public void reprocess(ConsumerRecord<String, OrderEvent> record) {
    kafkaTemplate.send("orders", record.key(), record.value());
}
```

Dùng khi: lỗi kỹ thuật đã được fix, muốn replay toàn bộ DLT tự động.

**Lưu ý**: Consumer gốc (`orders`) phải **idempotent** — message có thể đến 2 lần nếu có delay giữa DLT và topic gốc.

---

### Cách 2: Schedule resend có delay

```java
@DltHandler
public void handleDlt(OrderEvent event,
                      @Header(KafkaHeaders.EXCEPTION_FQCN) String exceptionClass) {
    if (isTechnicalError(exceptionClass)) {
        // Service tạm thời down → resend sau 30 phút
        scheduler.schedule(
            () -> kafkaTemplate.send("orders", event.getId(), event),
            30, TimeUnit.MINUTES
        );
    } else {
        // Lỗi business/data → lưu DB chờ fix thủ công
        deadLetterRepo.save(DeadLetterRecord.of(event, exceptionClass));
    }
}
```

Dùng khi: biết chắc downstream service sẽ recover sau một khoảng thời gian.

---

### Cách 3: Resend thủ công qua CLI (khi DLT tích lũy nhiều)

```bash
# Đọc toàn bộ DLT và republish vào topic gốc
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic orders-dlt \
  --from-beginning \
  --property print.key=true \
| kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --property parse.key=true
```

Dùng khi: bug đã fix, cần replay hàng loạt message từ DLT một lần.

---

### Quy trình reprocess an toàn

```
1. Fix bug hoặc đảm bảo downstream đã recover
2. Replay một vài message nhỏ trước → kiểm tra kết quả
3. Nếu OK → replay toàn bộ
4. Monitor consumer lag và error rate trong lúc replay
5. Sau khi xong → kiểm tra DLT còn message không
```

---

### Khi nào KHÔNG nên reprocess

| Loại lỗi | Reprocess? | Lý do |
|----------|-----------|-------|
| Service tạm thời down | Có | Retry sẽ thành công sau khi recover |
| Bug logic đã fix | Có | Kết quả sẽ khác sau khi fix |
| Bad data (format sai) | Không | Retry cũng fail, phải fix data trước |
| Business rule violation | Không | Vd: insufficient balance — cần notify user, không retry |
| Duplicate (đã xử lý rồi) | Không | Idempotency check sẽ skip, nhưng tốt nhất đừng resend |

---

## 7. Best Practices

### 6.1 Thiết kế DLQ topic

```
# Convention đặt tên rõ ràng
{service}-{original-topic}-dlt        # orders-service-payments-dlt
{original-topic}.DLT                  # payments.DLT (Confluent convention)

# Cùng số partition với topic gốc
# → Đảm bảo message của cùng một key đến cùng partition DLT
# → Dễ trace và reprocess theo partition

# Retention dài hơn topic gốc
# Topic gốc: 7 ngày
# DLT: 30 ngày (đủ thời gian review và fix)
```

### 6.2 Luôn enrich message với metadata lỗi

```java
// Tối thiểu phải có:
headers.add("error.message",     cause.getMessage())
headers.add("error.class",       cause.getClass().getName())
headers.add("original.topic",    record.topic())
headers.add("original.partition",String.valueOf(record.partition()))
headers.add("original.offset",   String.valueOf(record.offset()))
headers.add("failed.at",         Instant.now().toString())
headers.add("retry.count",       String.valueOf(retryCount))
headers.add("service.name",      "order-service")

// Nếu có distributed tracing:
headers.add("trace.id",          MDC.get("traceId"))
```

### 6.3 Alert ngay khi có message vào DLT

```java
@DltHandler
public void handleDlt(OrderEvent event, ...) {
    // Log đầy đủ
    log.error("DEAD_LETTER | topic={} | partition={} | offset={} | error={}",
              originalTopic, partition, offset, errorMessage);

    // Metric counter → alert khi vượt ngưỡng
    meterRegistry.counter("kafka.dlt.messages",
        "topic", originalTopic,
        "exception", exceptionClass
    ).increment();

    // Alert nếu DLT rate đột biến
    if (dlqRateMonitor.isAnomalous()) {
        alertService.critical("DLT spike detected on " + originalTopic);
    }
}
```

### 6.4 Không retry với non-retryable exception

```java
// SAI: retry tất cả exception → tốn thời gian, message vẫn vào DLT
@RetryableTopic(attempts = "5")

// ĐÚNG: chỉ retry exception mà retry có thể fix được
@RetryableTopic(
    attempts = "5",
    include = {
        ConnectException.class,          // network issue
        HttpServerErrorException.class,  // 5xx downstream
        OptimisticLockException.class    // DB conflict
    },
    exclude = {
        JsonParseException.class,        // data lỗi → retry vô ích
        IllegalArgumentException.class,  // bad input → retry vô ích
        BusinessRuleViolationException.class // business logic → retry vô ích
    }
)
```

### 6.5 Idempotency là bắt buộc khi có DLQ

Khi reprocess từ DLT, message có thể được deliver nhiều lần. Consumer **bắt buộc** phải idempotent:

```java
@KafkaListener(topics = "orders")
public void consume(OrderEvent event) {
    // Kiểm tra đã xử lý chưa (bằng eventId)
    if (processedEventRepo.exists(event.getEventId())) {
        log.info("Duplicate event, skipping. eventId={}", event.getEventId());
        return;
    }

    // Xử lý...
    orderService.process(event);

    // Đánh dấu đã xử lý — trong cùng transaction nếu có thể
    processedEventRepo.markProcessed(event.getEventId());
}
```

### 6.6 DLT không được là DLT của chính nó

```java
// Tránh: DLT consumer lại fail → push vào DLT → vòng lặp vô tận
@KafkaListener(topics = "orders-dlt")
@RetryableTopic(...)  // KHÔNG dùng @RetryableTopic cho DLT consumer
public void handleDlt(...) { ... }

// ĐÚNG: DLT consumer fail → chỉ log, alert, không push vào topic khác
@KafkaListener(topics = "orders-dlt")
public void handleDlt(OrderEvent event) {
    try {
        deadLetterRepo.save(...);
    } catch (Exception e) {
        log.error("Failed to save DLT message to DB. Manual intervention required.", e);
        // Alert, không ném exception → không retry, không loop
    }
}
```

---

## 7. DLQ trong RabbitMQ (so sánh)

RabbitMQ có DLQ native qua **Dead Letter Exchange (DLX)**:

```java
// Khai báo queue với DLX
@Bean
public Queue orderQueue() {
    return QueueBuilder.durable("orders")
        .withArgument("x-dead-letter-exchange", "orders.dlx")  // DLX
        .withArgument("x-dead-letter-routing-key", "orders.dlt")
        .withArgument("x-message-ttl", 30_000)   // message expire → vào DLT
        .build();
}

@Bean
public Queue deadLetterQueue() {
    return QueueBuilder.durable("orders.dlt").build();
}
```

**Message vào DLQ khi:**
1. Consumer `nack` (hoặc `reject`) với `requeue=false`
2. Message hết TTL
3. Queue full (khi đặt `x-max-length`)

| | Kafka DLQ | RabbitMQ DLQ |
|--|-----------|--------------|
| Native support | Không (phải tự implement) | Có (DLX) |
| Retry delay | Retry topic với delay | Plugin `rabbitmq-delayed-message` |
| Reprocess | Replay từ offset | Re-publish vào main queue |
| Visibility | Consumer phải subscribe DLT | Management UI built-in |
| Ordering | Giữ partition key | Không đảm bảo ordering |

---

## 8. Monitoring DLQ

### Metrics cần theo dõi

```
# Tốc độ message vào DLT — alert khi tăng đột biến
kafka.consumer.dlt.messages.total{topic="orders", exception="*"}

# DLT lag — số message chưa được xử lý trong DLT
kafka.consumer.fetch-manager.records-lag{topic="orders-dlt"}

# Retry rate — bao nhiêu % message phải retry
kafka.consumer.retry.rate{topic="orders"}
```

### Dashboard cần có

```
┌─────────────────────────────────────────┐
│  DLT Rate (msg/min)    [alert: > 10]    │
│  ████░░░░░░░░░░░░░░░░  3/min            │
├─────────────────────────────────────────┤
│  DLT Lag (unprocessed) [alert: > 100]   │
│  ████████░░░░░░░░░░░░  45 messages      │
├─────────────────────────────────────────┤
│  Top exception classes in DLT           │
│  ConnectException       → 60%           │
│  JsonParseException     → 30%           │
│  BusinessRuleViolation  → 10%           │
└─────────────────────────────────────────┘
```

---

## 9. Interview Q&A

**Q: DLQ là gì và tại sao cần trong Kafka?**

> DLQ (Dead Letter Queue) chứa message mà consumer không xử lý được sau nhiều lần retry. Kafka cần DLQ vì consumer phải commit offset để advance — nếu một message lỗi không được xử lý, partition bị block (poison pill). DLQ giải quyết: commit offset của message lỗi, lưu nó vào topic riêng để review/reprocess sau, không block luồng xử lý chính.

---

**Q: Kafka có DLQ native không? Nếu không, implement thế nào?**

> Kafka **không có DLQ native** (khác với RabbitMQ). Implement bằng cách tạo topic riêng (convention: `{topic}-dlt`). Spring Kafka hỗ trợ qua `@RetryableTopic` — tự động tạo retry topics với delay tăng dần, sau đó push vào DLT. Nếu không dùng Spring, phải tự publish vào DLT topic trong catch block và commit offset.

---

**Q: Retry topic và DLQ khác nhau thế nào?**

> **Retry topic**: message tạm thời bị fail, có thể thành công nếu thử lại — dùng nhiều retry topic với delay tăng dần (exponential backoff). **DLQ**: sau khi hết retry mà vẫn fail — message coi là "dead", cần can thiệp thủ công. Retry topic là bước trung gian trước DLQ.

---

**Q: Khi nào nên retry, khi nào nên đẩy thẳng vào DLQ?**

> **Retry** khi lỗi có thể tự phục hồi: network timeout, service 5xx, DB optimistic lock conflict.
> **DLQ ngay** khi lỗi không retry được: deserialization error (data lỗi), 4xx (bad request), business rule violation. Retry các lỗi này chỉ tốn thời gian và trì hoãn việc xử lý message tiếp theo.

---

**Q: Message trong DLQ có được xử lý tự động không?**

> Phụ thuộc vào loại lỗi. Lỗi kỹ thuật (service tạm thời down) → có thể schedule reprocess tự động sau khi service recover. Lỗi nghiệp vụ hoặc data → cần con người review, fix, rồi mới reprocess thủ công bằng cách replay từ DLT topic. Quan trọng: consumer chính phải **idempotent** vì message từ DLT có thể đến 2 lần.

---

**Q: Làm sao đảm bảo không mất message khi publish vào DLQ?**

> Publish vào DLT phải xong **trước khi** commit offset topic gốc. Dùng `kafkaTemplate.send(...).get()` (blocking) hoặc transaction để đảm bảo. Nếu publish DLT thất bại → không commit offset → message được retry sau khi consumer restart. Trade-off: có thể bị duplicate (at-least-once) nhưng không mất message.

---

**Q: Poison pill là gì? Xử lý thế nào?**

> Poison pill là message mà consumer **không bao giờ** xử lý được (ví dụ: không deserialize được do schema mismatch). Nếu không xử lý, consumer sẽ loop mãi mãi, block partition. Giải pháp: dùng `ErrorHandlingDeserializer` của Spring Kafka — wrap deserializer để catch exception, consumer nhận được record với flag lỗi, tự đẩy vào DLQ và commit offset để unblock partition.
