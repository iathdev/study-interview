# **Bảo mật Refresh Token và Access Token**
Việc bảo mật **access token** và **refresh token** là rất quan trọng để ngăn chặn các cuộc tấn công như đánh cắp token, sử dụng lại token, hoặc giả mạo. Dưới đây là các phương pháp bảo mật cụ thể:

#### **Bảo mật Access Token**
Access token thường có thời gian sống ngắn (5-15 phút) và được sử dụng để truy cập tài nguyên. Các cách bảo mật:
1. **Sử dụng HTTPS**:
   - Luôn truyền access token qua kênh mã hóa HTTPS để ngăn chặn các cuộc tấn công man-in-the-middle (MITM).
2. **Thời gian sống ngắn**:
   - Đặt `exp` (expiration) ngắn để giảm thiểu rủi ro nếu token bị đánh cắp.
   - Ví dụ: `expiresIn: '15m'` khi tạo token trong `jsonwebtoken`.
3. **Chữ ký mạnh**:
   - Sử dụng thuật toán mạnh như **RS256** (RSA với public/private key) thay vì **HS256** để tăng bảo mật.
   - Lưu private key an toàn trên server, chỉ chia sẻ public key.
4. **Hạn chế scope**:
   - Chỉ cấp quyền tối thiểu cần thiết trong claim `scope` của token.
   - Ví dụ: Nếu người dùng chỉ cần đọc dữ liệu, không cấp quyền ghi.
5. **Kiểm tra issuer và audience**:
   - Đảm bảo claim `iss` (issuer) và `aud` (audience) khớp với hệ thống của bạn để ngăn token từ nguồn không đáng tin cậy.
6. **Lưu trữ an toàn trên client**:
   - Lưu access token trong **HttpOnly, Secure cookie** hoặc **localStorage** với các biện pháp bảo vệ (như mã hóa hoặc chống XSS).
        - HttpOnly: Ngăn JavaScript truy cập cookie (giảm nguy cơ XSS).
        - Secure: Chỉ gửi cookie qua HTTPS.
        - SameSite=Strict: Ngăn cookie được gửi trong các yêu cầu cross-site, chống CSRF.
   - Tránh lưu trong **sessionStorage** hoặc nơi dễ bị truy cập bởi JavaScript.

#### **Bảo mật Refresh Token**
Refresh token có thời gian sống dài hơn (vài ngày hoặc tuần) và được sử dụng để tạo access token mới. Vì vậy, nó cần được bảo vệ cẩn thận hơn. Các cách bảo mật:
1. **Lưu trữ an toàn trên server**:
   - Lưu refresh token trong cơ sở dữ liệu hoặc Redis với mã hóa hoặc băm (hash) để ngăn truy cập trái phép.
   - Ví dụ: Lưu refresh token trong Redis với khóa như `refresh:<token>` và giá trị là user ID.
2. **Lưu trữ an toàn trên client**:
   - Sử dụng **HttpOnly, Secure, SameSite=Strict cookie** để lưu refresh token, ngăn chặn truy cập qua JavaScript và bảo vệ chống CSRF.
   - Tránh lưu refresh token trong localStorage hoặc nơi dễ bị đánh cắp qua XSS.
3. **Kiểm tra tính duy nhất**:
   - Đảm bảo mỗi refresh token chỉ được sử dụng một lần hoặc có cơ chế phát hiện sử dụng lại (dùng `jti` hoặc cơ chế tương tự).
   - Ví dụ: Khi refresh token được sử dụng, xóa token cũ và tạo token mới.
4. **Thu hồi khi cần**:
   - Khi người dùng đăng xuất hoặc nghi ngờ bị xâm phạm, xóa refresh token khỏi cơ sở dữ liệu hoặc đánh dấu là không hợp lệ.
   - Ví dụ trong Redis:
     ```javascript
     await client.del(`refresh:${refreshToken}`);
     ```
5. **Rotate Refresh Token**:
   - Mỗi khi refresh token được sử dụng để tạo access token mới, phát hành một refresh token mới và vô hiệu hóa token cũ.
   - Điều này giảm nguy cơ nếu refresh token bị đánh cắp.
     ```javascript
     app.post('/refresh-token', async (req, res) => {
       const { refreshToken } = req.body;
       const userId = await client.get(`refresh:${refreshToken}`);
       if (!userId) return res.status(401).send('Refresh token không hợp lệ');

       try {
         const decoded = jwt.verify(refreshToken, 'refresh-secret');
         // Xóa refresh token cũ
         await client.del(`refresh:${refreshToken}`);
         // Tạo access token và refresh token mới
         const newAccessToken = jwt.sign({ userId: decoded.userId }, 'access-secret', { expiresIn: '15m' });
         const newRefreshToken = jwt.sign({ userId: decoded.userId }, 'refresh-secret', { expiresIn: '7d' });
         await client.set(`refresh:${newRefreshToken}`, userId, 'EX', 7 * 24 * 60 * 60);
         res.json({ accessToken: newAccessToken, refreshToken: newRefreshToken });
       } catch (err) {
         res.status(401).send('Refresh token không hợp lệ');
       }
     });
     ```
6. **Sử dụng Binding**:
   - Liên kết refresh token với thông tin cụ thể của client (như IP, user-agent) để đảm bảo token chỉ được sử dụng từ thiết bị ban đầu.
   - Lưu ý: Cách này có thể gây bất tiện nếu người dùng thay đổi thiết bị hoặc mạng.

#### **Bảo mật chung cho cả Access Token và Refresh Token**
1. **Sử dụng JTI để quản lý**:
   - Gắn `jti` vào cả access token và refresh token để dễ dàng theo dõi và thu hồi.
   - Lưu `jti` trong blacklist khi cần thu hồi token.
2. **Phát hiện bất thường**:
   - Theo dõi các mẫu sử dụng token bất thường (ví dụ: nhiều yêu cầu từ các địa điểm khác nhau trong thời gian ngắn).
   - Có thể sử dụng hệ thống giám sát hoặc WAF (Web Application Firewall) để phát hiện.
3. **Cơ chế thu hồi khẩn cấp**:
   - Cung cấp API để người dùng hoặc admin thu hồi token ngay lập tức (như trong ví dụ ở câu hỏi trước).
4. **Bảo vệ khóa bí mật**:
   - Lưu secret key hoặc private key trong biến môi trường hoặc hệ thống quản lý bí mật (như AWS Secrets Manager, HashiCorp Vault).
   - Định kỳ thay đổi khóa và vô hiệu hóa token cũ nếu cần.
5. **Xử lý lỗi an toàn**:
   - Trả về thông báo lỗi chung như "Token không hợp lệ" thay vì chi tiết (ví dụ: "Token hết hạn") để tránh tiết lộ thông tin cho kẻ tấn công.
6. **Sử dụng JWKS cho RS256**:
   - Nếu dùng RS256, lấy public key từ **JWKS endpoint** của IdP (Identity Provider) để đảm bảo key luôn cập nhật.
   - Ví dụ trong Node.js:
     ```javascript
     const jwksClient = require('jwks-rsa');
     const client = jwksClient({
       jwksUri: 'https://your-idp/.well-known/jwks.json'
     });

     function getKey(header, callback) {
       client.getSigningKey(header.kid, (err, key) => {
         const signingKey = key.publicKey || key.rsaPublicKey;
         callback(null, signingKey);
       });
     }

     function verifyToken(token) {
       return new Promise((resolve, reject) => {
         jwt.verify(token, getKey, { algorithms: ['RS256'] }, (err, decoded) => {
           if (err) reject(err);
           resolve(decoded);
         });
       });
     }
     ```

---