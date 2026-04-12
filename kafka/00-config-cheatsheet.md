# Kafka Config Cheatsheet

> Tổng hợp toàn bộ config quan trọng từ các tài liệu kiến trúc, producer/consumer, EOS, storage, và bảo mật. Dùng làm tài liệu tham khảo nhanh khi phỏng vấn hoặc cấu hình hệ thống thực tế.

---

## 0. Config cần nắm chắc nhất

Bốn config này hay xuất hiện trong interview vì mỗi cái ảnh hưởng trực tiếp đến **độ bền dữ liệu, hành vi consumer, và khả năng scale** — set sai một trong bốn là hệ thống có thể mất message hoặc xử lý sai.

---

### `acks` (Producer)

**Tại sao quan trọng**: Quyết định trực tiếp **có mất message không**.

| Giá trị | Ý nghĩa | Rủi ro |
|---------|---------|--------|
| `0` | Fire-and-forget, không chờ broker xác nhận | Mất message nếu broker down |
| `1` | Chỉ cần leader xác nhận | Leader chết trước khi follower sync → mất message |
| `all` / `-1` | Toàn bộ ISR xác nhận | An toàn nhất, latency cao hơn |

```properties
acks=all
# Phải kết hợp với min.insync.replicas=2 mới có ý nghĩa thực sự
# acks=all + min.insync.replicas=1 = không mạnh hơn acks=1
```

---

### `group.id` (Consumer)

**Tại sao quan trọng**: Xác định **consumer group nào đang đọc topic** — Kafka lưu offset riêng cho từng group, mỗi group tự theo dõi đã đọc đến đâu, độc lập với group khác.

- Nhiều consumer cùng `group.id` → chia nhau partition, mỗi message chỉ xử lý **một lần** (load balancing)
- Khác `group.id` → mỗi group nhận **toàn bộ** message độc lập (fan-out)

```
Topic orders (3 partition):
  group.id="payment-service"   → Consumer A nhận P0, Consumer B nhận P1+P2
  group.id="audit-service"     → Consumer C nhận P0+P1+P2 (toàn bộ)
```

```properties
group.id=payment-service
# Thiếu group.id → mỗi lần restart là một consumer group mới
# → đọc lại từ đầu hoặc từ latest tuỳ auto.offset.reset
```

---

### `auto.offset.reset` (Consumer)

**Tại sao quan trọng**: Quyết định consumer đọc từ đâu khi **không tìm thấy committed offset** (consumer mới, group bị xóa, offset expired).

| Giá trị | Hành vi | Dùng khi |
|---------|---------|---------|
| `latest` (default) | Đọc từ message mới nhất, bỏ qua message cũ | Production — không cần xử lý lại quá khứ |
| `earliest` | Đọc từ đầu partition | Cần replay toàn bộ history, consumer mới cần catchup |
| `none` | Throw exception nếu không có offset | Muốn biết rõ mình đang ở đâu, không cho đoán |

```properties
auto.offset.reset=earliest
# Bẫy: set latest cho consumer mới của critical topic
# → bỏ qua toàn bộ message đã có trong topic trước khi consumer join
```

---

### `partition.assignment.strategy` (Consumer)

**Tại sao quan trọng**: Quyết định **thuật toán phân chia partition** cho consumer trong group — ảnh hưởng trực tiếp đến rebalance overhead và tính ổn định.

| Strategy | Thuật toán | Vấn đề |
|----------|-----------|--------|
| `RangeAssignor` (default cũ) | Chia theo range, topic by topic | Không đều nếu số consumer không chia hết số partition |
| `RoundRobinAssignor` | Chia vòng tròn qua tất cả partition | Đều hơn nhưng rebalance gán lại toàn bộ |
| `StickyAssignor` | Giữ assignment cũ tối đa, chỉ di chuyển khi cần | Ít xáo trộn hơn, nhưng vẫn stop-the-world |
| `CooperativeStickyAssignor` | Incremental rebalance — không stop-the-world | **Khuyến nghị từ Kafka 2.4+** |

```properties
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
# Mặc định từ Kafka 3.1+
# Rebalance không dừng toàn bộ consumer group
# Chỉ consumer bị ảnh hưởng mới tạm dừng
```

---

## 1. Producer Config

| Config | Default | Giá trị khuyến nghị | Ý nghĩa | Lưu ý quan trọng |
|--------|---------|---------------------|---------|------------------|
| `acks` | `all` (Kafka 3.0+), `1` (trước 3.0) | `all` | Số broker phải xác nhận trước khi coi write thành công. `0`=fire-forget, `1`=chỉ leader, `all`/`-1`=toàn bộ ISR | Chỉ có ý nghĩa thực sự khi kết hợp với `min.insync.replicas=2`. `acks=all` mà `min.insync.replicas=1` không mạnh hơn `acks=1` |
| `retries` | `Integer.MAX_VALUE` (Kafka 2.1+) | `Integer.MAX_VALUE` | Số lần retry khi gửi thất bại | Khi bật `enable.idempotence=true` sẽ tự được set. Không retry với lỗi non-retriable |
| `retry.backoff.ms` | `100` | `200-500` | Thời gian chờ (ms) giữa mỗi lần retry | Tăng để tránh hammering broker đang quá tải |
| `delivery.timeout.ms` | `120000` (2 phút) | `120000-300000` | Thời gian tối đa từ lúc `send()` đến lúc nhận ACK hoặc fail hẳn. Bao gồm cả thời gian chờ trong buffer | Phải lớn hơn `request.timeout.ms + linger.ms` |
| `request.timeout.ms` | `30000` | `30000` | Thời gian chờ response từ broker cho mỗi request | |
| `max.in.flight.requests.per.connection` | `5` | `1` (nếu cần ordering nghiêm ngặt) hoặc `5` (với idempotence) | Số request đang bay chưa nhận ACK trên một connection | **Bẫy**: `> 1` với retry có thể gây reorder message. Nếu `enable.idempotence=true` thì tối đa 5. Nếu `> 5` thì idempotence bị vô hiệu hóa |
| `enable.idempotence` | `true` (Kafka 3.0+) | `true` | Tự động set `acks=all`, `retries=MAX_VALUE`, `max.in.flight<=5`. Broker deduplicate message trùng lặp trong cùng session | Chỉ dedup trong một producer session. Producer restart sẽ nhận PID mới |
| `transactional.id` | `null` | Unique stable per producer instance | Bật exactly-once transactions. Phải unique và stable giữa các restart | Cần gọi `initTransactions()` trước. Fences zombie producer cùng `transactional.id` |
| `transaction.timeout.ms` | `60000` (1 phút) | `60000` | Thời gian broker chờ transaction commit/abort trước khi tự abort | Nếu producer crash mà không abort, broker sẽ tự abort sau thời gian này |
| `batch.size` | `16384` (16KB) | `65536-131072` (64-128KB) | Kích thước batch tối đa (bytes) trước khi gửi | Batch lớn hơn = throughput cao hơn, compression tốt hơn, latency cao hơn |
| `linger.ms` | `0` (Kafka < 4.0), `5` (Kafka 4.0+) | `5-20` (throughput), `0` (latency) | Thời gian chờ tích lũy thêm record vào batch trước khi gửi | `linger.ms=0` gửi ngay khi có message. Tăng để batching tốt hơn |
| `buffer.memory` | `33554432` (32MB) | `64MB-128MB` nếu producer bận | Tổng RAM để buffer record chưa gửi trong `RecordAccumulator` | Nếu buffer đầy, `send()` sẽ block đến `max.block.ms` rồi throw exception |
| `max.block.ms` | `60000` (1 phút) | `60000` | Thời gian tối đa `send()` block khi buffer đầy hoặc đang fetch metadata | |
| `compression.type` | `none` | `lz4` (latency) hoặc `zstd` (ratio) | Thuật toán nén batch trước khi gửi. `none`, `gzip`, `snappy`, `lz4`, `zstd` | `gzip`: nén tốt nhất, CPU cao. `lz4`: nhanh nhất. `zstd`: cân bằng tốt nhất. Nén phía producer → broker forward nguyên si (zero-copy vẫn dùng được) |
| `max.request.size` | `1048576` (1MB) | `1048576` hoặc tăng nếu message lớn | Kích thước tối đa của một ProduceRequest | Phải <= `message.max.bytes` trên broker |
| `bootstrap.servers` | *(bắt buộc)* | Liệt kê 3+ broker | Danh sách broker để bootstrap, không cần đủ tất cả | Chỉ cần 1 broker alive để discover toàn bộ cluster |
| `key.serializer` | *(bắt buộc)* | `StringSerializer` hoặc `ByteArraySerializer` | Class serialize key thành bytes | |
| `value.serializer` | *(bắt buộc)* | Tùy format (Avro, JSON, etc.) | Class serialize value thành bytes | |
| `partitioner.class` | `DefaultPartitioner` | Giữ default hoặc custom | Logic chọn partition. Default: hash(key) % numPartitions, sticky nếu không có key | Thay đổi số partition sẽ phá vỡ routing theo key |

---

## 2. Consumer Config

| Config | Default | Giá trị khuyến nghị | Ý nghĩa | Lưu ý quan trọng |
|--------|---------|---------------------|---------|------------------|
| `group.id` | `""` | Stable unique string per service | ID của consumer group. Offset được track per group | Bắt buộc nếu dùng offset commit hoặc subscribe |
| `auto.offset.reset` | `latest` | `earliest` (dev/replay), `latest` (production) | Hành vi khi không có committed offset: `latest`=bỏ qua message cũ, `earliest`=đọc từ đầu, `none`=throw exception | **Bẫy**: `latest` sẽ miss message được produce trước khi consumer group được tạo |
| `enable.auto.commit` | `true` | **`false`** | Tự động commit offset theo `auto.commit.interval.ms` | **Bẫy lớn**: Auto-commit có thể commit trước khi xử lý xong → mất message khi crash. Luôn dùng `false` và commit thủ công |
| `auto.commit.interval.ms` | `5000` (5s) | N/A (nên dùng manual commit) | Tần suất auto-commit offset | Chỉ relevant khi `enable.auto.commit=true` |
| `session.timeout.ms` | `10000` (10s, Kafka 3.0+: 45s) | `30000-45000` | Thời gian tối đa không nhận heartbeat trước khi consumer bị coi là dead và bị evict | **Bẫy**: Không nên nhầm với `max.poll.interval.ms`. Đây là về heartbeat thread. Thấp quá → evict do GC pause |
| `heartbeat.interval.ms` | `3000` (3s) | `1/3 * session.timeout.ms` | Tần suất gửi heartbeat lên broker | Phải nhỏ hơn `session.timeout.ms`. Rule of thumb: bằng 1/3 |
| `max.poll.interval.ms` | `300000` (5 phút) | `300000` hoặc tăng nếu xử lý nặng | Thời gian tối đa giữa hai lần gọi `poll()`. Vượt quá → consumer bị evict dù heartbeat vẫn OK | **Bẫy**: Heartbeat thread và poll thread là riêng biệt. Consumer có thể gửi heartbeat nhưng vẫn bị evict nếu không gọi `poll()` kịp |
| `max.poll.records` | `500` | `100-500` (nếu xử lý nặng), tăng cho batch processing | Số record tối đa trả về mỗi lần `poll()` | Giảm nếu xử lý mỗi record lâu để tránh vượt `max.poll.interval.ms` |
| `fetch.min.bytes` | `1` | `1MB` (throughput) hoặc `1` (latency) | Broker chờ đến khi có đủ số bytes này trước khi trả về fetch response | Tăng để giảm số request, tăng throughput |
| `fetch.max.wait.ms` | `500` | `500` | Thời gian broker chờ tối đa dù chưa đủ `fetch.min.bytes` | Cặp với `fetch.min.bytes`: sau thời gian này broker trả về dù chưa đủ |
| `fetch.max.bytes` | `52428800` (50MB) | `50MB` hoặc tăng nếu message lớn | Tổng data tối đa trả về mỗi fetch response | Phải >= `max.partition.fetch.bytes` |
| `max.partition.fetch.bytes` | `1048576` (1MB) | Match với kích thước batch điển hình | Giới hạn data per-partition trong mỗi fetch response | Nếu message size > giá trị này, consumer vẫn fetch được nhưng chỉ 1 message/request |
| `isolation.level` | `read_uncommitted` | `read_committed` (nếu dùng transactions) | `read_uncommitted`: đọc tất cả kể cả in-progress transaction. `read_committed`: chỉ đọc committed transaction | Dùng `read_committed` khi consume từ transactional producer. Consumer bị block tại LSO |
| `partition.assignment.strategy` | `RangeAssignor,CooperativeStickyAssignor` | `CooperativeStickyAssignor` | Thuật toán phân phối partition cho consumer trong group | Cooperative Sticky tránh stop-the-world rebalance. Mặc định từ Kafka 2.4+ |
| `group.instance.id` | `null` | Stable unique per consumer (hostname, pod name) | Bật static group membership. Consumer với ID này không trigger rebalance khi reconnect trong `session.timeout.ms` | Quan trọng với Kubernetes rolling deployment. Tránh rebalance không cần thiết do GC pause hay restart ngắn |
| `key.deserializer` | *(bắt buộc)* | Match với serializer của producer | Class deserialize key từ bytes | |
| `value.deserializer` | *(bắt buộc)* | Match với serializer của producer | Class deserialize value từ bytes | |
| `client.id` | `""` | Tên app có nghĩa | ID client để identify trong metrics và logs | Hữu ích cho debugging và quota management |

---

## 3. Broker Config

| Config | Default | Giá trị khuyến nghị | Ý nghĩa | Lưu ý quan trọng |
|--------|---------|---------------------|---------|------------------|
| `broker.id` | `-1` (tự động với KRaft) | Integer duy nhất per broker | ID broker trong cluster | Phải unique. Với KRaft có thể để `-1` để tự assign |
| `listeners` | `PLAINTEXT://:9092` | `SASL_SSL://0.0.0.0:9093,PLAINTEXT://127.0.0.1:9092` | Địa chỉ broker bind để lắng nghe kết nối | `0.0.0.0` = lắng nghe tất cả interface |
| `advertised.listeners` | Giống `listeners` | External IP/hostname broker | Địa chỉ được quảng cáo cho client trong Metadata Response | **Bẫy**: Trong Docker/K8s phải set đúng external address. Nếu set sai client sẽ nhận địa chỉ không reach được |
| `num.network.threads` | `3` | `3-8` tùy connection count | Số thread xử lý network request | Tăng khi có nhiều concurrent client connection |
| `num.io.threads` | `8` | Bằng số disk hoặc `2x CPU` | Số thread xử lý disk I/O | Tăng khi disk I/O là bottleneck. Kiểm tra `RequestHandlerAvgIdlePercent` |
| `socket.send.buffer.bytes` | `102400` (100KB) | `1048576` (1MB) cho mạng bandwidth cao | TCP send buffer size | Tăng cho high-bandwidth network |
| `socket.receive.buffer.bytes` | `102400` (100KB) | `1048576` (1MB) | TCP receive buffer size | |
| `socket.request.max.bytes` | `104857600` (100MB) | Giữ default | Kích thước tối đa của một socket request | |
| `log.dirs` | `/tmp/kafka-logs` | Ổ đĩa riêng cho Kafka | Thư mục lưu partition log. Nhiều path = nhiều disk, Kafka phân phối partition đều | Dùng SSD. Mount với `noatime`. Tách khỏi OS disk |
| `message.max.bytes` | `1048588` (~1MB) | `1MB` hoặc tăng đồng bộ với client | Kích thước message tối đa broker chấp nhận | Phải set đồng bộ: `message.max.bytes` (broker) = `max.request.size` (producer) = `max.partition.fetch.bytes` (consumer) |
| `replica.fetch.max.bytes` | `1048576` (1MB) | Phải `>= message.max.bytes` | Kích thước tối đa follower fetch từ leader mỗi request | **Bẫy**: Nếu nhỏ hơn `message.max.bytes` thì follower không thể replicate message lớn |
| `replica.lag.time.max.ms` | `30000` (30s) | `30000` | Follower bị loại khỏi ISR nếu không fetch từ leader trong khoảng thời gian này | Giảm để phát hiện lag sớm hơn nhưng tăng ISR churn. Config này là tiêu chí duy nhất để loại khỏi ISR từ Kafka 2.6+ |
| `unclean.leader.election.enable` | `false` | **`false`** | Cho phép replica out-of-sync được bầu làm leader khi ISR rỗng | **Bẫy**: Set `true` để tăng availability nhưng **có thể mất data**. Chỉ dùng khi availability quan trọng hơn durability |
| `auto.leader.rebalance.enable` | `true` | `true` | Tự động phân phối lại partition leadership để tránh skew | Disable nếu muốn kiểm soát hoàn toàn leader assignment |
| `leader.imbalance.check.interval.seconds` | `300` (5 phút) | `300` | Tần suất kiểm tra và rebalance leadership | |
| `leader.imbalance.per.broker.percentage` | `10` | `10` | % mất cân bằng leadership trigger rebalance tự động | |
| `default.replication.factor` | `1` | **`3`** | RF mặc định khi tạo topic mà không chỉ định | **Bẫy**: Default là 1 → không có redundancy. Luôn set 3 cho production |
| `min.insync.replicas` | `1` | **`2`** | Số replica tối thiểu trong ISR để broker chấp nhận write (chỉ tác dụng với `acks=all`) | **Bẫy lớn nhất**: Default là 1 nghĩa là `acks=all` không khác `acks=1`. Phải set `2` cho durability thực sự. KHÔNG set bằng RF |
| `offsets.topic.replication.factor` | `3` | `3` | RF của topic `__consumer_offsets` | Phải <= số broker. Nếu tăng sau thì phải reassign thủ công |
| `transaction.state.log.replication.factor` | `3` | `3` | RF của topic `__transaction_state` | Quan trọng cho EOS reliability |
| `transaction.state.log.min.isr` | `2` | `2` | `min.insync.replicas` cho `__transaction_state` | |
| `num.partitions` | `1` | `12-24` | Số partition mặc định khi tạo topic | Rule of thumb: bắt đầu 12-24, scale dựa trên consumer lag |
| `auto.create.topics.enable` | `true` | `false` (production) | Tự động tạo topic khi producer gửi đến topic không tồn tại | Disable ở production để tránh typo tạo topic rác |
| `delete.topic.enable` | `true` | `true` | Cho phép xóa topic | |
| `broker.rack` | `null` | AZ/rack label (ví dụ: `us-east-1a`) | Nhãn rack/AZ để Kafka phân phối replica đều qua các rack | Quan trọng cho rack-aware replication. Cần set trên tất cả broker |
| `log.retention.hours` | `168` (7 ngày) | `168` hoặc theo yêu cầu | Thời gian giữ message. Sau đó segment sẽ bị xóa | Chú ý dung lượng disk: `partitions × throughput × retention` |
| `log.retention.bytes` | `-1` (vô hạn) | Set theo capacity | Dung lượng tối đa per partition. `-1` = không giới hạn | Kết hợp với `log.retention.hours`: whichever first |
| `log.segment.bytes` | `1073741824` (1GB) | `1GB` | Kích thước tối đa của một active segment trước khi roll | Segment nhỏ hơn = rolling nhanh hơn nhưng nhiều file hơn |
| `log.roll.hours` | `168` (7 ngày) | `168` | Thời gian tối đa của một segment (kể cả chưa đầy) | |
| `log.index.size.max.bytes` | `10485760` (10MB) | `10MB` | Kích thước tối đa của `.index` file | |
| `log.index.interval.bytes` | `4096` | `4096` | Cứ sau bao nhiêu bytes append thì ghi thêm một entry vào sparse index | |
| `log.cleaner.enable` | `true` | `true` | Bật log compaction background thread | Cần thiết cho topic với `cleanup.policy=compact` |
| `log.cleaner.threads` | `1` | `2-4` nếu nhiều compact topic | Số log cleaner thread | |
| `log.cleaner.min.cleanable.ratio` | `0.5` | `0.5` | Tỉ lệ dirty/total bytes để trigger compaction | Giảm để compact thường xuyên hơn (tốn CPU hơn) |
| `allow.everyone.if.no.acl.found` | `true` | **`false`** (production với ACL) | Cho phép mọi truy cập nếu không có ACL nào match | Bắt buộc set `false` khi bật ACL authorization |
| `authorizer.class.name` | `""` | `org.apache.kafka.metadata.authorizer.StandardAuthorizer` (KRaft) | Class xử lý authorization | Set để bật ACL enforcement |
| `super.users` | `""` | `User:kafka-admin` | User được bypass mọi ACL check | |
| `controlled.shutdown.enable` | `true` | `true` | Broker graceful shutdown: migrate partition leadership trước khi dừng | |
| `controlled.shutdown.max.retries` | `3` | `3` | Số lần retry controlled shutdown | |

---

## 4. Topic Config (per-topic settings)

> Có thể set lúc tạo topic hoặc override sau bằng `kafka-configs.sh --entity-type topics`

| Config | Default (lấy từ broker) | Giá trị khuyến nghị | Ý nghĩa | Lưu ý quan trọng |
|--------|------------------------|---------------------|---------|------------------|
| `replication.factor` | `default.replication.factor` | **`3`** | Số bản sao mỗi partition. Phải `<= số broker` | RF=3 là sweet spot: chịu 1 broker chết, không tốn quá nhiều storage |
| `min.insync.replicas` | `min.insync.replicas` broker | **`2`** | Số replica tối thiểu trong ISR để chấp nhận write (với `acks=all`) | Topic-level override broker config. Fault tolerance = RF - min.isr = 1 |
| `num.partitions` | `num.partitions` broker | Tùy throughput, min `12-24` | Số partition của topic | Có thể tăng, **không bao giờ giảm được** |
| `cleanup.policy` | `delete` | `delete` hoặc `compact` hoặc `delete,compact` | `delete`: xóa theo thời gian/size. `compact`: giữ giá trị cuối cùng per key | `compact` biến topic thành key-value store |
| `retention.ms` | `604800000` (7 ngày) | Theo yêu cầu replay | Thời gian giữ message. `-1` = vô hạn | |
| `retention.bytes` | `-1` (vô hạn) | Set theo capacity | Dung lượng tối đa per partition | Kết hợp với `retention.ms` |
| `compression.type` | `producer` | `producer` | Thuật toán nén. `producer` = giữ nguyên compression của producer. Hoặc set broker-level compression | Giữ `producer` để zero-copy vẫn hoạt động |
| `max.message.bytes` | `message.max.bytes` broker | `1MB` hoặc tăng nếu cần | Kích thước message tối đa cho topic này | Topic-level override. Phải đồng bộ với producer `max.request.size` |
| `message.timestamp.type` | `CreateTime` | `CreateTime` | Loại timestamp: `CreateTime` (thời điểm producer tạo) hoặc `LogAppendTime` (broker nhận) | `LogAppendTime` hữu ích khi không tin tưởng producer clock |
| `segment.bytes` | `log.segment.bytes` broker | `1GB` | Kích thước tối đa active segment trước khi roll | |
| `segment.ms` | `log.roll.hours` broker | `604800000` (7 ngày) | Thời gian tối đa của một segment | |
| `delete.retention.ms` | `86400000` (24h) | `86400000` | Thời gian giữ tombstone record (value=null) trước khi xóa hoàn toàn khi compaction | Consumer cần consume tombstone trước khi bị xóa để biết key đã bị delete |
| `min.cleanable.dirty.ratio` | `0.5` | `0.5` | Tỉ lệ dirty/total để trigger compaction cho topic này | |
| `unclean.leader.election.enable` | `unclean.leader.election.enable` broker | `false` | Override broker setting cho topic cụ thể | Critical topic nên để `false` dù broker default là gì |
| `preallocate` | `false` | `false` | Preallocate disk space cho log segment | Hữu ích trên Windows, không cần thiết trên Linux |

---

## 5. KRaft / Controller Config

| Config | Default | Giá trị khuyến nghị | Ý nghĩa | Lưu ý quan trọng |
|--------|---------|---------------------|---------|------------------|
| `process.roles` | *(bắt buộc với KRaft)* | `broker`, `controller`, hoặc `broker,controller` | Vai trò của node này trong KRaft cluster | `broker,controller` = Combined mode (dev). `controller` = Dedicated mode (production lớn) |
| `node.id` | *(bắt buộc với KRaft)* | Integer duy nhất per node | ID node trong KRaft cluster. Thay thế `broker.id` | Phải unique trên cả broker và controller node |
| `controller.quorum.voters` | *(bắt buộc với KRaft)* | `1@ctrl1:9093,2@ctrl2:9093,3@ctrl3:9093` | Danh sách tất cả Voter trong Controller Quorum | Dùng 3 hoặc 5 voter (số lẻ) để bầu cử có majority. Công thức `2N+1` |
| `controller.listener.names` | `CONTROLLER` | `CONTROLLER` | Tên listener dành cho controller communication | |
| `listeners` (controller node) | *(bắt buộc)* | `CONTROLLER://0.0.0.0:9093` | Listener cho controller-to-controller và broker-to-controller | |
| `metadata.log.dir` | Giống `log.dirs` | Ổ riêng nếu có thể | Thư mục lưu `__cluster_metadata` log | Tách với data log để tránh I/O contention |
| `metadata.log.max.record.bytes.between.snapshots` | `20971520` (20MB) | `20MB` | Kích thước metadata log tích lũy trước khi snapshot | |
| `metadata.max.retention.bytes` | `104857600` (100MB) | `100MB` | Giới hạn dung lượng metadata log | |
| `metadata.max.retention.ms` | `604800000` (7 ngày) | `7 ngày` | Thời gian giữ metadata log | |
| `controller.quorum.election.timeout.ms` | `1000` | `1000` | Timeout trước khi Voter tự ứng cử làm controller | Randomized để tránh split vote |
| `controller.quorum.fetch.timeout.ms` | `2000` | `2000` | Timeout khi Voter fetch từ Active Controller | |
| `controller.quorum.request.timeout.ms` | `2000` | `2000` | Timeout cho quorum request | |

---

## 6. Security Config

### SSL/TLS

| Config | Default | Giá trị khuyến nghị | Ý nghĩa | Lưu ý quan trọng |
|--------|---------|---------------------|---------|------------------|
| `security.protocol` | `PLAINTEXT` | `SASL_SSL` (production) | Protocol bảo mật: `PLAINTEXT`, `SSL`, `SASL_PLAINTEXT`, `SASL_SSL` | `SASL_SSL` = TLS + SASL authentication. Nên dùng cho production |
| `ssl.keystore.location` | *(bắt buộc với SSL)* | Đường dẫn tuyệt đối đến `.jks` hoặc `.p12` | Keystore chứa certificate và private key của broker | |
| `ssl.keystore.password` | *(bắt buộc với SSL)* | *(secret)* | Password của keystore | Dùng secret management (Vault, K8s Secret) |
| `ssl.key.password` | *(bắt buộc với SSL)* | *(secret)* | Password của private key trong keystore | |
| `ssl.truststore.location` | *(bắt buộc với SSL)* | Đường dẫn đến truststore | Truststore chứa CA certificate để verify phía kia | |
| `ssl.truststore.password` | *(bắt buộc với SSL)* | *(secret)* | Password của truststore | |
| `ssl.client.auth` | `none` | `required` (mutual TLS) hoặc `none` | Broker có yêu cầu client certificate không. `none`, `requested`, `required` | `required` = mutual TLS (mTLS): client cũng phải có cert |
| `ssl.enabled.protocols` | `TLSv1.2,TLSv1.3` | `TLSv1.3,TLSv1.2` | Danh sách TLS version được phép | Loại bỏ TLS 1.0 và 1.1 |
| `ssl.protocol` | `TLSv1.3` | `TLSv1.3` | TLS version dùng để tạo SSLContext | |

### SASL

| Config | Default | Giá trị khuyến nghị | Ý nghĩa | Lưu ý quan trọng |
|--------|---------|---------------------|---------|------------------|
| `sasl.mechanism` | `GSSAPI` | `SCRAM-SHA-512` (production) hoặc `OAUTHBEARER` (cloud) | SASL mechanism phía client sử dụng | |
| `sasl.enabled.mechanisms` | `GSSAPI` | `SCRAM-SHA-512` hoặc `PLAIN,SCRAM-SHA-512` | Danh sách SASL mechanism broker hỗ trợ | Có thể hỗ trợ nhiều mechanism cùng lúc |
| `sasl.mechanism.inter.broker.protocol` | `GSSAPI` | Match với `sasl.enabled.mechanisms` | SASL mechanism dùng cho giao tiếp giữa các broker | |
| `sasl.jaas.config` | *(bắt buộc với SASL)* | Xem ví dụ trong file 06-ops-security.md | JAAS config string cho SASL login | Chứa credentials, nên dùng secret management |
| `sasl.kerberos.service.name` | `kafka` | `kafka` | Tên service Kerberos principal của broker | Chỉ cần với GSSAPI/Kerberos |
| `sasl.login.refresh.buffer.seconds` | `300` (5 phút) | `300` | Thời gian trước khi token hết hạn để refresh | Dành cho OAUTHBEARER |
| `sasl.login.refresh.min.period.seconds` | `60` | `60` | Thời gian tối thiểu giữa hai lần refresh | |

### ACL / Authorization

| Config | Default | Giá trị khuyến nghị | Ý nghĩa | Lưu ý quan trọng |
|--------|---------|---------------------|---------|------------------|
| `authorizer.class.name` | `""` (không có) | `org.apache.kafka.metadata.authorizer.StandardAuthorizer` | Class xử lý authorization. Để trống = không ACL | `StandardAuthorizer` cho KRaft. `AclAuthorizer` cho ZooKeeper mode (deprecated) |
| `allow.everyone.if.no.acl.found` | `true` | **`false`** | Cho phép tất cả khi không có ACL nào match | Phải `false` khi dùng ACL. `true` khiến ACL vô nghĩa |
| `super.users` | `""` | `User:kafka-admin` | User được bypass tất cả ACL check | |

---

## 7. Config Matrix — Kịch bản thường gặp

| Kịch bản | Producer Config | Consumer Config | Broker/Topic Config | Đánh đổi |
|----------|----------------|----------------|---------------------|----------|
| **Max Throughput** | `acks=1`, `batch.size=131072`, `linger.ms=20`, `compression.type=lz4`, `max.in.flight=5` | `fetch.min.bytes=1048576`, `fetch.max.wait.ms=500`, `max.poll.records=2000`, `enable.auto.commit=true` | `num.io.threads=16`, `num.network.threads=8` | Latency cao hơn, durability thấp hơn |
| **Exactly-Once (EOS)** | `enable.idempotence=true`, `transactional.id=<unique>`, `acks=all`, `max.in.flight<=5` | `isolation.level=read_committed`, `enable.auto.commit=false` | `replication.factor=3`, `min.insync.replicas=2`, `transaction.state.log.replication.factor=3`, `transaction.state.log.min.isr=2` | Throughput giảm ~3-20%, latency cao hơn |
| **At-Least-Once Durable** | `acks=all`, `enable.idempotence=true`, `retries=MAX_VALUE`, `max.in.flight<=5` | `enable.auto.commit=false`, `isolation.level=read_committed` (nếu có txn) | `replication.factor=3`, `min.insync.replicas=2`, `unclean.leader.election.enable=false` | Có thể xử lý duplicate, consumer phải idempotent |
| **Low Latency** | `acks=1`, `linger.ms=0`, `batch.size=16384`, `compression.type=none`, `max.in.flight=1` | `fetch.min.bytes=1`, `fetch.max.wait.ms=0`, `max.poll.records=10` | `num.replica.fetchers=4` | Throughput thấp hơn, durability thấp hơn |
| **High Durability (Critical Topic)** | `acks=all`, `enable.idempotence=true`, `retries=MAX_VALUE`, `delivery.timeout.ms=300000` | `enable.auto.commit=false`, `isolation.level=read_committed` | `replication.factor=5`, `min.insync.replicas=3`, `unclean.leader.election.enable=false`, `compression.type=producer` | Chi phí storage 5x, latency cao nhất |

---

## 8. Những config hay bị set sai (Gotchas)

### 1. Bẫy `min.insync.replicas` default = 1

**Vấn đề:**
```
# Setup tưởng là "durability cao" nhưng thực ra không bảo vệ gì:
acks=all
replication.factor=3
min.insync.replicas=1  ← KHÔNG SET, dùng default

→ Cả 2 follower đều chậm → bị kick khỏi ISR
→ ISR = [Leader only]
→ ISR size (1) >= min.insync.replicas (1) → vẫn OK
→ acks=all chờ "tất cả ISR" = chỉ mình leader → ACK ngay
→ Leader crash trước khi follower replicate → MẤT DATA
```

**Fix:**
```properties
replication.factor=3
min.insync.replicas=2   # PHẢI set tay, đừng dùng default
acks=all
```

Fault tolerance = RF - min.isr = 3 - 2 = **1 broker có thể chết**.

**Bẫy phụ:** Đừng set `min.insync.replicas = replication.factor` — 1 broker chết là toàn bộ write fail (zero fault tolerance).

---

### 2. Bẫy `enable.auto.commit = true`

**Vấn đề:**
```
Consumer fetch 100 records
→ Auto-commit chạy sau 5s → commit offset
→ Consumer xử lý record thứ 60 thì crash
→ Restart: bắt đầu từ record 101 (sau committed offset)
→ Record 61-100 bị BỎ QUA HOÀN TOÀN
```

**Fix:**
```java
// Tắt auto-commit
props.put("enable.auto.commit", "false");

// Commit thủ công sau khi xử lý xong
try {
    for (ConsumerRecord<?, ?> record : records) {
        process(record); // business logic
    }
    consumer.commitSync(); // commit sau khi xử lý toàn bộ batch
} catch (Exception e) {
    // Không commit → sẽ reprocess sau khi restart (at-least-once)
}
```

---

### 3. Bẫy `max.in.flight.requests.per.connection` với retry

**Vấn đề:**
```
max.in.flight.requests.per.connection=5 (default)
+ retry=true
+ enable.idempotence=false

→ Batch A (msg 1-10) gửi fail
→ Batch B (msg 11-20) gửi success ngay sau
→ Batch A retry thành công
→ Thứ tự trong log: [B][A] → ĐẢO NGƯỢC
```

**Fix (chọn một):**
```properties
# Option 1: Ép sequential (giảm throughput)
max.in.flight.requests.per.connection=1

# Option 2: Dùng idempotent producer (khuyến nghị)
enable.idempotence=true
# → tự set max.in.flight <= 5 VÀ broker deduplicate
# → ordering được đảm bảo dù max.in.flight=5
```

**Lưu ý thêm:** `max.in.flight.requests.per.connection > 5` sẽ **vô hiệu hóa idempotence** dù đã set `enable.idempotence=true`. Kafka sẽ throw `ConfigException` từ Kafka 3.0+.

---

### 4. Nhầm lẫn `session.timeout.ms` vs `max.poll.interval.ms`

**Vấn đề — hai timeout hoàn toàn khác nhau:**

| | `session.timeout.ms` | `max.poll.interval.ms` |
|---|---|---|
| Kiểm soát gì | Heartbeat thread | Poll thread (business logic) |
| Broker phát hiện sự kiện gì | Consumer bị mất kết nối / dead | Consumer xử lý quá chậm |
| Default | `10s` (KRaft: `45s`) | `5 phút` |
| Nên tăng khi nào | GC pause thường xuyên | Business logic xử lý lâu |

```
Kịch bản sai lầm:
Consumer xử lý mỗi batch mất 10 phút (gọi external API chậm)
→ Heartbeat vẫn gửi bình thường (session.timeout.ms không vi phạm)
→ Nhưng không gọi poll() trong 5 phút → max.poll.interval.ms bị vượt
→ Consumer bị EVICT dù heartbeat vẫn OK
```

**Fix:**
```properties
# Option 1: Tăng max.poll.interval.ms
max.poll.interval.ms=600000   # 10 phút nếu thực sự cần

# Option 2: Giảm lượng record mỗi poll để xử lý nhanh hơn
max.poll.records=10
```

---

### 5. Bẫy `unclean.leader.election.enable`

**Vấn đề:**
```
RF=3, ISR=[L, F1, F2]
→ F1 và F2 crash → ISR=[L]
→ L crash → ISR=[]

unclean.leader.election.enable=true (đôi khi set để tăng availability):
→ F1 hoặc F2 được bầu làm leader dù chưa replicate đầy đủ
→ Message tồn tại trên L nhưng chưa replicate sang F1/F2 → MẤT DATA
→ Consumer đã đọc message đó từ trước → data "biến mất"
```

**Luật:** Mặc định là `false` — **không bao giờ set `true`** cho production topic quan trọng. Chỉ cân nhắc khi:
- Availability quan trọng hơn durability (ví dụ: metric, log aggregation chấp nhận mất vài message)
- Topic đó là non-critical và mất data có thể chấp nhận được

---

### 6. Bẫy `advertised.listeners` trong container/cloud

**Vấn đề:**
```
Broker trong Docker container:
  bind: 0.0.0.0:9092
  container IP: 172.17.0.2

advertised.listeners=PLAINTEXT://172.17.0.2:9092  ← sai

→ Client ngoài container nhận metadata: "kết nối vào 172.17.0.2"
→ 172.17.0.2 không reachable từ ngoài → TimeoutException
```

**Fix:**
```properties
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://192.168.1.100:9092  # external IP
```

---

### 7. Bẫy `isolation.level` với hanging transaction

**Vấn đề:**
```
Consumer với isolation.level=read_committed
→ Producer mở transaction và crash mà không commit/abort
→ LSO (Last Stable Offset) bị đóng băng tại transaction đang mở
→ Consumer không thể đọc message mới dù producer khác vẫn đang produce phía trên LSO
→ Consumer lag tăng vô hạn
→ Log cleaner không thể advance → disk đầy
```

**Fix:**
```bash
# Tìm hanging transaction
kafka-transactions.sh --bootstrap-server :9092 find-hanging --broker 0

# Abort transaction thủ công
kafka-transactions.sh --bootstrap-server :9092 abort \
  --topic <topic> --partition <p> --start-offset <offset>
```

**Phòng ngừa:** Đặt `transaction.timeout.ms` hợp lý (mặc định 60s) để broker tự abort transaction treo.

---

### 8. Bẫy message size không đồng bộ

**Vấn đề:**
```
message.max.bytes=1MB (broker default)
max.request.size=10MB (producer)

→ Producer gửi message 5MB
→ Broker nhận ProduceRequest → từ chối vì > message.max.bytes
→ RecordTooLargeException

Hoặc ngược lại:
message.max.bytes=10MB (broker)
max.partition.fetch.bytes=1MB (consumer)
→ Broker có message 5MB
→ Consumer fetch → nhận message nhưng không thể đọc → stuck
```

**Fix:** Ba config này phải đồng bộ:
```properties
# Broker
message.max.bytes=10485760       # 10MB
replica.fetch.max.bytes=10485760 # phải >= message.max.bytes

# Topic (override broker)
max.message.bytes=10485760

# Producer
max.request.size=10485760

# Consumer
max.partition.fetch.bytes=10485760
fetch.max.bytes=52428800         # >= max.partition.fetch.bytes
```

---

### 9. Bẫy `auto.offset.reset=latest` khi deploy consumer mới

**Vấn đề:**
```
Producer đang produce vào topic "orders" từ lâu
→ Deploy consumer group "order-processor" lần đầu
→ auto.offset.reset=latest (default)
→ Consumer chỉ đọc message mới từ thời điểm deploy
→ Hàng nghìn order cũ bị BỎ QUA HOÀN TOÀN
```

**Fix:**
```properties
# Nếu cần xử lý từ đầu:
auto.offset.reset=earliest

# Nếu chỉ muốn xử lý message mới (intentional):
auto.offset.reset=latest   # nhưng phải biết mình đang bỏ qua message cũ
```

---

### 10. Bẫy rebalance storm với `session.timeout.ms` thấp

**Vấn đề:**
```
session.timeout.ms=10s (default cũ)
JVM Full GC xảy ra, pause 15s
→ Heartbeat bị chặn trong GC
→ Broker: consumer dead → evict → trigger rebalance
→ Rebalance stop-the-world → GC đang chạy lại
→ Heartbeat bị chặn lại → evict lại → rebalance lại
→ Vòng lặp vô tận, không có message nào được xử lý
```

**Fix:**
```properties
session.timeout.ms=45000      # Kafka 3.0+ default, đủ chịu GC pause
heartbeat.interval.ms=15000   # 1/3 session.timeout
```

Và tune JVM:
```
-XX:+UseG1GC
-XX:MaxGCPauseMillis=2000
```
