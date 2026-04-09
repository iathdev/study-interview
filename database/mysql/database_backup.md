| Phương pháp        | Tốc độ  | Độ an toàn | Dùng khi                        | Ghi chú                           |
| ------------------ | ------- | ---------- | ------------------------------- | --------------------------------- |
| Cold Backup        | Cao     | Rất cao    | Có downtime cho phép            | Dễ thực hiện, dừng toàn DB        |
| mysqldump          | Thấp    | Trung bình | Dữ liệu nhỏ/một phần            | Dễ dùng, chậm với dữ liệu lớn     |
| mysqlpump          | Trung   | Trung bình | Dữ liệu trung bình              | Hỗ trợ đa luồng, nhanh hơn        |
| Percona XtraBackup | Cao     | Cao        | Dữ liệu lớn, hoạt động liên tục | Phức tạp hơn, không hỗ trợ MyISAM |
| Binary Log         | Cao     | Cao        | Phục hồi theo thời gian         | Kết hợp với backup đầy đủ         |
| Replication        | Rất cao | Trung bình | Hệ thống lớn, cần HA            | Không thay thế backup thật sự     |
| Snapshot Volume    | Rất cao | Trung bình | Trên cloud, block-level         | Nhanh nhưng không chi tiết        |

## 1. Cold Backup (Sao lưu lạnh)

- **Định nghĩa**: Sao lưu khi hệ thống MySQL tắt hoàn toàn.
- **Ưu điểm**:
  - Đơn giản, dễ thực hiện.
  - Không cần cấu hình phức tạp.
  - Đảm bảo toàn vẹn dữ liệu tuyệt đối.
- **Nhược điểm**:
  - Hệ thống phải dừng hoạt động.
  - Không phù hợp cho hệ thống hoạt động liên tục.

## 2. Hot Backup (Sao lưu nóng)

Sao lưu trong khi hệ thống vẫn đang hoạt động.

### 2.1 `mysqldump`

- Sao lưu dưới dạng file SQL, tuần tự từng bảng.
- **Ưu điểm**:
  - Có sẵn trong MySQL.
  - Dễ sử dụng.
  - Không cần dừng hệ thống.
  - Hỗ trợ sao lưu toàn bộ hoặc từng phần.
- **Nhược điểm**:
  - Chậm khi dữ liệu lớn.
  - Dễ gây quá tải CPU/RAM nếu server yếu.

### 2.2 `mysqlpump`

- Giống `mysqldump` nhưng hỗ trợ sao lưu song song.
- **Ưu điểm**:
  - Nhanh hơn `mysqldump`.
  - Có sẵn trong MySQL.
- **Nhược điểm**:
  - Cũng chậm nếu dữ liệu quá lớn.
  - Có thể tiêu tốn nhiều tài nguyên.

### 2.3 Percona XtraBackup

- Công cụ sao lưu nóng mạnh mẽ, không khóa bảng.
- **Ưu điểm**:
  - Sao lưu mà không khóa bảng (InnoDB).
  - Phù hợp với cơ sở dữ liệu lớn, tải cao.
- **Nhược điểm**:
  - Không hỗ trợ MyISAM.
  - Yêu cầu cài đặt và cấu hình riêng.

## 3. Binary Log

- **Cách hoạt động**:
  - Bật `binlog` trong `my.cnf`.
  - Sao lưu định kỳ các file binlog.
  - Lưu tất cả các câu lệnh thay đổi dữ liệu
- **Ưu điểm**:
  - Hỗ trợ phục hồi dữ liệu đến thời điểm cụ thể (Point-in-time Recovery).
  - Giảm rủi ro mất dữ liệu.
  - Hỗ trợ cả replication
- **Nhược điểm**:
  - Tốn dung lượng nếu không quản lý log đúng cách.

## 4. Replication (Master - Slave)

- **Cơ chế**: Dữ liệu ghi trên master được sao chép thời gian thực sang slave.
- **Ưu điểm**:
  - Hạn chế downtime.
  - Tăng tính sẵn sàng, hỗ trợ phân tải.
  - Có thể dùng slave để backup.
- **Nhược điểm**:
  - Không phải backup thật sự (xóa nhầm trên master → mất trên slave).
  - Cần cấu hình và giám sát kỹ lưỡng.

| Tình huống                                 | Đọc từ đâu?                    |
| ------------------------------------------ | ------------------------------ |
| Sau khi ghi (INSERT, UPDATE...)            | Master                         |
| Truy vấn quan trọng, cần dữ liệu chính xác | Master                         |
| Dashboard, tìm kiếm, dữ liệu cache được    | Slave                          |
| Truy vấn trong transaction                 | Master                         |
| Muốn giảm tải cho master                   | Slave (chỉ với SELECT an toàn) |


## 5. Snapshot Volume

- Sao lưu theo block-level bằng snapshot (thường dùng trong cloud hoặc LVM).
- **Ưu điểm**:
  - Nhanh, không ảnh hưởng hiệu suất MySQL.
  - Triển khai đơn giản trên nền tảng cloud.
- **Nhược điểm**:
  - Phụ thuộc vào hệ thống lưu trữ hoặc cloud.
  - Không chi tiết theo mức thay đổi từng dòng dữ liệu.

