# Câu Hỏi Phỏng Vấn Về Việc Dùng AI Trong Lập Trình

## Tại sao nhà tuyển dụng hỏi về AI?

Năm 2025-2026, hầu hết công ty kỳ vọng developer biết dùng AI để tăng năng suất. Họ muốn biết:
1. Bạn có biết tận dụng AI để làm việc nhanh hơn không?
2. Bạn có đủ critical thinking để không bị phụ thuộc mù quáng không?
3. Bạn có hiểu rủi ro và biết quản lý chất lượng code không?

---

## Câu hỏi thường gặp và cách trả lời

---

### "Bạn dùng AI tool nào trong công việc hàng ngày?"

**Điều họ muốn biết:** Bạn có cập nhật với công nghệ không, và bạn dùng có mục đích không.

**Trả lời mẫu:**
> "Tôi dùng kết hợp nhiều tool tùy context. GitHub Copilot để autocomplete khi đang code, tiết kiệm thời gian gõ boilerplate. Với task phức tạp hơn như refactor hoặc debug khó, tôi dùng Claude Code vì nó có thể đọc nhiều file cùng lúc và hiểu context rộng hơn. Đôi khi tôi dùng ChatGPT hoặc Claude để brainstorm architecture hoặc explain concept mới."

**Lưu ý:** Đừng chỉ nói "ChatGPT" — nó nghe có vẻ không cập nhật và chung chung.

---

### "Làm sao bạn đảm bảo chất lượng khi dùng AI?"

**Điều họ muốn biết:** Bạn có bị phụ thuộc AI mù quáng không, bạn có process kiểm soát không.

**Trả lời mẫu:**
> "Tôi coi AI như junior developer viết code draft, và tôi là senior review lại. Quy trình của tôi là: AI đề xuất → tôi đọc hiểu từng dòng → chạy test → nếu không hiểu tại sao code đó hoạt động thì không commit. Với code liên quan security, tôi luôn tự viết hoặc review kỹ hơn vì AI hay bỏ qua edge case. Tôi cũng thỉnh thoảng verify thông tin bằng docs chính thức vì AI có thể hallucinate tên method hoặc API."

---

### "Bạn dùng AI như thế nào khi debug?"

**Trả lời mẫu:**
> "Khi gặp bug, tôi thường thử tự debug trước 10-15 phút. Nếu vẫn không ra, tôi paste stack trace + relevant code vào Claude với đủ context: 'Lỗi xảy ra ở đây, tôi đã kiểm tra X và Y rồi, nghi ngờ nguyên nhân là Z'. Điều này giúp AI đưa ra diagnosis chính xác hơn là hỏi chung chung. Tôi cũng học được từ giải thích của AI để lần sau tránh lỗi tương tự."

---

### "Dùng AI có làm bạn kém đi không?"

**Điều họ muốn biết:** Bạn có nhận thức được rủi ro này không, và bạn có chiến lược gì không.

**Trả lời mẫu:**
> "Đây là risk thực sự nếu dùng sai cách. Tôi chủ động tránh bằng cách: không dùng AI cho những thứ tôi muốn học sâu, luôn đọc và hiểu code AI tạo thay vì copy-paste, và định kỳ làm bài tập kỹ thuật không dùng AI để giữ sharp. Quan điểm của tôi là AI giỏi ở execution, nhưng tôi cần mạnh ở tư duy — thiết kế, architecture, tradeoff. Những thứ đó AI không thay thế được."

---

### "Bạn xử lý vấn đề bảo mật khi dùng AI như thế nào?"

**Trả lời mẫu:**
> "Tôi có quy tắc rõ ràng: không paste API key, password, customer data, hoặc thông tin nhạy cảm lên AI cloud. Nếu cần show ví dụ data, tôi anonymize trước. Với code security-critical như authentication, authorization, tôi viết tay và review kỹ hơn, vì AI hay bỏ qua timing attack hay improper validation. Tôi cũng check chính sách của công ty về việc dùng AI tool trước khi dùng."

---

### "Bạn có ví dụ cụ thể về lúc AI giúp bạn giải quyết vấn đề khó không?"

**Chuẩn bị 1-2 story thực tế. Cấu trúc: Tình huống → Dùng AI thế nào → Kết quả**

**Ví dụ mẫu:**
> "Một lần tôi cần refactor một class 2000 dòng trong Java, rất legacy, không có test. Tôi dùng Claude Code để đọc toàn bộ file, hiểu dependency, rồi đề xuất cách tách thành 4 class nhỏ hơn theo Single Responsibility. Claude tạo ra plan, tôi review và điều chỉnh một số chỗ không phù hợp với business rule, sau đó Claude thực hiện refactor và update imports. Công việc mà ước tính mất 2 ngày hoàn thành trong 4 tiếng. Sau đó tôi mất thêm 2 tiếng để viết test và review kỹ."

---

### "AI có thể thay thế developer không?"

**Trả lời mẫu:**
> "Trong tương lai gần, không. AI rất giỏi ở code execution nhưng kém ở những thứ quan trọng nhất: hiểu business context, quyết định architecture, quản lý tradeoff, làm việc với stakeholder, và xử lý uncertainty. Developer giỏi dùng AI để tăng tốc phần execution, để dành brain power cho những thứ AI không làm được. Trend hiện tại là một developer dùng AI giỏi có thể làm được việc của 2-3 developer không dùng AI — đó là opportunity, không phải threat."

---

### "Khi team có policy hạn chế dùng AI, bạn xử lý thế nào?"

**Trả lời mẫu:**
> "Tôi tôn trọng policy của công ty. Nếu chính sách cấm paste code lên AI bên ngoài, tôi sẽ dùng trong giới hạn cho phép — ví dụ chỉ hỏi về public APIs, generic patterns, không paste proprietary code. Nếu công ty có on-premise AI solution, tôi sẽ dùng đó. Quan trọng hơn, tôi không phụ thuộc vào AI đến mức không làm được việc nếu không có nó."

---

## Câu hỏi bạn nên hỏi lại nhà tuyển dụng

Hỏi ngược lại thể hiện bạn nghiêm túc và thực tế:

- "Team hiện tại dùng AI tool gì? Có policy cụ thể về cách dùng không?"
- "AI-generated code có được review theo quy trình riêng không?"
- "Công ty có invest vào AI tooling cho developer không?"
- "Tiêu chí đánh giá productivity của developer có tính đến việc dùng AI không?"

---

## Red flags cần tránh khi trả lời

- "Tôi dùng ChatGPT để viết tất cả code" — nghe như không review gì
- "Tôi không dùng AI vì không tin tưởng" — nghe như lạc hậu
- "AI có thể làm mọi thứ" — thiếu critical thinking
- Không có ví dụ cụ thể — nghe lý thuyết, không thực tế
- Không biết hạn chế của tool mình dùng — không hiểu sâu

---

## Tóm tắt: Positioning lý tưởng

> **"Tôi dùng AI như một công cụ mạnh trong toolbox, không phải cái gậy chống. Tôi biết khi nào dùng, khi nào không, và luôn chịu trách nhiệm về quality của output."**
