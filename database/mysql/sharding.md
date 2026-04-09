Sharding là kỹ thuật phân vùng cơ sở dữ liệu theo chiều ngang (horizontal partitioning) để phân phối dữ liệu qua nhiều máy chủ (shards), nhằm cải thiện hiệu suất và khả năng mở rộng cho hệ thống CRM xử lý hàng triệu khách hàng. Dưới đây là cách thiết kế và triển khai sharding cho cơ sở dữ liệu:

---

### **1. Sharding là gì?**
- Sharding chia dữ liệu thành các phần nhỏ (shards) dựa trên một khóa phân vùng (shard key). Mỗi shard chứa một tập hợp con dữ liệu và được lưu trữ trên một máy chủ riêng biệt.
- Mục tiêu:
  - Giảm tải cho từng máy chủ.
  - Tăng tốc độ truy vấn.
  - Hỗ trợ mở rộng quy mô lớn (hàng triệu khách hàng).

---

### **2. Thiết kế Sharding cho CRM**
Để sharding cơ sở dữ liệu trong hệ thống CRM, cần xác định các yếu tố sau:

#### **A. Chọn Shard Key**
Shard key là trường dữ liệu dùng để phân phối bản ghi qua các shards. Lựa chọn shard key rất quan trọng, vì nó ảnh hưởng đến hiệu suất và khả năng cân bằng tải.
- **Yêu cầu shard key**:
  - Phân phối dữ liệu đồng đều (tránh hotspot, tức là một shard nhận quá nhiều truy vấn).
  - Hỗ trợ các truy vấn phổ biến (giảm truy vấn cross-shard).
- **Ví dụ shard key cho CRM**:
  - **Customer ID**: Nếu mỗi khách hàng có ID duy nhất, đây là lựa chọn tốt vì phân phối đều và truy vấn thông tin khách hàng thường dựa trên ID.
  - **Khu vực địa lý (Region)**: Phù hợp nếu hệ thống phục vụ khách hàng toàn cầu, vì các truy vấn thường tập trung vào một khu vực.
  - **Organization ID**: Nếu CRM phục vụ nhiều tổ chức/doanh nghiệp (B2B), sharding theo tổ chức giúp cô lập dữ liệu.
- **Lưu ý**:
  - Tránh shard key có tính phân bố thấp (ví dụ: giới tính, vì chỉ có vài giá trị, dẫn đến phân phối không đều).
  - Kết hợp nhiều trường (compound shard key) nếu cần, ví dụ: `Region + Customer ID`.

#### **B. Loại Sharding**
- **Range-based Sharding**:
  - Dữ liệu được chia theo khoảng giá trị của shard key (ví dụ: Customer ID từ 1-1M ở shard 1, 1M-2M ở shard 2).
  - Ưu điểm: Dễ quản lý các truy vấn theo khoảng (range queries).
  - Nhược điểm: Có thể gây hotspot nếu dữ liệu không đồng đều.
- **Hash-based Sharding**:
  - Áp dụng hàm băm (hash function) lên shard key để phân phối dữ liệu (ví dụ: hash(Customer ID)).
  - Ưu điểm: Phân phối đồng đều hơn.
  - Nhược điểm: Khó thực hiện các truy vấn theo khoảng.
- **Geo-based Sharding**:
  - Phân phối dữ liệu theo vị trí địa lý (ví dụ: shard riêng cho Mỹ, châu Âu, châu Á).
  - Ưu điểm: Tối ưu cho truy vấn theo khu vực, giảm độ trễ.
  - Nhược điểm: Cần quản lý phức tạp nếu khách hàng di chuyển giữa các khu vực.

#### **C. Cơ sở dữ liệu hỗ trợ Sharding**
- **RDBMS**:
  - **PostgreSQL**: Hỗ trợ sharding thông qua extension như Citus hoặc tự triển khai bằng phân vùng (partitioning).
  - **MySQL**: Sử dụng Vitess để quản lý sharding.
- **NoSQL**:
  - **MongoDB**: Hỗ trợ sharding tích hợp với range hoặc hash-based sharding.
  - **Cassandra**: Sharding tự động dựa trên consistent hashing.
  - **CockroachDB**: Phân phối dữ liệu toàn cầu với sharding tự động.
- **Tìm kiếm**: Elasticsearch hỗ trợ sharding cho tìm kiếm toàn văn (text search).

---

### **3. Triển khai Sharding**
Dưới đây là các bước triển khai sharding cho CRM:

#### **Bước 1: Phân tích truy vấn**
- Xác định các truy vấn phổ biến trong CRM:
  - Lấy thông tin khách hàng theo ID.
  - Tìm kiếm khách hàng theo khu vực hoặc trạng thái.
  - Phân tích hành vi khách hàng (aggregation).
- Thiết kế shard key sao cho các truy vấn phổ biến chỉ truy cập một shard (tránh truy vấn cross-shard).

#### **Bước 2: Cấu hình Sharding**
- **Ví dụ với MongoDB**:
  - Chọn shard key: `customer_id`.
  - Cấu hình cluster:
    ```javascript
    sh.enableSharding("crm_database")
    sh.shardCollection("crm_database.customers", { "customer_id": "hashed" })
    ```
  - Thêm shards (máy chủ) vào cluster:
    ```javascript
    sh.addShard("shard_server_1:27017")
    sh.addShard("shard_server_2:27017")
    ```
- **Ví dụ với PostgreSQL (Citus)**:
  - Tạo bảng phân vùng:
    ```sql
    SELECT create_distributed_table('customers', 'customer_id');
    ```
  - Phân phối dữ liệu dựa trên `customer_id`.

#### **Bước 3: Quản lý Shards**
- **Cân bằng tải (Rebalancing)**:
  - Khi thêm hoặc xóa shard, cần di chuyển dữ liệu để đảm bảo phân phối đồng đều.
  - MongoDB và Cassandra tự động hóa quá trình này.
- **Replication**:
  - Mỗi shard nên có bản sao (replica) để đảm bảo tính sẵn sàng cao.
  - Ví dụ: Mỗi shard trong MongoDB có 3 bản sao (1 primary, 2 secondaries).
- **Backup**:
  - Sao lưu từng shard độc lập, sử dụng công cụ như `mongodump` hoặc AWS S3.

#### **Bước 4: Tối ưu hóa hiệu suất**
- **Index**: Tạo index trên shard key và các trường truy vấn thường xuyên.
- **Cache**: Sử dụng Redis/Memcached để cache dữ liệu từ các shard thường truy cập.
- **Query Routing**: Sử dụng API Gateway hoặc MongoDB Router (mongos) để định tuyến truy vấn đến shard đúng.

---

### **4. Thách thức và giải pháp**
- **Hotspot**:
  - Vấn đề: Một shard nhận quá nhiều truy vấn (ví dụ: khách hàng ở một khu vực đông dân hơn).
  - Giải pháp: Sử dụng hash-based sharding hoặc chia nhỏ shard (split shard).
- **Truy vấn Cross-Shard**:
  - Vấn đề: Các truy vấn như báo cáo tổng hợp toàn hệ thống cần lấy dữ liệu từ nhiều shard.
  - Giải pháp: Sử dụng data warehouse (Snowflake, BigQuery) cho báo cáo, hoặc tối ưu truy vấn bằng Elasticsearch.
- **Dữ liệu không đồng đều**:
  - Vấn đề: Một số shard chứa nhiều dữ liệu hơn các shard khác.
  - Giải pháp: Theo dõi và rebalance định kỳ, hoặc chọn shard key có tính phân bố cao.
- **Độ phức tạp quản lý**:
  - Vấn đề: Quản lý nhiều shard tăng chi phí vận hành.
  - Giải pháp: Sử dụng công cụ tự động hóa như Kubernetes hoặc Citus.

---

### **5. Ví dụ thực tế**
- **Hệ thống CRM với 10 triệu khách hàng**:
  - Shard key: `customer_id` (hashed).
  - 4 shards, mỗi shard chứa ~2.5M khách hàng.
  - Database: MongoDB cluster với 4 shard servers, mỗi shard có 3 bản sao.
  - Tìm kiếm: Elasticsearch cho tìm kiếm nhanh theo tên, email.
  - Cache: Redis lưu hồ sơ khách hàng thường truy cập.
  - Phân tích: Snowflake để tạo báo cáo hàng tháng.

---

### **6. Công nghệ đề xuất**
| **Công việc**          | **Công nghệ**                  |
|------------------------|--------------------------------|
| Sharding Database      | MongoDB, Cassandra, Citus     |
| Tìm kiếm               | Elasticsearch                 |
| Cache                  | Redis, Memcached              |
| Data Warehouse         | Snowflake, BigQuery           |
| Monitoring             | Prometheus, Grafana           |
