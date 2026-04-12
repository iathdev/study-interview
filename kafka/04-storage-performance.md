# Storage Internals và Performance Tuning

> Tổng hợp từ tài liệu chính thức, Confluent engineering blog, Strimzi, và các nguồn phỏng vấn senior.

---

## 5. Performance Tuning

### 5.1 Tuning Producer

| Config | Mặc định | Hướng tuning | Ghi chú |
|--------|----------|-------------|---------|
| `batch.size` | 16384 (16KB) | Tăng lên 65536-131072 cho throughput cao | Batch lớn hơn = compression tốt hơn, latency cao hơn |
| `linger.ms` | 0ms (Kafka < 4.0), 5ms (Kafka 4.0+) | Đặt 5-20ms cho throughput | Chờ đầy batch trước khi gửi |
| `compression.type` | none | `lz4` (tốc độ) hoặc `zstd` (ratio+tốc độ) | Giảm network I/O và disk broker |
| `max.in.flight.requests.per.connection` | 5 | Giữ <= 5 với idempotence | >5 vô hiệu hóa idempotence |
| `buffer.memory` | 33554432 (32MB) | Tăng nếu `send()` block thường xuyên | Tổng memory để buffer record |
| `acks` | all (Kafka 3.0+) | `1` cho latency thấp hơn nếu chấp nhận được | Đánh đổi durability vs tốc độ |

**Chọn compression:**
- `gzip`: Tỉ lệ nén tốt nhất, CPU cao nhất. Tốt cho cold storage topic.
- `snappy`: Tỉ lệ trung bình, CPU thấp. Cân bằng tốt.
- `lz4`: Nhanh nhất, tỉ lệ thấp nhất. Tốt cho high-throughput, latency-sensitive.
- `zstd`: Cân bằng ratio/tốc độ tốt nhất (khuyến nghị mặc định Kafka 3.0+).

---

### 5.2 Tuning Consumer

| Config | Mặc định | Hướng tuning | Ghi chú |
|--------|----------|-------------|---------|
| `fetch.min.bytes` | 1 | Tăng lên 1MB+ cho throughput | Broker chờ đến khi có đủ data này |
| `fetch.max.wait.ms` | 500ms | Tune cùng `fetch.min.bytes` | Thời gian chờ tối đa dù chưa đủ min bytes |
| `max.poll.records` | 500 | Tăng cho batch processing | Nhiều record mỗi poll = throughput cao hơn |
| `fetch.max.bytes` | 52428800 (50MB) | Tăng cho message lớn | Tổng data trả về mỗi fetch |
| `max.partition.fetch.bytes` | 1048576 (1MB) | Match với kích thước batch điển hình | Giới hạn per-partition trong mỗi fetch response |

---

### 5.3 Tuning Broker

| Config | Mặc định | Ghi chú |
|--------|----------|---------|
| `num.network.threads` | 3 | Tăng khi connection count cao |
| `num.io.threads` | 8 | Tăng khi disk I/O cao (đặt bằng số disk hoặc 2x CPU) |
| `socket.send.buffer.bytes` | 102400 | Tăng cho mạng bandwidth cao |
| `log.segment.bytes` | 1GB | Segment nhỏ hơn = log rolling nhanh hơn nhưng nhiều file hơn |
| `log.retention.hours` | 168 (7 ngày) | Đặt theo yêu cầu replay |
| `replica.fetch.max.bytes` | 1MB | Phải >= `message.max.bytes` |
| `auto.leader.rebalance.enable` | true | Ngăn leadership skew |
| `unclean.leader.election.enable` | false | Giữ false trừ khi availability > durability |

**Tuning OS:**
- Dùng filesystem XFS hoặc ext4
- Mount với `noatime` để tránh write amplification từ access time update
- Đặt `vm.swappiness = 1` (tối thiểu swapping, dựa vào page cache)
- Tăng `ulimit -n` (open file descriptors) lên 100000+
- Dùng network interface 10 GbE
- SSD cho broker log (hoặc RAID-10 HDD cho cost/capacity)
- Cấp JVM heap 4-6GB; phần còn lại để OS page cache (Kafka phụ thuộc nặng vào nó)

---

### 5.4 Chiến Lược Số Lượng Partition

**H: Topic nên có bao nhiêu partition?**

Rule of thumb: `partitions = max(throughput_cần / throughput_mỗi_partition, parallelism_mong_muốn)`

Cân nhắc:
- Nhiều partition hơn → parallelism cao hơn cho consumer
- Nhiều partition hơn → nhiều file handle hơn trên broker
- Nhiều partition hơn → controller failover chậm hơn (nhiều state cần transfer hơn)
- Nhiều partition hơn → end-to-end latency cho replication có thể TĂNG
- **Có thể tăng nhưng không bao giờ giảm partition count** — lên kế hoạch trước

Cho hầu hết workload production: bắt đầu với 12-24 partition mỗi topic. Scale up dựa trên consumer lag metric.

---

## 9. Storage Internals — Lưu Trữ Nội Bộ

### 9.1 Log Segments — Cấu Trúc File Trên Disk

**H: Kafka lưu message xuống disk như thế nào về mặt vật lý?**

Mỗi partition là một **directory** trên filesystem. Bên trong directory đó có một chuỗi **log segment**, mỗi segment gồm ba file chính:

```
/kafka-logs/my-topic-0/
    00000000000000000000.log        ← segment data
    00000000000000000000.index      ← sparse offset index
    00000000000000000000.timeindex  ← sparse time index
    00000000000000001024.log        ← segment mới (base offset = 1024)
    00000000000000001024.index
    00000000000000001024.timeindex
    leader-epoch-checkpoint
```

**Tên file** là **base offset** của segment đó, được zero-padded thành 20 chữ số. File `.log` cuối cùng là **active segment** — segment duy nhất đang được append. Các segment trước đó là **inactive** (immutable).

**Rolling segment**: Segment hiện tại được rolled (đóng lại, tạo segment mới) khi:
- Vượt quá `log.segment.bytes` (mặc định 1GB)
- Vượt quá `log.roll.ms` / `log.roll.hours` (mặc định 7 ngày)
- Index đầy (tính theo `log.index.size.max.bytes`, mặc định 10MB)

---

**H: Format của file `.log` (segment data) là gì?**

File `.log` là chuỗi các **record batch** (trước đây gọi là "message set") được append liên tiếp. Mỗi record batch có header:

```
RecordBatch:
  baseOffset:         int64    ← offset của record đầu tiên trong batch
  batchLength:        int32    ← tổng byte của batch (từ field tiếp theo đến cuối)
  partitionLeaderEpoch: int32  ← dùng để detect log divergence
  magic:              int8     ← version (hiện tại = 2)
  crc:                uint32   ← CRC32C checksum toàn bộ batch
  attributes:         int16    ← compression codec, timestamp type, transactional, control
  lastOffsetDelta:    int32    ← offset delta của record cuối trong batch
  baseTimestamp:      int64    ← timestamp của record đầu
  maxTimestamp:       int64    ← timestamp của record cuối
  producerId:         int64    ← PID (cho idempotent/transactional producer)
  producerEpoch:      int16    ← epoch của producer
  baseSequence:       int32    ← sequence number của record đầu
  records:            []Record ← danh sách record
```

Mỗi `Record` bên trong batch:
```
Record:
  length:         varint
  attributes:     int8
  timestampDelta: varint  ← delta so với baseTimestamp của batch
  offsetDelta:    varint  ← delta so với baseOffset của batch
  keyLength:      varint
  key:            bytes
  valueLength:    varint
  value:          bytes
  headers:        []Header
```

**Tại sao batch thay vì từng message?** Compression và CRC được áp dụng ở cấp batch, tiết kiệm overhead. Offset/timestamp dùng delta encoding nên nhỏ gọn hơn.

---

**H: File `.index` (offset index) hoạt động như thế nào?**

File `.index` là **sparse index** ánh xạ từ offset tương đối → vị trí vật lý (byte offset) trong file `.log`. Không phải mọi offset đều có entry — Kafka chỉ ghi một entry sau mỗi `log.index.interval.bytes` (mặc định 4096 bytes) được append.

Mỗi entry trong `.index` là 8 bytes:
```
relativeOffset: int32  ← offset - baseOffset của segment
position:       int32  ← byte position trong file .log
```

**Cách tìm kiếm offset cụ thể (ví dụ offset 1050):**
1. Kafka xác định segment chứa offset 1050 (tìm segment có base offset <= 1050)
2. Binary search trong `.index` để tìm entry có relativeOffset gần nhất và <= (1050 - baseOffset)
3. Seek đến byte position tương ứng trong `.log`
4. Scan tuần tự trong `.log` cho đến khi tìm đúng offset 1050

Vì index là sparse (không đầy đủ), bước 4 phải scan tối đa `log.index.interval.bytes` = 4096 bytes. Trên thực tế rất nhanh.

---

**H: File `.timeindex` (time index) khác gì `.index`?**

File `.timeindex` ánh xạ timestamp → offset. Mỗi entry 12 bytes:
```
timestamp:      int64  ← max timestamp của batch tính đến điểm này
relativeOffset: int32  ← offset tương ứng trong segment
```

Dùng cho `offsetsForTimes()` API — consumer muốn seek đến một thời điểm cụ thể (ví dụ: "cho tôi consume từ 2 giờ trước"). Kafka:
1. Binary search `.timeindex` để tìm entry gần nhất với timestamp yêu cầu
2. Dùng offset từ `.timeindex` để lookup `.index`
3. Seek đến vị trí vật lý trong `.log`

**H: Có cơ chế lấy offset theo logic cụ thể không? Ví dụ theo timestamp?**

Có — `offsetsForTimes()` API cho phép tìm offset tương ứng với một timestamp bất kỳ:

```java
Map<TopicPartition, Long> query = new HashMap<>();
query.put(new TopicPartition("my-topic", 0), timestamp); // unix ms

Map<TopicPartition, OffsetAndTimestamp> result = consumer.offsetsForTimes(query);

OffsetAndTimestamp offsetAndTs = result.get(partition);
consumer.seek(partition, offsetAndTs.offset()); // seek đến đúng vị trí
```

Trả về offset đầu tiên có timestamp **>= giá trị truyền vào**.

Ngoài timestamp còn có các cách seek khác:

| Method | Dùng khi |
|---|---|
| `seek(partition, offset)` | Biết chính xác offset |
| `seekToBeginning()` | Replay từ đầu |
| `seekToEnd()` | Bỏ qua hết, đọc từ mới nhất |
| `offsetsForTimes()` | Biết thời điểm, không biết offset |

**Các bài toán thực tế dùng `offsetsForTimes()`:**

- **Replay sau incident**: Production lỗi lúc 3:00 PM → `offsetsForTimes(2:55 PM)` để reprocess lại từ trước thời điểm lỗi
- **Disaster recovery**: Service xử lý sai data trong 1 tiếng → tìm offset lúc bắt đầu lỗi → reprocess lại đúng
- **Consumer lag theo thời gian**: Muốn biết consumer lag bao nhiêu phút thay vì số message → `offsetsForTimes(now)` lấy offset hiện tại → so sánh với offset consumer đang ở → tính ra lag tính bằng thời gian
- **Time-based windowing thủ công**: Chỉ xử lý data của ngày hôm qua → `offsetsForTimes(start of yesterday)` và `offsetsForTimes(end of yesterday)` → seek + consume đúng khoảng đó

---

### 9.2 Zero-Copy — `sendfile` Syscall

**H: Zero-copy trong Kafka là gì? Tại sao quan trọng cho performance?**

Khi consumer fetch message, Kafka cần đọc data từ disk và gửi qua network. Không có zero-copy, path thông thường là:

```
Disk → Kernel buffer (page cache)
     → User space (JVM heap) — copy 1
     → Kernel socket buffer  — copy 2
     → NIC                   — DMA
```

Có **2 lần copy dư thừa** và **2 lần context switch** (kernel ↔ user space).

Với zero-copy (Linux `sendfile(2)` syscall):
```
Disk → Kernel buffer (page cache)
     → NIC via DMA            — copy trực tiếp từ page cache, không qua user space
```

**Không có copy vào JVM heap.** Data di chuyển từ page cache đến NIC buffer hoàn toàn trong kernel space. Kafka gọi `FileChannel.transferTo()` trong Java, được JVM map sang `sendfile()` trên Linux.

**Lợi ích thực tế:**
- Giảm CPU usage đáng kể (không cần JVM serialize/deserialize cho việc đơn giản là forwarding)
- Tăng throughput consumer lên **2-4x** so với không có zero-copy
- Bộ nhớ JVM heap không bị áp lực bởi buffer data lớn

**Giới hạn quan trọng**: Zero-copy chỉ hoạt động khi message **không cần transformation** trên broker. Nếu broker cần decrypt (SSL/TLS) hoặc uncompress rồi recompress, zero-copy không áp dụng được — data phải đi qua user space. Đây là lý do Kafka khuyến nghị end-to-end compression (producer compress → broker forward nguyên si → consumer decompress) thay vì broker-level compression.

**SSL/TLS và zero-copy**: Khi bật SSL, Kafka **không thể dùng sendfile** vì cần encrypt data trong user space. Đây là chi phí ẩn của encryption — throughput giảm do extra CPU copy. Có thể giảm thiểu bằng hardware crypto acceleration.

---

### 9.3 Page Cache và Tại Sao Kafka Nhanh

**H: Tại sao Kafka có thể handle throughput rất cao dù ghi vào disk?**

Ba lý do cốt lõi:

**1. Sequential write, không random write**

Kafka chỉ append vào active segment — đây là sequential I/O. Trên HDD, sequential write nhanh hơn random write tới **100-1000x** vì không cần disk seek. Trên SSD, khoảng cách ít hơn nhưng sequential vẫn tốt hơn do giảm write amplification.

**2. Phụ thuộc vào OS page cache**

Kafka không tự quản lý memory cache. Thay vào đó, nó để OS Linux page cache làm việc:
- Write: Ghi vào page cache (nhanh như ghi RAM), OS flush ra disk theo lịch của riêng nó
- Read (hot data): OS phục vụ từ page cache, không cần disk I/O
- Consumer consume message mới: Message vẫn còn trong page cache từ lần write → zero disk read

Đây là lý do Kafka yêu cầu cấp **ít JVM heap, nhiều RAM** — heap lớn gây GC pressure; RAM thừa để OS dùng làm page cache.

```
JVM heap:       4-6 GB   (đủ cho Kafka metadata, không buffer message data)
Page cache:     phần còn lại của RAM (OS tự quản lý)
```

**3. Zero-copy sendfile** (đã phân tích ở trên)
