# Cấu Hình Producer, Delivery Semantics và Exactly-Once

> Tổng hợp từ tài liệu chính thức, Confluent engineering blog, Strimzi, và các nguồn phỏng vấn senior.

---

## 3. Cấu Hình Producer và Delivery Semantics

### 3.1 Acknowledgments (`acks`)

| acks | Hành vi | Durability | Latency |
|------|---------|------------|---------|
| `0` | Fire and forget. Không chờ response. | Thấp nhất (có thể mất data) | Thấp nhất |
| `1` | Chờ leader ghi vào local log. | Trung bình (mất data nếu leader fail trước khi replicate) | Trung bình |
| `all` / `-1` | Chờ tất cả ISR replica acknowledge. | Cao nhất (không mất data nếu `min.insync.replicas` được tuân thủ) | Cao nhất |

**H: Với `acks=1`, có thể mất data không?**

Có. Kịch bản: Producer gửi message → Leader ghi vào log → Leader gửi ACK → Leader crash **trước khi** follower replicate → Kafka bầu follower làm leader mới → Message chưa bao giờ tồn tại trên follower → **Mất data**.

---

### 3.2 Retry Producer và Ordering

**H: Rủi ro của retry với `acks=all` là gì?**

Với retry được bật và `max.in.flight.requests.per.connection > 1`, retry có thể gây **reorder message**:
1. Batch A (offset 10-20) được gửi, fail
2. Batch B (offset 21-30) được gửi và thành công
3. Batch A được retry và thành công
4. Thứ tự log cuối cùng: B rồi A — **lộn thứ tự**

**Fix**: Đặt `max.in.flight.requests.per.connection = 1` để ép gửi tuần tự. Hoặc dùng `enable.idempotence = true` (tự xử lý điều này với tối đa 5 in-flight).

---

### 3.3 Idempotent Producer

**H: Idempotent producer hoạt động như thế nào?**

Bật bằng `enable.idempotence = true`. Kafka tự động đặt:
- `acks = all`
- `retries = Integer.MAX_VALUE`
- `max.in.flight.requests.per.connection <= 5`

Cơ chế:
1. Broker gán mỗi producer một **Producer ID (PID)** duy nhất
2. Producer duy trì **sequence number** per partition, tăng cho mỗi batch
3. Broker theo dõi sequence number cuối cùng per (PID, partition)
4. Khi nhận batch: nếu `seq == last_seq + 1` → chấp nhận; nếu `seq <= last_seq` → duplicate, silently discard; nếu `seq > last_seq + 1` → `OutOfOrderSequenceException`
5. Sequence state được persist trong file `.snapshot` và sống qua broker restart

**Giới hạn**: Idempotence chỉ trong phạm vi **một session**. Producer restart sẽ nhận PID mới. Deduplication cross-partition và cross-session cần transactions API đầy đủ.

---

### 3.4 Tóm Tắt Delivery Semantics

| Semantic | Config | Rủi ro |
|----------|--------|--------|
| **At-most-once** | `acks=0`, không retry, commit trước khi xử lý | Mất message |
| **At-least-once** | `acks=all`, retry được bật, commit sau khi xử lý | Xử lý trùng lặp |
| **Exactly-once** | Idempotent producer + transaction HOẶC Kafka Streams EOS | Không mất, không trùng (trong Kafka) |

---

## 4. Exactly-Once Semantics (EOS) — Phân Tích Sâu

### 4.1 Transactions API

**H: Config nào bật exactly-once cho producer?**

```java
props.put("enable.idempotence", "true");
props.put("transactional.id", "my-transactional-producer-1"); // unique, stable per instance
```

Trong code:
```java
producer.initTransactions();
producer.beginTransaction();
producer.send(record1);
producer.send(record2);
producer.sendOffsetsToTransaction(offsets, consumerGroupMetadata); // cho consume-transform-produce
producer.commitTransaction(); // hoặc abortTransaction() khi có lỗi
```

**H: Transaction coordinator hoạt động như thế nào?**

- `transactional.id` được hash để xác định broker nào sở hữu partition `__transaction_state` — broker đó là **Transaction Coordinator**
- `initTransactions()` gửi `INIT_PRODUCER_ID` đến coordinator, coordinator gán cặp (PID, ProducerEpoch)
- ProducerEpoch **fences zombie producer**: nếu instance cũ của cùng `transactional.id` cố ghi, nó nhận `ProducerFencedException`
- Transaction metadata được persist trong `__transaction_state`

**H: Đi qua lifecycle của một transaction commit.**

1. `beginTransaction()` — chỉ thay đổi local state, không có RPC
2. `send()` đầu tiên đến partition P trigger `ADD_PARTITIONS_TO_TXN` đến coordinator → state chuyển sang "Ongoing"
3. Message được gửi qua `PRODUCE` request
4. `commitTransaction()` gửi `END_TXN` → coordinator ghi "PrepareCommit" vào `__transaction_state`
5. Coordinator gửi `WRITE_TXN_MARKERS` đến tất cả partition leader liên quan
6. Partition leader append một **control record** (commit marker) vào log của mình
7. Coordinator ghi "CompleteCommit" vào `__transaction_state`

**Đảm bảo atomicity**: Nếu coordinator crash sau bước 4, coordinator mới sẽ tìm thấy trạng thái "PrepareCommit" và tiếp tục commit khi recovery.

---

### 4.2 Consumer Isolation Levels

**H: Consumer tương tác với in-progress transaction như thế nào?**

- `isolation.level = read_uncommitted` (mặc định): Đọc tất cả record kể cả những cái trong transaction đang mở hoặc bị abort
- `isolation.level = read_committed`: Chỉ đọc record từ **committed** transaction. Bị block tại **Last Stable Offset (LSO)**.

**LSO (Last Stable Offset)**: Offset cao nhất mà tất cả transaction đến đó đã được quyết định (commit hoặc abort). Consumer với `read_committed` không thể advance qua LSO.

**First Unstable Offset (FUO)**: Offset sớm nhất của bất kỳ transaction đang mở (chưa commit). LSO = FUO - 1.

**Bẫy — Hanging Transaction**: Nếu producer mở transaction và không bao giờ commit/abort (do crash mà không cleanup đúng), LSO dừng advance. Consumer lag tăng vô hạn dù production tiếp tục phía trên. Gây ra:
- Topic compacted tăng không giới hạn (log cleaner không thể advance qua LSO)
- Cạn kiệt disk
- Gián đoạn service

Phát hiện:
```bash
kafka-transactions.sh --bootstrap-server :9092 find-hanging --broker 0
kafka-transactions.sh --bootstrap-server :9092 abort --topic <topic> --partition <p> --start-offset <offset>
```

---

### 4.3 Chi Phí Performance của EOS

| Nguồn Overhead | Tác động |
|----------------|---------|
| `initTransactions()` RPC + PID acquisition | Chi phí một lần khi startup |
| `ADD_PARTITIONS_TO_TXN` per transaction | Synchronization trước khi send đầu tiên per partition |
| `WRITE_TXN_MARKERS` khi commit | Request đến tất cả partition leader liên quan |
| LSO filtering | Consumer buffer cho đến khi nhận commit marker |
| Sequential transaction | Một open transaction per producer tại một thời điểm |

Số liệu benchmark:
- Idempotent-only producer: ~1-3% giảm throughput so với at-least-once
- Transactional producer: ~3% chậm hơn `acks=all` at-least-once; ~20% chậm hơn `acks=0`
- Kafka Streams với `exactly_once_v2` và commit interval 100ms: giảm 15-30% throughput
- Kafka Streams với commit interval 30s: gần như không có overhead với message >= 1KB

---

### 4.4 Giới Hạn Phạm Vi EOS

**Điểm phỏng vấn quan trọng**: Exactly-once trong Kafka chỉ áp dụng **trong Kafka**. Không bao gồm:
- Ghi vào database ngoài
- HTTP/RPC side effect
- Thao tác multi-cluster (transaction không span qua cluster)
- Consumer application logic (ví dụ: nếu consumer ghi vào 2 hệ thống ngoài, Kafka EOS không làm điều đó atomic)

Với external sink: cần idempotent consumer (deduplication bằng business key trong target system).
