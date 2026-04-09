Trong bối cảnh bài toán **concurrent booking** (đặt vé đồng thời), **khóa bi quan (pessimistic locking)** và **khóa lạc quan (optimistic locking)** là hai cơ chế phổ biến để đảm bảo không xảy ra **race condition** và **overbooking**. Dưới đây là so sánh chi tiết giữa hai phương pháp, tập trung vào đặc điểm, ưu điểm, nhược điểm, và cách áp dụng trong hệ thống chỉ sử dụng **MySQL** và **Redis**.

---

### 1. **Khóa bi quan (Pessimistic Locking)**

#### **Định nghĩa**
- Khóa bi quan giả định rằng xung đột giữa các giao dịch sẽ xảy ra, do đó khóa tài nguyên (ví dụ: hàng trong cơ sở dữ liệu hoặc key trong Redis) để ngăn các giao dịch khác truy cập đồng thời.
- Trong bài toán đặt vé, khóa bi quan đảm bảo chỉ một giao dịch được xử lý tại một thời điểm cho một sự kiện cụ thể.

#### **Cách hoạt động**
- **Trong MySQL**: Sử dụng `SELECT ... FOR UPDATE` để khóa hàng trong bảng `tickets`. Các giao dịch khác phải đợi cho đến khi khóa được giải phóng.
  ```sql
  START TRANSACTION;
  SELECT available_tickets FROM tickets WHERE event_id = 1 FOR UPDATE;
  -- Kiểm tra và cập nhật
  UPDATE tickets SET available_tickets = available_tickets - ? WHERE event_id = 1;
  INSERT INTO bookings (booking_id, user_id, event_id, tickets_booked) VALUES (?, ?, ?, ?);
  COMMIT;
  ```
- **Trong Redis**: Sử dụng khóa phân tán (distributed lock) với lệnh `SETNX` (set if not exists) để khóa key. Chỉ một giao dịch được phép xử lý tại một thời điểm.
  ```java
  Jedis jedis = new Jedis("localhost", 6379);
  if (jedis.setnx("lock:event:1", "locked") == 1) {
      jedis.expire("lock:event:1", 10); // TTL 10 giây
      // Xử lý đặt vé
      jedis.del("lock:event:1"); // Giải phóng khóa
  }
  ```

#### **Ưu điểm**
- **Đơn giản và đáng tin cậy**: Đảm bảo không có xung đột vì chỉ một giao dịch được xử lý tại một thời điểm.
- **Ngăn overbooking**: Do tài nguyên bị khóa, không có khả năng bán quá số vé.
- **Dễ triển khai trong MySQL**: `FOR UPDATE` là tính năng tích hợp sẵn, không cần logic phức tạp.

#### **Nhược điểm**
- **Hiệu suất thấp**: Khi có nhiều yêu cầu đồng thời, các giao dịch phải chờ đợi, dẫn đến độ trễ cao.
- **Tắc nghẽn (bottleneck)**: Trong hệ thống tải cao (ví dụ: hàng nghìn người đặt vé cùng lúc), hàng đợi chờ khóa có thể gây chậm hoặc sập hệ thống.
- **Deadlock**: Nếu nhiều giao dịch khóa nhiều tài nguyên theo thứ tự khác nhau, có thể xảy ra deadlock.
- **Không phù hợp với Redis**: Redis không hỗ trợ khóa bi quan mạnh như MySQL; khóa phân tán (`SETNX`) phức tạp và dễ bị lỗi nếu không quản lý TTL cẩn thận.

#### **Áp dụng trong bài toán**
- **MySQL**: Sử dụng `SELECT ... FOR UPDATE` để khóa hàng trong bảng `tickets` trước khi kiểm tra và cập nhật số vé.
- **Redis**: Ít được sử dụng cho khóa bi quan vì Redis thiên về hiệu suất và không có cơ chế khóa mạnh như MySQL.

---

### 2. **Khóa lạc quan (Optimistic Locking)**

#### **Định nghĩa**
- Khóa lạc quan giả định rằng xung đột hiếm khi xảy ra, cho phép nhiều giao dịch đọc dữ liệu đồng thời nhưng kiểm tra trước khi cập nhật để đảm bảo dữ liệu không bị thay đổi.
- Khi ghi (commit), hệ thống sẽ kiểm tra xem dữ liệu có bị thay đổi bởi tiến trình khác hay không:
  - Dựa trên version (số phiên bản) hoặc timestamp.
  - Nếu dữ liệu bị thay đổi => rollback.
  - Nếu không => ghi thành công.
- Trong bài toán đặt vé, sử dụng một cột `version` (hoặc tương tự trong Redis) để phát hiện xung đột.


#### **Cách hoạt động**
- **Trong MySQL**: Sử dụng cột `version` trong bảng `tickets`. Khi cập nhật, kiểm tra version có khớp không.
  ```sql
  START TRANSACTION;
  SELECT available_tickets, version FROM tickets WHERE event_id = 1;
  -- Kiểm tra số vé
  UPDATE tickets SET available_tickets = ?, version = version + 1 WHERE event_id = 1 AND version = ?;
  IF ROW_COUNT() = 0 THEN
      ROLLBACK; -- Xung đột, thử lại
  ELSE
      INSERT INTO bookings (booking_id, user_id, event_id, tickets_booked) VALUES (?, ?, ?, ?);
      COMMIT;
  END IF;
  ```
- **Trong Redis**: Sử dụng `WATCH`, `MULTI`, `EXEC` để theo dõi và cập nhật nguyên tử.
  ```java
  Jedis jedis = new Jedis("localhost", 6379);
  jedis.watch("event:1:available_tickets", "event:1:version");
  int availableTickets = Integer.parseInt(jedis.get("event:1:available_tickets"));
  int version = Integer.parseInt(jedis.get("event:1:version"));
  if (availableTickets >= ticketsRequested) {
      Pipeline pipeline = jedis.pipelined();
      pipeline.multi();
      pipeline.set("event:1:available_tickets", String.valueOf(availableTickets - ticketsRequested));
      pipeline.incr("event:1:version");
      if (pipeline.exec() != null) {
          // Cập nhật thành công
      } else {
          // Xung đột, thử lại
      }
  }
  ```

#### **Ưu điểm**
- **Hiệu suất cao**: Cho phép nhiều giao dịch đọc dữ liệu đồng thời, phù hợp với tải cao.
- **Tốt cho Redis**: Cơ chế `WATCH`/`MULTI`/`EXEC` được thiết kế để xử lý khóa lạc quan, tận dụng tốc độ của Redis.
- **Ít tắc nghẽn**: Không giữ khóa lâu, giảm thời gian chờ.
- **Phù hợp với hệ thống phân tán**: Dễ triển khai trong Redis khi kết hợp với MySQL.

#### **Nhược điểm**
- **Xử lý xung đột phức tạp**: Cần cơ chế thử lại (retry) khi xảy ra xung đột, làm tăng độ phức tạp.
- **Tỷ lệ thất bại cao trong tải cực cao**: Nếu nhiều giao dịch đồng thời, xác suất xung đột tăng, dẫn đến nhiều lần thử lại.
- **Yêu cầu đồng bộ**: Cần đảm bảo `version` nhất quán giữa Redis và MySQL.

#### **Áp dụng trong bài toán**
- **Redis**: Sử dụng `WATCH` để theo dõi `available_tickets` và `version`, cập nhật nguyên tử với `MULTI`/`EXEC`.
- **MySQL**: Cập nhật `tickets` với điều kiện `version`, đảm bảo tính nhất quán.

---

### So sánh chi tiết

| **Tiêu chí**                | **Khóa bi quan**                                                                 | **Khóa lạc quan**                                                               |
|-----------------------------|----------------------------------------------------------------------------------|--------------------------------------------------------------------------------|
| **Nguyên lý**               | Khóa tài nguyên trước khi xử lý, ngăn giao dịch khác truy cập.                   | Cho phép đọc đồng thời, kiểm tra xung đột trước khi cập nhật.                   |
| **MySQL**                   | Dùng `SELECT ... FOR UPDATE`.                                                   | Dùng cột `version` và kiểm tra điều kiện khi cập nhật.                          |
| **Redis**                   | Dùng khóa phân tán (`SETNX`), phức tạp và ít phổ biến.                           | Dùng `WATCH`/`MULTI`/`EXEC`, phù hợp với thiết kế của Redis.                    |
| **Hiệu suất**               | Thấp trong tải cao do chờ đợi khóa.                                              | Cao, cho phép đọc đồng thời, chỉ thử lại khi có xung đột.                      |
| **Xần xuất xung đột**       | Không có xung đột do khóa tài nguyên.                                           | Có thể xảy ra xung đột, cần thử lại.                                           |
| **Độ phức tạp**             | Đơn giản, ít logic xử lý xung đột.                                              | Phức tạp hơn, cần quản lý retry và version.                                    |
| **Tắc nghẽn**               | Dễ gây tắc nghẽn khi nhiều yêu cầu đồng thời.                                    | Ít tắc nghẽn, phù hợp với tải cao.                                             |
| **Deadlock**                | Có nguy cơ deadlock nếu không quản lý khóa cẩn thận.                             | Không có deadlock, nhưng có thể thất bại nếu xung đột nhiều.                   |
| **Phù hợp với tải**         | Tốt cho hệ thống nhỏ, ít giao dịch đồng thời.                                    | Tốt cho hệ thống lớn, nhiều giao dịch đọc.                                     |
| **Áp dụng trong bài toán**  | Khóa hàng trong MySQL, ít dùng trong Redis.                                      | Dùng Redis cho kiểm tra nhanh, đồng bộ với MySQL.                              |

---

### Áp dụng trong bài toán concurrent booking (MySQL + Redis)

#### **Khóa bi quan**
- **MySQL**: Sử dụng `SELECT ... FOR UPDATE` để khóa hàng trong bảng `tickets`. Redis có thể lưu cache số vé nhưng không cần thiết vì MySQL đã đảm bảo tính nhất quán.
  ```sql
  START TRANSACTION;
  SELECT available_tickets FROM tickets WHERE event_id = 1 FOR UPDATE;
  IF available_tickets >= ? THEN
      UPDATE tickets SET available_tickets = available_tickets - ? WHERE event_id = 1;
      INSERT INTO bookings (booking_id, user_id, event_id, tickets_booked) VALUES (?, ?, ?, ?);
      COMMIT;
  ELSE
      ROLLBACK;
  END IF;
  ```
- **Redis**: Nếu dùng, triển khai khóa phân tán (`SETNX`), nhưng phức tạp và ít hiệu quả hơn MySQL.
- **Nhược điểm**: Trong tải cao (ví dụ: hàng nghìn người đặt vé công viên cùng lúc), khóa bi quan gây chậm trễ lớn do các giao dịch phải chờ.

#### **Khóa lạc quan**
- **Redis**: Sử dụng `WATCH`/`MULTI`/`EXEC` - `theo dõi/đánh dấu start transaction/thực thi` để kiểm tra và cập nhật `available_tickets` và `version`.
- **MySQL**: Đồng bộ dữ liệu sau khi Redis cập nhật thành công, sử dụng giao dịch để ghi `bookings` và cập nhật `tickets`.
- **Ưu điểm**: Phù hợp với tải cao, tận dụng tốc độ của Redis để kiểm tra nhanh, giảm tải cho MySQL.
- **Nhược điểm**: Cần đồng bộ Redis và MySQL, xử lý retry khi có xung đột.

#### **Mã minh họa khóa lạc quan (từ mã bạn cung cấp)**
- Cập nhật Redis:
  ```java
  jedis.watch("event:" + eventId + ":available_tickets", "event:" + eventId + ":version");
  Pipeline pipeline = jedis.pipelined();
  pipeline.multi();
  pipeline.set("event:" + eventId + ":available_tickets", String.valueOf(availableTickets - ticketsRequested));
  pipeline.incr("event:" + eventId + ":version");
  Response<List<Object>> result = pipeline.exec();
  ```
- Đồng bộ MySQL:
  ```java
  try (PreparedStatement stmt = dbConn.prepareStatement(
          "UPDATE tickets SET available_tickets = ?, version = ? WHERE event_id = ?")) {
      stmt.setInt(1, availableTickets - ticketsRequested);
      stmt.setInt(2, currentVersion + 1);
      stmt.setInt(3, eventId);
      stmt.executeUpdate();
  }
  ```

---

### Đề xuất cho bài toán
- **Khóa lạc quan với Redis + MySQL** (như mã bạn cung cấp) là lựa chọn tốt hơn cho hệ thống tải cao:
  - Redis xử lý kiểm tra nhanh và phát hiện xung đột.
  - MySQL đảm bảo tính toàn vẹn dữ liệu lâu dài.
  - Phù hợp với bài toán đặt vé công viên, nơi có nhiều yêu cầu đồng thời.
- **Khóa bi quan** phù hợp hơn cho hệ thống nhỏ, ít giao dịch đồng thời, hoặc khi muốn đơn giản hóa logic (chỉ dùng MySQL, không cần Redis).

---

### Kết luận
- **Khóa bi quan**: Đơn giản, đáng tin cậy, nhưng kém hiệu quả trong tải cao do tắc nghẽn.
- **Khóa lạc quan**: Hiệu suất cao, phù hợp với Redis và hệ thống lớn, nhưng cần xử lý retry và đồng bộ.
- Trong bối cảnh chỉ dùng MySQL và Redis, **khóa lạc quan** với Redis (`WATCH`/`MULTI`/`EXEC`) kết hợp giao dịch MySQL là lựa chọn tối ưu cho bài toán **concurrent booking**, như đã triển khai trong mã của bạn.

Nếu bạn cần thêm ví dụ mã hoặc so sánh chi tiết hơn trong trường hợp cụ thể, hãy cho tôi biết!