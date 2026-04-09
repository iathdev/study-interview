
1. Creational Patterns (Tạo đối tượng)
Giúp bạn tạo object linh hoạt và tối ưu.

| Pattern              | Mô tả ngắn                                    | Ví dụ thực tế                  |
| -------------------- | --------------------------------------------- | ------------------------------ |
| **Singleton**        | Một lớp chỉ có 1 instance duy nhất            | Logger, Config                 |
| **Factory**          | Tạo object thông qua một interface/chọn class | Tạo shape: Circle, Square...   |
| **Abstract Factory** | Tạo ra họ object liên quan                    | UI Toolkit (Win vs Mac UI)     |
| **Builder**          | Tạo object phức tạp theo từng bước            | Tạo đối tượng `Pizza` tùy chọn |
| **Prototype**        | Sao chép object đã có                         | Clone file, tài liệu mẫu       |

2. Structural Patterns (Cấu trúc đối tượng)
Tổ chức mối quan hệ giữa các class và object.

| Pattern       | Mô tả ngắn                                        | Ví dụ thực tế                        |
| ------------- | ------------------------------------------------- | ------------------------------------ |
| **Adapter**   | "Chuyển định dạng" để tương thích interface       | Cổng sạc điện thoại                  |
| **Decorator** | Thêm hành vi mới mà không sửa class gốc           | Thêm topping vào trà sữa             |
| **Facade**    | Cung cấp interface đơn giản cho hệ thống phức tạp | Hệ thống bật tắt máy tính (Power ON) |
| **Proxy**     | Đại diện thay thế để điều khiển truy cập          | Đại lý bán vé, cache                 |
| **Composite** | Nhóm các object giống nhau theo cấu trúc cây      | Folder & File                        |
| **Bridge**    | Tách phần trừu tượng và triển khai                | Điều khiển TV qua remote             |
| **Flyweight** | Dùng lại object giống nhau để tiết kiệm bộ nhớ    | Ký tự trong word processor           |

3. Behavioral Patterns (Hành vi – cách object giao tiếp)

| Pattern                     | Mô tả ngắn                                     | Ví dụ thực tế                          |
| --------------------------- | ---------------------------------------------- | -------------------------------------- |
| **Observer**                | Theo dõi & thông báo thay đổi                  | Notification Facebook/Youtube          |
| **Strategy**                | Thay đổi thuật toán linh hoạt                  | Giảm giá theo mùa, tính phí vận chuyển |
| **Command**                 | Biến hành động thành object, dễ undo/redo      | Ctrl+Z/Redo                            |
| **State**                   | Thay đổi hành vi theo trạng thái               | Máy bán hàng tự động, traffic light    |
| **Template**                | Định nghĩa khung xử lý, cho phép override      | Giao dịch ngân hàng                    |
| **Chain of Responsibility** | Xử lý theo chuỗi                               | Hệ thống xử lý đơn bảo hành            |
| **Mediator**                | Điều phối giữa các object                      | Chatroom, Controller (MVC)             |
| **Visitor**                 | Tách hành vi ra khỏi đối tượng cần xử lý       | Tính thuế trên các loại sản phẩm       |
| **Iterator**                | Duyệt qua phần tử mà không lộ cấu trúc nội tại | Vòng lặp trong `List`, `Map`           |

```pgsql
Design Patterns
├── Creational
│   ├── Singleton
│   ├── Factory / Abstract Factory
│   ├── Builder
│   └── Prototype
├── Structural
│   ├── Adapter
│   ├── Decorator
│   ├── Facade
│   ├── Composite
│   └── Proxy / Bridge / Flyweight
└── Behavioral
    ├── Observer
    ├── Strategy
    ├── Command
    ├── State
    ├── Template
    ├── Chain of Responsibility
    └── Mediator / Visitor / Iterator
```