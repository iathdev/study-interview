# Replication

Cơ chế:
+ Bất đồng bộ
+ Tất cả các thay đổi (command) của master sẽ được lưu vào file binary log 
+ Slave đọc file binary log và thực hiện những thao tác trong file
+ I/O thread: đọc các sự kiện từ binary log trên master

| Vai trò    | Mô tả                                                                   |
| ---------- | ----------------------------------------------------------------------- |
| **Master** | Server chính, nơi xử lý **ghi dữ liệu (INSERT, UPDATE, DELETE)**        |
| **Slave**  | Một hoặc nhiều server phụ, dùng để **đọc dữ liệu (SELECT)** hoặc backup |

Mọi thao tác ghi trên master sẽ được log lại và gửi sang slave để slave thực hiện lại giống hệt.

## Cơ chế sao lưu & đồng bộ hoạt động như thế nào?

**Bước 1: Master ghi vào Binlog**
Khi có INSERT, UPDATE, DELETE, master không ghi thẳng sang slave mà ghi thao tác đó
vào file **binary log (binlog)** — đây là file nhật ký ghi lại mọi thay đổi dữ liệu theo thứ tự thời gian.

**Bước 2: Slave chủ động kéo binlog về (I/O Thread)**
Slave không ngồi chờ master đẩy sang. Thay vào đó, Slave chạy một **I/O Thread** liên tục
kết nối đến master và hỏi:
> “Tôi đã đọc đến position X, có gì mới không?”

Master gửi phần binlog mới → Slave copy về lưu vào **Relay Log** (bản sao local của binlog).

**Bước 3: Slave replay lại (SQL Thread)**
Slave chạy thêm một **SQL Thread** đọc Relay Log và thực thi lại từng lệnh y hệt master.
Kết quả: dữ liệu Slave giống Master gần như theo thời gian thực.

```
Client
  |
  | INSERT / UPDATE / DELETE
  v
Master ──── ghi vào Binlog (position tăng dần)
  |
  |  Slave I/O Thread hỏi: “Có gì mới từ position X không?”
  |  Master gửi phần mới
  v
Slave ───── lưu vào Relay Log
              |
              | SQL Thread đọc và replay
              v
           Slave DB (dữ liệu giống Master)
```

Vì là **async** nên giữa lúc master ghi xong và slave áp dụng xong có độ trễ nhỏ
→ **replication lag** → đọc từ slave có thể nhận dữ liệu cũ hơn master.

## Lợi ích

| Lợi ích            | Ghi chú                                          |
| ------------------ | ------------------------------------------------ |
| Tăng hiệu suất đọc | Cho phép các app đọc từ slave                    |
| Chống mất dữ liệu  | Nếu master hỏng, có thể phục hồi từ slave        |
| Hỗ trợ backup      | Có thể backup từ slave mà không ảnh hưởng master |
| Phân tải           | Tách read/write giúp server nhẹ hơn              |

## Lưu ý và rủi ro

| Vấn đề                           | Chi tiết                                                        |
| -------------------------------- | --------------------------------------------------------------- |
| **Replication lag**              | Slave có thể bị chậm hơn vài giây nếu nhiều ghi                 |
| **Không đồng bộ 2 chiều**        | Slave **chỉ đọc** – nếu bạn ghi vào slave → lỗi dữ liệu         |
| **Failover phức tạp**            | Nếu master chết, cần cấu hình lại để slave thành master         |
| **Không thay thế backup đầy đủ** | Replication ≠ backup (vì xóa nhầm trên master → slave cũng mất) |

## Khi master bị down thì làm cách nào để tự động chuyển slave thành master

Có 4 cách chính:

| Cách | Ai xử lý | Tự động failover | Scale read | Ghi chú |
|---|---|---|---|---|
| AWS RDS Multi-AZ | Cloud | Có | Không | Cần kết hợp Read Replicas |
| MySQL Group Replication | MySQL built-in | Có | Có (multi-primary) | Dùng Paxos bầu master mới |
| InnoDB Cluster | MySQL built-in | Có | Có | Bộ hoàn chỉnh hơn Group Replication |
| MHA / Orchestrator | Tool bên ngoài | Có | Không | Open source, tự deploy |

### 1. AWS RDS Multi-AZ (Cloud)
Master và Standby chạy ở 2 Availability Zone khác nhau, sync đồng bộ.
Khi master chết → AWS tự chuyển standby thành master + cập nhật DNS.

```
AZ-1: Primary (master) ──sync──> AZ-2: Standby
                                        |
                  master chết ──────────┘
                                        ↓
                               Standby lên làm master
                               DNS tự trỏ sang
```

Nhược điểm: không scale read, chi phí cao. Nên kết hợp thêm Read Replicas.

### 2. MySQL Group Replication (built-in)
Các node liên tục "bầu cử" với nhau theo giao thức Paxos.
Khi master chết, các node còn lại tự bầu ra master mới, không cần tool ngoài.

```
Node 1 (master) ─── Node 2 ─── Node 3
       |                           |
    chết ──────────────────────────┘
                                   ↓
                     Node 2 hoặc 3 được bầu làm master mới
```

Hỗ trợ 2 chế độ:
- **Single-primary**: 1 master ghi, các node còn lại chỉ đọc
- **Multi-primary**: nhiều node cùng ghi (phức tạp hơn, dễ conflict)

#### Paxos là gì?
Paxos là giao thức **đồng thuận phân tán** — giúp nhiều node thống nhất với nhau về
một quyết định dù có một số node bị chết hoặc mất kết nối.

Ý tưởng cốt lõi: **đa số thắng (majority)**
- Cluster 3 node → cần ít nhất 2 đồng ý
- Cluster 5 node → cần ít nhất 3 đồng ý
- Node chết < 50% thì cluster vẫn hoạt động bình thường

**Áp dụng vào bầu master mới:**

```
Bình thường:
Node1(master) ── Node2 ── Node3
  |                |         |
  └── ping OK ─────┘─────────┘

Node1 chết:
              ── Node2 ── Node3
                  |         |
Node2 gửi:   "Tôi muốn làm master, đồng ý không?"
Node3 trả:   "Đồng ý"
=> 2/3 đồng ý → Node2 được bầu làm master
```

**Tại sao cần đa số (majority)?**
Tránh "split-brain" — tình huống mạng bị chia đôi, 2 nhóm node không liên lạc được
với nhau, mỗi nhóm tự bầu master riêng → 2 master cùng nhận write → dữ liệu loạn.

```
Split-brain nếu không có majority:
Node1 ── Node2  |  Node3 ── Node4   (mạng bị cắt đôi)
   ↓                  ↓
master mới A    master mới B  ← 2 master cùng tồn tại → loạn
```

Với Paxos: nhóm nào không đủ majority thì không được bầu master → chỉ 1 master tồn tại.

### 3. InnoDB Cluster (built-in, hoàn chỉnh hơn)
Gói Group Replication + MySQL Router + MySQL Shell thành 1 bộ.
MySQL Router đứng trước cluster, tự redirect write sang master mới sau failover.
App chỉ connect vào Router, không cần quan tâm master là node nào.

```
App ──> MySQL Router ──> Node 1 (master)
                    └──> Node 2 (slave)
                    └──> Node 3 (slave)
        Router tự redirect khi master đổi
```

### 4. MHA / Orchestrator (tool bên ngoài, open source)
Monitor master liên tục. Khi master chết:
- Chọn slave có binlog mới nhất
- Promote slave đó lên làm master
- Redirect connection sang master mới

Phù hợp khi không dùng cloud và muốn tự kiểm soát hạ tầng.

# Binlog là gì?
Binlog (Binary Log) là file nhị phân trên máy chủ MySQL, ghi lại toàn bộ các thao tác ghi (write) ảnh hưởng đến dữ liệu.

Dùng cho:
- Replication (đồng bộ từ master → slave)
- Point-in-time recovery (khôi phục dữ liệu tới một thời điểm cụ thể)
- Audit (kiểm tra thay đổi dữ liệu)

## Binlog lưu cái gì?

| Loại ghi                          | Mô tả                               |
| --------------------------------- | ----------------------------------- |
| `INSERT`, `UPDATE`, `DELETE`      | Các thao tác làm thay đổi dữ liệu   |
| `CREATE TABLE`, `DROP`, `ALTER`   | Các thay đổi về schema (DDL)        |
| `BEGIN`, `COMMIT` (transaction)   | Ghi nhận phạm vi giao dịch          |
| `SET AUTOCOMMIT=1`, `USE db_name` | Thông tin môi trường                |
| Thông tin replication             | Position, server ID, GTID (nếu bật) |




