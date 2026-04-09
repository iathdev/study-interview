Trong **Apache Kafka**, các **message** (tin nhắn) trong một **partition** của một **topic** được lưu trữ dưới dạng **log bất biến** (append-only log), **không phải dạng stack** hay bất kỳ cấu trúc dữ liệu nào khác như queue hay stack. Dưới đây là giải thích chi tiết:

### 1. **Cách lưu trữ message trong partition**
- **Log bất biến (Append-only Log)**:
  - Mỗi partition trong một topic là một **log** được lưu trữ trên đĩa (disk) của **broker**. Log này hoạt động theo cách chỉ **thêm vào cuối** (append-only), nghĩa là các message mới được ghi vào cuối log và không bao giờ bị sửa đổi hay xóa (trừ khi hết thời gian lưu trữ hoặc giới hạn kích thước).
  - Mỗi message trong partition được gán một **offset** (một số nguyên tăng dần), đại diện cho vị trí của message trong log. Offset giúp consumer theo dõi vị trí đọc của mình.
  - Ví dụ: Nếu partition có 3 message, chúng sẽ được lưu với offset lần lượt là 0, 1, 2, và message mới sẽ được thêm vào với offset 3.

- **Cấu trúc vật lý**:
  - Trên đĩa, partition được lưu dưới dạng các **segment file** (tệp phân đoạn). Mỗi segment chứa một tập hợp các message, và Kafka chia log thành nhiều segment để quản lý hiệu quả (ví dụ: xóa các segment cũ khi hết thời gian lưu trữ).
  - Các message được lưu trữ dưới dạng **binary**, sử dụng định dạng nhị phân của Kafka (thường dựa trên **Protocol Buffers** hoặc cấu trúc tương tự), giúp tối ưu hóa kích thước và tốc độ xử lý.

- **Không phải Stack hay Queue**:
  - **Stack** (LIFO - Last In, First Out): Trong stack, phần tử được thêm vào cuối sẽ được lấy ra đầu tiên. Kafka không hoạt động theo cách này vì message được đọc theo thứ tự offset (từ đầu đến cuối log).
  - **Queue** (FIFO - First In, First Out): Mặc dù Kafka có thể được xem gần giống queue ở chỗ message được xử lý theo thứ tự, nhưng nó khác ở chỗ message không bị xóa sau khi consumer đọc (như queue truyền thống trong RabbitMQ). Thay vào đó, Kafka lưu trữ message lâu dài, cho phép replay.
  - Kafka giống một **log** hơn là queue hay stack, vì nó giữ lại toàn bộ lịch sử message (trong giới hạn cấu hình retention) và hỗ trợ consumer đọc từ bất kỳ offset nào.

### 2. **Cơ chế hoạt động**
- **Producer**: Khi một producer gửi message đến một topic, Kafka quyết định partition nào sẽ nhận message (dựa trên key hoặc round-robin). Message được thêm vào cuối log của partition đó với offset tiếp theo.
- **Consumer**: Consumer đọc message từ partition bằng cách chỉ định offset (ví dụ: từ offset 0 hoặc offset mới nhất). Consumer có thể đọc lại (replay) hoặc nhảy đến bất kỳ offset nào trong log.
- **Retention**: Kafka lưu trữ message trong partition cho đến khi vượt quá thời gian lưu trữ (**retention period**) hoặc giới hạn kích thước (**retention size**) được cấu hình. Sau đó, các segment cũ sẽ bị xóa.

### 3. **So sánh với Stack**
- **Stack** (LIFO): Trong stack, bạn chỉ có thể truy cập phần tử mới nhất (pop từ đỉnh). Kafka không hoạt động theo LIFO vì consumer có thể đọc từ bất kỳ offset nào, không bị giới hạn ở message mới nhất.
- **Kafka Log**: Message được lưu theo thứ tự thời gian (chronological order), và consumer thường đọc từ offset thấp đến cao (tương tự FIFO), nhưng có thể chọn đọc từ bất kỳ điểm nào trong log. Điều này làm Kafka linh hoạt hơn stack rất nhiều.

### 4. **Ví dụ minh họa**
Giả sử partition 0 của topic "orders" có các message:
```
Offset 0: { "order_id": 1, "item": "book" }
Offset 1: { "order_id": 2, "item": "pen" }
Offset 2: { "order_id": 3, "item": "laptop" }
```
- Một message mới `{ "order_id": 4, "item": "phone" }` sẽ được thêm vào cuối log với **offset 3**.
- Consumer có thể:
  - Đọc từ offset 0 (bắt đầu từ đầu log).
  - Đọc từ offset 2 (chỉ lấy message mới nhất).
  - Replay lại từ offset 1 để xử lý lại các message cũ.
- Message không bị xóa sau khi đọc, trừ khi hết thời gian retention (ví dụ: 7 ngày).

### 5. **Tại sao Kafka chọn Log bất biến thay vì Stack hay Queue?**
- **Khả năng replay**: Log bất biến cho phép consumer đọc lại dữ liệu cũ, rất hữu ích trong các kịch bản như phân tích dữ liệu, khắc phục lỗi, hoặc xử lý lại sự kiện.
- **Hiệu suất cao**: Ghi append-only nhanh hơn nhiều so với sửa đổi hoặc xóa dữ liệu (như trong queue/stack). Đọc tuần tự từ đĩa cũng rất hiệu quả.
- **Khả năng mở rộng**: Log bất biến kết hợp với partition cho phép Kafka xử lý song song và phân phối dữ liệu trên nhiều broker, phù hợp với big data và streaming.
- **Tính đơn giản**: Offset đơn giản hóa việc quản lý vị trí đọc của consumer, không cần cơ chế phức tạp như stack hay queue.

### 6. **So sánh với RabbitMQ**
- **RabbitMQ** (Queue-based): Tin nhắn thường được lưu trong queue và bị xóa sau khi consumer xử lý (trừ khi dùng RabbitMQ Streams). Queue giống FIFO hơn, nhưng không hỗ trợ replay dữ liệu dễ dàng như Kafka.
- **Kafka** (Log-based): Partition lưu trữ message dưới dạng log bất biến, cho phép lưu trữ lâu dài và replay linh hoạt, không giống queue hay stack.

### 7. **Tóm lại**
- Message trong partition của Kafka được lưu trữ dưới dạng **log bất biến (append-only log)**, không phải stack (LIFO) hay queue (FIFO truyền thống).
- Log bất biến lưu message theo thứ tự thời gian với offset tăng dần, cho phép consumer đọc linh hoạt từ bất kỳ vị trí nào và hỗ trợ replay.
- Điều này làm Kafka lý tưởng cho các ứng dụng streaming, big data, và xử lý dữ liệu lịch sử, khác với stack (chỉ truy cập phần tử mới nhất) hoặc queue (xóa sau khi tiêu thụ).

Nếu bạn cần thêm chi tiết, ví dụ mã, hoặc muốn tìm hiểu sâu hơn về cách partition hoạt động (như cấu hình retention, segment, hoặc consumer group), hãy cho mình biết!