# Claude Code — AI Agent cho Lập Trình Viên

## Claude Code là gì?

Claude Code là CLI tool của Anthropic, cho phép Claude (AI) **trực tiếp đọc file, chạy terminal, sửa code** trong project của bạn. Khác với ChatGPT (chỉ chat), Claude Code hành động như một developer thực sự ngồi làm việc cùng bạn trong terminal.

---

## Cài đặt

```bash
# Yêu cầu: Node.js 18+
npm install -g @anthropic-ai/claude-code

# Đặt API key
export ANTHROPIC_API_KEY="sk-ant-..."

# Chạy trong project
cd my-project
claude
```

API key lấy tại: https://console.anthropic.com

### Chi phí
- Dùng theo API (pay-as-you-go)
- Model mặc định: Claude Sonnet (~$3/1M input tokens, $15/1M output tokens)
- Một session trung bình: $0.10 - $0.50 tùy độ phức tạp

---

## Các lệnh quan trọng

```bash
claude                    # Khởi động interactive mode
claude "fix the bug in auth.js"   # Chạy task cụ thể
claude --continue         # Tiếp tục session trước
/help                     # Xem các lệnh trong session
/clear                    # Xóa context hiện tại
/compact                  # Nén context để tiết kiệm token
/model                    # Đổi model (Sonnet/Opus/Haiku)
```

---

## CLAUDE.md — File cấu hình project

Tạo file `CLAUDE.md` ở root project để Claude hiểu context:

```markdown
# Project: E-commerce API

## Tech stack
- Java 17, Spring Boot 3.x
- PostgreSQL, Redis
- Docker

## Quy tắc code
- Tất cả entity phải có @CreatedAt, @UpdatedAt
- Service layer không được gọi trực tiếp Repository từ Controller
- Test coverage tối thiểu 80%

## Lệnh hay dùng
- Build: ./mvnw clean package
- Test: ./mvnw test
- Run: docker-compose up
```

Claude sẽ đọc file này **mỗi khi bắt đầu session**, giúp không phải giải thích lại từ đầu.

---

## Workflow thực tế

### Debug bug
```
Bạn: "Có lỗi NullPointerException ở UserService.findByEmail(), giúp tôi fix"
Claude Code: [đọc UserService.java] [đọc UserRepository.java] [xem stack trace] → đề xuất fix → sửa trực tiếp
```

### Refactor
```
Bạn: "Refactor class OrderService, quá dài, tách thành các class nhỏ hơn"
Claude Code: [đọc toàn bộ file] [đề xuất cấu trúc] [tạo file mới] [cập nhật imports]
```

### Viết test
```
Bạn: "Viết unit test cho PaymentService"
Claude Code: [đọc PaymentService] [xem test hiện có] [tạo test file với các case đầy đủ]
```

---

## Tính năng nổi bật

### Context window lớn
Claude Sonnet có 200K token context — có thể đọc và hiểu cả codebase lớn trong một lần.

### Multi-file editing
Có thể chỉnh sửa nhiều file cùng lúc, tự động cập nhật imports, references.

### Terminal access
Chạy được lệnh shell: build, test, git, docker,... và phân tích output.

### Vision
Đọc được screenshot, diagram để debug UI hoặc hiểu requirement.

---

## Limitations (Hạn chế)

- **Không có memory** giữa các session (trừ CLAUDE.md)
- **Có thể sai** với logic phức tạp hoặc domain đặc thù — luôn review output
- **Tốn token** khi codebase lớn và prompt nhiều vòng
- **Không thay thế** được hiểu biết sâu về architecture
- **Rate limit** khi dùng nhiều liên tục
