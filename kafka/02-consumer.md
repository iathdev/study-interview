# Consumer Groups, Offsets và Rebalancing

> Tổng hợp từ tài liệu chính thức, Confluent engineering blog, Strimzi, và các nguồn phỏng vấn senior.

---

## 2. Consumer Groups, Offsets và Rebalancing

### 2.1 Consumer Groups

**H: Consumer group hoạt động như thế nào?**

Consumer group là tập hợp consumer cùng nhau consume một topic. Kafka đảm bảo:
- Mỗi partition được assign cho **đúng một consumer** trong group
- Các consumer group khác nhau mỗi cái duy trì offset độc lập — nhiều group có thể consume cùng topic độc lập

**Group Coordinator** là broker chịu trách nhiệm quản lý group membership. Được xác định theo công thức: `hash(groupId) % numPartitions` của topic `__consumer_offsets`.

**H: Group Leader là gì?**

Không nên nhầm với partition leader. Group Leader là một **consumer** trong group, được Group Coordinator bầu trong quá trình join. Group Leader:
- Nhận danh sách membership đầy đủ từ coordinator
- Chạy thuật toán partition assignment
- Gửi assignment về coordinator
- Thay đổi sau mỗi rebalance — là ephemeral

**H: `__consumer_offsets` lưu gì?**

- Committed offset theo (group, topic, partition)
- Group metadata (thành viên, protocol, assignment)
- Offset được lưu là **offset tiếp theo cần consume** (last processed + 1)

---

### 2.2 Quản Lý Offset

**H: Sự khác biệt giữa các giá trị `auto.offset.reset`?**

| Giá trị | Hành vi |
|---------|---------|
| `latest` (mặc định) | Bắt đầu từ cuối; bỏ qua message được produce trước khi consumer join |
| `earliest` | Bắt đầu từ đầu partition (offset 0 hoặc oldest được retain) |
| `none` | Throw exception nếu không tìm thấy committed offset |

**H: Rủi ro của `enable.auto.commit = true` là gì?**

Auto-commit kích hoạt mỗi `auto.commit.interval.ms` (mặc định 5000ms) bất kể trạng thái xử lý. Rủi ro:
- **Mất message**: Offset được commit trước khi message được xử lý hoàn toàn. Nếu consumer crash, message bị bỏ qua khi restart.
- **Duplicate tinh tế**: Auto-commit commit một batch; nếu một vài record cuối trong batch chưa xử lý trước khi commit, chúng bị bỏ qua.

**Best practice**: Dùng `enable.auto.commit = false` và gọi `commitSync()` sau khi xử lý thành công (at-least-once) hoặc dùng transaction (exactly-once).

---

### 2.3 Rebalancing Protocols

**H: Điều gì trigger rebalance?**

- Consumer join group
- Consumer rời group (graceful hoặc qua timeout)
- Partition mới được thêm vào topic đang subscribe
- Consumer gọi `subscribe()` với topic pattern khác

**H: Sự khác biệt giữa eager và cooperative rebalancing?**

**Eager (Legacy) Rebalancing** (Stop-the-World):
1. Tất cả consumer trong group revoke TẤT CẢ partition của mình
2. Tất cả consumer rejoin group
3. Group Leader tính lại assignment
4. Consumer nhận assignment mới và tiếp tục
- **Vấn đề**: Toàn bộ group dừng xử lý, kể cả consumer không thay đổi assignment

**Cooperative Sticky Rebalancing** (Incremental, Mặc định từ Kafka 2.4+):
1. Coordinator xác định partition nào cần chuyển
2. Chỉ consumer bị ảnh hưởng revoke những partition cụ thể đó
3. Một vòng rebalance thứ hai assign partition bị revoke cho owner mới
4. Consumer không bị ảnh hưởng **tiếp tục xử lý trong suốt quá trình**
- **Kết quả**: Gián đoạn tối thiểu; chỉ partition thực sự thay đổi chủ mới gây pause ngắn

Config qua: `partition.assignment.strategy = org.apache.kafka.clients.consumer.CooperativeStickyAssignor`

**H: Static group membership là gì và khi nào nên dùng?**

Đặt `group.instance.id` thành một giá trị stable, unique per consumer để bật static membership. Lợi ích:
- Consumer reconnect trong `session.timeout.ms` giữ nguyên partition assignment mà không trigger rebalance
- Quan trọng cho Kubernetes rolling deployment khi pod restart thường xuyên
- Loại bỏ rebalance không cần thiết do JVM GC pause hoặc network interruption ngắn

---

### 2.4 Cấu Hình Timeout Quan Trọng

| Config | Mặc định | Hướng dẫn |
|--------|----------|-----------|
| `session.timeout.ms` | 10s | Thời gian tối đa không có heartbeat trước khi consumer bị coi là dead. Đặt lớn hơn thời gian xử lý message đơn lẻ tối đa. Production: 30-45s |
| `heartbeat.interval.ms` | 3s | Tần suất gửi heartbeat. Phải < `session.timeout.ms`. Rule of thumb: 1/3 session timeout |
| `max.poll.interval.ms` | 300s (5 phút) | Thời gian tối đa giữa các lần gọi `poll()`. Nếu vượt quá, consumer bị coi là lỗi và bị remove khỏi group. **Tách biệt với heartbeat.** |

**Bẫy — Heartbeat vs Poll**: Heartbeat thread và poll thread là **riêng biệt** trong Java client. Consumer có thể gửi heartbeat (vẫn ở trong group) trong khi bị mắc kẹt ở `poll()` xử lý record. `max.poll.interval.ms` bắt trường hợp này — nếu không gọi `poll()` trong cửa sổ này, consumer bị evict dù heartbeat vẫn OK.

**Rebalancing storm**: Nếu `session.timeout.ms` quá thấp và GC pause vượt quá nó, consumer bị evict, rejoin, nhận partition, gây rebalance khác, bị evict lại — vòng lặp ngăn mọi progress.

---

## 12. Consumer Group Coordinator — Giao Thức JoinGroup/SyncGroup

### 12.1 Tìm Group Coordinator

**H: Consumer tìm Group Coordinator như thế nào?**

Group Coordinator là một broker cụ thể chịu trách nhiệm quản lý một consumer group. Được xác định bằng công thức:

```
coordinatorBroker = broker sở hữu partition:
  partitionIndex = abs(hash(groupId)) % numPartitions(__consumer_offsets)
```

Topic `__consumer_offsets` mặc định có 50 partition. Broker nào là leader của partition tương ứng sẽ là Group Coordinator.

**Flow để tìm Coordinator:**
```
1. Consumer gửi FindCoordinatorRequest đến bất kỳ broker nào:
   FindCoordinatorRequest {
     key: "my-consumer-group",
     keyType: GROUP (0)
   }

2. Broker tính toán và trả về:
   FindCoordinatorResponse {
     coordinator: {nodeId: 3, host: "broker-3", port: 9092}
   }

3. Consumer kết nối trực tiếp đến broker-3 cho tất cả group operations
```

---

### 12.2 JoinGroup Protocol — Từng Bước

**H: Quy trình JoinGroup/SyncGroup diễn ra như thế nào khi consumer join group?**

**Phase 1: JoinGroup**

```
Tất cả consumer trong group gửi JoinGroupRequest đến Coordinator:
  JoinGroupRequest {
    groupId: "my-group",
    sessionTimeoutMs: 30000,
    rebalanceTimeoutMs: 300000,
    memberId: "",             // "" nếu là lần đầu join
    groupInstanceId: null,    // null nếu không dùng static membership
    protocolType: "consumer",
    protocols: [
      {name: "cooperative-sticky", metadata: <serialized subscription>}
    ]
  }
```

**Coordinator chờ** tất cả member hiện tại gửi JoinGroupRequest (hoặc hết `rebalanceTimeoutMs`).

**Coordinator phản hồi JoinGroupResponse:**
```
Với member được chọn làm Group Leader:
  JoinGroupResponse {
    generationId: 5,          // tăng sau mỗi rebalance
    protocolName: "cooperative-sticky",
    leader: "consumer-1",
    memberId: "consumer-1",
    members: [                // chỉ leader nhận đầy đủ danh sách
      {memberId: "consumer-1", metadata: <subscription>},
      {memberId: "consumer-2", metadata: <subscription>},
      {memberId: "consumer-3", metadata: <subscription>}
    ]
  }

Với member thường:
  JoinGroupResponse {
    generationId: 5,
    protocolName: "cooperative-sticky",
    leader: "consumer-1",
    memberId: "consumer-2",
    members: []               // member thường không thấy danh sách
  }
```

**Phase 2: SyncGroup**

```
Group Leader tính toán partition assignment (chạy trên client, không phải broker)
→ Tạo ra map: {consumer-1: [p0, p3], consumer-2: [p1, p4], consumer-3: [p2, p5]}

Leader gửi SyncGroupRequest với assignment:
  SyncGroupRequest {
    groupId: "my-group",
    generationId: 5,
    memberId: "consumer-1",
    assignments: [
      {memberId: "consumer-1", assignment: <serialized [p0, p3]>},
      {memberId: "consumer-2", assignment: <serialized [p1, p4]>},
      {memberId: "consumer-3", assignment: <serialized [p2, p5]>}
    ]
  }

Member thường cũng gửi SyncGroupRequest (với assignments rỗng):
  SyncGroupRequest {
    groupId: "my-group",
    generationId: 5,
    memberId: "consumer-2",
    assignments: []
  }
```

**Coordinator phản hồi SyncGroupResponse cho mỗi member:**
```
SyncGroupResponse {
  assignment: <serialized [p1, p4]>  // assignment riêng của member này
}
```

**Phase 3: Normal operation**

```
Consumer bắt đầu fetch records từ assigned partitions
Consumer gửi Heartbeat định kỳ:
  HeartbeatRequest {groupId, generationId, memberId}
  HeartbeatResponse {errorCode: NONE}

Nếu Coordinator trả về error REBALANCE_IN_PROGRESS trong HeartbeatResponse
→ Consumer phải JoinGroup lại (trigger phase 1)
```

---

**H: Tại sao partition assignment logic chạy trên client (Group Leader) chứ không phải trên broker?**

Thiết kế có chủ đích:
1. **Extensibility**: Có thể plug-in custom `PartitionAssignor` ở phía client mà không cần thay đổi broker
2. **Broker simplicity**: Broker không cần biết gì về semantics của consumer group
3. **Flexibility**: Trong tương lai, cùng protocol có thể dùng cho loại workload khác

**Nhược điểm**: Nếu Group Leader chạy version client cũ hơn, nó có thể không hỗ trợ protocol assignment mới nhất. Rolling upgrade cần cẩn thận.

---

### 12.3 Cooperative Rebalancing — Cơ Chế Nội Bộ

**H: Cooperative Sticky Rebalancing khác Eager Rebalancing ở cơ chế nào?**

**Eager**: Tất cả member revoke partition trong JoinGroup round đầu, rồi nhận assignment mới trong SyncGroup. **Toàn bộ group bị pause.**

**Cooperative**: Diễn ra trong **2 vòng rebalance**:

```
Vòng 1:
  JoinGroup: Mỗi member gửi subscription + current assignment (partition đang giữ)
  Coordinator/Leader tính toán: partition nào cần chuyển chủ
  SyncGroup: Chỉ member CẦN REVOKE mới trả về danh sách partition cần nhả

  → Member không bị ảnh hưởng: tiếp tục fetch bình thường (không pause!)
  → Member cần revoke: dừng fetch partition đó, nhưng giữ partition khác

Vòng 2 (trigger ngay sau):
  JoinGroup lại (chỉ các member đã revoke)
  SyncGroup: Assign partition đã revoke cho owner mới
```

**Kết quả**: Chỉ partition thực sự thay đổi chủ mới bị pause ngắn. Consumer group lớn không bị ảnh hưởng toàn bộ.
