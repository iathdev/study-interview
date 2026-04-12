# Lưu Ý Khi Dùng AI Trong Lập Trình

## 1. Luôn review output của AI

AI có thể sai. Đặc biệt với:
- **Logic phức tạp** — AI có thể bỏ qua edge case
- **Business rule đặc thù** — AI không biết context của bạn
- **Code deprecated** — training data có thể lỗi thời
- **Security** — AI đôi khi tạo code có SQL injection, XSS, insecure random

**Quy tắc**: Không commit code AI tạo mà chưa đọc hiểu từng dòng.

---

## 2. Bảo mật thông tin

**KHÔNG bao giờ paste vào AI (trừ private/local model):**
- API keys, passwords, tokens
- Customer data, PII (email, số điện thoại, CCCD)
- Nội dung hợp đồng, thông tin kinh doanh nhạy cảm
- Internal architecture docs của công ty

**Thay thế**: Anonymize data trước khi hỏi
```
# Thay vì paste real data
{"email": "nguyen@company.com", "secret": "abc123"}

# Dùng placeholder
{"email": "user@example.com", "secret": "REDACTED"}
```

---

## 3. Hiểu code trước khi dùng

AI có thể tạo code chạy được nhưng bạn không hiểu. Điều này nguy hiểm vì:
- Không debug được khi production lỗi
- Không maintain được sau 6 tháng
- Phỏng vấn kỹ thuật sẽ fail nếu không giải thích được

**Quy tắc**: Nếu không giải thích được code cho đồng nghiệp, đừng merge.

---

## 4. Hallucination — AI bịa thông tin

AI có thể tự tin nói sai:
- Method/API không tồn tại
- Package version không có
- "Best practice" thực ra là anti-pattern

**Cách kiểm tra:**
```bash
# Luôn verify tên method/class
grep -r "methodName" src/
# Kiểm tra package tồn tại không
npm info package-name
# Đọc docs chính thức thay vì tin AI 100%
```

---

## 5. AI không thay thế kiến trúc tốt

AI giỏi ở **tactical** (viết function, fix bug) nhưng kém ở **strategic**:
- Quyết định microservices vs monolith
- Database schema design cho scale
- Trade-off về consistency vs availability
- Khi nào nên cache, khi nào không

Dùng AI như công cụ **thực thi** ý tưởng của bạn, không phải để **thay thế** tư duy kiến trúc.

---

## 6. Vấn đề với code quality

AI thường:
- Tạo code dài hơn cần thiết
- Không tận dụng utility có sẵn trong project
- Thiếu error handling edge case
- Đặt tên biến chung chung (`data`, `result`, `temp`)
- Thêm comment thừa không cần thiết

**Sau khi AI viết xong, hỏi thêm:**
```
"Code này có thể đơn giản hóa không?
Có utility nào trong project đã làm việc này chưa?"
```

---

## 7. Phụ thuộc quá mức vào AI

**Dấu hiệu nguy hiểm:**
- Không thể debug nếu không có AI
- Không biết tại sao code lại hoạt động
- Copy-paste từ AI mà không đọc
- Năng lực kỹ thuật không tăng theo thời gian

**Cách dùng AI mà vẫn giỏi lên:**
- Dùng AI để học, không chỉ để làm
- Tự thử trước, AI hỗ trợ sau khi stuck
- Đọc giải thích của AI để hiểu pattern
- Định kỳ làm bài tập không dùng AI

---

## 8. Vấn đề về bản quyền

- Code AI tạo ra có thể dựa trên code có bản quyền trong training data
- Một số công ty cấm dùng AI tool cho code proprietary
- Luôn kiểm tra chính sách công ty trước khi dùng

---

## 9. Khi nào KHÔNG nên dùng AI

| Tình huống | Lý do |
|-----------|-------|
| Học concept mới lần đầu | Mất cơ hội hiểu sâu |
| Thuật toán thi đấu (competitive programming) | Không phát triển kỹ năng |
| Security-critical code | Rủi ro cao, cần expert review |
| Task chỉ mất 2 phút | Overhead prompting không đáng |
| Code có NDA strict | Rủi ro leak thông tin |

---

## 10. Checklist trước khi commit AI-generated code

- [ ] Tôi đã đọc hiểu từng dòng code
- [ ] Không có hardcoded secret nào
- [ ] Test case đã chạy pass
- [ ] Không có package lạ được thêm vào
- [ ] Error handling đầy đủ
- [ ] Code tuân theo convention của project
- [ ] Tôi có thể giải thích code này cho đồng nghiệp
