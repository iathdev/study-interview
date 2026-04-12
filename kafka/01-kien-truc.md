# Kiến Trúc Cốt Lõi Kafka

> Tổng hợp từ tài liệu chính thức, Confluent engineering blog, Strimzi, và các nguồn phỏng vấn senior. Bao gồm kiến trúc nội bộ, các bẫy thường gặp, và system design patterns.

---

## 1. Kiến Trúc Cốt Lõi

### 1.1 Broker

**H: Kafka broker là gì và nó làm gì?**

Kafka broker là một server trong Kafka cluster. Nó:
- Nhận message từ producer và gán offset
- Lưu message xuống disk (append-only log segments)
- Phục vụ fetch request từ consumer
- Replicate dữ liệu partition từ/đến các broker khác

Một broker được bầu làm **Controller** (chế độ ZooKeeper) hoặc **Active Controller** (chế độ KRaft). Controller quản lý bầu chọn partition leader và đăng ký broker.

**H: Điều gì xảy ra khi controller broker bị lỗi?**

- **Chế độ ZooKeeper**: ZooKeeper phát hiện controller mất kết nối, broker khác giành quyền làm controller mới. Vấn đề: controller mới **không biết gì cả** — phải đọc lại toàn bộ partition metadata từ ZooKeeper (`O(số partition)`). Cluster có 500k partition → mất 30-60 giây. Trong thời gian đó không bầu được leader mới cho bất kỳ partition nào.
- **Chế độ KRaft**: Kafka tự lưu metadata trong topic nội bộ `__cluster_metadata`, replicate liên tục sang tất cả broker trong Quorum. Trong Quorum luôn có đúng **một Active Controller** (leader của quorum) — đây là broker duy nhất có quyền ra quyết định (bầu partition leader, quản lý broker registration...). Các broker còn lại trong quorum là Voter (follower). Khi **Active Controller chết**, các Voter bầu một Voter khác lên làm Active Controller mới qua Raft — vì Voter đó **đã có đầy đủ metadata rồi**, không cần đọc lại gì cả, tiếp tục làm việc ngay — chỉ mất vài milliseconds.

> **Quorum là gì?** Quorum là một nhóm nhỏ broker được chỉ định đặc biệt để lưu metadata và tham gia bầu chọn controller (gọi là Voter). Quyết định chỉ hợp lệ khi **đa số** (majority) trong quorum đồng ý — ví dụ quorum 3 broker thì cần 2/3 đồng ý. Cơ chế này đảm bảo không bao giờ có 2 controller cùng hoạt động (split-brain). Broker thông thường không tham gia quorum, chỉ fetch metadata từ controller (gọi là Observer).

**H: Active Controller nằm ở broker nào? Có random không?**

Không random — được bầu có tiêu chí rõ ràng:

- **Khai báo trong config**: Admin chỉ định trước broker nào được vào Quorum. Chỉ những broker này mới được bầu làm Active Controller:
  ```properties
  controller.quorum.voters=1@broker1:9093,2@broker2:9093,3@broker3:9093
  ```
- **Lần đầu khởi động**: Broker nào gửi vote request nhanh nhất (do randomized timeout) thắng, giữ vai trò Active Controller cho đến khi chết.
- **Khi Active Controller chết**: Raft bầu lại — Voter nào có **metadata log dài nhất và epoch cao nhất** thắng. Không random, có tiêu chí ưu tiên rõ ràng.

**H: Active Controller có hoạt động như broker bình thường không?**

Tùy vào cách setup:

**Combined Mode (cluster nhỏ)** — Broker làm Active Controller **vừa** lưu data như bình thường:
```
Broker 1 (Active Controller + lưu data partition)
Broker 2 (Voter + lưu data partition)
Broker 3 (Voter + lưu data partition)
```
Producer/consumer vẫn kết nối vào Broker 1 để đọc/ghi data bình thường.

**Dedicated Mode (production lớn)** — Controller node **chỉ làm quản lý cluster**, không lưu data:
```
Broker 1 (Active Controller — chỉ quản lý cluster)
Broker 2 (Voter — chỉ quản lý cluster)
Broker 3 (Voter — chỉ quản lý cluster)
Broker 4 (lưu data)
Broker 5 (lưu data)
Broker 6 (lưu data)
```
Producer/consumer không kết nối vào Broker 1/2/3. Controller tập trung xử lý metadata và bầu cử, không bị ảnh hưởng bởi I/O data.

Lý do phải tách ở production: Khi cluster có hàng triệu message/giây, broker lưu data rất bận (disk I/O, network). Nếu Active Controller cũng lưu data, nó có thể bị chậm → heartbeat timeout → cluster tưởng Controller chết → bầu lại không cần thiết → gây bất ổn toàn cluster.

---

### 1.2 Topics và Partitions

**H: Mối quan hệ giữa topic, partition và offset là gì?**

- **Topic**: Danh mục logic. Một named stream of records.
- **Partition**: Đơn vị lưu trữ vật lý. Topic được chia thành N partition. Mỗi partition là một ordered, immutable, append-only log.
- **Offset**: Số nguyên tăng đơn điệu định danh duy nhất mỗi record trong partition. Offset là per-partition — offset 5 ở partition 0 là message khác với offset 5 ở partition 1.

**H: Kafka xác định message đi vào partition nào như thế nào?**

1. Nếu có **key**: `partition = hash(key) % numPartitions` (dùng murmur2 hash mặc định)
2. Nếu **không có key**: Round-robin (phiên bản cũ) HOẶC sticky partitioning (Kafka 2.4+, lấp đầy một batch vào một partition trước khi chuyển)

**Bẫy**: Thay đổi số partition của topic sẽ phá vỡ mapping key → partition. Consumer phụ thuộc vào ordering theo key sẽ thấy vi phạm với các message mới. Kafka cho phép tăng partition nhưng **không bao giờ giảm**.

**H: Kafka đảm bảo ordering message như thế nào?**

- **Trong một partition**: Message được sắp xếp chặt chẽ theo offset.
- **Giữa các partition**: Không có đảm bảo ordering toàn cục.

Nếu cần tổng thứ tự cho tất cả message trong topic, dùng **single partition** — nhưng mất parallelism. Thiết kế đúng là chọn partition key sao cho các message cần ordering tương đối dùng cùng key (ví dụ: `user_id`, `order_id`).

---

### 1.3 Replication và ISR

**H: Replication trong Kafka hoạt động như thế nào?**

Mỗi partition có một **leader** và không hoặc nhiều **follower** (xác định bởi `replication.factor`). Tất cả read/write đều đi qua leader. Follower fetch message từ leader để sync.

**In-Sync Replicas (ISR)**: Tập hợp các replica đang "bắt kịp" leader trong `replica.lag.time.max.ms` (mặc định 30s). Replica bị loại khỏi ISR khi không fetch message đủ nhanh.

**High Watermark (HW)**: Offset cao nhất đã được replicate đến tất cả replica trong ISR. Consumer chỉ đọc được đến HW (không phải Log End Offset/LEO).

---

**H: Trong một Kafka cluster nên có bao nhiêu replica? Dùng công thức nào?**

Đây là câu hỏi có nhiều lớp — cần nắm **hai config phối hợp nhau**.

> **Đáp án: RF=3, min.insync.replicas=2, acks=all.** Công thức: `RF = f + 1` (f = số broker được phép chết). Muốn chịu 1 broker chết → RF=3. Fault tolerance khi ghi = `RF - min.insync.replicas = 1`.

### Công thức cốt lõi

> **Fault Tolerance** = khả năng chịu lỗi — cluster chịu được **bao nhiêu broker chết** mà vẫn hoạt động. Ví dụ Fault Tolerance = 1 nghĩa là chết 1 broker vẫn ổn, chết 2 thì sập.

> **Majority** = đa số — quá nửa số node phải đồng ý thì quyết định mới hợp lệ. 3 node thì majority = 2, 5 node thì majority = 3. Mục đích: đảm bảo không bao giờ có 2 nhóm cùng ra quyết định trái chiều nhau (split-brain).

Có **2 công thức** cho 2 ngữ cảnh khác nhau:

**1. Công thức `RF = f + 1` — dùng để chọn Replication Factor cho Partition**

Nguồn: [Apache Kafka Design Docs](https://kafka.apache.org/42/design/design/), [Confluent Replication Docs](https://docs.confluent.io/kafka/design/replication.html)

```
RF = f + 1
f = số broker được phép chết mà hệ thống vẫn hoạt động bình thường
```

| Muốn chịu | RF tối thiểu | Production dùng |
|---|---|---|
| 1 broker chết | 1+1 = 2 | **3** (xem lý do bên dưới) |
| 2 broker chết | 2+1 = 3 | **5** |

Tại sao production dùng RF=3 thay vì RF=2 dù công thức ra 2? Vì RF=2 không linh hoạt:
- `min.insync.replicas=1` → durability yếu (leader chết là mất data)
- `min.insync.replicas=2` → zero fault tolerance (1 broker chết là không ghi được)

RF=3 + `min.insync.replicas=2` mới cho phép vừa chịu được 1 broker chết, vừa đảm bảo durability.

Dùng công thức `RF - min.insync.replicas` để **kiểm tra** fault tolerance của setup đã chọn:
```
Fault Tolerance (writes) = RF - min.insync.replicas
Fault Tolerance (reads)  = RF - 1
```

**2. Công thức `2N+1` — dùng cho KRaft Controller Quorum (không phải partition)**

Nguồn: [Apache Kafka KRaft Operations](https://kafka.apache.org/41/operations/kraft/)

```
2N+1 controller node
N = số controller được phép chết mà vẫn bầu được leader mới (vẫn có majority)
```

| Muốn chịu | Controller nodes cần |
|---|---|
| 1 chết | 2(1)+1 = **3** |
| 2 chết | 2(2)+1 = **5** |

Áp dụng khi thiết kế **KRaft Controller Quorum** — bầu Active Controller cần majority đồng ý nên phải là số lẻ. **Không áp dụng cho partition replication.**

> **Tóm lại**: `RF = f+1` cho Partition Replication, `2N+1` cho KRaft Controller Quorum. Hai thứ hoàn toàn khác nhau.

### Tại sao cùng cluster mà dùng 2 công thức?

Broker vừa tham gia Controller Quorum, vừa chứa partition data — nhưng đang giải quyết **2 bài toán khác nhau**:

```
Kafka Cluster (3 broker — Combined mode)
│
├── Nhìn từ Controller Quorum (KRaft)  → công thức 2N+1
│   ├── Broker 1 (Active Controller)
│   ├── Broker 2 (Voter)
│   └── Broker 3 (Voter)
│   → 2(1)+1 = 3 node, chịu được 1 controller chết
│
└── Nhìn từ Partition Replication → công thức f+1
    ├── Broker 1 (chứa partition data)
    ├── Broker 2 (chứa partition data)
    └── Broker 3 (chứa partition data)
    → RF = 1+1 = 2 tối thiểu, dùng 3 cho linh hoạt
```

- **Controller Quorum** dùng `2N+1` vì bầu cử cần **majority vote** — phải có quá nửa đồng ý
- **Partition Replication** dùng `f+1` vì không cần majority vote — Controller chỉ chọn broker đầu tiên trong ISR

Kết quả đều ra **3** — đó là lý do RF=3 là con số "magic" cho production: **thỏa mãn cả 2 công thức cùng lúc** với cluster nhỏ nhất có thể.

### Setup chuẩn production

| Config | Giá trị | Ý nghĩa |
|--------|---------|---------|
| `replication.factor` | **3** | 3 bản sao mỗi partition |
| `min.insync.replicas` | **2** | Cần tối thiểu 2 replica đang sync thì producer mới ghi được |
| `acks` | **all** | Producer chờ tất cả ISR xác nhận |

→ Tolerate **1 broker chết** mà vẫn ghi được: `3 - 2 = 1`

### Tại sao RF=3, không phải 2 hay 5?

**RF=2:**
- Chết 1 broker → ISR chỉ còn 1 → nếu `min.insync.replicas=2` thì cluster **ngừng nhận write**
- Không có buffer nào cả

**RF=3 (sweet spot):**
- Chết 1 broker → ISR còn 2 → vẫn ghi được (2 >= min.insync.replicas)
- Chết 2 broker → mất write nhưng vẫn đọc được từ broker còn lại
- Chi phí storage tăng 3x nhưng đổi lại độ bền cao

**RF=5:**
- Dùng cho **critical topics** (financial, audit log)
- Tolerate 3 broker chết mà vẫn đọc được, 2 broker chết mà vẫn ghi được
- Chi phí cao, thường không cần thiết

### Ràng buộc quan trọng

```
replication.factor <= số broker trong cluster
```

Nếu RF=3 mà chỉ có 2 broker → Kafka sẽ **từ chối tạo topic**.

### Rack Awareness (hay hỏi ở level senior)

Kafka phân bổ replica theo rack/AZ:

```
Broker 1 (AZ-a) ← Leader
Broker 2 (AZ-b) ← Follower
Broker 3 (AZ-c) ← Follower
```

Nếu cả AZ-a chết → broker ở AZ-b hoặc AZ-c lên làm leader, không mất data. Cần config `broker.rack` trên mỗi broker.

### Tóm tắt trả lời phỏng vấn

> *"Production standard là RF=3, min.insync.replicas=2, acks=all. Công thức fault tolerance cho write là RF - min.insync.replicas = 1, nghĩa là chịu được 1 broker chết mà vẫn ghi được. RF không thể vượt quá số broker. Với hệ thống critical thì dùng RF=5. Kết hợp với rack awareness để đảm bảo replica trải đều qua các AZ."*

---

**H: `acks=all` nghĩa là chờ tất cả broker hay tất cả ISR?**

Chờ **tất cả broker trong ISR**, không phải tất cả broker được assign.

```
RF=3 → có 3 replica được assign
ISR = [Broker1, Broker2] (Broker3 bị lag, đã bị kick khỏi ISR)

acks=all → chỉ cần Broker1 + Broker2 confirm là đủ
Broker3 không tham gia dù đang sống
```

**H: `min.insync.replicas` là gì và tại sao quan trọng?**

`min.insync.replicas` (mặc định **1**) là ngưỡng tối thiểu của ISR. Khi ISR shrink xuống dưới ngưỡng này, broker **từ chối write hoàn toàn** và trả về `NotEnoughReplicasException` — không phải đợi rồi mới báo lỗi, mà từ chối ngay.

Chỉ có tác dụng khi `acks=all`. Với `acks=0` hoặc `acks=1`, config này bị **bỏ qua hoàn toàn**.

**Bẫy lớn nhất — `acks=all` mà không set `min.insync.replicas`:**

Mặc định `min.insync.replicas=1` — nghĩa là ISR chỉ cần còn 1 broker (chính leader) là đủ. Lúc này `acks=all` không mạnh hơn `acks=1` chút nào:

```
RF=3, min.insync.replicas=1 (default), acks=all

Follower1 và Follower2 đều chậm → bị kick khỏi ISR
ISR = [Leader only]
ISR size (1) >= min.insync.replicas (1) → vẫn OK
acks=all chờ "tất cả ISR" = chỉ mình leader → ACK ngay
Leader chết → MẤT DATA
```

**Setup đúng để có durability thực sự:**
```
replication.factor     = 3
min.insync.replicas    = 2   ← phải set tay, không dùng default
acks                   = all
```

**Bẫy khác**: Đặt `min.insync.replicas = replication.factor` nghĩa là **zero fault tolerance** — bất kỳ broker nào down là tất cả write đều fail.

**H: Điều gì xảy ra với partition khi leader broker bị lỗi?**

1. Controller phát hiện broker lỗi (qua ZooKeeper session expiry hoặc KRaft heartbeat timeout)
2. Controller bầu leader mới từ ISR
3. Nếu ISR rỗng và `unclean.leader.election.enable = true` (mặc định: false), replica out-of-sync có thể được bầu làm leader — rủi ro **mất data**
4. Producer/consumer nhận lỗi metadata refresh, retry và reconnect đến leader mới

---

**H: Cơ chế bầu leader hoạt động như thế nào?**

Có 2 loại bầu leader trong Kafka, hay bị nhầm lẫn:

**Loại 1 — Bầu Active Controller (Raft protocol)**

Xảy ra khi Active Controller chết. Các Voter trong Quorum tự bầu nhau:

```
1. Mỗi Voter có randomized timeout (Broker2=150ms, Broker3=300ms)
2. Broker2 hết timeout trước → tự ứng cử, tăng epoch (epoch=2), gửi vote request
3. Broker3 nhận request → kiểm tra:
   - Epoch của Broker2 cao hơn epoch mình đang biết? → YES
   - Log của Broker2 dài bằng hoặc hơn mình?       → YES
   → Bỏ phiếu cho Broker2
4. Broker2 nhận được majority → trở thành Active Controller
```

Điều kiện thắng: **epoch cao nhất + log dài nhất + nhận được majority vote**.
Randomized timeout để tránh tất cả ứng cử cùng lúc gây split vote.

**Loại 2 — Bầu Partition Leader (do Controller quyết định)**

Xảy ra khi partition leader chết. Không có vote — Controller đơn giản chọn broker đầu tiên trong ISR còn sống:

```
Topic A - Partition 0:
  ISR = [Broker4 (leader), Broker5, Broker6]

→ Broker4 chết

Controller nhìn vào ISR còn lại: [Broker5, Broker6]
→ Chọn broker đầu tiên trong danh sách: Broker5
→ Broker5 trở thành Partition Leader mới
```

| | Bầu Active Controller | Bầu Partition Leader |
|---|---|---|
| Ai quyết định | Các Voter tự bầu nhau (Raft) | Active Controller quyết định |
| Cơ chế | Vote, cần majority | Chọn broker đầu tiên trong ISR |
| Khi nào | Controller chết | Partition leader chết |
| Phức tạp | Cao (Raft protocol) | Đơn giản |

---

### 1.4 ZooKeeper vs KRaft

**H: ZooKeeper tạo ra những vấn đề gì cho Kafka?**

| Vấn đề | Chi tiết |
|--------|---------|
| **Giới hạn scalability** | ZooKeeper lưu toàn bộ partition metadata. Với ~200k partition, controller failover có thể mất 30+ giây do reload `O(partitions)` state |
| **Phức tạp vận hành** | Phải chạy và monitor một ZooKeeper ensemble riêng bên cạnh Kafka |
| **Single controller** | Chỉ có một active controller trong ZooKeeper mode; failover phải re-fetch toàn bộ state |
| **Phức tạp bảo mật** | Hai security surface riêng biệt (ZooKeeper ACLs + Kafka ACLs) |
| **Metadata inconsistency** | Có thể có lag giữa ZooKeeper state và in-memory state của broker |

**H: KRaft hoạt động nội bộ như thế nào?**

KRaft (Kafka Raft) lưu cluster metadata trong một Kafka topic nội bộ: `__cluster_metadata`. Các quyết định thiết kế chính:

- **Quorum of controllers**: Nhiều broker có thể được chỉ định làm controller. Một cái là active controller (leader); các cái còn lại là follower.
- **Raft consensus**: Controller dùng biến thể pull-based của giao thức Raft. Voter fetch từ leader (không push). Xử lý log reconciliation và leader isolation tốt hơn.
- **Event-sourced model**: Metadata được lưu dưới dạng event log, không phải snapshot. Snapshot được ghi định kỳ để giới hạn log growth.
- **Bầu chọn leader**: Candidate tăng epoch, request vote. Voter cấp vote dựa trên epoch validation và log length. Randomized timeout ngăn split vote.
- **Ba vai trò**: Leader (active controller), Voter (follower controller), Observer (broker thông thường fetch committed snapshots)

**H: `__cluster_metadata` lưu gì, lưu ở đâu, và quan hệ với controller thế nào?**

`__cluster_metadata` là topic nội bộ — "não" của toàn cluster. Lưu mọi thông tin về trạng thái cluster:

```
- Danh sách broker đang sống (broker ID, host, port)
- Danh sách topic và partition
- Ai là leader của từng partition
- ISR list của từng partition
- Replication factor của từng topic
- ACLs, quota config
- Controller epoch hiện tại
```

**Lưu ở đâu:** Trên disk của các Controller node (Voter) dưới dạng Kafka log bình thường. Topic này chỉ có **1 partition duy nhất**.

```
Broker 1 (Active Controller) → bản ghi đầy đủ, được WRITE vào
Broker 2 (Voter)             → bản sao, sync liên tục từ Active Controller
Broker 3 (Voter)             → bản sao, sync liên tục từ Active Controller
Broker 4 (Observer/data)     → KHÔNG có, chỉ fetch snapshot định kỳ
```

**Quan hệ với Controller:**

- **Active Controller** là người **duy nhất được WRITE** vào `__cluster_metadata` — mọi thay đổi (broker join/leave, leader election, ISR update) đều phải qua đây
- **Voter** nhận log từ Active Controller liên tục → luôn có bản sao đầy đủ, sẵn sàng lên thay bất cứ lúc nào
- **Observer** (broker thường) fetch snapshot định kỳ → biết partition nào đang ở broker nào, không tham gia bầu cử

Khi Active Controller chết → Voter thắng bầu cử → đã có đầy đủ `__cluster_metadata` → làm việc ngay, không cần đọc lại gì cả.

**H: Timeline áp dụng KRaft?**

| Phiên bản | Milestone |
|-----------|-----------|
| 2.8 | Early access (chưa production-ready) |
| 3.3 | KRaft production-ready (KIP-500) |
| 3.7 | KRaft mặc định cho cluster mới |
| **4.0** | **Bỏ hoàn toàn ZooKeeper support** |

**Bẫy**: Kể từ Kafka 4.0 (phát hành 2024), tất cả cluster mới chạy KRaft. Cluster ZooKeeper-mode phải migrate.

---

**H: Broker bình thường có quan hệ gì với Active Controller?**

Broker thường (Observer) có 3 quan hệ chính:

**1. Đăng ký khi join cluster**
```
Broker mới khởi động
→ Gửi BrokerRegistration request lên Active Controller
→ Controller ghi vào __cluster_metadata
→ Controller thông báo cho các broker khác biết có thành viên mới
```

**2. Nhận lệnh từ Controller**
```
Active Controller ra quyết định:
- Partition leader mới là ai
- ISR thay đổi
→ Đẩy lệnh xuống broker liên quan
→ Broker thực thi và báo lại
```

**3. Fetch metadata định kỳ**
```
Broker fetch snapshot từ __cluster_metadata
→ Biết được toàn bộ trạng thái cluster
→ Producer/consumer hỏi broker nào cũng trả về metadata đúng
```

Broker thường **không bao giờ tự ra quyết định** — chỉ nhận lệnh từ Controller, thực thi, và báo cáo lại. Mọi quyền lực tập trung ở Active Controller.

---

## 10. Network Layer — Cách Client Tìm Đúng Broker

### 10.1 Bootstrap và Metadata

**H: Client kết nối vào Kafka lần đầu như thế nào?**

Vấn đề cần giải: Client cần biết partition nào đang ở broker nào để gửi/nhận đúng chỗ. Nhưng cluster có nhiều broker, client không thể hardcode hết. Kafka giải quyết bằng **2 bước**:

**Bước 1 — Bootstrap (chỉ làm một lần):** Client kết nối vào 1 broker bất kỳ trong danh sách `bootstrap.servers` và hỏi: *"Cho tôi biết toàn bộ cluster trông như thế nào?"*

**Bước 2 — Nhận Metadata:** Broker trả lời: *"Cluster có 3 broker, topic X có 3 partition, partition 0 đang ở broker-2, partition 1 ở broker-3..."*

Sau đó client lưu thông tin này lại và **kết nối thẳng** đến đúng broker chứa partition cần dùng — không qua trung gian.

```
Client                    Broker-1 (bootstrap)      Broker-2
  |                            |                       |
  |--- TCP connect ----------->|                       |
  |--- "Cho tôi xem metadata"->|                       |
  |<-- Danh sách brokers,      |                       |
  |    partition → broker map  |                       |
  |                            |                       |
  |--- (ghi partition 0) -------------------------------->|
  |    (kết nối thẳng, không qua Broker-1 nữa)
```

> `bootstrap.servers` không cần liệt kê hết tất cả broker — chỉ cần 1 broker còn sống là đủ để lấy metadata.

---

**H: Client cập nhật lại thông tin khi nào?**

Thông tin metadata có thể lỗi thời — ví dụ broker-2 chết, leader của partition 0 đã chuyển sang broker-3. Client cần biết để không gửi nhầm chỗ.

Có 4 tình huống client tự cập nhật:

| Tình huống | Điều gì xảy ra |
|---------|-------|
| Gửi đến broker, nhận lỗi "tôi không còn là leader" | Broker báo thẳng, client hỏi lại ngay |
| Hỏi topic không tồn tại | Có thể topic vừa được tạo, hỏi lại cho chắc |
| Cứ mỗi 5 phút (mặc định) | Proactive — tự hỏi lại dù không có lỗi |
| Mất kết nối TCP | Kết nối lại và hỏi metadata mới |

**Ví dụ thực tế — broker-2 crash:**
```
Producer đang gửi → broker-2 (leader partition 0)
  → broker-2 crash → TCP bị đứt
  → Producer hỏi metadata từ broker-1 hoặc broker-3 (còn sống)
  → "broker-3 là leader mới của partition 0"
  → Producer gửi lại đến broker-3
```

---

**H: `advertised.listeners` là gì? Tại sao hay bị lỗi trong Docker/Kubernetes?**

Khi broker trả metadata cho client, nó quảng cáo địa chỉ của chính nó. `advertised.listeners` là địa chỉ broker **tự khai** trong metadata đó.

Vấn đề: Địa chỉ broker dùng để lắng nghe kết nối ≠ địa chỉ client bên ngoài có thể kết nối vào.

```
Broker trong Docker:
  - Broker nghe trên: 0.0.0.0:9092 (tất cả interface trong container)
  - IP container:     172.17.0.2  (chỉ trong mạng Docker nội bộ)
  - IP máy host:      192.168.1.100

Nếu không set advertised.listeners:
  → Broker tự khai địa chỉ là 172.17.0.2:9092
  → Client ngoài Docker nhận metadata: "kết nối vào 172.17.0.2:9092"
  → Client không thể reach được → lỗi
```

**Cách fix:**
```properties
listeners=PLAINTEXT://0.0.0.0:9092          # broker nghe trên tất cả interface
advertised.listeners=PLAINTEXT://192.168.1.100:9092  # khai địa chỉ mà client reach được
```

Trong Kubernetes: expose broker bằng NodePort/LoadBalancer và khai địa chỉ external đó trong `advertised.listeners`.

---

### 10.2 Producer Gửi Message Như Thế Nào

**H: Từ lúc gọi `producer.send()` đến lúc message ra mạng, có những bước gì?**

Producer không gửi từng message một — nó gom thành batch rồi mới gửi. Có **2 thread** làm việc song song:

```
Thread 1 (code của bạn)         Thread 2 (nền, Kafka tự quản lý)
─────────────────────────        ─────────────────────────────────
producer.send(msg1)          →   Gom batch theo (topic, partition)
producer.send(msg2)          →   Khi đủ điều kiện → gửi đi
producer.send(msg3)          →   Nhận phản hồi từ broker
```

**Batch được gửi khi nào?** Kafka không gửi ngay sau mỗi `.send()`. Batch chờ đến khi:
- Đủ `batch.size` bytes (mặc định 16KB) — batch đầy thì gửi
- Hoặc đợi `linger.ms` ms (mặc định 0ms) — hết giờ chờ thì gửi dù chưa đầy

Tại sao cần batch? Gửi 1000 message trong 1 batch nhanh hơn nhiều so với gửi 1000 lần riêng lẻ (giảm network round-trip, tăng throughput).

**`linger.ms` là gì?** Thời gian chờ thêm để gom nhiều message vào 1 batch. Tăng `linger.ms` → batch to hơn → throughput cao hơn, nhưng latency tăng. Mặc định 0ms (gửi ngay khi có).

---

## 11. ISR — Cơ Chế Đồng Bộ Dữ Liệu Giữa Các Broker

### 11.1 Follower Lấy Dữ Liệu Từ Leader Như Thế Nào

**H: Follower đồng bộ dữ liệu bằng cách nào?**

Follower **không nhận** dữ liệu bị push từ leader. Thay vào đó, follower **chủ động hỏi** leader: *"Cho tôi các message từ offset X trở đi"* — giống consumer fetch, nhưng là follower hỏi.

```
Vòng lặp liên tục trên mỗi follower:

Follower                      Leader
  |                              |
  |--- "Cho tôi từ offset 1050"->|
  |<-- Message 1050, 1051, 1052--|
  |    (kèm theo HW hiện tại)    |
  |                              |
  |--- "Cho tôi từ offset 1053"->|
  |<-- Message 1053, 1054 -------|
  |                              |
  ... (lặp liên tục, không nghỉ)
```

Mỗi partition có một thread riêng (`ReplicaFetcherThread`) làm việc này để giữ lag gần bằng 0.

**Leader biết follower đang ở đâu:** Mỗi lần follower hỏi, nó khai báo "tôi đang cần từ offset X" — leader dùng thông tin này để tính **High Watermark** (xem mục dưới).

---

### 11.2 Ba Mốc Offset Cần Biết

**H: LEO, HW, LSO khác nhau như thế nào?**

Hãy hình dung log của một partition như một cuộn băng. Có 3 vị trí đánh dấu:

```
Offset:  0   1   2   3   4   5   6   7   8   9   10 ...
         [msg][msg][msg][msg][msg][msg][msg][msg][ ][ ] ...
                              ↑           ↑        ↑
                             LSO         HW       LEO
```

| Mốc | Tên đầy đủ | Ý nghĩa đơn giản |
|-----|-----------|-----------------|
| **LEO** | Log End Offset | Offset của message tiếp theo sẽ được ghi vào. Tức là "đã ghi đến đây rồi" |
| **HW** | High Watermark | Offset cao nhất đã được **tất cả** follower trong ISR chép xong. Consumer chỉ đọc đến đây |
| **LSO** | Last Stable Offset | Chỉ liên quan đến transaction — offset cao nhất mà mọi transaction đều đã xong (commit hoặc abort) |

**Tại sao consumer không đọc qua HW?**

Giả sử leader có offset 95-99 nhưng follower f2 chỉ mới có đến 94. Nếu leader chết và f2 lên làm leader mới, nó không có offset 95-99. Consumer đã đọc 95-99 rồi sẽ thấy dữ liệu "biến mất" — inconsistent.

HW ngăn điều đó: consumer chỉ đọc đến HW — tức là chỉ đọc những message đã được đủ ISR chép xong.

```
Leader LEO = 100
f1 LEO     = 98
f2 LEO     = 95

HW = min(100, 98, 95) = 95
→ Consumer chỉ đọc đến offset 94
→ Offset 95-99 tồn tại trên leader nhưng consumer chưa thấy
```

---

### 11.3 Khi Nào Follower Bị Đuổi Khỏi ISR

**H: ISR shrink xảy ra khi nào?**

Đơn giản: Nếu follower **không hỏi leader trong 30 giây** (config `replica.lag.time.max.ms`), leader đuổi nó ra khỏi ISR.

Leader theo dõi "lần cuối follower hỏi tôi là bao giờ". Cứ mỗi 15 giây (= 30s / 2) leader kiểm tra:
- Nếu follower im lặng > 30s → bị đuổi ra khỏi ISR, thông báo cho Controller

**Nguyên nhân follower im lặng:**
- Follower broker bị crash
- Broker đang bận (GC pause kéo dài, disk chậm)
- Mạng bị đứt giữa follower và leader
- Follower đang restart

> **Lưu ý:** Trước Kafka 2.6 có thêm tiêu chí về số message bị lag (`replica.lag.max.messages`). Config này đã bỏ vì hay gây nhầm: lúc traffic đột biến, follower lag về số message dù đang fetch đều đặn → bị đuổi oan.

---

### 11.4 Follower Bị Đuổi Có Vào Lại ISR Được Không

**H: Follower quay lại ISR như thế nào?**

Được. Khi follower khởi động lại và bắt kịp leader, nó tự động được thêm trở lại ISR.

```
Follower crash và restart:

  Trước crash: follower ở offset 950
  Leader hiện tại: offset 1200

  Follower restart:
  → Đọc log local: "tôi đang ở offset 950"
  → Bắt đầu hỏi leader liên tục từ offset 950

  Leader theo dõi tiến trình catch-up:
  → Khi follower đã hỏi đến offset 1200 (bắt kịp)
  → Nếu từ lúc bắt đầu catch-up đến giờ < 30 giây
  → Leader thêm follower vào lại ISR
  → Thông báo Controller cập nhật metadata
```

**Trong khi follower đang catch-up:** HW không tăng vì follower chưa trong ISR. Chỉ sau khi vào lại ISR, follower mới tham gia vào tính toán HW.

**Vấn đề log bị lệch sau leader election:**

Khi follower reconnect, nó khai báo `leaderEpoch` (số thứ tự của đợt leader hiện tại). Leader dùng số này để kiểm tra: "log của mày có bị lệch không?"

Ví dụ: Leader cũ đã ghi offset 980-990 nhưng follower không kịp nhận. Leader cũ crash, leader mới lên. Leader mới có thể không có offset 980-990 đó. Follower quay lại với offset 990 → leader mới bảo: "epoch không khớp, mày cần xóa từ offset 980 trở đi rồi fetch lại từ đó."
