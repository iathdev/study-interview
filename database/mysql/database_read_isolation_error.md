# Isolation

| Tên lỗi             | Nguyên nhân                                             | Cách tránh                                |
| ------------------- | ------------------------------------------------------- | ----------------------------------------- |
| Dirty Read          | Đọc dữ liệu từ giao dịch chưa `COMMIT`                  | Dùng `READ COMMITTED` trở lên             |
| Non-repeatable Read | Đọc cùng bản ghi nhiều lần nhưng thấy kết quả khác nhau | Dùng `REPEATABLE READ` trở lên            |
| Phantom Read        | Kết quả truy vấn thay đổi do bản ghi mới được thêm/xóa  | Dùng `SERIALIZABLE` hoặc gap lock (MySQL) |


## 1. Dirty Read (Đọc bẩn)

### Định nghĩa:
Dirty Read xảy ra khi **một giao dịch đọc dữ liệu đã được ghi bởi một giao dịch khác nhưng chưa `COMMIT`**. Nếu giao dịch kia bị `ROLLBACK`, dữ liệu đã đọc là **sai** và **không tồn tại** trong thực tế.

### Ví dụ:

1. Giao dịch A:
   ```sql
   UPDATE accounts SET balance = balance - 500 WHERE id = 1;
   -- chưa COMMIT
```

2.Giao dịch B:
   ```sql
   SELECT balance FROM accounts WHERE id = 1;
-- thấy balance đã trừ 500
```

3. Sau đó giao dịch A bị ROLLBACK.

⇒ Giao dịch B đã đọc một giá trị không hợp lệ, vì thay đổi đó không được xác nhận.

Cách tránh:
Dùng isolation level **READ COMMITTED** trở lên.

## 2. Non-repeatable Read (Đọc không lặp lại)

### Định nghĩa:
Non-repeatable Read xảy ra khi **một giao dịch đọc cùng một bản ghi nhiều lần** và nhận được **kết quả khác nhau** vì một giao dịch khác đã `UPDATE` hoặc `DELETE` bản ghi đó **trong khi giao dịch đầu đang thực thi**.

### Ví dụ:

1. Giao dịch A:
```sql
   SELECT balance FROM accounts WHERE id = 1;
   -- balance = 1000
```

2. Giao dịch B:
```sql
UPDATE accounts SET balance = 500 WHERE id = 1;
COMMIT;
```

3. Giao dịch A (tiếp tục):
```sql
SELECT balance FROM accounts WHERE id = 1;
-- balance = 500
```


⇒ Cùng một câu truy vấn, nhưng Giao dịch A thấy hai kết quả khác nhau.

Cách tránh:
Dùng isolation level **REPEATABLE READ** hoặc **SERIALIZABLE**.

## 3. Phantom Read (Đọc bóng ma)

### Định nghĩa:
Phantom Read xảy ra khi **cùng một truy vấn với điều kiện nhất định được thực hiện nhiều lần** trong một giao dịch, nhưng mỗi lần truy vấn lại trả về **tập kết quả khác nhau** do **giao dịch khác đã thêm hoặc xóa bản ghi mới** thỏa điều kiện đó.

### Ví dụ:

1. Giao dịch A:
   ```sql
   SELECT * FROM orders WHERE total > 1000;
   -- Trả về 2 đơn hàng
```

2. Giao dịch B:
```sql
INSERT INTO orders (total) VALUES (1500);
COMMIT;
```

3. Giao dịch A (tiếp tục):
```sql
SELECT * FROM orders WHERE total > 1000;
-- Trả về 3 đơn hàng
```

⇒ Một "bản ghi bóng ma" xuất hiện trong kết quả – do nó mới được thêm vào sau lần truy vấn đầu.

Cách tránh:
Dùng isolation level **SERIALIZABLE** để khóa phạm vi truy vấn.
Trong MySQL (InnoDB), có thể dùng **REPEATABLE READ** kết hợp với **gap locking** để ngăn phantom read.




