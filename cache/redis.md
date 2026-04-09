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

#### Ưu điểm:
- **Scale read & write**
- **Failover tự động**
- Phân tán dữ liệu → giảm tải

#### Nhược điểm:
- Phức tạp hơn
- Cost cao hơn
- Data inconsistency vẫn tồn tại

#### Hash Slot:
- Redis Cluster chia dữ liệu thành **16,384 hash slots**
- Mỗi master quản lý một phần hash slot → dữ liệu được phân phối đều

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

### Các chế độ:
- `noeviction`: không xóa, trả lỗi
- `allkeys-random`: xóa key ngẫu nhiên
- `volatile-ttl`: xóa key có TTL sắp hết hạn
- `allkeys-lru`: xóa key ít được dùng gần đây nhất
- `allkeys-lfu`: xóa key ít được truy cập nhất

---

## 9. Persistence (Sao lưu dữ liệu)

### 9.1 AOF (Append Only File)
- Ghi **mọi lệnh write** xuống file `.aof`
- Có thể phục hồi bằng cách thực hiện lại lệnh
- **Ưu điểm**: Ít mất dữ liệu
- **Nhược điểm**: File lớn, khôi phục lâu

### 9.2 RDB (Redis Database Backup - Snapshot)
- Redis định kỳ chụp nhanh (snapshot) toàn bộ dữ liệu và lưu vào file .rdb
- Snapshot dữ liệu theo chu kỳ
- Lưu trữ dạng binary `.rdb`

| Ưu điểm          | Nhược điểm                         |
|------------------|-------------------------------------|
| Tốc độ khôi phục nhanh | Không real-time, có thể mất dữ liệu |
| File nhỏ hơn AOF       | Snapshot lớn → hiệu suất giảm        |

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
Quá trình sao lưu dữ liệu từ Master sang Slave trong Redis bao gồm hai giai đoạn chính: 
- Đồng bộ hóa toàn bộ (Full Synchronization) 
- Sao chép lệnh (Command Propagation).

Tóm tắt cơ chế sao lưu dữ liệu từ Master sang Slave trong Redis
- Master-Slave Replication trong Redis sao chép dữ liệu từ 
- Master (xử lý ghi/đọc) sang Slave (chỉ đọc) để cân bằng tải, đảm bảo sẵn sàng cao và dự phòng dữ liệu. Cơ chế gồm:

#### 11.1 Đồng bộ hóa toàn bộ (Full Synchronization):
Khi một Slave kết nối lần đầu với Master hoặc khi kết nối bị gián đoạn quá lâu, quá trình đồng bộ hóa toàn bộ được thực hiện:
- Slave gửi `PSYNC/SYNC` đến Master.
- Master tạo và gửi snapshot RDB, Slave tải và áp dụng.
- Master gửi các lệnh ghi tích lũy trong quá trình truyền.

#### 11.2 Sao chép lệnh (Command Propagation):
Sau khi hoàn tất đồng bộ hóa toàn bộ, Redis chuyển sang giai đoạn sao chép lệnh để giữ cho Slave luôn đồng bộ với Master:
- Khi client gửi một lệnh ghi đến Master (ví dụ: SET key value, DEL key, LPUSH list value), Master ghi log các lệnh này vào Replication buffer.
- Các lệnh này được gửi theo thời gian thực (qua TCP stream) từ Master tới tất cả các Slave đang kết nối.
- Slave không tự thực hiện ghi từ client mà chỉ nhận lệnh từ Master và thực thi lại.
- Mỗi Slave nhận và thực thi từng lệnh Redis y như Master.
- Điều này đảm bảo Slave có trạng thái dữ liệu giống hệt Master, nhưng chỉ ở chế độ đọc (read-only).

#### Hỗ trợ đồng bộ một phần – Partial Resynchronization (PSYNC)
- Khi Slave mất kết nối với Master (ví dụ mạng chập chờn):
- Nếu kết nối khôi phục sớm và Master vẫn giữ replication buffer chưa bị ghi đè, thì Slave có thể yêu cầu đồng bộ một phần:
  - Gửi PSYNC replication_id offset đến Master.
  - Master chỉ gửi phần lệnh mới phát sinh từ offset đó.
  - Không cần đồng bộ lại toàn bộ dữ liệu.
- Nếu buffer bị ghi đè hoặc Master khởi động lại → fallback về full resync (bắt đầu từ đầu).

#### Đặc điểm:
- Không đồng bộ (asynchronous), có thể gây độ trễ nhỏ.
- Slave chỉ đọc, hỗ trợ cân bằng tải và sao lưu.
- Tốn tài nguyên khi đồng bộ hóa toàn bộ.