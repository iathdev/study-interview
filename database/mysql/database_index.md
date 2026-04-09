# MySQL Index Summary

## Index Overview

* Index là cấu trúc dữ liệu lưu trên đĩa (disk)
* Dùng để tăng tốc truy vấn SELECT bằng cách tìm kiếm nhanh hơn dữ liệu

---

## Phân Loại Index

### 1. Theo cấu trúc dữ liệu:

* B-tree
* B+ tree
* Hash
* R-tree (cho Spatial Index)
* Fulltext (Inverted Index)

### 2. Theo cách lưu trữ vật lý:

* Clustered Index
* Non-clustered (Secondary) Index

### 3. Theo số cột:

* Single-column Index
* Composite Index (cover index)

### 4. Theo tính chất:

* Primary Key Index
* Unique Index
* Prefix Index

### So sánh các loại index

| Phân loại           | Mô tả                               | Ghi chú                           |
| ------------------- | ----------------------------------- | --------------------------------- |
| B-tree/B+ tree      | Cấu trúc dạng cây nhị phân tìm kiếm, cây tự cân bằng | Dùng phổ biến nhất                |
| Hash                | Sử dụng hàm băm để tra cứu nhanh    | Chỉ dùng cho `=`                  |
| Clustered Index     | Chỉ có 1, trùng với primary key     | Xác định vị trí dữ liệu trên disk |
| Non-clustered Index | Thứ cấp, có thể nhiều cái           | Trỏ về primary key                |
| Single-column Index | Index trên 1 cột                    | Dễ quản lý                        |
| Composite Index     | Index trên nhiều cột                | Hiệu quả nếu dùng đúng thứ tự     |
| Fulltext            | Tìm kiếm văn bản                    | Dạng Inverted Index               |
| Spatial Index       | Index không gian địa lý             | PostGIS/PostgreSQL thường dùng    |

---

## Data Structure

### B-Tree Index

* Độ phức tạp: `O(log N)`
* Hỗ trợ: `=`, `!=`, `>`, `<`, `LIKE`, `RANGE`
* Dùng `LIKE` chỉ tác dụng khi `%` ở cuối: VD: `'name%'`

### B+ Tree

* Tối ưu cho truy vấn dạng RANGE
* Node lá được link với nhau => truy vấn nhanh

### Hash Index

* Tánh hash key => hash table
* Chỉ hỗ trợ: so sánh `=`
* Nhược điểm: không hợp với RANGE

---

## Physical Storage

### Clustered Index

* Là primary index quyết định vị trí vật lý dữ liệu trên disk
* Mỗi bảng chỉ có 1 clustered index
* Primary Key sẽ sinh clustered index ngầm

### Non-clustered Index (Secondary)

* Mỗi bảng có thể có nhiều secondary index
* Node lá của B-tree secondary index lưu GIÁ TRỊ PRIMARY KEY (không phải địa chỉ disk)

### Tại sao không lưu địa chỉ?

* Địa chỉ disk có thể thay đổi khi rebuild index, tái phân vùng

---

## Characteristics

### Khác nhau giữa Key vs Index

* **Key**: Ràng buộc (PK, FK)
* **Index**: Tối ưu tìm kiếm dữ liệu

### Primary Index

* Luôn unique, không NULL
* Tăng dần sẽ hiệu quả hơn với B+ Tree

### Unique Index

* Unique, nhưng chấp nhận NULL

---

## Number of Columns

### Composite Index

* Index nhiều cột
* Query PHẢI chứa cột đầu tiên
* Hiệu quả cao khi query có điều kiện `=` với cột đầu

### Skip Scan Index > mysql 8
- Cơ chế bỏ qua giá trị đầu tiên trong composite index khi không có điều kiện WHERE với cột đó
- Cho phép sử dụng phần còn lại của index mà không cần phải có điều kiện đầy đủ từ cột đầu tiên
- Hiệu quả với dữ liệu có tính phân tán đều ở các cột trong index

### Thứ tự trong index:

* Dựa theo **Cardinality** (số lượng giá trị khác nhau)
* Cột có cardinality cao đặt trước

---

## Other Index Types

* Fulltext (Inverted Index): search text
* Spatial Index: GIST (Postgres), R-tree
* Bitmap Index: Thường dùng trong DWH


---

## Practices

### Nên dùng Index khi:

* Query có `WHERE`, `JOIN`, `GROUP BY`
* Truy vấn trả về tập dữ liệu nhỏ
* Table có nhiều dữ liệu
* Trường unique hoặc truy vấn nhiều

### KHÔNG nên dùng khi:

* Bảng nhỏ, dữ liệu ít
* Cột update thường xuyên
* Trường low-cardinality (ví dụ: tuổi, boolean)
* Trường có nhiều NULL

---

## Best Practices

* Giới hạn số index trong 1 bảng
* Tránh index trên cột NULL nhiều
* Index trên primary key nên tăng dần
* Dùng Covering Index: giảm I/O disk
* Prefix Index: dùng cho TEXT dài
* Monitor & optimize thường xuyên

---

## EXPLAIN - Các column quan trọng

Khi dùng `EXPLAIN`, bạn cần chú ý đến các trường sau để phân tích hiệu suất truy vấn:

| Cột             | Ý nghĩa                                                             |
| --------------- | ------------------------------------------------------------------- |
| `id`            | Thứ tự thực thi của phần truy vấn (subquery)                        |
| `select_type`   | Loại truy vấn: SIMPLE, PRIMARY, SUBQUERY, DERIVED, UNION            |
| `table`         | Tên bảng hoặc alias được truy vấn                                   |
| `type`          | Kiểu truy cập dữ liệu: const, ref, range, index, ALL                |
| `possible_keys` | Những index có thể dùng                                             |
| `key`           | Index thực tế được sử dụng                                          |
| `key_len`       | Độ dài key được dùng                                                |
| `ref`           | Cột hoặc hằng số dùng để so sánh với index                          |
| `rows`          | Số dòng ước lượng cần duyệt                                         |
| `filtered`      | Tỷ lệ phần trăm dòng sau khi áp dụng điều kiện WHERE                |
| `Extra`         | Thông tin thêm: Using where, Using index, Using filesort, temporary |


## Dùng Covering Index giúp giảm I/O disk là bởi vì:

### Bản chất I/O disk là gì?
- Khi MySQL cần lấy dữ liệu từ bảng, nó đọc từ đĩa (disk) nếu dữ liệu chưa có sẵn trong RAM (buffer pool).
- Mỗi lần truy cập bảng gốc để lấy dữ liệu => một lần I/O disk.

### Covering Index giải quyết vấn đề như thế nào?
- Một Covering Index là một index mà chứa tất cả các cột được dùng trong câu SELECT, nghĩa là:
- MySQL không cần truy cập clustered index nữa.
- Nó lấy dữ liệu trực tiếp từ index.

| Truy vấn bình thường        | Truy vấn với Covering Index |
| --------------------------- | --------------------------- |
| 1. Dò index để lọc          | 1. Dò index để lọc          |
| 2. Truy cập bảng gốc (disk) | 2. Không cần truy cập clustered index  |
| 3. Lấy dữ liệu từ bảng      | 3. Lấy dữ liệu từ index     |


