ACID:
+ Atomicity: nguyên tử: 1 giao dịch có nhiều action thì tất cả action phải được xử lý. Hoặc ko action nào được xử lý
+ Consistency: nhất quán: không hoàn thành giao dịch một cách dở dang. Tạo ra trạng thái mới, retry và phải rollback về
trạng thái ban đầu
+ Isolation: cách ly: hai hay nhiều giao dịch không trộn lẫn nhau, ko được tạo ra race condition khi nhiều transaction xảy
ra
+ Durability: bền vững: trong trường hợp thất bại hoặc ứng dụng khởi động lại thì có thể phục hồi sự kết thúc bất thường

| Chữ cái | Viết tắt    | Ý nghĩa                                                                           |
| ------- | ----------- | --------------------------------------------------------------------------------- |
| A       | Atomicity   | Tính nguyên tử – giao dịch phải thực hiện **toàn bộ hoặc không gì cả**            |
| C       | Consistency | Tính nhất quán – dữ liệu luôn ở trạng thái **hợp lệ** trước và sau giao dịch, có lỗi phải rollback |
| I       | Isolation   | Tính độc lập – các giao dịch không **ảnh hưởng lẫn nhau** khi thực hiện đồng thời |
| D       | Durability  | Tính bền vững – khi giao dịch thành công, **dữ liệu sẽ được lưu vĩnh viễn**       |


## Giải thích chi tiết từng yếu tố

### 1. Atomicity (Tính nguyên tử)

- Một giao dịch gồm nhiều thao tác, nhưng phải được **thực hiện trọn vẹn** hoặc **không thực hiện gì cả**.
- Nếu có bất kỳ lỗi nào xảy ra trong quá trình, hệ thống sẽ **rollback toàn bộ giao dịch**.

**Ví dụ**:
```sql
-- Chuyển tiền từ A → B
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
COMMIT;
```

Nếu bước thứ hai (cộng tiền vào tài khoản B) thất bại, thì bước đầu tiên (trừ tiền từ A) cũng sẽ bị rollback → không mất tiền.

### 2. Consistency (Tính nhất quán)

- Giao dịch phải đảm bảo dữ liệu **luôn tuân thủ các quy tắc ràng buộc** (constraints, khóa chính, khóa ngoại, domain value...).
- Trước và sau mỗi giao dịch, dữ liệu phải ở trạng thái **hợp lệ**.
- Nếu bất kỳ bước nào trong giao dịch làm vi phạm tính nhất quán, toàn bộ giao dịch sẽ bị hủy (rollback).

**Ví dụ**:

- Giả sử hệ thống không cho phép số dư tài khoản < 0.
- Giao dịch sau đây sẽ bị từ chối nếu số dư không đủ:

```sql
UPDATE accounts SET balance = balance - 1_000_000 WHERE id = 'A';
```

Nếu kết quả làm cho balance < 0, giao dịch sẽ bị rollback để đảm bảo tính nhất quán.


### 3. Isolation (Tính cô lập)

- Mỗi giao dịch nên được **thực hiện độc lập** với các giao dịch khác đang chạy song song.
- Isolation đảm bảo dữ liệu không bị ảnh hưởng nếu có nhiều người dùng thao tác cùng lúc.
- Tùy vào cấp độ cô lập (Isolation Level), hệ thống có thể phòng tránh các lỗi sau:
  - **Dirty Read**: đọc dữ liệu mà giao dịch khác chưa commit.
  - **Non-repeatable Read**: cùng một truy vấn cho kết quả khác nhau trong cùng một giao dịch.
  - **Phantom Read**: một truy vấn trả về số dòng khác nhau do giao dịch khác đã thêm hoặc xóa dữ liệu.

**Ví dụ**:

Giả sử Giao dịch A đang cập nhật số dư:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
-- chưa COMMIT
```

Nếu Giao dịch B cùng lúc chạy:

```sql
SELECT balance FROM accounts WHERE id = 'A';
```

Nếu cấp độ cô lập là READ UNCOMMITTED, giao dịch B có thể đọc được số dư đã bị trừ dù giao dịch A chưa COMMIT. Đây là lỗi Dirty Read.
Nếu dùng READ COMMITTED hoặc cao hơn, B sẽ không thấy thay đổi, giúp tránh được lỗi.

Các mức Isolation phổ biến và lỗi tránh được

| Isolation Level  | Tránh Dirty Read | Tránh Non-repeatable Read | Tránh Phantom Read                 |
| ---------------- | ---------------- | ------------------------- | ---------------------------------- |
| READ UNCOMMITTED | ✖                | ✖                         | ✖                                  |
| READ COMMITTED   | ✔                | ✖                         | ✖                                  |
| REPEATABLE READ  | ✔                | ✔                         | ✖ (MySQL tránh được bằng gap lock) |
| SERIALIZABLE     | ✔                | ✔                         | ✔                                  |

### 4. Durability (Tính bền vững)

- Durability đảm bảo rằng **khi một giao dịch đã `COMMIT` thành công**, mọi thay đổi của nó sẽ được **lưu vĩnh viễn** vào bộ nhớ ổn định (disk).
- Dữ liệu sẽ **không bị mất** ngay cả khi hệ thống gặp sự cố như mất điện, crash server, hoặc shutdown đột ngột.

---

**Cơ chế thực hiện trong DBMS**:

- Hệ quản trị CSDL thường sử dụng:
  - **Write-Ahead Logging (WAL)**: ghi log trước rồi mới ghi dữ liệu.
  - **Redo/Undo logs**: để khôi phục hoặc phục hồi sau khi hệ thống gặp lỗi.
  - **Checkpointing**: định kỳ ghi dữ liệu đang giữ tạm ra đĩa.

---

**Ví dụ thực tế**:

1. Giao dịch cập nhật số dư tài khoản và `COMMIT` thành công.
2. Ngay sau đó, hệ thống **bị mất điện** hoặc **crash**.
3. Khi khởi động lại, hệ thống kiểm tra log và khôi phục lại trạng thái đã `COMMIT` trước đó → đảm bảo không mất dữ liệu.

---

**So sánh với các hệ thống thiếu Durability**:

| Tình huống                                 | Có Durability     | Không có Durability       |
|--------------------------------------------|--------------------|----------------------------|
| Sau khi `COMMIT`, hệ thống sập đột ngột   | Dữ liệu được lưu   | Dữ liệu có thể bị mất      |
| Mất điện sau khi chuyển khoản thành công   | Số dư vẫn đúng     | Có thể bị mất hoặc sai lệch|

> Durability là yếu tố then chốt giúp hệ thống **đáng tin cậy**, đặc biệt với các ứng dụng tài chính, ngân hàng, thương mại điện tử.

---

**Ghi chú**:
- Trong MySQL (InnoDB), durability phụ thuộc vào cấu hình:
  - `innodb_flush_log_at_trx_commit = 1`: an toàn nhất, ghi log ngay khi `COMMIT`.
  - `innodb_flush_log_at_trx_commit = 2` hoặc `0`: hiệu năng cao hơn nhưng có thể mất dữ liệu nếu crash.



