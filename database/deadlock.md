### **Tổng quan về Deadlock**

Deadlock (tạm dịch: bế tắc) là một tình trạng trong lập trình đa luồng (multithreading) hoặc hệ thống phân tán, nơi hai hoặc nhiều tiến trình (process) hoặc luồng (thread) bị kẹt lại do mỗi tiến trình/luồng đang giữ một tài nguyên và chờ đợi tài nguyên khác mà các tiến trình/luồng khác đang giữ. Kết quả là không tiến trình nào có thể tiếp tục thực thi, dẫn đến hệ thống bị đình trệ.

Deadlock thường xảy ra trong các hệ thống quản lý tài nguyên, như cơ sở dữ liệu, hệ điều hành, hoặc ứng dụng sử dụng nhiều luồng. Trong ngữ cảnh hệ thống IAM (Identity and Access Management) hoặc API Gateway tích hợp OAuth 2.0, deadlock có thể xuất hiện khi quản lý truy cập tài nguyên đồng thời (như token, session) nếu không được thiết kế cẩn thận.

Dưới đây là giải thích chi tiết về deadlock, nguyên nhân, điều kiện xảy ra, cách phát hiện, phòng tránh và xử lý.

---

### **1. Deadlock là gì?**

Deadlock là tình trạng mà một tập hợp các tiến trình/luồng bị khóa vô thời hạn vì mỗi tiến trình/luồng:
- Đang giữ ít nhất một tài nguyên.
- Đang chờ để có được tài nguyên bổ sung, mà tài nguyên này lại đang được giữ bởi một tiến trình/luồng khác trong tập hợp.
- Không có tiến trình nào chịu nhả tài nguyên đang giữ, dẫn đến vòng chờ (circular wait).

Ví dụ: 
- Tiến trình A giữ tài nguyên X và chờ tài nguyên Y.
- Tiến trình B giữ tài nguyên Y và chờ tài nguyên X.
- Cả A và B không thể tiếp tục, dẫn đến deadlock.

---

### **2. Các điều kiện dẫn đến Deadlock**

Theo lý thuyết của Coffman, để xảy ra deadlock, cần thỏa mãn đồng thời **bốn điều kiện** sau:

1. **Mutual Exclusion (Loại trừ lẫn nhau)**:
   - Tài nguyên chỉ có thể được giữ bởi một tiến trình/luồng tại một thời điểm. Các tiến trình khác muốn sử dụng tài nguyên phải chờ.
   - Ví dụ: Một khóa (lock) trong cơ sở dữ liệu chỉ cho phép một giao dịch truy cập tại một thời điểm.

2. **Hold and Wait (Giữ và chờ)**:
   - Một tiến trình đang giữ ít nhất một tài nguyên và chờ đợi tài nguyên khác đang bị chiếm giữ bởi tiến trình khác.
   - Ví dụ: Một luồng giữ khóa trên bảng A và chờ khóa trên bảng B.

3. **No Preemption (Không thể cướp quyền)**:
   - Tài nguyên không thể bị lấy đi (cướp quyền) từ tiến trình đang giữ nó, trừ khi tiến trình tự nguyện nhả tài nguyên.
   - Ví dụ: Một luồng không thể bị buộc nhả khóa cơ sở dữ liệu trừ khi giao dịch hoàn tất.

4. **Circular Wait (Vòng chờ)**:
   - Một tập hợp tiến trình tạo thành một chuỗi vòng, trong đó mỗi tiến trình giữ tài nguyên mà tiến trình tiếp theo cần.
   - Ví dụ: A chờ B, B chờ C, C chờ A.

---

### **3. Deadlock trong hệ thống IAM và OAuth 2.0**

Trong hệ thống IAM hoặc API Gateway tích hợp OAuth 2.0, deadlock có thể xảy ra trong các tình huống sau:

- **Quản lý token**:
  - Nhiều luồng hoặc dịch vụ cố gắng truy cập hoặc làm mới refresh token cùng lúc trong Authorization Server, dẫn đến tranh chấp khóa cơ sở dữ liệu (database locks).
  - Ví dụ: Hai luồng đồng thời cố gắng cập nhật trạng thái của cùng một refresh token trong Keycloak, gây ra deadlock nếu cơ sở dữ liệu không được tối ưu.

- **API Gateway**:
  - API Gateway xử lý nhiều yêu cầu đồng thời đến cùng một tài nguyên (như cache token hoặc session), dẫn đến tranh chấp khóa (lock contention).
  - Ví dụ: Nếu Gateway sử dụng cache đồng bộ để lưu trữ access token và nhiều luồng cùng cố gắng đọc/ghi cache, có thể xảy ra deadlock.

- **Cơ sở dữ liệu**:
  - Trong hệ thống IAM, các giao dịch cơ sở dữ liệu (như cập nhật thông tin người dùng, cấp token) có thể gây deadlock nếu truy cập các bảng theo thứ tự không nhất quán.
  - Ví dụ: Một giao dịch giữ khóa trên bảng `users` và chờ bảng `tokens`, trong khi giao dịch khác giữ khóa trên `tokens` và chờ `users`.

---

### **4. Cách phát hiện Deadlock**

Phát hiện deadlock là một bước quan trọng để xử lý và giảm thiểu tác động. Một số phương pháp phát hiện bao gồm:

1. **Sử dụng công cụ giám sát**:
   - Các hệ thống như Keycloak, Okta, hoặc cơ sở dữ liệu (MySQL, PostgreSQL) cung cấp công cụ giám sát để phát hiện deadlock.
   - Ví dụ: PostgreSQL ghi log deadlock trong tệp log của hệ thống.

2. **Biểu đồ chờ (Wait-for Graph)**:
   - Trong hệ điều hành hoặc hệ thống phân tán, sử dụng biểu đồ chờ để phát hiện vòng chờ (circular wait).
   - Nếu biểu đồ có chu trình (cycle), hệ thống đang ở trạng thái deadlock.

3. **Theo dõi hiệu suất**:
   - Quan sát các dấu hiệu như yêu cầu API bị treo, thời gian phản hồi tăng cao, hoặc luồng bị kẹt trong hệ thống.

4. **Công cụ phân tích**:
   - Sử dụng các công cụ như JStack (cho Java), VisualVM, hoặc profiling tools để phát hiện luồng bị kẹt trong ứng dụng.

---

### **5. Cách phòng tránh Deadlock**

Để ngăn chặn deadlock, cần phá vỡ ít nhất một trong bốn điều kiện dẫn đến deadlock:

1. **Phá vỡ Mutual Exclusion**:
   - Khó thực hiện, vì nhiều tài nguyên (như khóa cơ sở dữ liệu, token) yêu cầu loại trừ lẫn nhau để đảm bảo tính toàn vẹn.

2. **Phá vỡ Hold and Wait**:
   - Yêu cầu tiến trình lấy tất cả tài nguyên cần thiết cùng một lúc trước khi thực thi, thay vì giữ một tài nguyên và chờ thêm.
   - Ví dụ: Trong cơ sở dữ liệu, yêu cầu tất cả các khóa cần thiết trong một giao dịch trước khi bắt đầu.

3. **Cho phép Preemption**:
   - Cho phép hệ thống lấy lại tài nguyên từ một tiến trình nếu cần thiết.
   - Ví dụ: Trong hệ thống IAM, thu hồi refresh token nếu phát hiện xung đột truy cập.

4. **Phá vỡ Circular Wait**:
   - Áp đặt thứ tự cố định khi yêu cầu tài nguyên (resource ordering).
   - Ví dụ: Luôn khóa bảng `users` trước bảng `tokens` trong các giao dịch cơ sở dữ liệu.

Các kỹ thuật phòng tránh cụ thể:
- **Sử dụng timeout**: Đặt thời gian chờ cho các khóa (lock timeout) để tự động hủy bỏ nếu không thể lấy tài nguyên.
- **Tối ưu hóa giao dịch**: Giảm thời gian giữ khóa bằng cách tối ưu hóa truy vấn cơ sở dữ liệu hoặc sử dụng giao dịch ngắn gọn.
- **Sử dụng cơ chế khóa không blocking**: Áp dụng các cơ chế như optimistic locking (khóa lạc quan) thay vì pessimistic locking (khóa bi quan).
- **Tích hợp với API Gateway**:
  - Sử dụng rate limiting và caching trong API Gateway để giảm tranh chấp tài nguyên.
  - Đảm bảo Authorization Server xử lý token đồng thời một cách an toàn (ví dụ: sử dụng Redis để lưu trữ token thay vì cơ sở dữ liệu quan hệ).

---

### **6. Cách xử lý Deadlock**

Khi deadlock xảy ra, có thể áp dụng các chiến lược sau để xử lý:

1. **Phát hiện và khôi phục**:
   - Hệ thống (như cơ sở dữ liệu) tự động phát hiện deadlock và hủy một trong các tiến trình để phá vỡ vòng chờ.
   - Ví dụ: PostgreSQL sẽ chọn một giao dịch để hủy và trả về lỗi `deadlock detected`.

2. **Thu hồi tài nguyên**:
   - Hủy bỏ một tiến trình/luồng và giải phóng tài nguyên của nó để các tiến trình khác tiếp tục.

3. **Tái thử (Retry)**:
   - Sau khi phát hiện deadlock, thử lại giao dịch hoặc yêu cầu với cơ chế backoff (tăng thời gian chờ giữa các lần thử).

4. **Cảnh báo và giám sát**:
   - Ghi log deadlock và gửi cảnh báo đến đội vận hành để phân tích nguyên nhân.

---

### **7. Ví dụ thực tế trong IAM và OAuth 2.0**

Giả sử bạn sử dụng Keycloak và API Gateway (như Kong) trong hệ thống IAM:
- **Tình huống deadlock**:
  - Hai luồng trong Keycloak cố gắng cập nhật cùng một refresh token trong cơ sở dữ liệu.
  - Luồng A giữ khóa trên bảng `tokens` và chờ bảng `users`.
  - Luồng B giữ khóa trên bảng `users` và chờ bảng `tokens`.
- **Giải pháp**:
  - Áp đặt thứ tự khóa: Luôn khóa `users` trước `tokens` trong mọi giao dịch.
  - Sử dụng timeout cho các giao dịch cơ sở dữ liệu trong Keycloak (ví dụ: 5 giây).
  - Tích hợp Redis để lưu trữ token tạm thời, giảm tranh chấp cơ sở dữ liệu.
- **Trong API Gateway**:
  - Sử dụng caching (như Redis) để lưu trữ access token, tránh truy cập cơ sở dữ liệu liên tục.
  - Cấu hình rate limiting để giới hạn số lượng yêu cầu đồng thời từ client.

