# 1.5 Consumer Group

## Vấn đề
Producers gửi rất nhiều dữ liệu vào topic mà chỉ có 1 consumer để xử lý => consumer không đủ khả năng xử lý -> Cần tăng số lượng consumer -> Để đảm bảo mỗi message trong 1 topic được xử lý **một lần duy nhất** -> sinh ra khái niệm **Consumer Group**.

## Quy tắc cốt lõi
> **Mỗi partition chỉ được gán cho đúng 1 consumer trong cùng 1 group.**

Đây là quy tắc quan trọng nhất để hiểu 2 trường hợp bên dưới.

## Hai trường hợp cần lưu ý

### TH1: Số consumer < số partition — một consumer đọc nhiều partition

```
Topic: orders (6 partitions)
Consumer Group: group-A (3 consumers)

Consumer 1 ← Partition 0, Partition 1
Consumer 2 ← Partition 2, Partition 3
Consumer 3 ← Partition 4, Partition 5
```

**Tại sao?** Kafka phải đảm bảo mọi partition đều có consumer đọc (không được bỏ sót message). Mà số consumer ít hơn partition → một consumer phải "gánh" nhiều partition.

**Hệ quả:**
- Consumer phải xử lý nhiều hơn → tải không đều, **có thể bị quá tải** nếu lượng message lớn mà consumer không đủ khả năng xử lý kịp
- Consumer xử lý không kịp → message bị tồn đọng (lag) trong partition → tăng độ trễ (latency) của hệ thống
- Nhưng **không mất message** — mọi partition đều được đọc, chỉ là chậm

**Giải pháp:** Thêm consumer vào group (tối đa bằng số partition) để phân tải đều hơn, giảm nguy cơ quá tải.

---

### TH2: Số consumer > số partition — có consumer bị inactive

```
Topic: orders (3 partitions)
Consumer Group: group-A (5 consumers)

Consumer 1 ← Partition 0
Consumer 2 ← Partition 1
Consumer 3 ← Partition 2
Consumer 4 ← (không có partition → IDLE)
Consumer 5 ← (không có partition → IDLE)
```

**Tại sao?** Vì quy tắc "1 partition chỉ gán cho 1 consumer", mà chỉ có 3 partition → chỉ cần 3 consumer. 2 consumer thừa không có partition nào để đọc → ngồi chơi (idle).

**Hệ quả:**
- Consumer idle **lãng phí tài nguyên** (RAM, CPU, network connection) mà không làm gì
- Không giúp tăng throughput — vì partition là đơn vị song song nhỏ nhất

**Giải pháp:**
- Tăng số partition của topic (nếu cần scale thêm consumer)
- Hoặc giảm consumer cho bằng số partition

---

### Tóm tắt

| Trường hợp | Kết quả | Vấn đề |
|---|---|---|
| Consumer = Partition | Lý tưởng, 1:1 | Không |
| Consumer < Partition | 1 consumer đọc nhiều partition | Tải không đều, có thể chậm |
| Consumer > Partition | Consumer thừa bị idle | Lãng phí tài nguyên |

> **Quy tắc vàng: Số consumer trong group nên ≤ số partition. Lý tưởng nhất là bằng nhau.**

## Rebalance — cơ chế phân phối lại partition

Khi consumer join/leave group, Kafka tự động **rebalance** — phân phối lại partition cho các consumer còn lại.

Ví dụ:
- Topic có 4 partitions, group có 2 consumers → mỗi consumer đọc 2 partitions
- 1 consumer chết → consumer còn lại nhận cả 4 partitions
- Thêm 1 consumer mới → rebalance lại thành 2-2

## Tại sao cần consumer group chứ không chỉ thêm consumer?

Nếu 2 consumer đọc cùng 1 partition mà **không thuộc group** → cả 2 đều nhận message → **xử lý trùng lặp**. Consumer group giải quyết bằng cách đảm bảo mỗi partition chỉ gán cho 1 consumer trong group.

---

# `__consumer_offsets` — Internal Topic

## Khái niệm
Khi consumer đọc message từ Kafka, nó cần ghi nhớ **offset (vị trí)** cuối cùng đã đọc, để khi khởi động lại có thể tiếp tục từ đúng điểm đó.

`__consumer_offsets` là internal topic trong Kafka, lưu trữ offset của consumer groups.

## Hình dung dễ hiểu

```
__consumer_offsets giống như cuốn sổ đánh dấu trang (bookmark):

- Bạn đang đọc 3 cuốn sách (3 partitions)
- Mỗi lần đọc xong 1 trang, bạn ghi lại số trang vào sổ bookmark
- Nếu bạn ngủ quên (consumer restart), mở sổ bookmark ra → biết đọc tiếp từ đâu
- Nếu bạn nhờ bạn khác đọc thay (rebalance), bạn đó cũng mở sổ bookmark → biết đọc từ đâu
```

## Cấu trúc dữ liệu
- Lưu dạng key-value: `(group_id, topic, partition) → offset`
- Mặc định có **50 partitions** (config `offsets.topic.num.partitions`)

## Cách consumer commit offset

| Cách | Cơ chế | Ưu điểm | Nhược điểm |
|---|---|---|---|
| **Auto commit** (`enable.auto.commit=true`) | Tự động commit mỗi 5s | Đơn giản, không cần code thêm | Có thể xử lý trùng nếu consumer chết giữa chừng |
| **Manual commit** | Tự kiểm soát khi nào commit | Chính xác hơn | Phải code thêm logic commit |

## Rủi ro khi commit offset

| Chiến lược | Rủi ro |
|---|---|
| Commit **trước** khi xử lý xong | **Mất message** — commit offset 10, nhưng chết ở message 8 → message 8,9 không được xử lý lại |
| Commit **sau** khi xử lý xong | **Xử lý trùng** — xử lý xong message 10 nhưng chưa kịp commit thì chết → restart đọc lại từ offset cũ |

Đây là lý do Kafka đảm bảo **at-least-once** delivery mặc định. Muốn **exactly-once** phải dùng thêm `idempotent producer` + `transactional API`.
