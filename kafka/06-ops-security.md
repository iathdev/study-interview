# Vận Hành, Bảo Mật, Quota và Monitoring

> Tổng hợp từ tài liệu chính thức, Confluent engineering blog, Strimzi, và các nguồn phỏng vấn senior.

---

## 13. Kafka Security — Bảo Mật

### 13.1 Transport Layer Security (SSL/TLS)

**H: Kafka hỗ trợ encryption như thế nào?**

Kafka dùng SSL/TLS để encrypt data in-transit giữa:
- Client (producer/consumer) ↔ Broker
- Broker ↔ Broker (inter-broker communication)

**Cấu hình broker:**
```properties
# Listener với SSL
listeners=SSL://0.0.0.0:9093
advertised.listeners=SSL://broker-1:9093
ssl.keystore.location=/var/ssl/kafka.server.keystore.jks
ssl.keystore.password=keystore-password
ssl.key.password=key-password
ssl.truststore.location=/var/ssl/kafka.server.truststore.jks
ssl.truststore.password=truststore-password

# Yêu cầu client certificate (mutual TLS)
ssl.client.auth=required   # hoặc requested / none
```

**Cấu hình client:**
```properties
security.protocol=SSL
ssl.truststore.location=/var/ssl/client.truststore.jks
ssl.truststore.password=truststore-password
# Nếu broker yêu cầu client cert:
ssl.keystore.location=/var/ssl/client.keystore.jks
ssl.keystore.password=keystore-password
ssl.key.password=key-password
```

**Lưu ý performance**: SSL làm giảm throughput ~20-30% so với PLAINTEXT do:
1. Handshake overhead (một lần per connection)
2. Encrypt/decrypt mỗi message (CPU)
3. Không dùng được zero-copy sendfile (data phải qua user space để decrypt)

Dùng TLS 1.3 (mặc định từ Kafka 3.x) để giảm handshake latency.

---

### 13.2 SASL Authentication

**H: Các cơ chế SASL trong Kafka là gì? Khi nào dùng cái nào?**

SASL (Simple Authentication and Security Layer) cung cấp authentication. Kafka hỗ trợ:

**SASL/PLAIN**
```
Cơ chế: Username/password dạng plaintext truyền qua kết nối
Bảo mật: PHẢI kết hợp với SSL (không thì password đi dạng plaintext)
Dùng khi: Development, testing, hoặc internal cluster với SSL
Config broker:
  sasl.enabled.mechanisms=PLAIN
  sasl.mechanism.inter.broker.protocol=PLAIN
  listener.name.sasl_ssl.plain.sasl.jaas.config=\
    org.apache.kafka.common.security.plain.PlainLoginModule required \
    username="admin" password="admin-secret" \
    user_admin="admin-secret" user_alice="alice-secret";
```

**SASL/SCRAM-SHA-256 và SCRAM-SHA-512**
```
Cơ chế: Challenge-response, password không bao giờ truyền plaintext
Lưu trữ: Credential được hash và lưu trong ZooKeeper/KRaft
Dùng khi: Production khi không có Kerberos infrastructure
Tạo user:
  kafka-configs.sh --bootstrap-server :9092 --alter \
    --add-config 'SCRAM-SHA-512=[iterations=8192,password=alice-secret]' \
    --entity-type users --entity-name alice

Config broker:
  sasl.enabled.mechanisms=SCRAM-SHA-512
  listener.name.sasl_ssl.scram-sha-512.sasl.jaas.config=\
    org.apache.kafka.common.security.scram.ScramLoginModule required;
```

**SASL/GSSAPI (Kerberos)**
```
Cơ chế: Kerberos ticket-based authentication
Dùng khi: Enterprise với Active Directory / Kerberos KDC đã có sẵn
Ưu điểm: Single Sign-On, không lưu password trong Kafka
Nhược điểm: Cần Kerberos KDC, phức tạp hơn để setup
Config broker:
  sasl.enabled.mechanisms=GSSAPI
  sasl.kerberos.service.name=kafka
  listener.name.sasl_ssl.gssapi.sasl.jaas.config=\
    com.sun.security.auth.module.Krb5LoginModule required \
    useKeyTab=true storeKey=true \
    keyTab="/etc/kafka/kafka.keytab" \
    principal="kafka/broker-1.example.com@EXAMPLE.COM";
```

**SASL/OAUTHBEARER**
```
Cơ chế: JWT token từ OAuth 2.0 provider (Okta, Keycloak, AWS Cognito)
Dùng khi: Cloud-native, microservice với existing OAuth infrastructure
Ưu điểm: Token rotation, short-lived credentials
Config: Cần custom callback để validate token với OAuth provider
```

**So sánh tổng quan:**

| Mechanism | Complexity | Enterprise Support | Token Rotation | Khi dùng |
|-----------|-----------|-------------------|----------------|----------|
| PLAIN | Thấp | Không | Không | Dev/test, simple prod với SSL |
| SCRAM-SHA-512 | Trung bình | Một phần | Thủ công | Production không có Kerberos |
| GSSAPI/Kerberos | Cao | Đầy đủ | Tự động | Enterprise với AD/Kerberos |
| OAUTHBEARER | Trung bình | Modern cloud | Tự động | Cloud-native, microservice |

---

### 13.3 Authorization — ACLs

**H: Kafka ACL hoạt động như thế nào?**

Kafka dùng ACL (Access Control List) để authorize sau khi authenticate. Default authorizer: `AclAuthorizer` (lưu ACL trong ZooKeeper) hoặc `StandardAuthorizer` (KRaft).

**Cấu trúc ACL:**
```
Principal: User:alice           (ai)
Permission: ALLOW / DENY        (cho phép hay từ chối)
Operation: READ / WRITE / CREATE / DELETE / ALTER / DESCRIBE / ...
Resource: Topic:orders          (tài nguyên nào)
Host: *                         (từ host nào, * = mọi host)
```

**Quản lý ACL qua CLI:**
```bash
# Cho phép alice produce vào topic "orders"
kafka-acls.sh --bootstrap-server :9092 \
  --add --allow-principal User:alice \
  --operation Write --topic orders

# Cho phép nhóm "payments-consumers" consume từ topic "orders"
kafka-acls.sh --bootstrap-server :9092 \
  --add --allow-principal User:payments-service \
  --operation Read --topic orders \
  --group payments-consumers

# Consumer cần cả READ trên topic VÀ READ trên consumer group
kafka-acls.sh --bootstrap-server :9092 \
  --add --allow-principal User:payments-service \
  --operation Read --group payments-consumers

# Liệt kê ACL
kafka-acls.sh --bootstrap-server :9092 --list --topic orders

# Xóa ACL
kafka-acls.sh --bootstrap-server :9092 \
  --remove --allow-principal User:alice \
  --operation Write --topic orders
```

**Bật ACL trong broker:**
```properties
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
# Kafka 3.x KRaft:
super.users=User:admin
# Nếu không có ACL matching → DENY by default
allow.everyone.if.no.acl.found=false
```

**Các operation cần thiết cho consumer:**

| Operation | Resource | Lý do |
|-----------|---------|-------|
| READ | Topic | Fetch records |
| READ | ConsumerGroup | Commit offset, JoinGroup |
| DESCRIBE | Topic | Metadata request |

**Các operation cần thiết cho producer:**

| Operation | Resource | Lý do |
|-----------|---------|-------|
| WRITE | Topic | Produce records |
| CREATE | Topic | Auto-create topic (nếu bật) |
| DESCRIBE | Topic | Metadata request |

---

### 13.4 Kết Hợp Security Layers

**H: Cách deploy Kafka với đầy đủ security (encrypt + authenticate + authorize)?**

```
listeners=SASL_SSL://0.0.0.0:9093,PLAINTEXT://127.0.0.1:9092
# SASL_SSL: cho external client (encrypt + authenticate)
# PLAINTEXT: chỉ local (monitoring tool trên cùng host)

advertised.listeners=SASL_SSL://broker-1:9093

# SSL config
ssl.keystore.location=...
ssl.truststore.location=...

# SASL config
sasl.enabled.mechanisms=SCRAM-SHA-512
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512

# Authorization
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
super.users=User:kafka-admin
```

**Security protocol options:**

| Protocol | Transport | Authentication |
|----------|-----------|---------------|
| `PLAINTEXT` | None | None |
| `SSL` | TLS | Optionally mTLS |
| `SASL_PLAINTEXT` | None | SASL |
| `SASL_SSL` | TLS | SASL + optionally mTLS |

Production nên dùng `SASL_SSL` hoặc ít nhất `SASL_PLAINTEXT` cho internal network.

---

## 16. Quota Management — Quản Lý Hạn Mức

### 16.1 Các Loại Quota

**H: Kafka hỗ trợ những loại quota nào?**

Kafka có ba loại quota để ngăn một client độc chiếm tài nguyên cluster:

| Loại Quota | Mô tả | Đơn vị |
|-----------|-------|--------|
| **Producer byte-rate** | Tốc độ produce tối đa | bytes/giây |
| **Consumer byte-rate** | Tốc độ fetch tối đa | bytes/giây |
| **Request rate** | % CPU time được dùng cho request handling | % (0.0 - 1.0) |

Quota được áp dụng theo thứ tự ưu tiên (từ cụ thể đến chung):
1. `(user, client-id)` — cụ thể nhất
2. `(user, default-client-id)` — mọi client-id của user này
3. `(default-user, client-id)` — user này với client-id cụ thể
4. `(default-user, default-client-id)` — mặc định cho tất cả

---

### 16.2 Thiết Lập Quota

**H: Cách set quota cho producer/consumer?**

```bash
# Giới hạn producer "analytics-app" produce tối đa 10MB/s
kafka-configs.sh --bootstrap-server :9092 \
  --alter --add-config 'producer_byte_rate=10485760' \
  --entity-type clients --entity-name analytics-app

# Giới hạn consumer "reporting-service" fetch tối đa 5MB/s
kafka-configs.sh --bootstrap-server :9092 \
  --alter --add-config 'consumer_byte_rate=5242880' \
  --entity-type clients --entity-name reporting-service

# Giới hạn theo user (kết hợp SASL)
kafka-configs.sh --bootstrap-server :9092 \
  --alter --add-config 'producer_byte_rate=52428800,consumer_byte_rate=52428800' \
  --entity-type users --entity-name alice

# Kết hợp user + client-id (quota chi tiết nhất)
kafka-configs.sh --bootstrap-server :9092 \
  --alter --add-config 'producer_byte_rate=1048576' \
  --entity-type users --entity-name alice \
  --entity-name analytics-client

# Set quota mặc định cho tất cả user
kafka-configs.sh --bootstrap-server :9092 \
  --alter --add-config 'producer_byte_rate=104857600' \
  --entity-type users --entity-default

# Xem quota hiện tại
kafka-configs.sh --bootstrap-server :9092 \
  --describe --entity-type clients --entity-name analytics-app
```

---

### 16.3 Cơ Chế Throttling

**H: Khi client vượt quota, Kafka xử lý như thế nào?**

Kafka **không reject** request khi vượt quota — thay vào đó nó **throttle** (làm chậm):

**Cơ chế:**
```
1. Broker tính toán quota violation:
   - Đo tốc độ thực tế trong cửa sổ thời gian (mặc định 30 window * 1s = 30s)
   - So sánh với quota được cấu hình

2. Broker tính toán delay time cần thiết:
   delay_ms = (bytes_sent_above_quota / quota_rate) * 1000

3. Broker trả về response bình thường NHƯNG thêm throttle_time_ms vào header
   (không reject, không drop message)

4. Client Java nhận throttle_time_ms:
   - Trước Kafka 2.0: sleep trước khi gửi request tiếp theo
   - Từ Kafka 2.0+: broker tự delay response (server-side throttling)
     để client không bị block polling loop

5. Metric tăng: throttle-time-avg, throttle-time-max
```

**Server-side throttling (Kafka 2.0+):** Broker giữ connection mở nhưng delay response. Cách này tốt hơn vì:
- Client không cần biết về quota logic
- Không làm timeout heartbeat thread
- Đơn giản hơn từ phía client

**Xem throttling đang xảy ra:**
```bash
# Metric trên broker
kafka.server:type=ClientQuotaManager,user=alice,client-id=analytics
  → produce-throttle-time-avg
  → produce-throttle-time-max
  → fetch-throttle-time-avg

# Metric trên producer client
kafka.producer:type=producer-metrics,client-id=analytics
  → produce-throttle-time-avg
```

---

## 17. Monitoring — Giám Sát Hệ Thống

### 17.1 Consumer Lag Monitoring

**H: Consumer lag là gì và cách monitor đúng cách?**

**Consumer lag** = Offset của message mới nhất trong partition − Committed offset của consumer group

```
Log End Offset (LEO):    1000   ← message mới nhất được produce
Consumer committed:       950   ← consumer đã xử lý đến đây
Consumer lag:              50   ← consumer đang bị lùi 50 messages
```

**Cách đo lag:**

```bash
# Command line
kafka-consumer-groups.sh --bootstrap-server :9092 \
  --describe --group my-group

OUTPUT:
GROUP        TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
my-group     orders   0          950             1000            50
my-group     orders   1          1200            1200            0
my-group     orders   2          800             900             100
```

**Metric JMX quan trọng:**
```
# Consumer-side (đo từ client)
kafka.consumer:type=consumer-fetch-manager-metrics,client-id=<id>
  records-lag-max          ← lag tối đa trong tất cả partition đang assign
  records-lag-avg          ← lag trung bình
  records-lag              ← lag của partition cụ thể (per partition)

# Tốt hơn: đo từ external (không phụ thuộc consumer còn sống)
→ Dùng Burrow (LinkedIn) hoặc kafka-consumer-groups.sh
→ So sánh committed offset với LEO bằng Admin Client API
```

**Lag alert thực tế:**

```
Alert nếu lag tăng liên tục (không phải chỉ lag cao):
  - Lag cao + stable → consumer đang catch up được, OK
  - Lag tăng liên tục → consumer chậm hơn producer, cần scale

Alert threshold thực tế:
  - Warn: lag > 10,000 messages VÀ tăng liên tục 5 phút
  - Critical: lag > 100,000 messages hoặc lag > Xh của production rate
```

---

### 17.2 Key JMX Metrics — Cheat Sheet Đầy Đủ

**H: Những JMX metric nào quan trọng nhất cho monitoring Kafka?**

**Broker — Health**

| Metric | Giá trị bình thường | Alert khi |
|--------|-------------------|-----------|
| `kafka.controller:type=KafkaController,name=ActiveControllerCount` | 1 (trên toàn cluster) | != 1 |
| `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions` | 0 | > 0 kéo dài > 30s |
| `kafka.server:type=ReplicaManager,name=UnderMinIsrPartitionCount` | 0 | > 0 |
| `kafka.server:type=ReplicaManager,name=OfflinePartitionsCount` | 0 | > 0 |
| `kafka.server:type=ReplicaManager,name=IsrShrinksPerSec` | 0 | > 0 |
| `kafka.server:type=ReplicaManager,name=IsrExpandsPerSec` | 0 (hoặc thấp) | Tăng cao sau shrink |

**Broker — Performance**

| Metric | Ý nghĩa |
|--------|---------|
| `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec` | Ingestion rate (bytes/s) |
| `kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec` | Consumption rate (bytes/s) |
| `kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec` | Message ingestion rate |
| `kafka.network:type=RequestMetrics,name=RequestsPerSec,request=Produce` | Produce request rate |
| `kafka.network:type=RequestMetrics,name=RequestsPerSec,request=FetchConsumer` | Consumer fetch rate |
| `kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce` | Produce request latency |
| `kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent` | % thời gian handler rảnh (< 30% = cần thêm `num.io.threads`) |

**Broker — Disk và OS**

| Metric | Alert khi |
|--------|----------|
| Disk usage per broker | > 70% (để margin cho replication) |
| `kafka.log:type=LogManager,name=OfflineLogDirectoryCount` | > 0 (disk fail) |
| Network throughput | Gần giới hạn NIC |
| Page cache hit rate | Thấp → disk I/O cao → cần thêm RAM |

**Producer Metrics**

| Metric | Ý nghĩa |
|--------|---------|
| `record-error-rate` | Tỉ lệ message fail / tổng message |
| `record-retry-rate` | Tỉ lệ retry |
| `request-latency-avg` | Latency trung bình từ send đến ACK |
| `batch-size-avg` | Kích thước batch trung bình (nhỏ → tăng `linger.ms`) |
| `compression-rate-avg` | Tỉ lệ nén (1.0 = không nén) |
| `record-queue-time-avg` | Thời gian chờ trong `RecordAccumulator` |

**Consumer Metrics**

| Metric | Ý nghĩa |
|--------|---------|
| `records-lag-max` | Lag lớn nhất qua tất cả partition |
| `records-consumed-rate` | Message consume per second |
| `fetch-latency-avg` | Latency fetch request |
| `commit-latency-avg` | Latency offset commit |
| `rebalance-rate-per-hour` | Tần suất rebalance (cao = vấn đề stability) |
| `last-rebalance-seconds-ago` | Thời gian kể từ rebalance gần nhất |

---

### 17.3 Under-Replicated Partitions — Cách Xử Lý

**H: `UnderReplicatedPartitions` alert lên, cần làm gì?**

```bash
# 1. Xác định partition nào bị under-replicated
kafka-topics.sh --bootstrap-server :9092 --describe \
  --under-replicated-partitions

OUTPUT (ví dụ):
Topic: orders  Partition: 3  Leader: 2  Replicas: 2,1,3  Isr: 2,1
# → Broker 3 bị rơi khỏi ISR của partition này

# 2. Kiểm tra broker 3
kafka-broker-api-versions.sh --bootstrap-server broker-3:9092
# Nếu không response → broker 3 đang down

# 3. Kiểm tra ISR shrink log trên leader (broker 2)
grep "Shrinking ISR" /var/log/kafka/server.log | grep "orders-3"

# 4. Kiểm tra replica lag
kafka-log-dirs.sh --bootstrap-server :9092 \
  --broker-list 3 --topic-list orders
```

**Nguyên nhân phổ biến và giải pháp:**

| Nguyên nhân | Triệu chứng | Giải pháp |
|-------------|------------|-----------|
| Broker down | Broker không response | Restart broker |
| Disk full | Log "disk quota exceeded" | Tăng disk hoặc giảm retention |
| GC pressure | GC log > 30s pauses | Tune JVM, giảm heap |
| Network lag giữa broker | Ping latency cao | Kiểm tra network, rack config |
| High load | `RequestHandlerAvgIdlePercent` < 30% | Scale thêm broker, giảm partition per broker |

---

## 18. Các Vấn Đề Vận Hành Thường Gặp

### 18.1 Consumer Lag Tăng Không Kiểm Soát

**H: Consumer lag tăng liên tục, làm thế nào để xử lý?**

**Bước 1: Xác định bottleneck**
```bash
# Kiểm tra lag theo partition
kafka-consumer-groups.sh --bootstrap-server :9092 \
  --describe --group my-group

# Nếu lag tập trung vào ít partition → hotspot (key skew)
# Nếu lag đều trên tất cả partition → consumer chậm hoặc producer tăng đột biến
```

**Bước 2: Kiểm tra consumer throughput**
```
Kiểm tra metric records-consumed-rate:
- Thấp hơn bình thường → consumer bị slow (DB bottleneck, external API slow, GC)
- Bình thường → producer đang produce nhanh hơn → cần scale consumer
```

**Bước 3: Giải pháp theo nguyên nhân**

| Nguyên nhân | Giải pháp |
|-------------|-----------|
| Consumer xử lý chậm (heavy business logic) | Tối ưu code, async processing, batch external calls |
| DB insert bottleneck | Batch insert, connection pool tuning |
| Consumer lag do rebalance thường xuyên | Bật static membership, tăng `session.timeout.ms` |
| Không đủ consumer instance | Thêm consumer instance (tối đa = số partition) |
| Partition không đủ | Tăng partition count (cẩn thận: không giảm được) |
| Key hotspot | Redesign partition key, thêm composite key |

**Khi không thể scale ngay lập tức:**
```bash
# Tạm thời skip lag bằng cách reset offset về cuối (mất message!)
kafka-consumer-groups.sh --bootstrap-server :9092 \
  --group my-group --reset-offsets --to-latest \
  --topic my-topic --execute

# Chỉ dùng khi chấp nhận mất message và SLA cho phép
```

---

### 18.2 Partition Rebalancing Issues

**H: Consumer group liên tục rebalance, cách diagnose và fix?**

**Triệu chứng:**
- Log: `Rebalance in progress` hoặc `Member ... has left the group`
- Metric: `rebalance-rate-per-hour` cao
- Consumer lag tăng vọt mỗi vài phút rồi giảm (do rebalance stop-the-world)

**Nguyên nhân phổ biến:**

**1. `max.poll.interval.ms` bị vượt quá**
```
Consumer đang xử lý batch nặng, không kịp gọi poll() trong 5 phút
→ Consumer bị evict → rebalance

Fix:
  - Giảm max.poll.records để xử lý ít hơn mỗi lần
  - Tăng max.poll.interval.ms nếu business logic thực sự cần lâu hơn
  - Tách processing ra thread riêng, gọi poll() định kỳ
```

**2. GC pause vượt `session.timeout.ms`**
```
Full GC > 30s → heartbeat thread bị dừng → broker coi consumer là dead

Fix:
  - Tăng session.timeout.ms (nhưng detection chậm hơn)
  - Tune JVM GC (G1GC với -XX:MaxGCPauseMillis=2000)
  - Giảm JVM heap (ít heap = ít GC pressure; dùng off-heap cho cache)
```

**3. Network intermittent**
```
Heartbeat packet bị drop → session timeout

Fix:
  - Kiểm tra network stability
  - Tăng session.timeout.ms
  - Bật static membership (group.instance.id)
```

**4. Rolling deployment không đúng**
```
Mỗi lần restart pod → member leave → rebalance

Fix:
  - Dùng static membership (group.instance.id = stable pod name)
  - Consumer rejoins trong session.timeout.ms mà không trigger rebalance
  - Đặt group.instance.id = hostname hoặc pod name
```

---

### 18.3 Disk Full Trên Broker

**H: Disk của broker sắp đầy, cần làm gì ngay?**

**Immediate actions (trong 15 phút):**
```bash
# 1. Xác định topic nào chiếm nhiều disk nhất
du -sh /kafka-logs/* | sort -rh | head -20

# 2. Giảm retention tạm thời cho topic lớn nhất
kafka-configs.sh --bootstrap-server :9092 \
  --alter --add-config 'retention.ms=3600000' \    # 1 giờ thay vì 7 ngày
  --entity-type topics --entity-name big-topic

# Hoặc giảm retention bytes
kafka-configs.sh --bootstrap-server :9092 \
  --alter --add-config 'retention.bytes=1073741824' \  # 1GB per partition
  --entity-type topics --entity-name big-topic

# 3. Trigger log cleanup thủ công
kafka-log-dirs.sh --bootstrap-server :9092 \
  --broker-list 1 --describe --topic-list big-topic

# 4. Nếu vẫn không đủ: tạm dừng producer vào topic đó
# (extreme case) hoặc move partition sang broker khác
kafka-reassign-partitions.sh ...
```

**Tại sao disk đầy nhanh — các nguyên nhân ẩn:**

| Nguyên nhân | Kiểm tra | Giải pháp |
|-------------|---------|-----------|
| Hanging transaction | `kafka-transactions.sh find-hanging` | Abort transaction thủ công |
| LSO không advance | Consumer với `read_committed` bị stuck | Abort hanging transaction → LSO advance → log cleaner chạy |
| Replication lag cao | `kafka-topics.sh --under-replicated-partitions` | Fix replica, HW advance, segment cũ có thể xóa |
| Log compaction không chạy | Check `min.cleanable.dirty.ratio` | Giảm threshold, check log cleaner thread |
| Producer compression không bật | Metric `compression-rate-avg` = 1.0 | Bật `compression.type=lz4` hoặc `zstd` |

**Phòng ngừa dài hạn:**
```properties
# Alert trước khi đầy
disk_alert_warn=70%
disk_alert_critical=85%

# Đặt retention hợp lý từ đầu
log.retention.hours=168     # 7 ngày mặc định
log.retention.bytes=107374182400  # 100GB per partition

# Monitor partition count × retention × throughput = disk cần thiết
```

---

### 18.4 Broker Không Join Được Cluster

**H: Broker mới không join được vào cluster sau khi restart, cách debug?**

**Checklist debug:**

```bash
# 1. Kiểm tra log broker
tail -f /var/log/kafka/server.log | grep -E "ERROR|WARN|exception"

# Lỗi phổ biến:
# "Broker 3 is not registered"
# → KRaft: controller chưa nhận registration
# → Check controller log

# "LEADER_NOT_AVAILABLE for __cluster_metadata"
# → Quorum controller chưa elected leader
# → Check nếu có đủ quorum (majority of voters alive)

# 2. Kiểm tra cluster ID
cat /kafka-logs/meta.properties
# cluster.id phải khớp với cluster đang chạy
# Nếu khác → format lại storage bị sai

# 3. KRaft: kiểm tra quorum health
kafka-metadata-quorum.sh --bootstrap-server :9092 describe --status

# 4. Kiểm tra connectivity
nc -zv broker-1 9092   # TCP connectivity
nc -zv broker-1 9093   # Controller port

# 5. Check certificate nếu dùng SSL
openssl s_client -connect broker-1:9093 -verify 2
```

---

### 18.5 Message Size Quá Lớn

**H: Producer nhận `RecordTooLargeException`, cần config gì?**

Message size limit phải được set **đồng bộ** ở cả producer, broker, và consumer:

```properties
# Broker (topic level override hoặc broker default)
message.max.bytes=10485760         # 10MB (broker default: 1MB)
replica.fetch.max.bytes=10485760   # Phải >= message.max.bytes

# Topic level (ưu tiên hơn broker)
kafka-configs.sh --bootstrap-server :9092 \
  --alter --add-config 'max.message.bytes=10485760' \
  --entity-type topics --entity-name my-topic

# Producer
max.request.size=10485760          # Mặc định 1MB

# Consumer
max.partition.fetch.bytes=10485760 # Mặc định 1MB
fetch.max.bytes=52428800           # Tổng fetch response (phải >= max.partition.fetch.bytes)
```

**Best practice**: Message > 1MB thường là anti-pattern. Cân nhắc:
- Lưu payload lớn vào object storage (S3), chỉ gửi reference qua Kafka (**Claim Check pattern**)
- Chia message thành nhiều phần nhỏ với correlation ID
- Nén message trước khi gửi
