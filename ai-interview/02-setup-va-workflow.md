# Setup & Workflow Dùng AI Hiệu Quả

## 1. Thiết lập môi trường

### GitHub Copilot
```bash
# VSCode: cài extension "GitHub Copilot"
# JetBrains: Settings → Plugins → GitHub Copilot
# Đăng nhập GitHub account (cần subscription $10/tháng hoặc miễn phí cho student)
```

### Cursor IDE
```bash
# Download tại cursor.sh
# Import settings từ VSCode (extensions, keybindings)
# Ctrl+K: inline edit | Ctrl+L: chat panel | Ctrl+I: composer (multi-file)
```

### Claude Code
```bash
npm install -g @anthropic-ai/claude-code
export ANTHROPIC_API_KEY="sk-ant-..."

# Thêm vào ~/.zshrc hoặc ~/.bashrc để không phải set lại
echo 'export ANTHROPIC_API_KEY="sk-ant-..."' >> ~/.zshrc
```

---

## 2. Cấu trúc CLAUDE.md tốt

```markdown
# [Tên Project]

## Mục đích
[Mô tả ngắn project làm gì]

## Tech Stack
- Backend: [ngôn ngữ, framework, version]
- Database: [loại DB, ORM]
- Infrastructure: [Docker, K8s, cloud...]

## Quy ước code
- [Convention đặt tên]
- [Pattern hay dùng]
- [Những thứ KHÔNG làm]

## Commands
- Build: [lệnh]
- Test: [lệnh]
- Deploy: [lệnh]

## Context quan trọng
- [Quirk đặc thù của project]
- [Business rule quan trọng]
```

---

## 3. Kỹ thuật prompt hiệu quả

### Cung cấp đủ context
```
# Tệ
"Fix lỗi login"

# Tốt
"Hàm login() trong AuthService.java trả về 403 khi user nhập đúng password.
Stack trace: [paste stack trace]. DB query đã kiểm tra, user tồn tại.
Nghi ngờ vấn đề ở phần so sánh password hash BCrypt."
```

### Chia nhỏ task phức tạp
```
# Thay vì: "Xây dựng hệ thống thanh toán"

# Làm từng bước:
1. "Thiết kế schema cho bảng payments và transactions"
2. "Viết PaymentService với method createPayment()"
3. "Tích hợp Stripe webhook handler"
4. "Viết test cho các case: success, failure, timeout"
```

### Yêu cầu giải thích
```
"Sửa bug này VÀ giải thích tại sao lại bị lỗi,
để lần sau tôi tránh được"
```

### Đặt ràng buộc
```
"Viết theo pattern hiện có trong project, đừng thêm dependency mới,
giữ backward compatible với API v1"
```

---

## 4. Workflow hàng ngày

### Buổi sáng — Bắt đầu task mới
```bash
cd project
claude
# "Tôi cần implement feature X. Đọc file Y và Z để hiểu context,
#  sau đó đề xuất approach trước khi code"
```

### Trong lúc code — Copilot / Cursor
- Gõ comment mô tả function → Copilot tự complete
- `Ctrl+K` trong Cursor → sửa inline nhanh
- Không accept ngay — đọc code được gợi ý

### Debug — Claude Code hoặc Chat
```bash
# Paste error + relevant code
"Lỗi này nghĩa là gì và cách fix?"
```

### Review trước commit
```bash
git diff | claude "Review code này, tìm bug tiềm ẩn và security issue"
```

---

## 5. Tích hợp vào Git workflow

```bash
# Sinh commit message
git diff --staged | claude "Viết commit message theo conventional commits"

# Review PR
gh pr diff 123 | claude "Review PR này, focus vào correctness và performance"

# Sinh changelog
git log v1.0..HEAD --oneline | claude "Tóm tắt thay đổi cho release notes"
```

---

## 6. Quản lý chi phí API

- Dùng **Haiku** cho task đơn giản (rẻ hơn 10-20x)
- Dùng **Sonnet** cho task vừa (mặc định của Claude Code)
- Dùng **Opus** chỉ khi cần reasoning sâu
- Set `/compact` thường xuyên để giảm token
- Không để context quá dài vô ích — `/clear` khi đổi task
