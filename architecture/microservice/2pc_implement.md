Trong Cam kết Hai Giai đoạn (Two-Phase Commit - 2PC), **Coordinator** ghi trạng thái giao dịch để đảm bảo tính bền vững và hỗ trợ khôi phục sau lỗi (failure recovery). Việc ghi trạng thái, nơi lưu trữ và cách thực hiện phụ thuộc vào hệ thống và công nghệ được sử dụng. Dưới đây là giải thích chi tiết bằng tiếng Việt:

### Coordinator ghi trạng thái như thế nào?
Coordinator ghi lại các trạng thái quan trọng của giao dịch trong suốt quá trình 2PC để đảm bảo rằng, ngay cả khi xảy ra lỗi (như sập máy hoặc mất kết nối), hệ thống vẫn có thể khôi phục và hoàn thành giao dịch một cách nhất quán. Các trạng thái thường được ghi bao gồm:

1. **Bắt đầu giao dịch (Transaction Start)**: Khi Coordinator khởi tạo giao dịch.
2. **Giai đoạn Chuẩn bị (Prepare Phase)**: 
   - Ghi lại danh sách các Participants và trạng thái yêu cầu "prepare" đã gửi.
   - Ghi lại phản hồi từ các Participants ("Sẵn sàng" hoặc "Hủy").
3. **Quyết định giao dịch (Decision)**: Ghi lại quyết định cuối cùng ("cam kết" hoặc "hủy").
4. **Giai đoạn Cam kết/Hủy (Commit/Abort Phase)**: Ghi lại trạng thái khi gửi lệnh "commit" hoặc "rollback" đến các Participants.
5. **Hoàn tất (Completion)**: Ghi lại khi giao dịch hoàn tất hoặc bị hủy.

**Mục đích**:
- Đảm bảo tính **bền vững (durability)**: Trạng thái được lưu trữ để không bị mất khi xảy ra lỗi.
- Hỗ trợ **khôi phục (recovery)**: Coordinator có thể kiểm tra trạng thái cuối cùng để tiếp tục hoặc hoàn tác giao dịch sau khi khởi động lại.

### Lưu trạng thái vào đâu?
Coordinator lưu trạng thái giao dịch vào một **bộ nhớ bền vững (persistent storage)** để đảm bảo dữ liệu không bị mất khi hệ thống gặp sự cố. Các tùy chọn lưu trữ phổ biến bao gồm:

1. **Cơ sở dữ liệu**:
   - Sử dụng một bảng cơ sở dữ liệu (ví dụ: MySQL, PostgreSQL, MongoDB) để lưu trạng thái giao dịch.
   - Ví dụ: Một bảng `TransactionLog` với các cột như `TransactionID`, `State` (prepare, commit, abort), `Participants`, `Timestamp`.
   - Ưu điểm: Dễ tích hợp với hệ thống hiện có, hỗ trợ truy vấn và quản lý.
   - Nhược điểm: Thêm độ trễ do truy cập cơ sở dữ liệu.

2. **Hệ thống tệp (File System)**:
   - Ghi trạng thái vào tệp nhật ký (log file) trên đĩa.
   - Ví dụ: Một tệp `transaction.log` với các dòng như: `[Timestamp] TransactionID:123 Prepare Sent`, `[Timestamp] TransactionID:123 All Ready`.
   - Ưu điểm: Đơn giản, nhanh cho hệ thống nhỏ.
   - Nhược điểm: Khó mở rộng trong hệ thống lớn, cần quản lý đồng bộ hóa khi ghi.

3. **Hệ thống lưu trữ phân tán**:
   - Sử dụng các hệ thống như **ZooKeeper**, **etcd**, hoặc **Redis** để lưu trạng thái.
   - Ví dụ: Trong ZooKeeper, trạng thái được lưu dưới dạng các node (znode) với key như `/transactions/123/state`.
   - Ưu điểm: Hỗ trợ hệ thống phân tán, đảm bảo tính sẵn sàng cao và đồng bộ.
   - Nhược điểm: Phức tạp hơn, yêu cầu thiết lập thêm cơ sở hạ tầng.

4. **Hàng đợi tin nhắn (Message Queue)**:
   - Sử dụng **Kafka**, **RabbitMQ**, hoặc **ActiveMQ** để lưu trạng thái dưới dạng tin nhắn.
   - Ví dụ: Coordinator đẩy trạng thái giao dịch vào một topic Kafka, như `transaction-log`.
   - Ưu điểm: Phù hợp cho hệ thống microservices, dễ tích hợp với luồng xử lý sự kiện.
   - Nhược điểm: Cần cơ chế đảm bảo tin nhắn được xử lý chính xác một lần (exactly-once).

5. **Bộ nhớ trong (In-Memory) với sao lưu**:
   - Lưu trạng thái trong RAM (ví dụ: Redis) và định kỳ sao lưu ra đĩa.
   - Ưu điểm: Nhanh, phù hợp cho hiệu suất cao.
   - Nhược điểm: Cần cơ chế sao lưu cẩn thận để tránh mất dữ liệu.

### Làm thế nào để ghi trạng thái?
Cách Coordinator ghi trạng thái phụ thuộc vào công nghệ và thiết kế hệ thống. Dưới đây là các bước và ví dụ triển khai:

#### 1. Quy trình ghi trạng thái
- **Bước 1: Khởi tạo giao dịch**:
  - Tạo một ID giao dịch duy nhất (`TransactionID`) và ghi trạng thái "bắt đầu" vào bộ nhớ bền vững.
- **Bước 2: Giai đoạn Chuẩn bị**:
  - Ghi danh sách Participants và trạng thái "prepare sent".
  - Sau khi nhận phản hồi từ Participants, ghi trạng thái "all ready" hoặc "abort" dựa trên phản hồi.
- **Bước 3: Quyết định giao dịch**:
  - Ghi quyết định "commit" hoặc "abort" trước khi gửi lệnh đến Participants.
- **Bước 4: Hoàn tất**:
  - Ghi trạng thái "committed" hoặc "aborted" sau khi tất cả Participants xác nhận.

#### 2. Ví dụ triển khai
##### a. Ghi vào cơ sở dữ liệu (MySQL)
- **Thiết kế bảng**:
  ```sql
  CREATE TABLE TransactionLog (
      TransactionID VARCHAR(50) PRIMARY KEY,
      State VARCHAR(20), -- prepare, commit, abort, completed
      Participants TEXT, -- JSON hoặc danh sách Participants
      Timestamp DATETIME
  );
  ```
- **Triển khai bằng Java (Spring/JDBC)**:
  ```java
  @Service
  public class CoordinatorService {
      @Autowired
      private JdbcTemplate jdbcTemplate;

      public void logTransactionState(String txId, String state, String participants) {
          String sql = "INSERT INTO TransactionLog (TransactionID, State, Participants, Timestamp) VALUES (?, ?, ?, NOW())";
          jdbcTemplate.update(sql, txId, state, participants);
      }

      public void preparePhase(String txId, String participants) {
          logTransactionState(txId, "prepare", participants);
          // Gửi yêu cầu prepare đến Participants
          boolean allReady = sendPrepareRequests(participants);
          logTransactionState(txId, allReady ? "all_ready" : "abort", participants);
      }

      public void commitPhase(String txId, String participants) {
          logTransactionState(txId, "commit", participants);
          // Gửi yêu cầu commit
          sendCommitRequests(participants);
          logTransactionState(txId, "committed", participants);
      }
  }
  ```
- **Khôi phục**: Khi Coordinator khởi động lại, truy vấn bảng `TransactionLog` để tìm các giao dịch chưa hoàn tất và tiếp tục (commit hoặc rollback).

##### b. Ghi vào ZooKeeper
- **Cấu trúc znode**:
  - `/transactions/tx123/prepare`: Lưu trạng thái prepare và danh sách Participants.
  - `/transactions/tx123/decision`: Lưu quyết định (commit/abort).
  - `/transactions/tx123/completed`: Đánh dấu giao dịch hoàn tất.
- **Triển khai bằng Go**:
  ```go
  import "github.com/go-zookeeper/zk"

  type Coordinator struct {
      zkConn *zk.Conn
  }

  func (c *Coordinator) logState(txID, state, participants string) error {
      path := "/transactions/" + txID + "/" + state
      data := []byte(participants)
      _, err := c.zkConn.Create(path, data, 0)
      return err
  }

  func (c *Coordinator) preparePhase(txID, participants string) error {
      err := c.logState(txID, "prepare", participants)
      if err != nil {
          return err
      }
      // Gửi yêu cầu prepare
      allReady := sendPrepareRequests(participants)
      state := "abort"
      if allReady {
          state = "all_ready"
      }
      return c.logState(txID, state, participants)
  }
  ```
- **Khôi phục**: Kiểm tra các znode trong `/transactions` để tìm giao dịch chưa hoàn tất.

##### c. Ghi vào Kafka
- **Cấu trúc topic**:
  - Topic `transaction-log` lưu các tin nhắn như `{ "txId": "123", "state": "prepare", "participants": "..." }`.
- **Triển khai bằng Python (Kafka)**:
  ```python
  from kafka import KafkaProducer
  import json

  class Coordinator:
      def __init__(self):
          self.producer = KafkaProducer(bootstrap_servers=['localhost:9092'])

      def log_state(self, tx_id, state, participants):
          message = {
              'txId': tx_id,
              'state': state,
              'participants': participants,
              'timestamp': str(datetime.now())
          }
          self.producer.send('transaction-log', json.dumps(message).encode('utf-8'))
          self.producer.flush()

      def prepare_phase(self, tx_id, participants):
          self.log_state(tx_id, 'prepare', participants)
          all_ready = send_prepare_requests(participants)
          state = 'all_ready' if all_ready else 'abort'
          self.log_state(tx_id, state, participants)
  ```
- **Khôi phục**: Consumer Kafka đọc topic `transaction-log` để tìm trạng thái cuối cùng của giao dịch.

### Lưu ý khi ghi trạng thái
1. **Bền vững**: Đảm bảo trạng thái được ghi vào bộ nhớ không bay hơi (non-volatile) trước khi chuyển sang bước tiếp theo.
2. **Đồng bộ hóa**: Trong hệ thống phân tán, sử dụng khóa (lock) hoặc cơ chế đồng thuận (như ZooKeeper) để tránh xung đột khi ghi.
3. **Hiệu suất**: Tối ưu hóa ghi bằng cách sử dụng bộ nhớ đệm (như Redis) kết hợp với sao lưu định kỳ.
4. **Khôi phục lỗi**: Coordinator cần kiểm tra nhật ký khi khởi động lại để xác định trạng thái giao dịch và thực hiện các hành động cần thiết (commit/rollback).
5. **Bảo mật**: Mã hóa dữ liệu trạng thái nếu lưu trên hệ thống phân tán (như Kafka hoặc ZooKeeper).
