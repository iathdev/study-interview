# Chủ Đề Nâng Cao: Streams, Connect, Schema Registry, Compaction, MirrorMaker 2 và So Sánh

> Tổng hợp từ tài liệu chính thức, Confluent engineering blog, Strimzi, và các nguồn phỏng vấn senior.

---

## 6. Chủ Đề Nâng Cao

### 6.1 Kafka Streams

**H: Kafka Streams là gì và khác gì stream processor khác?**

Kafka Streams là một **client library** (không phải cluster riêng) để build stream processing application. Điểm khác biệt chính:
- **Không có cluster riêng**: Chạy trong JVM của application
- **Kafka-native**: Input và output là Kafka topic
- **Fault-tolerant**: State store được backup bởi changelog Kafka topic
- **Nhẹ**: Không cần quản lý Spark/Flink cluster

| Khía cạnh | Kafka Streams | Apache Flink | Spark Streaming |
|-----------|--------------|--------------|-----------------|
| Deployment | Library trong app | Cluster riêng | Cluster riêng |
| State | RocksDB + changelog topic | Managed checkpoint | In-memory + checkpoint |
| Latency | Sub-second | Sub-second | Micro-batch (giây) |
| EOS | Có (`exactly_once_v2`) | Có | Có |
| Use case | Microservice, Kafka-native | Complex CEP, quy mô lớn | Hybrid batch + streaming |

**H: Stream topology là gì?**

Là một DAG (directed acyclic graph) gồm source processor → stream processor → sink processor. Định nghĩa qua DSL (`StreamsBuilder`) hoặc Processor API. Kafka Streams song song hóa thực thi bằng cách assign stream task cho application instance — mỗi task xử lý một partition.

---

### 6.2 KStream vs KTable vs GlobalKTable

| Abstraction | Semantics | Storage | Join Requirement |
|-------------|-----------|---------|-----------------|
| `KStream` | Stream sự kiện vô hạn (insert-only) | Stateless (hoặc windowed state) | Cần co-partitioning |
| `KTable` | Changelog stream — giá trị mới nhất per key (upsert) | State store (phân vùng) | Cần co-partitioning |
| `GlobalKTable` | Bản sao đầy đủ của topic trên mỗi instance | State store (replicated toàn cục) | Không cần co-partitioning |

**H: Khi nào dùng GlobalKTable vs KTable?**

Dùng `GlobalKTable` khi:
- Table đủ nhỏ để fit trên mỗi instance
- Cần join một stream lớn với một reference table nhỏ
- Reference table ít được cập nhật

Tránh `GlobalKTable` cho dataset lớn — mỗi instance phải giữ toàn bộ table trong memory/disk.

---

### 6.3 Windowing

| Loại Window | Định nghĩa | Use Case |
|-------------|------------|---------|
| **Tumbling** | Fixed-size, không chồng lấp | Đếm sự kiện mỗi bucket 5 phút |
| **Hopping** | Fixed-size, chồng lấp | Window 5 phút, advance mỗi 1 phút |
| **Sliding** | Window xác định bởi khoảng thời gian giữa sự kiện | Sự kiện trong vòng 10s của nhau |
| **Session** | Dựa trên activity, giới hạn bởi gap | Group hoạt động user với idle gap 30s |

**Grace period**: Thời gian sau khi window đóng mà Kafka Streams vẫn chấp nhận record đến muộn. Sau grace period, record muộn bị drop. Cần thiết để xử lý out-of-order event từ distributed producer.

---

### 6.4 Kafka Connect

**H: Kafka Connect là gì và connector là gì?**

Kafka Connect là framework để di chuyển data đáng tin cậy giữa Kafka và hệ thống ngoài mà không cần viết code tùy chỉnh.

- **Source Connector**: Đọc từ hệ thống ngoài và ghi vào Kafka (ví dụ: Debezium cho CDC từ PostgreSQL)
- **Sink Connector**: Đọc từ Kafka và ghi vào hệ thống ngoài (ví dụ: JDBC sink to MySQL, S3 sink)

**H: Sự khác biệt giữa standalone và distributed Connect mode?**

| Mode | Use Case | Scalability | Fault Tolerance |
|------|----------|-------------|-----------------|
| Standalone | Development, single-node | Single process | Không có |
| Distributed | Production | Nhiều worker chia sẻ task | Worker có thể fail, task được phân phối lại |

Trong distributed mode, connector config và offset được lưu trong Kafka topic (`connect-configs`, `connect-offsets`, `connect-status`).

**H: Debezium là gì và tại sao thường dùng với Kafka?**

Debezium là CDC (Change Data Capture) source connector. Nó đọc database transaction log (MySQL binlog, PostgreSQL WAL, MongoDB oplog) và phát mỗi thay đổi row-level dưới dạng Kafka message. Đây là approach chuẩn cho:
- Pipeline database → Kafka mà không cần polling
- Triển khai outbox pattern
- Database migration zero-downtime
- Event sourcing qua CDC mà không cần sửa code application

---

### 6.5 Schema Registry

**H: Schema Registry giải quyết vấn đề gì?**

Không có Schema Registry, producer và consumer phải đồng thuận về message format ngoài băng tần. Schema Registry cung cấp:
- Lưu trữ schema tập trung (Avro, Protobuf, JSON Schema)
- Versioning schema và enforcement compatibility
- Wire format: message chứa prefix 5 byte (magic byte + 4-byte schema ID)

Consumer tra cứu schema ID từ registry để deserialize message, tách consumer và producer khỏi việc cần cùng schema version cùng lúc.

**H: Các compatibility mode là gì?**

| Mode | Đảm bảo | Thứ tự upgrade |
|------|---------|----------------|
| `BACKWARD` (mặc định) | Consumer mới đọc được data cũ | Consumer trước |
| `BACKWARD_TRANSITIVE` | Consumer mới đọc được toàn bộ data lịch sử | Consumer trước |
| `FORWARD` | Consumer cũ đọc được data mới | Producer trước |
| `FORWARD_TRANSITIVE` | Tất cả consumer cũ đọc được data mới | Producer trước |
| `FULL` | Cả backward và forward (adjacent) | Thứ tự tùy ý |
| `FULL_TRANSITIVE` | Cả backward và forward (tất cả version) | Thứ tự tùy ý |
| `NONE` | Không kiểm tra | Phối hợp thủ công |

**Quy tắc Avro BACKWARD compatibility:**
- Thêm field với default value: được phép
- Xóa field với default value: được phép
- Thêm required field (không có default): KHÔNG được phép
- Xóa required field: KHÔNG được phép (nhưng được phép dưới FORWARD)
- Đổi tên field: KHÔNG được phép (bị coi là xóa + thêm)

**Bẫy**: Mode `BACKWARD` mặc định là non-transitive — chỉ kiểm tra với **schema version mới nhất**. Schema X compatible với X-1, nhưng không nhất thiết với X-2. Dùng `BACKWARD_TRANSITIVE` nếu cần consumer trên version cũ vẫn hoạt động.

**Kafka Streams gotcha**: Kafka Streams chỉ hỗ trợ compatibility `BACKWARD`. Streams app phải được upgrade trước producer upstream vì phải đọc được cả schema input topic mới lẫn schema state store cũ.

---

### 6.6 Log Compaction

**H: Log compaction là gì và hoạt động như thế nào?**

Log compaction là retention policy thay thế (`log.cleanup.policy = compact`) giữ lại ít nhất **giá trị cuối cùng cho mỗi key unique** trong partition. Biến topic thành key-value store.

**Cách hoạt động:**
1. Mỗi partition log được chia thành phần **clean** (đã compacted) và **dirty** (record mới chưa compact)
2. Log Cleaner (background thread) đọc phần dirty và build offset map key → offset mới nhất
3. Một segment compact mới được ghi với chỉ giá trị mới nhất per key
4. Segment cũ được thay thế

**Tombstone record**: Record với key không null và **value null** là **tombstone**. Quá trình compaction giữ tombstone trong khoảng thời gian config (`delete.retention.ms`, mặc định 24h), sau đó xóa hoàn toàn. Báo hiệu consumer rằng key đã bị xóa.

**Use case:**
- Database change log (giữ state mới nhất của mỗi row)
- Topic user profile (state mới nhất per user ID)
- Event sourcing với snapshot
- Kafka như distributed key-value store

**Bẫy**: Compaction KHÔNG xảy ra ngay lập tức hay theo khoảng thời gian có thể đoán được. Tỉ lệ dirty phải vượt `min.cleanable.dirty.ratio` (mặc định 0.5). Có thể có lag đáng kể trước khi record cũ bị xóa. Đừng dựa vào compaction để xóa ngay lập tức.

**Bẫy khác**: Nếu `isolation.level=read_committed` được đặt và có hanging transaction, LSO không thể advance qua transaction đang mở, block log compaction. Gây topic tăng không giới hạn.

---

## 14. MirrorMaker 2 — Replication Liên Datacenter

### 14.1 Tổng Quan MirrorMaker 2

**H: MirrorMaker 2 là gì và khác gì MirrorMaker 1?**

**MirrorMaker 2** (MM2) là giải pháp replication liên cluster/datacenter của Kafka, được build trên nền Kafka Connect framework (KIP-382). Thay thế hoàn toàn MirrorMaker 1 (deprecated).

**Điểm khác biệt chính:**

| Khía cạnh | MirrorMaker 1 | MirrorMaker 2 |
|-----------|--------------|--------------|
| Architecture | Consumer + Producer riêng biệt | Kafka Connect cluster |
| Offset translation | Không có — phải reset từ đầu | Tự động dịch offset giữa cluster |
| Topic management | Không sync config | Sync topic config, ACL, offset |
| Replication lag monitoring | Thủ công | Built-in metrics |
| Active-active | Không hỗ trợ | Hỗ trợ (với topic rename) |
| Failure recovery | Mất offset khi restart | Resume từ chỗ dừng |

**MM2 gồm ba connector chính:**
```
MirrorSourceConnector:    Replicate records từ source → target cluster
MirrorCheckpointConnector: Replicate consumer group offset checkpoints
MirrorHeartbeatConnector:  Emit heartbeat để đo replication lag
```

---

### 14.2 Topic Namespace và Offset Translation

**H: MM2 xử lý naming conflict và offset translation như thế nào?**

**Topic renaming**: MM2 tự động prefix topic name với source cluster alias:
```
Source cluster "dc1", topic "orders"
→ Target cluster topic: "dc1.orders"

Config:
  source.cluster.alias=dc1
  target.cluster.alias=dc2
  replication.policy.class=org.apache.kafka.connect.mirror.DefaultReplicationPolicy
```

**Tại sao cần rename?** Tránh infinite replication loop trong active-active: nếu topic cùng tên được replicate cả hai chiều, sẽ có vòng lặp vô tận.

**Offset translation**: Offset trong dc1 và dc2 có thể khác nhau (do compaction, retention). MM2 lưu mapping offset vào internal topic `mm2-offset-syncs.dc1.internal` và expose qua `RemoteClusterUtils.translateOffsets()`.

Khi consumer group failover từ dc1 sang dc2:
```bash
# Tool tự động translate offset
kafka-consumer-groups.sh --bootstrap-server dc2:9092 \
  --group my-group --reset-offsets \
  --to-latest --execute
# Hoặc dùng MirrorCheckpointConnector để tự động sync
```

---

### 14.3 Active-Active vs Active-Passive

**H: Active-Active và Active-Passive replication khác nhau thế nào? Khi nào dùng cái nào?**

**Active-Passive (Disaster Recovery)**
```
                    replication
DC1 (Primary) ─────────────────→ DC2 (Standby)
Producer → DC1                    (không có producer)
Consumer → DC1                    Consumer không đọc bình thường

Mục đích: DR site. Khi DC1 fail, failover consumer sang DC2.
```

Config MM2 cho active-passive:
```properties
# Chạy MM2 ở DC2
clusters=dc1,dc2
dc1.bootstrap.servers=broker-dc1:9092
dc2.bootstrap.servers=broker-dc2:9092

dc1->dc2.enabled=true    # replication từ DC1 sang DC2
dc2->dc1.enabled=false   # không replication ngược lại

dc1->dc2.topics=.*       # replication tất cả topic
```

**Ưu điểm Active-Passive:**
- Đơn giản, không conflict
- RTO thấp (failover nhanh)
- Không có write conflict

**Nhược điểm:**
- DC2 "lãng phí" khi DC1 hoạt động bình thường
- RPO không bằng zero (replication lag = có thể mất data)

---

**Active-Active (Multi-region)**
```
DC1 ←──────────────────→ DC2
Producer → DC1               Producer → DC2
Consumer → DC1               Consumer → DC2
```

Config:
```properties
dc1->dc2.enabled=true
dc2->dc1.enabled=true

# Tránh replication loop: exclude topic đã được replicated
dc1->dc2.topics=.*
dc2->dc1.topics=.*
dc1->dc2.topics.exclude=dc2\\..*   # không re-replicate topic từ dc2
dc2->dc1.topics.exclude=dc1\\..*   # không re-replicate topic từ dc1
```

**Luồng active-active:**
```
DC1: topic "orders" (gốc từ dc1 producer)
DC2: topic "dc1.orders" (replicated từ dc1)
     topic "orders" (gốc từ dc2 producer)
DC1: topic "dc2.orders" (replicated từ dc2)
```

**Thách thức active-active:**
- **Conflict resolution**: Cùng business entity được update ở cả hai DC → phải có strategy (last-write-wins, CRDT, application-level merge)
- **Consumer consistency**: Consumer đọc từ local DC có thể không thấy update từ remote DC ngay lập tức
- **Offset complexity**: Hai topic namespace riêng biệt, consumer cần subscribe cả hai

**Khi nào dùng:**

| Yêu cầu | Pattern | Lý do |
|---------|---------|-------|
| DR đơn giản, RPO vài giây | Active-Passive | Đơn giản, ít risk |
| Low-latency cho user toàn cầu | Active-Active | User viết vào DC gần nhất |
| Compliance data residency | Active-Active with filtering | Mỗi DC giữ dữ liệu riêng |
| Zero downtime migration | Active-Active tạm thời | Replication cả hai chiều khi chuyển cluster |

---

### 14.4 Monitoring MM2

```bash
# Replication lag (giây)
kafka.connect:type=MirrorSourceConnector,target=dc2,topic=orders,partition=0
  → replication-latency-ms

# Heartbeat lag
kafka.connect:type=MirrorHeartbeatConnector
  → replication-latency-ms

# Kiểm tra offset sync topic
kafka-console-consumer.sh --bootstrap-server dc2:9092 \
  --topic mm2-offset-syncs.dc1.internal --from-beginning
```

---

## 15. So Sánh: Kafka vs RabbitMQ vs Pulsar

### 15.1 Kafka vs RabbitMQ

**H: Khi nào dùng Kafka, khi nào dùng RabbitMQ?**

**Mô hình nền tảng khác biệt:**

| Khía cạnh | Kafka | RabbitMQ |
|-----------|-------|----------|
| **Mô hình** | Log-based, pull | Message queue, push |
| **Retention** | Giữ message sau consume (configurable) | Xóa message sau consume |
| **Consumer** | Pull, manage offset tự | Push, broker manage delivery |
| **Ordering** | Đảm bảo trong partition | Đảm bảo trong queue (không requeue) |
| **Replay** | Có, consumer seek về offset cũ | Không (message đã xóa) |
| **Throughput** | Rất cao (triệu msg/s) | Cao nhưng thấp hơn (hàng chục nghìn/s per queue) |
| **Latency** | Milliseconds (default), sub-ms với tuning | Sub-millisecond |
| **Routing** | Simple (hash by key) | Flexible (exchange, binding, routing key) |
| **Message size** | Tốt nhất với small-medium (<1MB) | Tốt với bất kỳ size |
| **Protocol** | Kafka binary protocol | AMQP, MQTT, STOMP |

**Dùng Kafka khi:**
- Cần **event streaming / event sourcing** — giữ lịch sử, replay được
- Throughput rất cao (IoT, clickstream, log aggregation)
- Nhiều consumer group độc lập consume cùng data
- Cần audit trail hoặc compliance logging
- Stream processing với Kafka Streams hoặc Flink
- Microservice choreography với loose coupling

**Dùng RabbitMQ khi:**
- **Task queue** — worker pool xử lý job, mỗi job cần được xử lý đúng một lần
- **Complex routing** — dùng exchange/binding để route message theo nhiều tiêu chí
- **Low latency** — RabbitMQ có thể đạt sub-millisecond latency
- **Request-reply pattern** — RabbitMQ có hỗ trợ native RPC
- Message nhỏ, không cần replay
- Đội ngũ không muốn quản lý Kafka cluster phức tạp

**Ví dụ minh họa:**
```
Kafka: "Mỗi lần user đặt hàng, lưu event, cho phép inventory/analytics/email
        đều consume lịch sử để rebuild state nếu cần"

RabbitMQ: "Phân phối email job cho 10 worker, mỗi email chỉ gửi một lần,
           nếu worker fail thì worker khác lấy lại job"
```

---

### 15.2 Kafka vs Apache Pulsar

**H: Apache Pulsar khác Kafka ở điểm gì? Khi nào nên chọn Pulsar?**

**Kiến trúc cốt lõi:**

| Khía cạnh | Kafka | Pulsar |
|-----------|-------|--------|
| **Storage** | Broker lưu trữ data trực tiếp | Tách biệt: Broker stateless, data lưu trong **BookKeeper** |
| **Broker state** | Stateful (mỗi broker giữ partition data) | Stateless (broker chỉ serve) |
| **Scaling** | Scale broker = scale storage + compute cùng lúc | Scale compute (broker) và storage (BookKeeper) độc lập |
| **Topic model** | Partition | Topic, Partition, Subscription |
| **Subscription model** | Consumer group (pull) | Exclusive, Shared, Failover, Key_Shared |
| **Geo-replication** | MirrorMaker 2 (external tool) | Built-in native geo-replication |
| **Multi-tenancy** | Không native (phải dùng naming convention) | Native (tenant/namespace/topic hierarchy) |
| **Functions** | Kafka Streams (external library) | Pulsar Functions (built-in, lightweight serverless) |
| **Protocol** | Kafka binary | Pulsar binary + AMQP + MQTT + Kafka protocol (compatibility) |

**Ưu điểm Pulsar:**
1. **Tách compute/storage**: Scale broker mà không cần thêm disk, scale storage mà không cần thêm compute
2. **Zero copy rebalance**: Khi broker fail, không cần move data — stateless broker khác serve ngay
3. **Native geo-replication**: Built-in, không cần external tool
4. **Multi-tenancy**: Phân chia tenant/namespace với quota riêng, không cần separate cluster
5. **Subscription flexibility**: Pulsar hỗ trợ cả queue và pub/sub model trong một system

**Nhược điểm Pulsar:**
1. **Phức tạp hơn**: Phải quản lý cả Pulsar brokers + BookKeeper + ZooKeeper
2. **Ecosystem nhỏ hơn**: Ít connector, tool, và community support hơn Kafka
3. **BookKeeper overhead**: Thêm một layer storage = thêm latency và complexity
4. **Kafka compatibility không hoàn hảo**: Kafka protocol adapter có giới hạn

**Dùng Pulsar khi:**
- Cần **elastic scaling** tách biệt compute/storage (cloud-native workload)
- **Multi-tenancy** mạnh là yêu cầu (SaaS platform với nhiều tenant)
- Cần **native geo-replication** đơn giản
- Workload mix: vừa cần queue (task distribution) vừa cần pub/sub
- Tổ chức sẵn sàng đầu tư vào learning curve

**Dùng Kafka khi:**
- Ecosystem và tooling là ưu tiên (Confluent, Debezium, Kafka Streams, Flink integration)
- Team đã có Kafka expertise
- Cần throughput cực cao với latency thấp
- Không cần multi-tenancy phức tạp

---

### 15.3 Tóm Tắt Decision Tree

```
Cần replay message?
  Không → RabbitMQ (hoặc SQS/ActiveMQ)
  Có → tiếp tục

Cần throughput > 100k msg/s hoặc event sourcing?
  Có → Kafka hoặc Pulsar

Cần tách storage/compute hoặc multi-tenancy native?
  Có → Pulsar
  Không → Kafka

Cần complex routing logic (fanout, content-based)?
  Có → RabbitMQ hoặc Kafka + custom consumer logic

Cần sub-millisecond latency + task queue semantics?
  Có → RabbitMQ
```
