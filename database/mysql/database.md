# 1. Thiết kế cơ sở dữ liệu:
Lý thuyết:
Thiết kế theo các chuẩn 1NF, 2NF, 3NF, BCNF

## 1NF - Chuẩn hoá cột
Quy tắc:
- Mỗi cột chỉ chứa một giá trị nguyên tử
- Không lồng nhiều giá trị vào một cột


Vi phạm:

| StudentID | Name    | Subjects           |
|-----------|---------|--------------------|
| 1         | An      | Math, Physics      |
| 2         | Binh    | Literature, History|

Subjects chứa nhiều giá trị → vi phạm 1NF

Đáp ứng:

| StudentID | Name  | Subject     |
|-----------|-------|-------------|
| 1         | An    | Math        |
| 1         | An    | Physics     |
| 2         | Binh  | Literature  |
| 2         | Binh  | History     |


## 2NF - Chuẩn hoá khoá chính
Quy tắc:
- Đáp ứng 1NF
- Các cột phải phụ thuộc vào khoá chính
- Nếu có nhiều cột tạo thành khoá chính thì các cột còn lại phải phụ thuộc vào khoá chính này. Không được phụ thuộc vào chỉ 1 thành phần của khoá chính -> cần tách bảng riêng

Vi phạm:

| StudentID | CourseID | StudentName |
|-----------|----------|-------------|
| 1         | C1       | An          |
| 2         | C1       | Binh        |

Khóa chính = (StudentID, CourseID)
StudentName chỉ phụ thuộc vào StudentID → phụ thuộc từng phần → vi phạm 2NF

Đáp ứng: tách 2 bảng riêng

-- Student table
| StudentID | StudentName |
|-----------|-------------|
| 1         | An          |
| 2         | Binh        |

-- Enrollment table
| StudentID | CourseID |
|-----------|----------|
| 1         | C1       |
| 2         | C1       |


## 3NF - Chuẩn hoá bảng
Quy tắc:
- Đáp ứng 2NF
- Không có thuộc tính phụ thuộc bắc cầu
- Xảy ra khi một trường thay vì phụ thuộc vào khoá chính lại phụ thuộc vào trường khác (thường là khoá ngoại)


Vi phạm:

| StudentID | StudentName | Department | DepartmentLocation |
|-----------|-------------|------------|---------------------|
| 1         | An          | IT         | Building A          |
| 2         | Binh        | Math       | Building B          |

StudentID → Department
Department → DepartmentLocation ⇒ StudentID → DepartmentLocation (phụ thuộc bắc cầu)

Đáp ứng:

-- Student table
| StudentID | StudentName | Department |
|-----------|-------------|------------|
| 1         | An          | IT         |

-- Department table
| Department | Location   |
|------------|------------|
| IT         | Building A |


## Về kinh nghiệm cần thiết kế DB đảm bảo các tính chất
+ Tránh trùng lặp dữ liệu
+ Khoá chính & định danh duy nhất: đảm bảo tính duy nhất của mỗi bảng, các thuộc tính của bảng phụ thuộc vào khoá
chính
+ Tính toàn vẹn, nhất quán của dữ liệu
+ Tính nguyên tử: tránh làm phức tạp dữ liệu
+ Chuẩn hoá
+ Bảo mật: password, hoặc những thông tin quan trọng khác cần được mã hoá
+ Dễ bảo trì
+ Tính mở rộng cao

## Câu hỏi:
1. Làm sao để thiết kế DB có tính mở rộng
+ Đảm bảo các nguyên tắc thiết kế CSDL:
- Chuẩn hoá dữ liệu
- Chia nhỏ bảng theo từng chức năng
- Replication

VD: 
Polymorphic relationship (nhiều loại liên kết về 1 bảng)
-- comments table
id | commentable_type | commentable_id | content
---|------------------|----------------|--------
1  | "posts"          | 10             | "Hay quá"
2  | "videos"         | 5              | "Nice video"

Metadata / JSON column
-- salary
id | user_id | salary_detail (JSON)
---|---------|-------------------
1  | 1       | {"hobby": "reading", "gender": "male"}

