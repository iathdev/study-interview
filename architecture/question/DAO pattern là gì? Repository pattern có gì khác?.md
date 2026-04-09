# So sánh DAO Pattern và Repository Pattern

## **DAO Pattern (Data Access Object)**
- **Định nghĩa**: DAO cung cấp giao diện trừu tượng để tương tác với nguồn dữ liệu (database, file, API), gói gọn các thao tác CRUD (Create, Read, Update, Delete) và ẩn chi tiết triển khai.
- **Mục đích**: Tách biệt logic nghiệp vụ khỏi truy cập dữ liệu, dễ thay đổi công nghệ lưu trữ.
- **Cách hoạt động**: 
  - Mỗi DAO tương ứng với một bảng/thực thể trong cơ sở dữ liệu.
  - Cung cấp phương thức như `save()`, `findById()`, `update()`, `delete()`.

## **Repository Pattern**
- **Định nghĩa**: Repository là trung gian giữa Domain Layer và Data Access Layer, cung cấp giao diện giống "bộ sưu tập" để làm việc với đối tượng miền.
- **Mục đích**: Tập trung vào Domain, ẩn chi tiết lưu trữ, cung cấp phương thức theo ngữ cảnh nghiệp vụ.
- **Cách hoạt động**:
  - Làm việc với **Aggregate** trong Domain-Driven Design (DDD).
  - Phương thức theo ngôn ngữ miền, ví dụ: `findActiveUsers()`, `findOrdersByCustomerId()`.

## **So sánh DAO và Repository**

| **Tiêu chí**            | **DAO**                                      | **Repository**                              |
|-------------------------|---------------------------------------------|--------------------------------------------|
| **Mức độ trừu tượng**   | Thấp, tập trung vào thao tác dữ liệu (CRUD). | Cao, tập trung vào miền nghiệp vụ (Domain). |
| **Mục tiêu**            | Ẩn chi tiết truy cập dữ liệu.               | Ẩn cơ chế lưu trữ, giao diện giống bộ sưu tập. |
| **Phạm vi**             | Ánh xạ 1:1 với bảng trong database.         | Làm việc với Aggregate, không phụ thuộc database. |
| **Ngôn ngữ**            | Phương thức dựa trên thao tác dữ liệu.      | Phương thức dựa trên ngữ cảnh nghiệp vụ.   |
| **Ứng dụng**            | Phù hợp dự án đơn giản, không dùng DDD.     | Phù hợp dự án phức tạp, áp dụng DDD.       |
| **Phụ thuộc**           | Có thể phụ thuộc vào công nghệ database.   | Hoàn toàn độc lập với công nghệ database.  |

## **Tóm lại**
- **DAO**: Đơn giản, tập trung vào truy cập dữ liệu, phù hợp dự án nhỏ.
- **Repository**: Trừu tượng, tập trung vào nghiệp vụ, lý tưởng cho dự án lớn dùng DDD.

