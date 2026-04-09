# Partition

🔹 1. Range Partitioning (Phân vùng theo khoảng)
Dữ liệu được chia theo khoảng giá trị của một cột (thường là số hoặc ngày tháng).

Ví dụ: Phân chia khách hàng theo vùng (region) dựa trên ID vùng:

```sql
Sao chép
Chỉnh sửa
PARTITION BY RANGE (region_id) (
    PARTITION p_north VALUES LESS THAN (100),
    PARTITION p_central VALUES LESS THAN (200),
    PARTITION p_south VALUES LESS THAN (MAXVALUE)
)
```

🔹 2. List Partitioning (Phân vùng theo danh sách)
Chia dữ liệu theo danh sách giá trị cụ thể.

Ví dụ: Phân chia theo tên vùng:

```sql
Sao chép
Chỉnh sửa
PARTITION BY LIST (region_name) (
    PARTITION p_north VALUES IN ('North', 'NorthEast'),
    PARTITION p_central VALUES IN ('Central'),
    PARTITION p_south VALUES IN ('South', 'SouthWest')
)
```

🔹 3. Column Partitioning (Phân vùng theo cột - Vertical Partitioning)
Dữ liệu được chia theo chiều dọc: chia thành nhiều bảng chứa các cột khác nhau. Không phải là partition trong nghĩa cơ sở dữ liệu vật lý, nhưng thường dùng trong thiết kế CSDL.

Ví dụ:

Customer_Info(id, name, region)

Customer_Orders(id, order_date, total)

🔹 4. Hash Partitioning (Phân vùng băm)
Dữ liệu được chia dựa trên hàm băm của một cột (phân bố đều).

Ví dụ:

```sql
Sao chép
Chỉnh sửa
PARTITION BY HASH(region_id) PARTITIONS 4;
```
Các giá trị region_id sẽ được chia vào 4 partition một cách ngẫu nhiên nhưng đều.

🔹 5. Key Partitioning
Tương tự Hash, nhưng dùng khóa chính hoặc khóa định nghĩa bởi hệ thống, đôi khi kết hợp nhiều cột.

Ví dụ (MySQL InnoDB):

```sql
Sao chép
Chỉnh sửa
PARTITION BY KEY(region_id) PARTITIONS 3;
```

📌 Gợi ý phân vùng theo region
Nếu vùng miền được xác định rõ (VD: Bắc, Trung, Nam), nên dùng List Partitioning vì rõ ràng và dễ kiểm soát.

Nếu vùng được mã hóa bằng số theo khoảng, có thể dùng Range Partitioning.

