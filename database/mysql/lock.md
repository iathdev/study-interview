# Các loại Lock trong MySQL

Trong MySQL, có nhiều loại lock (khóa) khác nhau được sử dụng để đảm bảo tính nhất quán và toàn vẹn dữ liệu khi nhiều truy vấn được thực hiện đồng thời. Tùy vào storage engine (ví dụ: InnoDB, MyISAM) mà MySQL sử dụng loại khóa khác nhau.

## 1. Table Lock (Khóa bảng)

- **Áp dụng cho**: MyISAM, MEMORY, MERGE (và cũng có thể được dùng tạm thời trong InnoDB)
- **Loại khóa**:
  - `READ LOCK`: các truy vấn đọc có thể thực thi đồng thời, ghi bị chặn.
  - `WRITE LOCK`: chỉ một truy vấn ghi được thực thi, chặn cả đọc lẫn ghi khác.
- **Dùng trong trường hợp**:
  - Khi bạn muốn đảm bảo toàn bộ bảng không bị thay đổi trong lúc đang đọc hoặc ghi.
  - Dùng nhiều trong MyISAM vì MyISAM không hỗ trợ row-level locking.
- **Ví dụ**:
```sql
  LOCK TABLES employees READ;
  -- thực hiện truy vấn SELECT
  UNLOCK TABLES;
```

## 2. Row-level Lock (Khóa dòng)

- **Áp dụng cho**: InnoDB
- **Loại khóa**:
  - Shared Lock (`S`): nhiều transaction có thể đọc cùng lúc.
    - Dùng khi: Đọc dữ liệu (SELECT)
    - Cho phép: Nhiều transaction được cấp shared lock cùng lúc → nhiều transaction có thể đọc cùng lúc.
    - Không cho phép: Không có transaction nào được ghi (write) vào dòng dữ liệu đó cho đến khi các shared lock được giải phóng.
  - Exclusive Lock (`X`): chỉ một transaction có thể ghi.
    - Dùng khi: Ghi dữ liệu (INSERT, UPDATE, DELETE)
    - Cho phép: Chỉ 1 transaction được cấp exclusive lock trên dòng đó.
    - Không cho phép: Không transaction nào khác có thể đọc hoặc ghi dòng dữ liệu đó cho đến khi lock được giải phóng.
- **Dùng trong**:
  - Hệ thống yêu cầu tính đồng thời cao.
  - Giao dịch (transaction) phức tạp có nhiều người dùng cùng thao tác trên cùng bảng nhưng khác dòng.
  - Đảm bảo chỉ lock chính xác dòng cần thiết, tránh ảnh hưởng các dòng khác như table lock.
- **Đặc điểm**:
  - Chỉ được kích hoạt khi sử dụng trong transaction (`BEGIN`, `START TRANSACTION`).
  - Hiệu quả hơn table lock trong các hệ thống OLTP (Online Transaction Processing).
  - Có thể gây deadlock nếu không kiểm soát tốt thứ tự lock.
- **Tương tác với các isolation level**:
  - Dưới `READ COMMITTED`: chỉ lock dòng khi ghi (update/delete).
  - Dưới `REPEATABLE READ`: có thể kết hợp thêm gap lock hoặc next-key lock.
- **Tối ưu hóa**:
  - Nên có chỉ mục phù hợp để tránh lock toàn bảng (trường hợp thiếu index, InnoDB có thể lock nhiều dòng hơn dự kiến).

### Shared Lock (S) – Khóa chia sẻ
Mục đích:
- Cho phép nhiều transaction cùng đọc dữ liệu nhưng không ai được ghi (UPDATE/DELETE) dữ liệu đang bị khóa.

Cơ chế hoạt động:
- Một transaction đặt Shared Lock (S) lên một dòng dữ liệu.
- Các transaction khác vẫn được phép đặt Shared Lock lên cùng dòng đó.
- Nhưng không transaction nào được đặt Exclusive Lock (tức là không thể UPDATE/DELETE dòng đó cho đến khi các shared lock được giải phóng).

Dùng khi nào:
- Khi cần đọc dữ liệu có tính ổn định trong một transaction.
- Khi muốn đảm bảo không có transaction khác thay đổi dữ liệu trong lúc đang kiểm tra để đưa ra quyết định.

```sql
SELECT ... LOCK IN SHARE MODE;
-- Dữ liệu này không thể bị UPDATE/DELETE bởi transaction khác cho đến khi commit/rollback
```

#### Ngữ cảnh: Kiểm tra tồn kho sản phẩm trước khi cho đặt hàng
Yêu cầu:
- Trước khi cho phép khách đặt hàng, hệ thống phải kiểm tra còn đủ hàng không. Nhưng trong lúc kiểm tra, không ai được phép cập nhật số lượng hàng (tránh trường hợp over-selling).

```sql
START TRANSACTION;
SELECT stock FROM products WHERE id = 1 LOCK IN SHARE MODE;
-- kiểm tra stock >= 1 thì cho phép tiếp tục
-- không ai được UPDATE dòng này trong lúc này
-- sau đó COMMIT hoặc ROLLBACK
```
Lợi ích:
- Đảm bảo dữ liệu ổn định (consistent) trong suốt thời gian kiểm tra.
- Không bị người khác UPDATE trong lúc mình đang xem stock.

### Exclusive Lock (X) – Khóa độc quyền
Mục đích:
- Chỉ cho phép một transaction ghi (UPDATE/DELETE) vào dữ liệu được khóa, và không ai được đọc kiểu có lock hoặc ghi vào nó.

Cơ chế hoạt động:
- Một transaction đặt Exclusive Lock (X) lên dòng dữ liệu.
- Không transaction nào khác được:
  - Đặt Shared Lock trên dòng đó
  - Đặt thêm Exclusive Lock
- Tức là dòng đó bị “độc chiếm” cho đến khi transaction kia commit hoặc rollback.

Dùng khi nào:
- Khi transaction cần thay đổi dữ liệu, và phải đảm bảo không ai khác can thiệp vào dữ liệu đó trong suốt thời gian thực hiện transaction.
- Khi cần kiểm tra rồi mới quyết định ghi

```sql
SELECT ... FOR UPDATE;
UPDATE ...
DELETE ...
-- Những dòng này bị lock hoàn toàn (cả đọc/ghi có lock)
```

#### Ngữ cảnh: Chuyển tiền giữa 2 tài khoản
Yêu cầu:
- Chuyển 1.000đ từ tài khoản Alice sang Bob phải đảm bảo:
- Không bị người khác can thiệp cùng lúc (vd: cũng đang chuyển tiền cho Alice)
- Toàn bộ thao tác là atomic

```sql
START TRANSACTION;

-- Khóa dòng tài khoản Alice để trừ tiền
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- Khóa dòng tài khoản Bob để cộng tiền
SELECT * FROM accounts WHERE id = 2 FOR UPDATE;

-- Thực hiện chuyển tiền
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;

COMMIT;
```

Lợi ích:
- Không có transaction nào khác được đọc/ghi dòng của Alice hoặc Bob trong suốt quá trình.
- Tránh race condition (tranh chấp dữ liệu).

| Đặc điểm                | Shared Lock (S)                   | Exclusive Lock (X)                 |
| ----------------------- | --------------------------------- | ---------------------------------- |
| Cho phép nhiều truy cập | Có (nhiều transaction có thể đọc) | Không (chỉ 1 transaction được ghi) |
| Cho phép ghi            | Không                             | Có                                 |
| Cho phép đọc            | Có (nhiều transaction)            | Không (chặn cả đọc nếu cần lock)   |
| Gây xung đột với        | Exclusive Lock                    | Shared Lock và Exclusive Lock      |
| Dùng cho                | `LOCK IN SHARE MODE`              | `FOR UPDATE`, `UPDATE`, `DELETE`   |


## 3. Intention Lock

- **Áp dụng cho**: InnoDB
- **Loại**:
  - Intention Shared (`IS`)
  - Intention Exclusive (`IX`)
- **Mục đích**:
  - Thông báo cho hệ thống rằng một transaction có ý định đặt Shared/Exclusive lock trên một hoặc nhiều dòng của bảng.
  - Cho phép InnoDB kiểm tra xung đột lock ở cấp bảng một cách nhanh chóng.
- **Ví dụ tình huống**:
  - Khi một transaction muốn đặt row-level `S` lock, InnoDB sẽ tự động đặt `IS` lock ở cấp bảng.
  - Khi dùng `SELECT ... FOR UPDATE`, InnoDB sẽ đặt `IX` lock ở bảng, `X` lock ở dòng.
- **Không cần lập trình viên đặt bằng tay**, được MySQL quản lý tự động.
- **Giúp giảm deadlock** khi có nhiều transaction đồng thời truy cập cùng bảng.

## 4. Gap Lock

- **Áp dụng cho**: InnoDB, ở mức `REPEATABLE READ`
- **Khóa khoảng** giữa các giá trị, không khóa bản ghi hiện tại.
- **Mục đích**:
  - Ngăn chặn việc insert vào khoảng giữa các dòng đã khóa.
  - Ngăn phantom row xuất hiện khi thực hiện lại cùng truy vấn trong transaction.
- **Tình huống sử dụng**:
  - Khi thực hiện `SELECT ... FOR UPDATE` trên một điều kiện có phạm vi (range).
- **Đặc điểm**:
  - Không ngăn update/delete bản ghi đã tồn tại.
  - Có thể gây lock nhiều dòng hơn cần thiết nếu truy vấn không dùng index.

## 5. Next-Key Lock

- **Áp dụng cho**: InnoDB, ở mức `REPEATABLE READ`
- **Là sự kết hợp giữa** row-level lock và gap lock.
- **Mục đích**:
  - Tránh phantom read bằng cách khóa cả dòng hiện tại và khoảng giữa dòng hiện tại với dòng kế tiếp.
- **Ví dụ**:
  - Truy vấn `SELECT * FROM products WHERE price > 100 FOR UPDATE;`
    => sẽ khóa bản ghi có `price > 100` và ngăn insert mới trong khoảng đó.
- **Chỉ áp dụng nếu dùng index phù hợp**.
- Nếu không có index, có thể gây lock toàn bộ bảng (table-level).

## 6. Metadata Lock (MDL)

- **Áp dụng cho**: Tất cả các engine (InnoDB, MyISAM,...)
- **Tự động áp dụng** khi bất kỳ truy vấn nào truy cập đến cấu trúc bảng (SELECT, INSERT, UPDATE, v.v.)
- **Dùng để**:
  - Ngăn việc thay đổi cấu trúc bảng trong khi có truy vấn đang sử dụng bảng đó.
  - Ví dụ: không thể `ALTER TABLE` nếu có truy vấn đang `SELECT` chưa kết thúc.
- **Đặc điểm**:
  - Không hiển thị qua `SHOW ENGINE INNODB STATUS`.
  - Dễ gây treo session nếu một truy vấn `ALTER` chờ truy vấn khác `SELECT` quá lâu.
- **Giải pháp**:
  - Hạn chế long-running queries.
  - Kiểm tra session treo bằng `SHOW PROCESSLIST`.

## 7. Auto-Inc Lock (Khóa tự tăng)

- **Áp dụng cho**:
  - InnoDB: dùng một mutex nội bộ, kiểm soát linh hoạt hơn.
  - MyISAM: dùng table-level lock.
- **Dùng để**:
  - Đảm bảo giá trị sinh ra từ AUTO_INCREMENT là duy nhất và chính xác.
- **Hoạt động**:
  - Trong InnoDB, chế độ mặc định là `auto-inc lock mode = 1 (consecutive)`:
    - Truy vấn `INSERT` giữ một lock nhẹ cho đến khi commit.
  - Có thể thay đổi qua cấu hình `innodb_autoinc_lock_mode`:
    - `0`: traditional (table-level lock)
    - `1`: consecutive (mặc định)
    - `2`: interleaved (tăng concurrency cao, ít lock)

## Tổng kết so sánh các loại lock

| Lock Type       | Storage Engine | Cấp độ        | Mục đích sử dụng                             |
|-----------------|----------------|----------------|----------------------------------------------|
| Table Lock      | MyISAM         | Bảng           | Đọc/ghi toàn bảng                            |
| Row Lock        | InnoDB         | Dòng           | Giao dịch chỉ lock dòng cần thiết            |
| Intention Lock  | InnoDB         | Bảng (logic)   | Phối hợp với row lock để tránh xung đột      |
| Gap Lock        | InnoDB         | Khoảng         | Ngăn insert giá trị vào vùng đang truy vấn   |
| Next-Key Lock   | InnoDB         | Dòng + Khoảng  | Tránh phantom read trong isolation cao       |
| Metadata Lock   | Tất cả         | Metadata       | Ngăn thay đổi cấu trúc bảng khi đang dùng    |
| Auto-inc Lock   | InnoDB/MyISAM  | Tự tăng        | Quản lý giá trị AUTO_INCREMENT an toàn       |


