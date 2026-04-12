# PostgreSQL vs MySQL

---

## 1. Tổng quan

| | PostgreSQL | MySQL |
|--|-----------|-------|
| **Loại** | Object-Relational Database | Relational Database |
| **Triết lý** | Đúng đắn trước, tính năng phong phú | Đơn giản, nhanh, dễ dùng |
| **Phổ biến trong** | Fintech, analytics, hệ thống phức tạp | Web app, e-commerce, startup |
| **License** | PostgreSQL License (tự do) | GPL (community) / Commercial (enterprise) |
| **Backing** | Cộng đồng mã nguồn mở | Oracle (từ 2010) |

---

## 2. Kiến trúc

| | PostgreSQL | MySQL |
|--|-----------|-------|
| **Storage engine** | Một engine duy nhất (heap-based) | Nhiều engine: InnoDB (mặc định), MyISAM, Memory... |
| **Process model** | Multi-process — mỗi connection là một process riêng | Multi-thread — các connection share thread pool |
| **MVCC** | Có — dùng tuple versioning trong heap | Có — dùng undo log (InnoDB) |
| **WAL** | Write-Ahead Log | Redo log + Binlog |

---

## 3. SQL Compliance

PostgreSQL tuân thủ chuẩn SQL nghiêm ngặt hơn MySQL.

| Tính năng | PostgreSQL | MySQL |
|-----------|-----------|-------|
| **Full SQL standard** | Gần như đầy đủ | Bỏ qua một số chuẩn |
| **Window functions** | Có, đầy đủ | Có (từ 8.0) |
| **CTE (WITH clause)** | Có, hỗ trợ recursive | Có (từ 8.0) |
| **FULL OUTER JOIN** | Có | Không có — phải dùng UNION |
| **CHECK constraint** | Có, thực thi đầy đủ | Có (từ 8.0.16), trước đó parse nhưng bỏ qua |
| **RETURNING clause** | Có — INSERT/UPDATE/DELETE trả về row vừa thay đổi | Không có |
| **UPSERT** | `INSERT ... ON CONFLICT` | `INSERT ... ON DUPLICATE KEY UPDATE` |

---

## 4. Kiểu dữ liệu

PostgreSQL có hệ thống type phong phú hơn nhiều:

| Kiểu | PostgreSQL | MySQL |
|------|-----------|-------|
| **JSON** | `JSON` và `JSONB` (binary, có index) | `JSON` (không index trực tiếp) |
| **Array** | Có — `integer[]`, `text[]`... | Không có |
| **UUID** | Có native type | Phải dùng `CHAR(36)` hoặc `BINARY(16)` |
| **Enum** | Có, nhưng khó alter | Có, dễ dùng hơn |
| **Range type** | Có — `daterange`, `int4range`... | Không có |
| **Geometric** | Có — point, polygon, circle... | Không có |
| **Custom type** | Có — tự định nghĩa type | Không có |
| **Full-text search** | Có, mạnh hơn | Có, cơ bản hơn |

---

## 5. Index

| | PostgreSQL | MySQL (InnoDB) |
|--|-----------|----------------|
| **B-Tree** | Có | Có |
| **Hash** | Có | Có (in-memory only) |
| **GIN** | Có — tốt cho JSON, Array, full-text | Không có |
| **GiST** | Có — tốt cho geometric, range | Không có |
| **BRIN** | Có — hiệu quả cho data tuần tự (timestamp) | Không có |
| **Partial index** | Có — index chỉ một phần row | Không có |
| **Expression index** | Có — index trên `lower(email)` | Có (từ 8.0) |
| **Clustered index** | Không có mặc định (heap table) | Có — primary key là clustered index |

**Điểm quan trọng về clustered index:**
- MySQL InnoDB: primary key quyết định vật lý sắp xếp data trên disk → query theo PK rất nhanh, nhưng insert random PK (UUID) gây fragmentation
- PostgreSQL: heap table, data không sắp theo PK → cần CLUSTER command thủ công nếu muốn

---

## 6. Transaction & Concurrency

| | PostgreSQL | MySQL (InnoDB) |
|--|-----------|----------------|
| **MVCC** | Tuple versioning — version cũ nằm trong cùng heap | Undo log — version cũ nằm trong undo tablespace |
| **Isolation levels** | Read Uncommitted, Read Committed, Repeatable Read, Serializable | Tương tự, nhưng mặc định Repeatable Read |
| **Mặc định isolation** | Read Committed | Repeatable Read |
| **Deadlock detection** | Có | Có |
| **Advisory locks** | Có — lock tùy ý theo business logic | Không có |
| **Row-level locking** | Có | Có |
| **DDL transaction** | Có — `ALTER TABLE` có thể rollback | Không — DDL tự động commit |

**DDL transaction** là điểm khác biệt lớn:
- PostgreSQL: `ALTER TABLE`, `CREATE INDEX` có thể wrap trong transaction, rollback nếu lỗi
- MySQL: chạy `ALTER TABLE` rồi lỗi → không rollback được, phải xử lý thủ công

---

## 7. Replication

| | PostgreSQL | MySQL |
|--|-----------|-------|
| **Physical replication** | Streaming replication (WAL-based) | Không có |
| **Logical replication** | Có (từ 10) — replication theo row changes | Binlog-based replication |
| **Synchronous replication** | Có | Có (semi-sync) |
| **Multi-master** | Không native — cần BDR, Citus | Group Replication (từ 5.7) |
| **Failover** | Patroni, repmgr | Orchestrator, MHA |

---

## 8. Performance

Không có câu trả lời tuyệt đối — phụ thuộc vào workload:

| Workload | Winner | Lý do |
|---------|--------|-------|
| **Simple read/write, high concurrency** | MySQL | Thread model nhẹ hơn, connection overhead thấp |
| **Complex query, analytics** | PostgreSQL | Query planner mạnh hơn, nhiều index type hơn |
| **JSON query** | PostgreSQL | JSONB + GIN index nhanh hơn nhiều |
| **Write-heavy, simple schema** | MySQL | InnoDB clustered index tối ưu sequential write |
| **Geospatial** | PostgreSQL | PostGIS extension |
| **Full-text search** | PostgreSQL | Mạnh hơn, hoặc dùng Elasticsearch cho cả hai |

---

## 9. Extensibility

PostgreSQL nổi tiếng với khả năng mở rộng:

- **PostGIS** — geospatial data, GIS queries
- **TimescaleDB** — time-series data
- **pgvector** — vector embeddings, AI/ML similarity search
- **Citus** — horizontal sharding
- **Custom functions** — viết bằng PL/pgSQL, Python, C...
- **Custom type, operator, index method**

MySQL không có hệ sinh thái extension tương đương.

---

## 10. Khi nào chọn gì

**Chọn PostgreSQL khi:**
- Cần query phức tạp, nhiều JOIN, subquery
- Lưu JSON và cần query/index vào trong JSON
- Cần tính năng địa lý (PostGIS)
- Cần DDL transaction, data integrity cao
- Dùng cho analytics, reporting
- Hệ thống tài chính cần ACID nghiêm ngặt

**Chọn MySQL khi:**
- Web app truyền thống, schema đơn giản
- Team đã quen MySQL, ecosystem có sẵn
- Cần tích hợp với stack phổ biến (WordPress, Laravel mặc định)
- Read-heavy với query đơn giản
- Hosting giá rẻ (MySQL phổ biến hơn trên shared hosting)
