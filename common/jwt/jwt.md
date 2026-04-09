**JWT** (JSON Web Token) là một chuẩn mở (RFC 7519) dùng để truyền thông tin giữa các bên dưới dạng một chuỗi JSON an toàn. Nó thường được sử dụng để **xác thực** (authentication) và **trao đổi thông tin** trong các ứng dụng web hoặc API. Dưới đây là giải thích ngắn gọn, dễ hiểu, liên hệ với các ngữ cảnh như **event-driven** và **Nginx** từ các câu hỏi trước nếu cần.

### 1. **JWT là gì?**
- JWT là một **token** (chuỗi ký tự) được mã hóa, chứa thông tin (claims) về người dùng hoặc phiên làm việc, được sử dụng để:
  - **Xác thực**: Kiểm tra danh tính người dùng (ví dụ: đăng nhập API).
  - **Trao đổi thông tin**: Gửi dữ liệu an toàn giữa client và server.
- JWT được mã hóa dưới dạng **Base64** và thường được gửi trong header HTTP (thường là `Authorization: Bearer <token>`).

### 2. **Cấu trúc của JWT**
JWT gồm **3 phần chính**, được nối với nhau bằng dấu chấm (`.`):
- **Header**: Chứa thông tin về thuật toán mã hóa (thường là HMAC SHA256 hoặc RSA) và loại token (JWT).
  - Ví dụ: `{"alg": "HS256", "typ": "JWT"}`
- **Payload**: Chứa các **claims** (thông tin) như ID người dùng, vai trò, thời gian hết hạn, v.v.
  - Ví dụ: `{"sub": "user123", "role": "admin", "exp": 1697059200}`
- **Signature**: Được tạo bằng cách mã hóa Header và Payload với một **secret key** (hoặc cặp public/private key), dùng để kiểm tra tính toàn vẹn.
  - Ví dụ: `HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)`

**Định dạng JWT**: `Header.Payload.Signature` (Base64-encoded)
- Ví dụ: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyMTIzIiwicm9sZSI6ImFkbWluIn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c`

### 3. **Cách hoạt động của JWT**
1. **Tạo token**:
   - Server tạo JWT khi người dùng đăng nhập thành công (ví dụ: kiểm tra username/password).
   - Token chứa thông tin (như ID người dùng) và được ký bằng secret key.
2. **Gửi token**:
   - Client lưu token (thường trong localStorage hoặc cookie) và gửi kèm trong header HTTP cho các yêu cầu sau (`Authorization: Bearer <token>`).
3. **Xác minh token**:
   - Server nhận token, kiểm tra chữ ký (signature) để đảm bảo token hợp lệ và chưa bị thay đổi.
   - Nếu token hợp lệ, server cho phép truy cập tài nguyên.

### 4. **Ưu điểm của JWT**
- **Không trạng thái (Stateless)**: Server không cần lưu trạng thái phiên (session), token chứa đủ thông tin để xác thực.
- **Khả năng mở rộng**: Phù hợp cho hệ thống phân tán, microservices, vì token có thể được xác minh bởi bất kỳ server nào có secret key.
- **Đa nền tảng**: JWT hoạt động trên nhiều ngôn ngữ và framework (Java, Node.js, Python, v.v.).
- **Tích hợp tốt với API**: Thường dùng trong RESTful API hoặc GraphQL.

### 5. **Nhược điểm của JWT**
- **Khó thu hồi token**: Vì JWT là stateless, không dễ thu hồi token trước khi hết hạn (trừ khi dùng blacklist).
- **Kích thước lớn**: Token có thể dài, làm tăng kích thước request.
- **Bảo mật**: Nếu secret key bị lộ, token có thể bị giả mạo. Cần lưu trữ token an toàn ở client.
- **Hết hạn**: Token có thời hạn, cần cơ chế refresh token để duy trì phiên.

### 6. **Liên hệ với Event-Driven và Nginx**
- **Event-Driven**:
  - Trong một hệ thống **event-driven** (như Java NIO server hoặc hệ thống Pub/Sub), JWT có thể được gửi kèm trong các sự kiện (events) để xác thực nguồn gốc của sự kiện.
  - Ví dụ: Một client gửi message vào **Message Queue** (như RabbitMQ) hoặc **Pub/Sub** (như Redis), kèm theo JWT để xác minh danh tính.
- **Nginx**:
  - Nginx có thể được cấu hình để kiểm tra JWT trong header HTTP trước khi chuyển yêu cầu đến backend (ví dụ: dùng module `ngx_http_auth_jwt_module`).
  - Trong hệ thống event-driven, Nginx xử lý các yêu cầu HTTP chứa JWT bằng **event loop** và **epoll**, đảm bảo hiệu suất cao khi xác minh token cho hàng nghìn client.

### 7. **Ví dụ code (Java với JWT)**
Dùng thư viện `jjwt` để tạo và xác minh JWT:

```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import java.util.Date;

public class JwtExample {
    public static void main(String[] args) {
        String secretKey = "mySecretKey";

        // Tạo JWT
        String jwt = Jwts.builder()
                .setSubject("user123")
                .claim("role", "admin")
                .setExpiration(new Date(System.currentTimeMillis() + 86400000)) // Hết hạn sau 1 ngày
                .signWith(SignatureAlgorithm.HS256, secretKey.getBytes())
                .compact();
        System.out.println("JWT: " + jwt);

        // Xác minh JWT
        try {
            Jwts.parser().setSigningKey(secretKey.getBytes()).parseClaimsJws(jwt);
            System.out.println("JWT is valid");
        } catch (Exception e) {
            System.out.println("JWT is invalid: " + e.getMessage());
        }
    }
}
```

- **Cài đặt**: Thêm thư viện `jjwt` vào dự án (dùng Maven hoặc Gradle).
- **Giải thích**: Code tạo một JWT với thông tin người dùng và xác minh tính hợp lệ của nó.

### 8. **Ứng dụng thực tế**
- **Xác thực API**: Client gửi JWT trong header `Authorization` để truy cập API bảo mật.
- **Single Sign-On (SSO)**: JWT được dùng để chia sẻ thông tin xác thực giữa các dịch vụ.
- **Microservices**: Các dịch vụ xác minh JWT để đảm bảo an toàn khi giao tiếp.
- **Event-Driven Systems**: JWT được gửi kèm trong messages (Message Queue) hoặc events (Pub/Sub) để xác minh nguồn gốc.

### 9. **Tóm tắt**
- **JWT** là một chuỗi JSON mã hóa, dùng để xác thực và trao đổi thông tin an toàn.
- **Cấu trúc**: Header.Payload.Signature, chứa thông tin và chữ ký để đảm bảo tính toàn vẹn.
- **Ưu điểm**: Stateless, mở rộng tốt, phù hợp cho API và hệ thống phân tán.
- **Nhược điểm**: Khó thu hồi, kích thước lớn, cần bảo mật secret key.
- **Liên hệ với Event-Driven/Nginx**: JWT có thể được xử lý trong event loop (Nginx, Java NIO) hoặc gửi qua Message Queue/Pub/Sub để xác thực.

