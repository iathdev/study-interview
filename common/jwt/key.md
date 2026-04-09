Để giải thích cách **tạo** và **xác minh** JWT (JSON Web Token) trong Java cho cả hai trường hợp **khóa đối xứng** (symmetric key) và **khóa bất đối xứng** (asymmetric key), tôi sẽ cung cấp hướng dẫn rõ ràng, ngắn gọn, kèm ví dụ code sử dụng thư viện `jjwt`. Tôi sẽ liên hệ với ngữ cảnh **event-driven** và **Nginx** từ các câu hỏi trước khi cần thiết, đồng thời đảm bảo câu trả lời dễ hiểu và tập trung vào câu hỏi của bạn.

### 1. **Khóa đối xứng (Symmetric Key)**
- **Định nghĩa**: Sử dụng một **secret key** duy nhất để cả tạo và xác minh JWT. Thường dùng thuật toán **HMAC** (như HS256, HS384, HS512).
- **Ứng dụng**: Phù hợp cho hệ thống mà server tạo và xác minh token đều có quyền truy cập vào cùng secret key (ví dụ: trong một ứng dụng đơn lẻ hoặc microservices nội bộ).
- **Ưu điểm**: Đơn giản, nhanh, dễ triển khai.
- **Nhược điểm**: Nếu secret key bị lộ, token có thể bị giả mạo.

#### **Tạo JWT với khóa đối xứng**
Dùng thư viện `jjwt` để tạo token với thuật toán HS256:

```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import java.util.Date;

public class SymmetricJwtExample {
    public static void main(String[] args) {
        String secretKey = "mySecretKey1234567890"; // Khóa bí mật (ít nhất 32 ký tự cho HS256)

        // Tạo JWT
        String jwt = Jwts.builder()
                .setSubject("user123") // ID người dùng
                .claim("role", "admin") // Thông tin bổ sung
                .setIssuedAt(new Date()) // Thời gian phát hành
                .setExpiration(new Date(System.currentTimeMillis() + 86400000)) // Hết hạn sau 1 ngày
                .signWith(SignatureAlgorithm.HS256, secretKey.getBytes()) // Ký với HS256
                .compact();

        System.out.println("JWT: " + jwt);
    }
}
```

- **Giải thích**:
  - `setSubject`: Đặt ID người dùng (có thể là email, username).
  - `claim`: Thêm thông tin tùy chỉnh (như vai trò).
  - `setExpiration`: Đặt thời gian hết hạn.
  - `signWith`: Ký token bằng secret key và thuật toán HS256.

#### **Xác minh JWT với khóa đối xứng**
Dùng cùng secret key để kiểm tra tính hợp lệ của token:

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureException;

public class VerifySymmetricJwt {
    public static void main(String[] args) {
        String secretKey = "mySecretKey1234567890";
        String jwt = "your_jwt_here"; // Thay bằng token từ bước tạo

        try {
            Claims claims = Jwts.parser()
                    .setSigningKey(secretKey.getBytes())
                    .parseClaimsJws(jwt)
                    .getBody();
            System.out.println("Subject: " + claims.getSubject());
            System.out.println("Role: " + claims.get("role"));
            System.out.println("JWT is valid");
        } catch (Exception e) {
            System.out.println("JWT is invalid: " + e.getMessage());
        }
    }
}
```

- **Giải thích**:
  - `setSigningKey`: Dùng cùng secret key để xác minh chữ ký.
  - `parseClaimsJws`: Giải mã token, kiểm tra chữ ký và trả về payload (claims).
  - Nếu token không hợp lệ (sai key, hết hạn, hoặc bị chỉnh sửa), sẽ ném ngoại lệ.

### 2. **Khóa bất đối xứng (Asymmetric Key)**
- **Định nghĩa**: Sử dụng cặp khóa **public/private key** để tạo và xác minh JWT. Private key dùng để ký token, public key dùng để xác minh. Thường dùng thuật toán **RSA** (như RS256) hoặc **ECDSA**.
- **Ứng dụng**: Phù hợp cho hệ thống phân tán, nơi private key được giữ bí mật ở server tạo token, và public key được chia sẻ cho các server khác để xác minh (ví dụ: trong microservices hoặc với bên thứ ba).
- **Ưu điểm**: An toàn hơn, vì chỉ private key được dùng để ký, public key chỉ xác minh.
- **Nhược điểm**: Phức tạp hơn, cần quản lý cặp khóa.

#### **Tạo cặp khóa RSA**
Trước tiên, tạo cặp khóa RSA (private key và public key) bằng công cụ như `keytool` hoặc Java:

```java
import java.security.KeyPair;
import java.security.KeyPairGenerator;

public class GenerateRsaKeyPair {
    public static void main(String[] args) throws Exception {
        KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA");
        keyGen.initialize(2048); // Độ dài khóa 2048 bit
        KeyPair keyPair = keyGen.generateKeyPair();

        // Lưu private key và public key (có thể lưu vào file hoặc sử dụng trực tiếp)
        System.out.println("Private Key: " + keyPair.getPrivate());
        System.out.println("Public Key: " + keyPair.getPublic());
    }
}
```

#### **Tạo JWT với khóa bất đối xứng**
Sử dụng private key để ký token:

```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.util.Date;

public class AsymmetricJwtExample {
    public static void main(String[] args) throws Exception {
        // Tạo cặp khóa RSA
        KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA");
        keyGen.initialize(2048);
        KeyPair keyPair = keyGen.generateKeyPair();

        // Tạo JWT
        String jwt = Jwts.builder()
                .setSubject("user123")
                .claim("role", "admin")
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + 86400000))
                .signWith(SignatureAlgorithm.RS256, keyPair.getPrivate()) // Ký với private key
                .compact();

        System.out.println("JWT: " + jwt);
    }
}
```

- **Giải thích**:
  - `signWith`: Dùng private key và thuật toán RS256 để ký token.
  - Token chứa chữ ký được tạo từ private key, chỉ có thể xác minh bằng public key tương ứng.

#### **Xác minh JWT với khóa bất đối xứng**
Sử dụng public key để kiểm tra tính hợp lệ:

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import java.security.KeyPair;
import java.security.KeyPairGenerator;

public class VerifyAsymmetricJwt {
    public static void main(String[] args) throws Exception {
        // Tạo cặp khóa RSA (trong thực tế, lấy public key từ file hoặc nguồn khác)
        KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA");
        keyGen.initialize(2048);
        KeyPair keyPair = keyGen.generateKeyPair();

        String jwt = "your_jwt_here"; // Thay bằng token từ bước tạo

        try {
            Claims claims = Jwts.parser()
                    .setSigningKey(keyPair.getPublic()) // Xác minh bằng public key
                    .parseClaimsJws(jwt)
                    .getBody();
            System.out.println("Subject: " + claims.getSubject());
            System.out.println("Role: " + claims.get("role"));
            System.out.println("JWT is valid");
        } catch (Exception e) {
            System.out.println("JWT is invalid: " + e.getMessage());
        }
    }
}
```

- **Giải thích**:
  - `setSigningKey`: Dùng public key để xác minh chữ ký của token.
  - Nếu token được ký bằng private key tương ứng, xác minh sẽ thành công.

### 3. **So sánh khóa đối xứng và bất đối xứng**

| **Tiêu chí**             | **Khóa đối xứng (HS256)**                          | **Khóa bất đối xứng (RS256)**                     |
|--------------------------|---------------------------------------------------|--------------------------------------------------|
| **Khóa sử dụng**         | Một secret key cho cả ký và xác minh.            | Private key để ký, public key để xác minh.       |
| **Thuật toán**           | HMAC (HS256, HS384, HS512).                      | RSA, ECDSA (RS256, ES256).                      |
| **Bảo mật**              | Dễ bị lộ nếu secret key không được bảo vệ.        | An toàn hơn, chỉ private key cần được bảo mật.   |
| **Hiệu suất**            | Nhanh hơn, vì HMAC đơn giản hơn RSA.              | Chậm hơn, do tính toán RSA phức tạp.            |
| **Ứng dụng**             | Hệ thống nội bộ, nơi secret key dễ chia sẻ an toàn. | Microservices, SSO, chia sẻ token với bên thứ ba. |
| **Quản lý khóa**         | Chỉ cần quản lý một khóa.                        | Cần quản lý cặp khóa public/private.            |


