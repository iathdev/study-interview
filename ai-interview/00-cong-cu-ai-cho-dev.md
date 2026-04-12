# Công Cụ AI Cho Lập Trình Viên

## Phân loại công cụ AI

### 1. Code Completion (Tự động hoàn thành code)
- **GitHub Copilot** — tích hợp trực tiếp vào IDE, gợi ý theo context
- **Tabnine** — tự host được, phù hợp enterprise bảo mật cao
- **Codeium** — miễn phí, hỗ trợ nhiều ngôn ngữ

### 2. Chat-based AI (Hỏi đáp)
- **ChatGPT (GPT-4o)** — tổng quát, giải thích code, viết docs
- **Claude (claude.ai)** — lý luận sâu, context window lớn, an toàn hơn
- **Gemini** — tích hợp Google ecosystem

### 3. AI Code Agent (Tự thực hiện nhiệm vụ)
- **Claude Code** — CLI agent của Anthropic, thao tác file/terminal
- **Cursor** — IDE fork từ VSCode, tích hợp AI sâu
- **Windsurf (Codeium)** — tương tự Cursor
- **Aider** — CLI agent mã nguồn mở
- **Devin** — agent tự động hoàn toàn (đắt tiền)

### 4. Specialized Tools
- **v0.dev** — sinh UI component (React/Tailwind)
- **Bolt.new** — fullstack app từ prompt
- **Perplexity** — tìm kiếm + AI reasoning

---

## So sánh nhanh

| Tool | Miễn phí | IDE Integration | Agent | Tốt nhất cho |
|------|----------|-----------------|-------|---------------|
| GitHub Copilot | Giới hạn | VSCode, JetBrains | Không | Code completion daily |
| Claude Code | API pay-as-go | Terminal | Có | Refactor, debug lớn |
| Cursor | Có (giới hạn) | Built-in IDE | Có | Viết code nhanh |
| ChatGPT | Có | Plugin | Không | Giải thích, brainstorm |
| Tabnine | Có | Nhiều IDE | Không | Enterprise, offline |

---

## Khi nào dùng gì

**Viết feature mới nhanh** → Cursor hoặc Claude Code
**Review code, explain bug** → ChatGPT hoặc Claude
**Autocomplete khi typing** → Copilot hoặc Codeium
**Refactor codebase lớn** → Claude Code
**Sinh UI component** → v0.dev
**Tìm hiểu thư viện mới** → Perplexity hoặc Claude
