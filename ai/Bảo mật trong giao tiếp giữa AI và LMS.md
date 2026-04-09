## I. CÁC MỐI NGUY GÂY RÒ RỈ (Common Threats)
| Mối nguy                            | Mô tả                                             |
| ----------------------------------- | ------------------------------------------------- |
| 🔓 Dữ liệu bị nghe lén              | Giao tiếp không mã hóa (HTTP, không TLS)          |
| 🧑‍💻 API bị lạm dụng               | Gọi sai mục đích, spam endpoint AI                |
| 🧬 Dữ liệu bị giả mạo hoặc thay đổi | Dữ liệu từ LMS/AI bị sửa giữa đường (MITM attack) |
| 🤖 Webhook bị giả mạo/giả callback  | Một hệ thống khác giả danh AI gửi kết quả về LMS  |
| 🔁 Replay attack                    | Kẻ xấu gửi lại dữ liệu cũ nhiều lần               |
| 💽 Rò rỉ token/API key              | Token hardcode hoặc log lộ ra ngoài               |


## II. CÁC LỚP BẢO MẬT BẠN NÊN ÁP DỤNG
### 1. Xác thực (Authentication)
- Sử dụng JWT (JSON Web Token) cho REST API
- Hoặc dùng OAuth2 Client Credential Flow
- Nếu dùng Webhook: xác minh bằng HMAC Signature

### 2. Mã hoá (Encryption)
- Bắt buộc dùng HTTPS/TLS 1.2+ cho mọi giao tiếp
- Không cho phép HTTP thường

#### Kafka:
- Dùng TLS cho Producer & Consumer (SSL encryption)
- Có thể bật SASL để xác thực user

### 3. Phân quyền & Hạn chế truy cập (Authorization & Access Scope)
- Phân quyền theo vai trò: LMS chỉ được gọi /analyze-skill, không được gọi /admin/train
- Hạn chế theo IP / CIDR / API Gateway
- Token chỉ có hiệu lực trong 15 phút – 1 giờ

### 4. Rate limiting / Throttling
- Giới hạn số lần gọi API từ LMS → AI
- Tránh bị spam gây DDoS
- Có thể dùng: API Gateway, Nginx rate limit, Cloudflare API shield

### 5.  Logging & Audit
- Log lại toàn bộ call: IP, token ID, endpoint, thời gian, trạng thái
- Dùng log để audit hành vi bất thường (có AI model lạ gọi LMS?)
- Gắn trace-id / request-id để trace end-to-end


