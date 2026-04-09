# Connection Pool là gì?

## 1. Định nghĩa
**Connection Pool** (bể kết nối) là kỹ thuật tái sử dụng các kết nối đến hệ thống như:
- Cơ sở dữ liệu (MySQL, PostgreSQL...)
- Redis, Kafka, RabbitMQ...
- Các dịch vụ API khác

Mục tiêu: Giảm chi phí mở/đóng kết nối, tăng hiệu suất và tiết kiệm tài nguyên.

---

## 2. Vì sao cần Connection Pool?

### Không dùng Pool:
- Mỗi request phải mở và đóng kết nối mới
- Gây tốn thời gian và tài nguyên

### Dùng Pool:
- Mở sẵn N kết nối
- Khi cần: lấy 1 kết nối có sẵn → dùng xong → trả về pool
- Không tạo/đóng lại kết nối mỗi lần

---

## 3. Cách hoạt động

```
+---------+ +----------------+
| Client |---------> | Connection Pool|
+---------+ +----------------+
|
| lấy kết nối sẵn có
v
+----------+
| Database |
+----------+
```

### Lợi ích

| Lợi ích             | Mô tả                                  |
| ------------------- | -------------------------------------- |
| Nhanh hơn           | Không mất thời gian tạo kết nối mới    |
| Tái sử dụng         | Kết nối được reuse hiệu quả            |
| Giảm lỗi            | Hạn chế kết nối vượt quá giới hạn      |
| Tăng khả năng scale | Hệ thống chịu tải cao hơn, ổn định hơn |


### Rủi ro nếu không quản lý tốt

| Vấn đề          | Nguyên nhân                              |
| --------------- | ---------------------------------------- |
| Exhausted Pool  | Không đủ kết nối phục vụ, thread bị treo |
| Connection Leak | Không trả kết nối về pool sau khi dùng   |
| Overload DB     | Tạo quá nhiều kết nối đồng thời          |
