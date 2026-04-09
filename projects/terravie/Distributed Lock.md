Distributed Lock (Khóa phân tán) là một cơ chế được sử dụng trong các hệ thống phân tán để đảm bảo rằng chỉ một tiến trình hoặc nút (node) trong hệ thống có thể truy cập hoặc thực thi một tác vụ cụ thể tại một thời điểm, tránh xung đột hoặc trạng thái không nhất quán. 

### Ý nghĩa và mục đích:
- **Đồng bộ hóa**: Đảm bảo rằng các tài nguyên dùng chung (như cơ sở dữ liệu, tệp, hoặc dịch vụ) chỉ được truy cập bởi một thực thể duy nhất tại một thời điểm.
- **Tránh xung đột**: Ngăn chặn nhiều tiến trình thực hiện cùng một thao tác có thể gây ra lỗi hoặc dữ liệu không nhất quán.
- **Tính nhất quán**: Duy trì trạng thái nhất quán trong các hệ thống phân tán, nơi nhiều máy tính hoặc container hoạt động đồng thời.

### Cách hoạt động:
Distributed Lock thường được triển khai thông qua một hệ thống trung gian (như ZooKeeper, Redis, etcd, hoặc Consul) để quản lý khóa. Quy trình cơ bản:
1. **Yêu cầu khóa**: Một tiến trình cố gắng "khóa" một tài nguyên bằng cách ghi một giá trị (lock) vào hệ thống quản lý khóa.
2. **Kiểm tra khóa**: Nếu khóa đã được giữ bởi một tiến trình khác, tiến trình yêu cầu sẽ chờ hoặc thử lại sau.
3. **Thả khóa**: Sau khi hoàn thành tác vụ, tiến trình giải phóng khóa để các tiến trình khác có thể sử dụng.

### Ví dụ công cụ hỗ trợ:
- **Redis**: Sử dụng lệnh `SETNX` (Set if Not Exists) hoặc Redlock để triển khai khóa phân tán.
- **ZooKeeper**: Sử dụng các "znodes" để quản lý khóa, đảm bảo tính nhất quán cao.
- **etcd**: Dùng cơ chế lease và key-value store để hỗ trợ khóa phân tán.

### Ứng dụng thực tế:
- **Cơ sở dữ liệu**: Đảm bảo chỉ một tiến trình cập nhật dữ liệu tại một thời điểm.
- **Hệ thống thanh toán**: Ngăn chặn việc xử lý trùng lặp một giao dịch.
- **Tác vụ định kỳ**: Đảm bảo một công việc định kỳ (cron job) chỉ chạy trên một máy trong cụm.

### Thách thức:
- **Hiệu suất**: Việc quản lý khóa trong hệ thống phân tán có thể gây độ trễ.
- **Tính sẵn sàng**: Nếu hệ thống quản lý khóa (như Redis) gặp sự cố, toàn bộ hệ thống có thể bị ảnh hưởng.
- **Deadlock**: Cần cơ chế để xử lý khi một tiến trình giữ khóa quá lâu hoặc bị crash.

