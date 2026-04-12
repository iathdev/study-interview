# Command

| Lệnh                 | Mô tả                            |
| -------------------- | -------------------------------- |
| `SET key value`      | Đặt giá trị cho key              |
| `GET key`            | Lấy giá trị từ key               |
| `DEL key`            | Xóa key                          |
| `EXPIRE key seconds` | Đặt thời gian sống (TTL) cho key |
| `TTL key`            | Xem TTL còn lại                  |
| `INCR key`           | Tăng giá trị số nguyên lên 1     |
| `DECR key`           | Giảm giá trị số nguyên xuống 1   |
| `APPEND key value`   | Nối thêm vào giá trị key         |


| Lệnh                    | Mô tả                      |
| ----------------------- | -------------------------- |
| `LPUSH list value`      | Thêm vào đầu danh sách     |
| `RPUSH list value`      | Thêm vào cuối danh sách    |
| `LPOP list`             | Lấy và xóa phần tử đầu     |
| `RPOP list`             | Lấy và xóa phần tử cuối    |
| `LRANGE list start end` | Lấy dải phần tử trong list |
| `LLEN list`             | Độ dài list                |


| Lệnh                   | Mô tả                             |
| ---------------------- | --------------------------------- |
| `SADD set member`      | Thêm phần tử vào set              |
| `SREM set member`      | Xóa phần tử khỏi set              |
| `SMEMBERS set`         | Lấy tất cả phần tử trong set      |
| `SISMEMBER set member` | Kiểm tra phần tử có tồn tại không |
| `SUNION set1 set2`     | Hợp 2 set                         |
| `SINTER set1 set2`     | Giao 2 set                        |
| `SDIFF set1 set2`      | Hiệu set1 - set2                  |


| Lệnh                     | Mô tả                            |
| ------------------------ | -------------------------------- |
| `HSET hash field value`  | Set giá trị của field trong hash |
| `HGET hash field`        | Lấy giá trị của field            |
| `HDEL hash field`        | Xóa field khỏi hash              |
| `HGETALL hash`           | Lấy tất cả field và giá trị      |
| `HINCRBY hash field num` | Tăng giá trị số trong field      |
| `HEXISTS hash field`     | Kiểm tra field có tồn tại không  |


| Lệnh                       | Mô tả                                  |
| -------------------------- | -------------------------------------- |
| `ZADD zset score member`   | Thêm phần tử kèm điểm (score)          |
| `ZRANGE zset start end`    | Lấy phần tử theo thứ tự tăng dần score |
| `ZREVRANGE zset start end` | Theo thứ tự giảm dần score             |
| `ZREM zset member`         | Xóa phần tử                            |
| `ZSCORE zset member`       | Lấy điểm của phần tử                   |


| Lệnh                      | Mô tả                   |
| ------------------------- | ----------------------- |
| `PUBLISH channel message` | Gửi message vào channel |
| `SUBSCRIBE channel`       | Lắng nghe channel       |
| `UNSUBSCRIBE channel`     | Rời khỏi channel        |


| Lệnh                               | Mô tả                               |
| ---------------------------------- | ----------------------------------- |
| `XADD mystream * key value`        | Thêm bản ghi vào stream             |
| `XRANGE mystream - +`              | Lấy toàn bộ bản ghi từ đầu đến cuối |
| `XREAD COUNT 2 STREAMS mystream 0` | Đọc dữ liệu từ stream               |


| Lệnh           | Mô tả                                 |
| -------------- | ------------------------------------- |
| `KEYS pattern` | Tìm key theo pattern (`*`, `user:*`)  |
| `FLUSHALL`     | Xóa toàn bộ dữ liệu trong tất cả DB   |
| `FLUSHDB`      | Xóa dữ liệu trong DB hiện tại         |
| `INFO`         | Xem thông tin Redis server            |
| `MONITOR`      | Giám sát tất cả request gửi đến Redis |
| `PING`         | Kiểm tra Redis hoạt động không        |


# Redis Cheat Sheet

## 1. Redis là gì?
- Redis là một **database in-memory**, mã nguồn mở.
- Được sử dụng như một **remote cache** để lưu trữ dữ liệu tạm thời và truy xuất nhanh.
- Hỗ trợ nhiều **cấu trúc dữ liệu** như: `String`, `Hash`, `List`, `Set`, `Sorted Set`, `Bitmap`, `HyperLogLog`, `Stream`.
- Tính năng nổi bật:
  - Transaction
  - Pub/Sub
  - Lua Script
  - Stream
  - Persistence (sao lưu)
  - Cluster & Sentinel

## 2. Use Cases của Redis
- Caching
- Counter
- Pub/Sub
- Queue
- Key-Value Database

---

## 3. Các kiểu dữ liệu và Use-case

| Kiểu dữ liệu    | Use-case ví dụ                                  |
|----------------|--------------------------------------------------|
| **String**      | Lưu key-value đơn giản, token, config           |
| **List**        | Queue, Stack (push/pop), logs, chat messages    |
| **Set**         | Tags, danh sách không trùng                     |
| **Hash**        | Lưu object dạng JSON, user profile              |
| **Sorted Set**  | Leaderboard, ranking theo điểm số               |
| **Bitmap**      | Lưu trữ boolean: online/offline, flags          |
| **Stream**      | Message queue, event logs                       |
| **HyperLogLog** | Đếm số lượng phần tử unique (tương đối)         |

---

## 4. Ưu điểm của Redis
- **Tốc độ cao** vì in-memory.
- **Hỗ trợ nhiều kiểu dữ liệu phong phú.**
- **Tính năng mở rộng tốt (scale tốt)**.
- **Nhiều tính năng mạnh** như transaction, pub/sub, stream, script.
- **Cơ chế loại bỏ dữ liệu không dùng đến** khi bộ nhớ đầy.
- **Redis Cluster** hỗ trợ tính sẵn sàng và phân tán dữ liệu.

---

## 5. Redis vs Memcache

### 5.1 Giống nhau
- In-memory database
- Được dùng làm remote cache
- Có thể đặt TTL cho key
- Hiệu suất cao (rất nhanh)

### 5.2 Khác nhau

| So sánh       | Redis                             | Memcache                        |
|---------------|------------------------------------|----------------------------------|
| **Threading** | Single-thread                     | Multi-thread                    |
| **Read/Write**| Tối ưu read                        | Tối ưu write                    |
| **Data Types**| Nhiều cấu trúc dữ liệu             | Chủ yếu chỉ là key-value        |
| **Persistence**| Có (RDB, AOF)                     | Không hỗ trợ                    |
| **Cluster**   | Có cluster mode                   | Không hỗ trợ                    |
| **Khác**      | Pub/Sub, Stream, Lua Script...    | Không có                        |

👉 **Redis phổ biến hơn Memcache** vì tính năng phong phú và linh hoạt.

---

## 6. Redis Architecture

### 6.1 Master - Slave
- **Ghi và đọc tại master**
- **Slave chỉ đọc**
- **Sao lưu bất đồng bộ từ master → slave**

#### Ưu điểm:
- Tăng khả năng mở rộng đọc (read scalability)
- Tính sẵn sàng cao

#### Nhược điểm:
- Dữ liệu giữa master - slave có thể không đồng bộ → **data inconsistency**
- Không tự động failover

#### Failover là gì?
Failover là cơ chế tự động chuyển đổi sang hệ thống dự phòng (backup) khi hệ thống chính gặp sự cố (crash, lỗi phần cứng, mất kết nối, v.v). Mục tiêu là duy trì dịch vụ liên tục, giảm thời gian gián đoạn (downtime) và tăng tính sẵn sàng (high availability).

---

### 6.2 Sentinel
- Thêm **Sentinel node** để giám sát master/slave
- Sentinel tự động chọn slave làm master mới khi master cũ down

#### Ưu điểm:
- **Tự động failover**

#### Nhược điểm:
- Cần thêm tài nguyên (chi phí tăng)
- Vẫn có nguy cơ data inconsistency
- Không scale được write (vì vẫn 1 master ghi)

---

### 6.3 Redis Cluster
- Có nhiều **master nodes**, mỗi master giữ một phần dữ liệu
- Có thể **đọc/ghi phân tán**

**Vấn đề của Sentinel:** vẫn chỉ có 1 master ghi → bottleneck khi write nhiều, scale được read nhưng không scale được write.

Redis Cluster giải quyết bằng cách có nhiều master:
```
Master 1 ──── Slave 1    (quản lý slot 0 → 5460)
Master 2 ──── Slave 2    (quản lý slot 5461 → 10922)
Master 3 ──── Slave 3    (quản lý slot 10923 → 16383)
```

#### Hash Slot:
- Redis Cluster chia dữ liệu thành **16,384 hash slots**
- Khi `SET user:123`, Redis tính `CRC16("user:123") % 16384` → ra slot số bao nhiêu → biết ghi vào master nào
- Mỗi master chỉ chịu trách nhiệm một phần slot → dữ liệu phân tán đều

#### Failover tự động:
- Mỗi master có slave riêng
- Master chết → slave của nó tự lên làm master, không cần Sentinel

#### Ưu điểm:
- **Scale read & write**
- **Failover tự động**
- Phân tán dữ liệu → giảm tải

#### Nhược điểm:
- Phức tạp hơn
- Cost cao hơn
- Data inconsistency vẫn tồn tại

#### So sánh 3 kiến trúc:

| | Master-Slave | Sentinel | Cluster |
|---|---|---|---|
| Scale write | Không | Không | Có |
| Scale read | Có | Có | Có |
| Tự động failover | Không | Có | Có |
| Độ phức tạp | Thấp | Trung bình | Cao |

---

## 7. Tại sao Redis nhanh?

### 7.1 Bottlenecks ở đâu?
- **Memory**: lưu trên RAM → đắt tiền
- **Network bandwidth**: nếu data lớn
- **Không nằm ở CPU**: Redis dùng 1 core (single-thread)

### 7.2 Tại sao dùng single-thread?
- Đảm bảo **toàn vẹn dữ liệu**
- Giảm độ phức tạp của hệ thống

### 7.3 Vì sao nhanh?
- In-memory: đọc/ghi RAM nhanh
- Single-thread
- Multiplexing I/O: 1 thread xử lý nhiều connection
- Cấu trúc dữ liệu hiệu quả cao

---

## 8. Quản lý bộ nhớ khi Redis đầy

Redis sẽ thực hiện theo `maxmemory-policy`

**LRU (Least Recently Used):** xóa key lâu nhất chưa được truy cập
```
Key A truy cập lúc: 10:00
Key B truy cập lúc: 10:05
Key C truy cập lúc: 10:10
=> RAM đầy => xóa Key A (lâu nhất chưa dùng)
```

**LFU (Least Frequently Used):** xóa key có tần suất truy cập thấp nhất
```
Key A được truy cập: 100 lần
Key B được truy cập: 3 lần
Key C được truy cập: 50 lần
=> RAM đầy => xóa Key B (ít được dùng nhất)
```

| | LRU | LFU |
|---|---|---|
| Dựa vào | Thời điểm dùng gần nhất | Số lần truy cập |
| Key lâu không dùng | Bị xóa | Giữ nếu dùng nhiều lần trước đó |
| Phù hợp | Cache ngắn hạn, web session | Hot key, dữ liệu phổ biến lâu dài |

### Các chế độ:
- `noeviction`: không xóa, trả lỗi
- `allkeys-random`: xóa key ngẫu nhiên
- `volatile-ttl`: xóa key có TTL sắp hết hạn nhất
- `allkeys-lru`: xóa key ít được dùng gần đây nhất (LRU)
- `allkeys-lfu`: xóa key ít được truy cập nhất (LFU)

---

## 9. Persistence (Sao lưu dữ liệu)

Redis lưu dữ liệu trên RAM → khi server crash hoặc restart, dữ liệu mất hết.
Persistence giúp Redis lưu dữ liệu xuống disk để có thể phục hồi lại sau khi restart.

### 9.1 AOF (Append Only File)
Ghi lại **từng lệnh write** vào file `.aof` theo thứ tự thời gian, giống như transaction log.

```
SET user:1 "Alice"   ← ghi vào .aof
SET user:2 "Bob"     ← ghi vào .aof
DEL user:1           ← ghi vào .aof
...
```

Khi khôi phục: Redis đọc file `.aof` và **thực thi lại từng lệnh** từ đầu đến cuối.

Ưu điểm: ít mất dữ liệu nhất (có thể cấu hình ghi mỗi lệnh, mỗi giây, hoặc để OS tự quyết)
Nhược điểm: file ngày càng lớn, khôi phục lâu vì phải replay toàn bộ lịch sử lệnh

### 9.2 RDB (Redis Database Backup - Snapshot)
Redis định kỳ **chụp toàn bộ dữ liệu** tại một thời điểm và lưu vào file `.rdb` dạng binary.

```
10:00 → snapshot → lưu file rdb
10:05 → snapshot → lưu file rdb (ghi đè)
10:10 → Redis crash
=> Khôi phục từ snapshot 10:05, mất dữ liệu 5 phút
```

Ưu điểm: file nhỏ gọn, khôi phục nhanh (load binary trực tiếp vào RAM)
Nhược điểm: mất dữ liệu trong khoảng thời gian giữa 2 snapshot

### So sánh AOF vs RDB

| | AOF | RDB |
|---|---|---|
| Cơ chế | Ghi từng lệnh | Snapshot định kỳ |
| Mất dữ liệu tối đa | Rất ít (vài giây) | Nhiều hơn (vài phút) |
| Tốc độ khôi phục | Chậm (replay lệnh) | Nhanh (load binary) |
| Kích thước file | Lớn | Nhỏ |
| Dùng khi | Cần an toàn dữ liệu | Cần khôi phục nhanh |

Thực tế thường **kết hợp cả hai**: AOF để đảm bảo ít mất dữ liệu, RDB để khôi phục nhanh khi cần.

---

## 10. Cơ chế xóa dữ liệu hết hạn

### 1. Lazy Deletion (Xóa Lười)

- Khi **truy cập một key** (GET, SET, v.v.), Redis **kiểm tra xem key đó đã hết hạn chưa**
- Nếu đã hết hạn, **Redis sẽ xóa ngay lập tức**
- Nếu không có ai truy cập vào key → **nó vẫn tồn tại** trong bộ nhớ

### 2. Active Expiration Cycle (Xóa Chủ Động Định Kỳ)
- Redis sẽ định kỳ quét ngẫu nhiên các key có thiết lập thời gian hết hạn (EXPIRE)
- Nếu phát hiện key đã hết hạn → xóa
- Redis giới hạn số lượng key được quét mỗi lần để không gây quá tải CPU
- Redis không quét toàn bộ key vì điều đó tốn tài nguyên lớn

### 3. Eviction (Khi Thiếu Bộ Nhớ)
- Khi Redis đạt đến maxmemory (giới hạn bộ nhớ), nó sẽ kích hoạt cơ chế eviction
- Redis sẽ xóa key dựa trên chính sách được cấu hình trong maxmemory-policy
- Trong nhiều chính sách, Redis ưu tiên xóa các key đã hết hạn trước

Chính sách:
- LRU (Least Recently Used) – Ít được sử dụng gần đây nhất
  - Ưu tiên xóa các key nào lâu rồi không được truy cập
- LFU (Least Frequently Used) – Ít được truy cập thường xuyên nhất
  - Ưu tiên xóa những key nào bị truy cập ít nhất, bất kể gần đây hay không

| Tiêu chí                 | LRU                        | LFU                              |
| ------------------------ | -------------------------- | -------------------------------- |
| Dựa vào                  | Thời điểm sử dụng gần nhất | Tần suất sử dụng (frequency)     |
| Dữ liệu “lâu không dùng” | Sẽ bị xóa                  | Có thể giữ nếu dùng nhiều        |
| Dữ liệu “vừa mới dùng”   | Có thể giữ                 | Có thể bị xóa nếu dùng ít        |
| Phù hợp với              | Web cache ngắn hạn         | Hệ thống có hot key, phân bổ đều |
| Cần theo dõi             | Thời gian gần nhất sử dụng | Số lần sử dụng                   |

Tổng quan cơ chế xoá dữ liệu hết hạn

| Cơ chế               | Khi nào kích hoạt     | Ưu điểm                    | Nhược điểm                    |
| -------------------- | --------------------- | -------------------------- | ----------------------------- |
| Lazy deletion        | Khi key được truy cập | Không tốn tài nguyên       | Key chết vẫn chiếm RAM        |
| Active expiration    | Chạy định kỳ nội bộ   | Tự động dọn rác nhẹ nhàng  | Có thể không xóa ngay lập tức |
| Eviction khi đầy RAM | Khi vượt `maxmemory`  | Tránh crash khi hết bộ nhớ | Có thể xóa key chưa hết hạn   |


### 11. Cơ chế sao lưu dữ liệu từ Master sang Slave

Mục tiêu: giữ cho Slave luôn có dữ liệu giống Master để phục vụ read và sẵn sàng thay thế Master khi cần.

Có 2 giai đoạn chính:

#### 11.1 Full Synchronization - Đồng bộ toàn bộ
Xảy ra khi Slave kết nối lần đầu hoặc mất kết nối quá lâu.

**Bước 1:** Slave gửi `PSYNC/SYNC` lên Master để yêu cầu đồng bộ.

**Bước 2:** Master tạo snapshot RDB — chụp toàn bộ dữ liệu hiện tại xuống file `.rdb`.
Trong lúc đang tạo RDB, client vẫn gửi lệnh write đến Master bình thường.
Các lệnh write này được Master tạm ghi vào **Replication Buffer** để không bị mất.

**Bước 3:** Master gửi file RDB sang Slave. Slave xóa dữ liệu cũ và load toàn bộ RDB vào RAM.

**Bước 4:** Master gửi tiếp các lệnh trong Replication Buffer sang Slave.
Slave thực thi lại từng lệnh → bắt kịp những thay đổi xảy ra trong lúc đang truyền RDB.

```
Client                Master                  Slave
  |                     |                       |
  |                     |   <-- PSYNC/SYNC ---  |   Bước 1: Slave yêu cầu đồng bộ
  |                     |                       |
  |                     |  (tạo snapshot RDB)   |   Bước 2: Master chụp toàn bộ data
  |                     |                       |
  |  SET key value -->  |                       |
  |                     |  (ghi vào repl buf)   |   Bước 2: lệnh write tạm lưu vào buffer
  |                     |                       |
  |                     |  --- RDB file -----> |   Bước 3: Slave load toàn bộ data
  |                     |                       |
  |                     |  --- buffer cmds --> |   Bước 4: Slave áp dụng lệnh tích lũy
  |                     |                       |
  |                     |     [đồng bộ xong]    |
```

Sau bước 4, Slave đã có dữ liệu giống hệt Master → chuyển sang Command Propagation.

#### 11.2 Command Propagation - Sao chép lệnh
Sau khi Full Sync xong, Slave không cần sync lại từ đầu nữa. Thay vào đó mỗi lệnh write
từ client gửi lên Master sẽ được Master **forward ngay lập tức** xuống tất cả Slave qua TCP stream.

Slave nhận lệnh và thực thi lại y hệt Master → dữ liệu luôn được cập nhật theo thời gian thực.
Slave **không nhận write trực tiếp từ client**, chỉ nhận từ Master.

```
Client                Master                  Slave 1       Slave 2
  |                     |                       |               |
  |  SET user "Alice" ->|                       |               |
  |                     |--- SET user "Alice" ->|               |
  |                     |--- SET user "Alice" --------------- ->|
  |                     |                       |               |
  |  DEL user:2 ------->|                       |               |
  |                     |--- DEL user:2 ------->|               |
  |                     |--- DEL user:2 --------------------- ->|
```

Vì là **async** nên Slave có thể trễ hơn Master vài ms (replication lag) → đọc từ Slave
có thể nhận dữ liệu cũ hơn Master một chút.

#### 11.3 Partial Resynchronization (PSYNC) - Đồng bộ một phần
Khi Slave mất kết nối tạm thời (mạng chập chờn) rồi kết nối lại, nếu làm Full Sync lại
thì rất tốn tài nguyên. PSYNC giải quyết bằng cách chỉ sync phần còn thiếu.

Master luôn giữ một **Replication Buffer** (vòng tròn, có giới hạn kích thước) chứa các
lệnh write gần đây. Mỗi Slave theo dõi **offset** — vị trí lệnh cuối cùng nó đã nhận.

Khi Slave kết nối lại, gửi `PSYNC <replication_id> <offset>`:
- Master kiểm tra offset đó còn trong buffer không
- Còn → chỉ gửi phần lệnh từ offset đó đến hiện tại, không cần full sync
- Mất (buffer bị ghi đè) → fallback về Full Synchronization

```
Replication Buffer của Master (giới hạn kích thước):
[cmd1][cmd2][cmd3][cmd4][cmd5]
                  ^
            offset Slave đang ở (đã nhận đến cmd3)

Slave kết nối lại: PSYNC <id> <offset=cmd3>
=> Master chỉ gửi [cmd4][cmd5]
```

#### Đặc điểm:
- **Async**: Slave có thể trễ so với Master → data inconsistency tạm thời
- **Slave read-only**: không nhận write từ client
- Full sync tốn tài nguyên → hạn chế số lần Slave reconnect