# 1. **JTI là gì?**
**JTI** (JSON Token Identifier) là một claim tùy chọn trong JWT (JSON Web Token), được sử dụng để định danh duy nhất một token. Nó thường là một chuỗi ngẫu nhiên hoặc UUID (Universally Unique Identifier) được gán khi token được tạo ra. 

#### Vai trò của JTI:
- **Theo dõi token**: JTI giúp phân biệt các token khác nhau, đặc biệt khi cần quản lý hoặc thu hồi token.
- **Hỗ trợ blacklist**: Khi muốn thu hồi một token cụ thể, JTI được sử dụng làm khóa để lưu vào danh sách đen (blacklist).
- **Ngăn lạm dụng**: JTI đảm bảo rằng mỗi token có một định danh riêng, giúp phát hiện và ngăn chặn việc sử dụng lại token cũ.

#### Ví dụ:
Trong payload của JWT, JTI có dạng:
```json
{
  "sub": "user123",
  "iat": 1698765432,
  "exp": 1698769032,
  "jti": "a1b2c3d4-e5f6-7890-abcd-1234567890ab"
}
```

Khi verify token, hệ thống có thể kiểm tra `jti` trong blacklist để xác định xem token có bị thu hồi hay không.

