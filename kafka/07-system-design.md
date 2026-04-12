# Use Case Thực Tế, System Design, Câu Hỏi Hóc Búa và Cheat Sheet

> Tổng hợp từ tài liệu chính thức, Confluent engineering blog, Strimzi, và các nguồn phỏng vấn senior.

---

## 7. Use Case Thực Tế và System Design

### 7.1 Kafka như Event Bus (Pub/Sub ở Scale)

**Pattern**: Nhiều producer ghi domain event. Nhiều consumer group độc lập mỗi cái xử lý stream cho mục đích khác nhau.

**Ví dụ**: E-commerce order service:
- Producer: Order service emit event `OrderPlaced`
- Consumer Group 1: Inventory service đặt trước stock
- Consumer Group 2: Notification service gửi email/SMS
- Consumer Group 3: Analytics pipeline ghi vào data warehouse

Key: Mỗi group duy trì offset riêng, xử lý độc lập, theo tốc độ của mình.

---

### 7.2 Event Sourcing và CQRS

**Pattern**: Write của application được mô hình hóa là immutable event. Event log LÀ source of truth. Materialized view được tạo bằng cách replay event.

**Với Kafka:**
- Write side: produce event vào compacted topic với `transactional.id` (atomicity)
- Read side: Kafka Streams hoặc consumer build projection vào database/cache
- Replay: Consumer group với `auto.offset.reset=earliest` replay toàn bộ event log
- CQRS: Read model riêng biệt được build bởi consumer group riêng biệt

**Bẫy**: Retention mặc định của Kafka (7 ngày) giới hạn khoảng replay. Cho event sourcing thực sự với lịch sử vô hạn, config `log.retention.ms = -1` (không giới hạn) trên topic, hoặc archive sang S3/HDFS và reload qua Kafka Connect.

---

### 7.3 Change Data Capture (CDC)

**Pattern**: Dùng Debezium (Kafka Connect source) để capture mỗi thay đổi database row và push vào Kafka topic.

```
PostgreSQL WAL → Debezium → Kafka Topic (một per table) → Consumer Services
```

Lợi ích:
- Không polling; propagation gần real-time
- Tách microservice khỏi coupling trực tiếp với DB
- Audit trail đầy đủ của mọi thay đổi row

**System design gotcha**: CDC topic thường nên dùng log compaction per table topic để giữ state row mới nhất per primary key — mirror semantics của source table.

---

### 7.4 Saga Pattern / Distributed Transaction

**Pattern**: Kafka điều phối distributed transaction nhiều bước. Mỗi bước produce event success/failure; một saga orchestrator service subscribe và drive workflow.

**Ví dụ**: Payment processing saga:
1. `OrderCreated` → Đặt trước inventory
2. `InventoryReserved` → Charge payment
3. `PaymentCharged` → Confirm order
4. `PaymentFailed` → Bù trừ inventory reservation

Với Kafka exactly-once semantics và transactional API, state transition có thể được thực hiện đáng tin cậy.

---

### 7.5 System Design Interview: Real-Time Analytics Pipeline

**Bài toán**: Thiết kế hệ thống ingestion 1 triệu event/giây từ IoT device, xử lý real-time cho anomaly detection, và lưu kết quả để dashboard.

**Giải pháp:**
```
IoT Devices → Kafka (raw-events topic, 100 partitions)
           → Kafka Streams (anomaly detection, windowed aggregation)
           → Kafka (alerts topic) → Alert Service
           → Kafka (aggregated-metrics topic) → Druid/ClickHouse (dashboard)
           → S3 (long-term storage qua Kafka Connect S3 Sink)
```

Quyết định thiết kế chính:
- **Partition key**: `device_id` — đảm bảo tất cả event từ một device đến cùng partition để xử lý có thứ tự
- **Retention**: 3 ngày trên raw topic (replay window), vô hạn trên aggregated
- **Compression**: `lz4` trên raw (tốc độ), `zstd` trên aggregated (tỉ lệ)
- **Consumer scaling**: 100 partition = tối đa 100 consumer song song per group
- **EOS**: Dùng `exactly_once_v2` trong Kafka Streams để ngăn double-counting trong metric

---

## 8. Câu Hỏi Hóc Búa và Những Cái Bẫy

### Bẫy "Mất Message"

**H: Có thể mất message với `acks=all` và `replication.factor=3` không?**

Có, nếu `unclean.leader.election.enable = true`. Kịch bản:
1. Cả 3 replica trong ISR, message được ACK bởi tất cả
2. 2 follower crash đồng thời — chỉ còn leader
3. Leader crash; không còn thành viên ISR
4. `unclean.leader.election.enable = true` cho phép broker đầu tiên quay lại (dù bị behind) trở thành leader
5. Broker đó có thể không có message mới nhất → **mất data**

Phòng ngừa: Giữ `unclean.leader.election.enable = false` trong production.

---

### Khoảng Hở "Xử Lý Trùng Lặp"

**H: Bạn dùng `acks=all`, `enable.idempotence=true`, và commit offset thủ công sau xử lý. Vẫn có thể nhận duplicate không?**

Có. Kịch bản:
1. Consumer fetch record và xử lý chúng
2. Consumer ghi kết quả vào database
3. Consumer gọi `commitSync()` — fail (network issue)
4. Consumer restart, fetch lại cùng record
5. Record được xử lý lại và ghi vào database lần nữa

Giải pháp: Dùng transactional exactly-once semantics (giữ offset commit trong cùng Kafka transaction), hoặc làm idempotent downstream write (upsert bằng business key).

---

### Partition Key Hotspot

**H: Tại sao bạn thấy consumer lag chỉ ở một số partition trong khi các cái khác idle?**

Key partition bị skewed. Nếu tất cả message có cùng key (hoặc key low-cardinality), chúng đều hash vào cùng partition. Một consumer xử lý toàn bộ load; các cái còn lại idle.

Giải pháp:
- Dùng key cardinality cao hơn
- Dùng composite key
- Thêm random suffix để phân tán load (nhưng mất ordering guarantee cho key gốc)
- Dùng custom partitioner

---

### Bẫy Ordering `max.in.flight.requests`

**H: Có thể có cả `retries > 0` lẫn strict ordering mà không cần idempotence không?**

Không. Với `retries > 0` và `max.in.flight.requests.per.connection > 1`, retry của batch fail có thể đến broker sau khi batch sau đó đã được ghi. Điều này reorder message.

**Cấu hình đúng:**
- Ordering với retry (legacy): `max.in.flight.requests.per.connection = 1`
- Ordering với retry (hiện đại): `enable.idempotence = true` (cho phép tối đa 5 in-flight)

---

### Consumer Rebalance Performance Cliff

**H: Bạn có 50 consumer trong group với eager rebalancing. Một consumer được thêm vào. Điều gì xảy ra?**

Tất cả 50 consumer dừng xử lý đồng thời (stop-the-world). Tất cả revoke partition, rejoin và chờ reassignment. Trong thời gian này, **không có message nào được consume** trong toàn bộ group. Đây là lý do cooperative rebalancing cần thiết cho consumer group lớn.

---

### Trade-off Partition Count và Latency

**H: Tăng partition count có luôn cải thiện performance không?**

Không. Trade-off:
- Nhiều partition hơn → consumer song song hơn → throughput cao hơn
- Nhiều partition hơn → nhiều open file handle hơn per broker
- Nhiều partition hơn → end-to-end latency CÓ THỂ TĂNG vì replication là per-partition và nhiều partition hơn nghĩa là nhiều replication operation đồng thời hơn
- Nhiều partition hơn → controller failover chậm hơn (nhiều metadata cần xử lý hơn)
- Nhiều partition hơn → Kafka Streams app có thể cần nhiều memory hơn (một state store per partition)

Rule: Scale partition count để match **yêu cầu consumer parallelism peak**, không hơn.

---

### Schema Compatibility Assumption

**H: Bạn đặt compatibility `BACKWARD`. Consumer của bạn ở schema version 3. Producer upgrade lên schema version 5. Tất cả consumer có an toàn không?**

Chưa chắc. `BACKWARD` (non-transitive) đảm bảo:
- v5 compatible với v4 (version mới nhất trước đó)
- v4 compatible với v3 khi nó được đăng ký

Nhưng **không đảm bảo** v5 có thể đọc được trực tiếp bởi consumer v3. Cho điều đó, cần `BACKWARD_TRANSITIVE`.

---

### Bẫy "Kafka Không Phải Database"

**H: Một team muốn dùng Kafka compacted topic như persistent lookup table được query theo key lúc runtime. Có phù hợp không?**

Một phần. Compacted Kafka topic tốt để **bootstrap state** (ví dụ: Kafka Streams state store, consumer initialization). Nhưng random key lookup lúc query time cần consume toàn bộ topic và build in-memory hoặc RocksDB-backed map. Kafka không có server-side key lookup API.

Cho lookup table đúng nghĩa, consume compacted topic vào Redis, DynamoDB, hoặc Kafka Streams `GlobalKTable` (có thể query qua interactive queries).

---

### EOS Qua Hệ Thống Ngoài

**H: Kafka Streams app của bạn consume từ Kafka, gọi external REST API, và produce kết quả về Kafka. `exactly_once_v2` có cho bạn exactly-once end-to-end không?**

Không. `exactly_once_v2` đảm bảo exactly-once cho chu trình read-process-write **hoàn toàn trong Kafka**. REST call ngoài nằm ngoài transaction boundary của Kafka. Nếu Streams task fail sau REST call nhưng trước khi commit Kafka transaction, REST call sẽ xảy ra lại khi retry.

Cho external side effect: hoặc làm external API idempotent (truyền correlation ID từ Kafka), dùng outbox pattern, hoặc chấp nhận at-least-once cho external call.

---

## Cheat Sheet Kafka Metrics

| Metric | Nguồn | Ý nghĩa |
|--------|-------|---------|
| `kafka.consumer:type=consumer-fetch-manager-metrics,records-lag-max` | Consumer | Lag tối đa qua tất cả partition — health indicator chính |
| `kafka.server:type=ReplicaManager,name=IsrShrinksPerSec` | Broker | Replica rơi khỏi ISR — tín hiệu bất ổn |
| `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec` | Broker | Tỉ lệ ingestion |
| `kafka.controller:type=KafkaController,name=ActiveControllerCount` | Broker | Phải đúng là 1 trong cluster |
| `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions` | Broker | Partition có ít thành viên ISR hơn replication factor |
| `kafka.network:type=RequestMetrics,name=RequestsPerSec` | Broker | Tỉ lệ request theo API key |
| Producer `record-error-rate` | Producer | Send thất bại |
| Producer `request-latency-avg` | Producer | End-to-end producer latency |

---

## Quick Reference: Config Matrix cho Các Kịch Bản Thường Gặp

| Kịch bản | `acks` | `enable.idempotence` | `transactional.id` | `isolation.level` |
|----------|--------|---------------------|--------------------|------------------|
| Throughput tối đa, chấp nhận mất | `0` | false | — | — |
| Throughput cao, một ít durability | `1` | false | — | — |
| Durable, at-least-once | `all` | false | — | `read_uncommitted` |
| Không duplicate (single session) | `all` | true | — | `read_uncommitted` |
| Exactly-once (consume-transform-produce) | `all` | true | bắt buộc | `read_committed` |
| Kafka Streams EOS | Đặt bởi `processing.guarantee=exactly_once_v2` | auto | auto | auto |

---

*Nguồn tham khảo: [Confluent EOS Blog](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/), [Strimzi Transactions Deep Dive](https://strimzi.io/blog/2023/05/03/kafka-transactions/), [Hello Interview Kafka Deep Dive](https://www.hellointerview.com/learn/system-design/deep-dives/kafka), [Confluent KRaft Blog](https://www.confluent.io/blog/why-replace-zookeeper-with-kafka-raft-the-log-of-all-logs/), [AutoMQ EOS Implementation](https://www.automq.com/blog/kafka-exactly-once-semantics-implementation-idempotence-and-transactional-messages), [Conduktor Consumer Groups](https://www.conduktor.io/glossary/kafka-consumer-groups-explained), [Confluent Schema Evolution](https://docs.confluent.io/platform/current/schema-registry/fundamentals/schema-evolution.html), [Redpanda Performance Tuning](https://www.redpanda.com/guides/kafka-performance-kafka-performance-tuning), [Kafka Storage Internals](https://kafka.apache.org/documentation/#log), [Confluent MirrorMaker 2](https://docs.confluent.io/platform/current/multi-dc-deployments/mirrormaker.html), [Apache Pulsar vs Kafka](https://pulsar.apache.org/docs/en/compare/), [Kafka Security](https://kafka.apache.org/documentation/#security)*
