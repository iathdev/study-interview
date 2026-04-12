# Interview Q&A — What I Did, Challenges & Lessons

> Focus: Kinh nghiệm thực tế, khó khăn, thử thách, cách giải quyết.
> Gợi ý trả lời theo format STAR (Situation, Task, Action, Result) khi phù hợp.

---

## I. PREP Technology — B2B English Learning & Examination Platform

---

### Q1: Mô tả tổng quan hệ thống B2B English learning platform mà bạn xây dựng.

**Gợi ý trả lời:**

Hệ thống gồm 2 domain chính được tách riêng:
- **Management domain (Laravel):** quản lý đề thi, user, tổ chức, báo cáo — traffic thấp, logic CRUD phức tạp, phù hợp với Laravel ecosystem (Eloquent, Queue, Nova...).
- **Exam domain (Go):** phục vụ candidate làm bài thi online — yêu cầu low latency, high concurrency, real-time processing.

Hai domain giao tiếp qua **event-driven architecture** — management publish events (exam created, exam started) và exam domain subscribe để xử lý.

Ngoài ra có **Learning Profile service** dùng Kafka event streams để aggregate hoạt động học tập từ nhiều services.

---

### Q2: Tại sao chọn tách Laravel (management) và Go (exam) thay vì dùng 1 stack duy nhất?

**Gợi ý trả lời:**

- **Management domain:** logic nghiệp vụ phức tạp (CRUD đề thi, quản lý tổ chức, phân quyền) — Laravel mạnh ở rapid development, ORM, ecosystem sẵn có. Traffic thấp, không cần optimize performance cực độ.
- **Exam domain:** hàng nghìn candidate thi cùng lúc, mỗi candidate gửi events liên tục (tab switch, focus loss, submission) — cần **low latency, high throughput**. Go có goroutine model nhẹ, compile thành binary, memory footprint nhỏ, phù hợp cho concurrent workload.
- **Thử thách:** phải thiết kế rõ ràng service boundary và communication protocol giữa 2 stack khác nhau. Team cũng cần biết cả 2 ngôn ngữ.

---

### Q3: Anti-cheating service hoạt động như thế nào? Khó khăn lớn nhất khi build?

**Gợi ý trả lời:**

**Cách hoạt động:**
- Client-side SDK capture các events: tab switch, window focus loss, copy-paste, rapid submission, idle detection.
- Events được gửi lên Go service với tần suất cao (high-frequency).
- Go service dùng **Redis làm event buffer** — gom events theo candidate session, tính toán pattern trong sliding window.
- Khi detect suspicious behavior (vd: chuyển tab >3 lần trong 1 phút, submit quá nhanh), hệ thống flag candidate và notify proctor.

**Khó khăn:**
1. **False positive:** Candidate bị flag oan vì lý do hợp lệ (vd: notification popup, Alt+Tab vô tình). Phải tune threshold nhiều lần, dùng **combination of signals** thay vì single event.
2. **High-frequency events gây pressure lên Redis:** Hàng nghìn candidate x nhiều events/giây. Phải design buffering strategy — batch write vào Redis, dùng pipeline commands, set TTL hợp lý để tự cleanup.
3. **Real-time vs accuracy trade-off:** Detect nhanh nhưng phải đủ chính xác. Giải quyết bằng 2 layers: real-time alerting (loose threshold) cho proctor xem, và post-exam analysis (strict threshold) cho final decision.

---

### Q4: Tại sao chọn Redis làm event buffer thay vì ghi thẳng vào Kafka hoặc database?

**Gợi ý trả lời:**

- **Latency:** Redis in-memory, write ~1ms. Kafka write tuy nhanh nhưng có overhead của batching, ack, replication (~5-10ms). Với high-frequency events cần real-time detection, Redis phù hợp hơn cho buffering layer.
- **Aggregation tại chỗ:** Redis Sorted Set / Hash cho phép aggregate events per candidate ngay tại buffer layer (đếm số lần tab switch trong window), không cần pull data ra rồi tính.
- **Ephemeral data:** Anti-cheat events chỉ cần giữ trong thời gian thi. Redis TTL tự cleanup, không cần maintenance. Database/Kafka thì data persist lâu dài không cần thiết cho use case này.
- **Trade-off chấp nhận:** Nếu Redis down, mất buffer data. Giải quyết bằng: Redis Sentinel cho HA, và fallback ghi raw events vào Kafka để post-analysis.

---

### Q5: Event-driven architecture giữa management và exam domain — cụ thể flow thế nào? Khó khăn gì?

**Gợi ý trả lời:**

**Flow ví dụ — tạo kỳ thi:**
1. Admin tạo exam trên management (Laravel) → lưu DB → publish event `exam.created` chứa exam config, questions, time limit.
2. Exam domain (Go) subscribe → lưu local copy của exam data → sẵn sàng serve candidate.
3. Admin start exam → publish `exam.started` → Exam domain open access cho candidate.
4. Candidate submit → Exam domain publish `submission.completed` → Management domain nhận để scoring, reporting.

**Khó khăn:**
1. **Data consistency:** Exam data phải sync giữa 2 domains. Nếu event bị mất hoặc delay, candidate vào thi mà exam chưa có data. Giải quyết bằng **idempotent consumers + retry mechanism + health check endpoint** để verify data sync trước khi open exam.
2. **Event ordering:** Nếu `exam.updated` đến trước `exam.created` (race condition khi dùng multiple partitions) → data bị corrupt. Giải quyết bằng **partition key = exam_id** để đảm bảo ordering per exam.
3. **Debugging khó:** Khi có bug, phải trace event flow across services. Thiết lập **correlation ID** gắn theo mỗi event chain để trace end-to-end.

---

### Q6: Asynchronous scoring pipeline — tại sao cần async? Thiết kế thế nào?

**Gợi ý trả lời:**

**Tại sao async:**
- Khi hàng nghìn candidate submit cùng lúc (cuối giờ thi), nếu scoring synchronous → database overload, response timeout, candidate không biết submit thành công hay chưa.
- Tách scoring ra async: candidate submit → nhận ACK ngay lập tức → background process scoring.

**Pipeline flow:**
1. Candidate submit → Exam service validate + lưu raw answer → publish `submission.received` event → return 200 OK cho candidate.
2. Scoring worker consume event → chấm điểm (match answer key, tính điểm partial...) → lưu result → publish `scoring.completed`.
3. Analytics worker consume → aggregate statistics (điểm TB, phân bố, thời gian làm bài...) → lưu vào analytics DB.
4. Notification worker → gửi email/notification cho candidate và admin.

**Khó khăn:**
- **Đảm bảo exactly-once scoring:** Nếu worker crash giữa chừng rồi retry → candidate bị chấm 2 lần. Giải quyết bằng **idempotency key (submission_id)** — check trước khi chấm, nếu đã có result thì skip.
- **Latency expectation:** Admin muốn xem kết quả ngay sau khi thi kết thúc, nhưng scoring pipeline cần thời gian. Giải quyết bằng **progress tracking** — dashboard hiển thị % đã chấm, real-time update.

---

### Q7: Bạn contribute gì cho Learning Profile service? Thử thách chính là gì?

**Gợi ý trả lời:**

**Contribution:**
- Learning Profile service aggregate hoạt động học tập từ nhiều sources: exam results, lesson completion, practice exercises, time spent.
- Dùng **Kafka event streams** — mỗi service publish events (lesson.completed, exam.scored, exercise.submitted) → Learning Profile service consume và maintain learner state.
- Data model: mỗi learner có profile chứa progress per skill/topic, strengths, weaknesses, recent activities.

**Thử thách:**
1. **Event schema khác nhau từ nhiều services:** Mỗi service define event format riêng. Phải design **canonical event format** hoặc dùng adapter pattern để normalize trước khi aggregate.
2. **Eventual consistency:** Profile không reflect ngay khi learner hoàn thành activity. User complain "tôi vừa xong bài mà profile chưa update". Giải quyết bằng **optimistic UI update** ở client-side + backend eventual consistency.
3. **State rebuild:** Khi cần fix bug trong aggregation logic, phải replay events để rebuild profile. Thiết kế hệ thống có khả năng **replay từ Kafka** (retention policy đủ dài).

---

### Q8: Concurrent large-scale online assessment — cụ thể khó khăn gì về technical?

**Gợi ý trả lời:**

**Thử thách chính:**

1. **Thundering herd khi exam start:** 5000 candidate cùng load đề thi trong 1 giây. Giải quyết: **cache exam data ở Redis**, serve từ cache thay vì hit DB. Thêm **jitter** — client-side random delay 0-5s để spread load.

2. **Write spike khi exam end:** Tất cả submit trong vài phút cuối. Giải quyết: async submission pipeline (đã nói trên), **write buffer** — batch insert thay vì insert từng record.

3. **Connection limit:** Mỗi candidate maintain WebSocket/long-polling cho anti-cheat monitoring. Thousands concurrent connections. Go xử lý tốt nhờ goroutine, nhưng phải tune **file descriptor limits, TCP keepalive, connection pooling**.

4. **Exam integrity khi network unstable:** Candidate mất mạng giữa chừng → reconnect → phải restore đúng state (đã trả lời câu nào, còn bao nhiêu thời gian). Giải quyết: **server-side state management** — mọi answer được persist khi submit từng câu (auto-save), timer tính server-side không client-side.

---

## II. GHTK — GAM & HRM Systems

---

### Q9: Bạn áp dụng DDD và Hexagonal Architecture tại GHTK thế nào? Tại sao cần restructure?

**Gợi ý trả lời:**

**Situation:** Codebase hiện tại là monolith, business logic nằm rải rác trong controllers và services, high coupling giữa các modules (HRM gọi thẳng vào internal của GAM, shared database tables).

**Problem gây ra:**
- Thay đổi 1 module → break module khác (regression).
- Không test được business logic riêng lẻ — phải setup toàn bộ infrastructure.
- Onboard developer mới rất khó hiểu flow.

**Action:**
- Áp dụng **DDD** để xác định bounded contexts: Working Schedule, Leave Management, Employee, Authorization...
- Mỗi bounded context có riêng Entity, Value Object, Repository interface, Domain Service.
- Áp dụng **Hexagonal Architecture**: business logic ở core, không depend vào framework hay infrastructure.
  - **Ports (interfaces):** ScheduleRepositoryPort, NotificationPort...
  - **Adapters:** JpaScheduleRepository, KafkaNotificationAdapter...
- Controller → Application Service → Domain Service → Port → Adapter.

**Result:**
- Module có thể develop và test independently.
- Giảm regression khi thay đổi.
- Swap infrastructure dễ dàng (vd: đổi notification từ email sang Kafka chỉ cần thay adapter).

**Khó khăn:**
- **Team buy-in:** Không phải ai cũng quen DDD terminology. Phải tổ chức knowledge sharing, pair programming.
- **Over-engineering risk:** Không phải mọi module đều cần full DDD. Modules đơn giản (CRUD config) vẫn giữ simple.
- **Migration dần dần:** Không thể rewrite toàn bộ. Áp dụng **Strangler Fig pattern** — module mới viết theo DDD, module cũ refactor dần.

---

### Q10: Production deadlocks tại GHTK — root cause và cách giải quyết?

**Gợi ý trả lời:**

**Situation:** HRM system có tính năng duyệt đơn nghỉ phép. Khi nhiều manager approve đồng thời, hoặc khi batch job chạy song song với user request → deadlock.

**Root cause analysis:**
- Transaction A: lock row employee → update leave balance → lock row leave_request → update status.
- Transaction B: lock row leave_request → update status → lock row employee → update leave balance.
- **Circular lock dependency** → deadlock.

**Cách detect:**
- MySQL `SHOW ENGINE INNODB STATUS` → xem deadlock log.
- Thêm monitoring: alert khi số deadlock/phút vượt threshold.
- Application log: catch `DeadlockLoserDataAccessException` (Spring) và log context.

**Giải quyết:**
1. **Consistent lock ordering:** Luôn lock employee trước, leave_request sau — tất cả transactions follow cùng thứ tự → phá vỡ circular dependency.
2. **Reduce lock scope:** Tách transaction lớn thành nhiều transaction nhỏ hơn. Chỉ lock khi thực sự cần write.
3. **Optimistic locking** cho một số case: dùng version column, retry khi conflict thay vì giữ lock lâu.
4. **Retry mechanism:** Khi deadlock xảy ra, application tự retry (max 3 lần với backoff).

**Result:** Deadlock giảm ~95%. Các case còn lại được handle bởi retry mechanism, user không bị ảnh hưởng.

---

### Q11: Query optimization tại GHTK — ví dụ cụ thể?

**Gợi ý trả lời:**

**Ví dụ 1: Working Schedule query chậm**

**Situation:** API lấy lịch làm việc của tất cả nhân viên trong 1 phòng ban, 1 tháng. Với phòng ban 200 người, query mất 3-5 giây.

**Investigation:**
- `EXPLAIN ANALYZE` → full table scan trên bảng `schedules` (millions records).
- N+1 query: load department → loop qua từng employee → query schedule per employee.

**Action:**
1. Thêm **composite index** `(department_id, employee_id, date)` — match đúng pattern query.
2. Rewrite thành **single query** với JOIN thay vì N+1.
3. Thêm **Redis cache** cho schedule data (invalidate khi có thay đổi).

**Result:** Query giảm từ 3-5s xuống <100ms.

**Ví dụ 2: Report query gây slow toàn bộ system**

**Situation:** Admin chạy báo cáo tổng hợp (aggregate nhiều bảng) → query chạy 30s → lock tables → các request khác bị chậm theo.

**Action:**
1. Chuyển report queries sang **read replica** — không ảnh hưởng primary.
2. Tối ưu query: dùng **materialized view / summary table** được update bởi background job thay vì aggregate real-time.
3. Thêm **query timeout** để prevent runaway queries.

---

### Q12: OAuth2 và IAM service bạn implement thế nào? Khó khăn gì?

**Gợi ý trả lời:**

**Scope:** Internal enterprise platforms (GAM, HRM, và các internal tools khác) cần single sign-on và centralized authorization.

**Implementation:**
- **OAuth2 Authorization Code flow** cho web apps (employee login).
- **Client Credentials flow** cho service-to-service communication.
- IAM service quản lý: users, roles, permissions, resource-based policies.
- JWT access token chứa user info + roles → services validate locally, không cần call IAM mỗi request.

**Khó khăn:**
1. **Permission model phức tạp:** GAM cần permission theo resource (project A, project B), HRM cần permission theo org hierarchy (manager chỉ xem nhân viên của mình). Phải design flexible permission model — kết hợp RBAC + resource-based.
2. **Token invalidation:** JWT stateless nên khó revoke. Giải quyết bằng **short-lived access token (15 min) + refresh token rotation + blacklist** cho trường hợp cần revoke ngay.
3. **Migration:** Các hệ thống cũ dùng session-based auth riêng lẻ. Phải migrate dần — support cả 2 cơ chế trong transition period.

---

### Q13: Kafka và Redis tại GHTK dùng cho use case gì cụ thể?

**Gợi ý trả lời:**

**Kafka:**
- **Event-driven communication giữa GAM và HRM:** Employee thay đổi phòng ban (GAM event) → HRM update working schedule, leave quota.
- **Audit log:** Mọi thay đổi quan trọng publish event → audit service consume và lưu trữ.
- **Notification pipeline:** Approve/reject request → publish event → notification service gửi email, push notification.

**Redis:**
- **Caching:** User sessions, permission data (tránh query IAM mỗi request), frequently accessed config.
- **Distributed lock:** Prevent duplicate processing khi multiple instances handle cùng 1 request (vd: approve leave request).
- **Rate limiting:** Protect APIs khỏi abuse — dùng Redis INCR + EXPIRE cho sliding window counter.

**Khó khăn:**
- **Kafka consumer ordering:** Events từ GAM đến HRM phải đúng thứ tự (transfer department trước, rồi mới update schedule). Partition key = employee_id đảm bảo ordering per employee.
- **Cache invalidation:** Permission data cached ở Redis, nhưng khi admin thay đổi role → phải invalidate ngay. Dùng **pub/sub** để broadcast invalidation signal tới tất cả instances.

---

### Q14: High-concurrency workloads tại GHTK — traffic level và cách handle?

**Gợi ý trả lời:**

**Context:** GHTK là công ty logistics lớn, hàng nghìn nhân viên dùng internal tools đồng thời. Peak traffic vào đầu tháng (chấm công, duyệt lương), đầu tuần (phân ca), cuối năm (tổng kết).

**Challenges:**
1. **Chấm công peak:** Sáng 8h, hàng nghìn nhân viên check-in cùng lúc → write spike vào attendance table. Giải quyết: **batch insert + message queue buffer** — check-in request vào queue, worker batch insert mỗi 1-2 giây.
2. **Report generation:** Nhiều manager chạy report cùng lúc → DB CPU spike. Giải quyết: **pre-computed reports** bởi background job, cache kết quả, **read replica** cho report queries.
3. **API response time degradation:** Dưới load cao, response time tăng. Giải quyết: **connection pool tuning** (HikariCP), **query optimization**, **circuit breaker** cho downstream services.

---

## III. Tomosia — E-commerce, Entertainment, Digital Transformation

---

### Q15: Data collection pipeline dùng AWS Lambda + SQS + OpenSearch + Puppeteer — mô tả chi tiết?

**Gợi ý trả lời:**

**Use case:** Thu thập dữ liệu sản phẩm từ nhiều e-commerce sources, index vào OpenSearch để phục vụ search và analytics.

**Architecture:**
1. **Scheduler (CloudWatch Events)** trigger Lambda function định kỳ.
2. **Lambda (Puppeteer)** → headless browser → navigate tới target pages → scrape data (tên, giá, mô tả, hình ảnh...).
3. Scraped data → push vào **SQS queue** (decouple scraping và processing).
4. **Processing Lambda** consume SQS → clean data, normalize format, enrich metadata.
5. Processed data → index vào **OpenSearch** → phục vụ search API.

**Khó khăn:**
1. **Lambda cold start + Puppeteer heavy:** Puppeteer (headless Chrome) chiếm ~300MB+ memory, Lambda cold start lên đến 5-10 giây. Giải quyết: **provisioned concurrency** cho critical functions, optimize Puppeteer binary size (dùng chromium minimal build).
2. **Dynamic content:** Nhiều trang load data bằng JavaScript, cần wait cho DOM render xong. Phải implement **smart waiting strategy** — wait for specific selectors thay vì fixed timeout.
3. **Rate limiting bởi target sites:** Scrape quá nhanh bị block. Giải quyết: **SQS delay queue** để space out requests, rotate User-Agent, respect robots.txt.
4. **Data quality:** Dữ liệu scrape không consistent (missing fields, wrong format). Implement **validation layer** trước khi index, alert khi error rate cao.
5. **OpenSearch indexing performance:** Bulk indexing millions documents. Dùng **bulk API**, tune refresh interval, optimize mapping (keyword vs text fields).

---

### Q16: Amusement park digital transformation — ticket management system thế nào?

**Gợi ý trả lời:**

**Scope:** Platform số hóa cho công viên giải trí — bán vé online, quản lý cổng vào, thống kê lượng khách.

**Chức năng chính:**
- Bán vé online (nhiều loại: vé ngày, vé combo, vé nhóm).
- QR code check-in tại cổng.
- Dashboard real-time hiển thị lượng khách, doanh thu, công suất.
- Quản lý operational (nhân viên, thiết bị, sự kiện).

**Technical decisions:**
- Deploy trên **AWS** (EC2, RDS, S3, CloudFront).
- Backend: Laravel (phù hợp rapid development cho project timeline).
- QR code generation và validation.

**Khó khăn:**
1. **Peak traffic vào ngày lễ/cuối tuần:** Lượng mua vé tăng đột biến. Giải quyết: **auto-scaling group** trên AWS, **CloudFront CDN** cho static assets, **Redis cache** cho ticket inventory.
2. **Concurrency khi bán vé giới hạn:** 100 vé VIP, 500 người cùng mua. Giải quyết: **optimistic locking + Redis atomic decrement** cho inventory count — tránh overselling.
3. **Offline scenario:** Khu vui chơi có thể mất internet → cổng check-in phải vẫn hoạt động. Giải quyết: **local cache** danh sách vé hợp lệ trên thiết bị check-in, sync khi có mạng.
4. **QR code security:** Prevent fake QR / reuse QR. Mỗi QR chứa signed token (HMAC), server validate + mark as used atomically.

---

### Q17: Bạn lead backend development ở Tomosia — kinh nghiệm leading?

**Gợi ý trả lời:**

**Scope:** Lead backend cho nhiều projects, coordinate với frontend team và PM.

**Responsibilities:**
- Technical decision making: chọn architecture, tech stack, database design.
- Code review và đảm bảo code quality.
- Task breakdown và estimation.
- Mentoring junior developers.

**Khó khăn:**
1. **Multiple projects cùng lúc:** Phải context switch giữa e-commerce, entertainment, data pipeline. Giải quyết: documentation rõ ràng cho mỗi project, standardize tech stack và coding conventions across projects.
2. **Junior team members:** Nhiều members mới, code quality không đồng đều. Giải quyết: establish code review process, viết coding guidelines, pair programming sessions.
3. **Client requirement thay đổi liên tục:** Scope creep. Giải quyết: push back hợp lý, document change requests, estimate impact trước khi accept.

---

### Q18: CloudWatch monitoring và database optimization ở production — cụ thể?

**Gợi ý trả lời:**

**CloudWatch setup:**
- **Custom metrics:** API response time (p50, p95, p99), error rate, queue depth.
- **Alarms:** Response time p95 > 1s, error rate > 1%, CPU > 80%, memory > 85%.
- **Dashboard:** Real-time view cho từng service — request count, latency distribution, error breakdown.
- **Log Insights:** Query structured logs để investigate issues.

**Database optimization:**
1. **Slow query identification:** Enable slow query log (>500ms), review weekly.
2. **Index optimization:** Analyze query patterns → add missing indexes, remove unused indexes (index cũng có write cost).
3. **Connection pool tuning:** Monitor active connections, adjust pool size based on load pattern.
4. **Query rewrite:** Replace subqueries với JOINs khi phù hợp, avoid SELECT *, sử dụng pagination cho large result sets.

**Khó khăn:**
- **Alert fatigue:** Ban đầu set threshold quá sensitive → quá nhiều false alerts → team bắt đầu ignore. Phải tune threshold dựa trên historical data, phân biệt **warning vs critical**.
- **Production debugging:** Không thể reproduce locally. Phải rely on logs + metrics. Học cách đọc CloudWatch metrics correlation — CPU spike cùng lúc với slow queries → identify problematic query.

---

## IV. Cross-Company — Technical Challenges

---

### Q19: Khó khăn lớn nhất khi chuyển từ PHP sang Java/Go?

**Gợi ý trả lời:**

**PHP → Java:**
- **Mindset shift:** PHP là request-based (mỗi request fresh state), Java application sống lâu (stateful, memory management matters). Phải học cách nghĩ về memory leak, thread safety, connection pooling.
- **Complexity:** Spring ecosystem rất lớn — Spring Boot, Spring Security, Spring Data, AOP... Learning curve cao nhưng powerful.
- **Type system:** PHP loosely typed (dù có strict types), Java strict typing. Ban đầu thấy verbose nhưng sau thấy lợi ích cho maintainability.

**PHP → Go:**
- **Error handling:** Không có try-catch, phải check error ở mỗi bước. Ban đầu thấy verbose nhưng forces bạn handle errors explicitly.
- **No OOP traditional:** Go không có class, inheritance. Phải tư duy composition, interface implicitly satisfied.
- **Concurrency model:** Goroutine + channel khác hoàn toàn Thread model. Phải học cách tư duy concurrent programming đúng cách, tránh race conditions.

**Bài học:** Mỗi ngôn ngữ có philosophy riêng. Không nên viết Java style trong Go hay PHP style trong Java. Embrace idioms của từng ngôn ngữ.

---

### Q20: Kinh nghiệm với event-driven architecture — bài học rút ra sau nhiều projects?

**Gợi ý trả lời:**

**Đã dùng ở:** GHTK (Kafka giữa GAM-HRM), PREP (Laravel-Go communication, Learning Profile).

**Bài học:**

1. **Event design là quyết định quan trọng nhất.** Event schema rất khó thay đổi sau khi đã có consumers. Phải invest thời gian thiết kế event format, versioning strategy từ đầu.

2. **Debugging distributed event flows rất khó.** Luôn cần: correlation ID, structured logging, event tracing. Thiếu observability → mất hàng giờ debug.

3. **Eventual consistency gây confuse cho users.** Phải design UI cho phù hợp — show pending states, progress indicators, refresh mechanisms.

4. **Idempotency là bắt buộc, không phải nice-to-have.** Events có thể delivered nhiều lần (retry, rebalance). Consumer phải handle duplicate.

5. **Không phải mọi thứ đều cần event-driven.** Simple CRUD hoặc synchronous flow thì REST call đơn giản hơn. Event-driven cho decouple, async processing, event sourcing — dùng khi thực sự cần.

6. **Dead letter queue là lifesaver.** Events fail processing phải có nơi chứa để investigate, không được drop silently.

---

### Q21: Kinh nghiệm về database — bài học từ production incidents?

**Gợi ý trả lời:**

1. **Deadlock ở GHTK:** (đã mô tả Q10) → Bài học: luôn có consistent lock ordering, minimize transaction scope.

2. **Missing index gây outage:** Query chạy tốt lúc data nhỏ, nhưng table grow lên millions rows → full table scan → CPU 100% → toàn bộ system chậm. Bài học: **load test với production-like data volume**, monitor slow queries proactively.

3. **Migration gây downtime:** ALTER TABLE trên large table (add column) lock table hàng phút. Bài học: dùng **online schema change tools** (pt-online-schema-change, gh-ost), hoặc design migration thành multiple non-locking steps.

4. **Connection pool exhaustion:** Application hold connections quá lâu (long-running queries, forgot to close) → pool hết → new requests fail. Bài học: **set connection timeout, max lifetime**, monitor pool metrics.

5. **Read-after-write inconsistency:** Write vào primary, read từ replica ngay lập tức → data chưa replicate → user thấy data cũ. Bài học: critical reads phải hit primary, hoặc dùng **read-your-writes consistency** (sticky session to primary after write).

---

### Q22: Cách bạn approach system design cho một feature mới?

**Gợi ý trả lời:**

**Process thực tế:**

1. **Understand requirements deeply:** Clarify với PM/stakeholders. Phân biệt must-have vs nice-to-have. Hỏi về expected scale (users, data volume, growth rate).

2. **Identify constraints:** Deadline, team skill set, existing infrastructure, budget.

3. **High-level design:** Vẽ component diagram — services nào, communicate thế nào, data flow. Review với team.

4. **Data model first:** Database schema thường là quyết định khó thay đổi nhất. Design carefully, consider access patterns.

5. **Identify risks & unknowns:** Performance bottleneck ở đâu? Single point of failure? New technology team chưa dùng?

6. **Prototype risky parts:** Nếu có component chưa chắc chắn (vd: Redis event buffer cho anti-cheat), prototype và benchmark trước.

7. **Incremental delivery:** Không build toàn bộ rồi mới deliver. Chia thành milestones, mỗi milestone có giá trị riêng.

**Ví dụ thực tế:** Anti-cheating service ở PREP — prototype Redis buffering approach trước, benchmark với simulated load, confirm viable rồi mới design full system.

---

### Q23: Cách bạn handle technical debt?

**Gợi ý trả lời:**

**Tại Tomosia (4 năm):** Chứng kiến tech debt tích lũy qua nhiều projects. Bài học: tech debt giống nợ tài chính — một ít thì OK, quá nhiều thì trả lãi (development slow down) nhiều hơn benefit.

**Approach:**
1. **Track tech debt:** Tạo backlog item cho mỗi debt, ghi rõ impact và effort to fix.
2. **Boy Scout Rule:** Mỗi khi touch code, improve một chút (rename unclear variable, add missing test, fix small code smell). Không cần ticket riêng.
3. **Dedicate capacity:** Thuyết phục team dành ~15-20% sprint capacity cho tech debt. Justify bằng metrics: "query này chạy 5s, optimize xuống 200ms → user experience tốt hơn, giảm support tickets."
4. **Prioritize ruthlessly:** Không phải mọi debt đều cần trả. Focus vào debt gây pain thực sự (bugs, slow development, poor performance).

**Ví dụ tại GHTK:** DDD + Hexagonal Architecture restructure chính là cách trả tech debt có hệ thống — không rewrite from scratch mà refactor dần bằng Strangler Fig pattern.

---

## V. Architecture & Design Decisions

---

### Q24: So sánh kinh nghiệm với Monolith (Laravel) vs Microservices — khi nào chọn gì?

**Gợi ý trả lời:**

**Kinh nghiệm thực tế:**
- **Tomosia:** Chủ yếu monolith Laravel — phù hợp vì team nhỏ, project scope rõ ràng, deadline gấp.
- **GHTK:** Monolith đang transition sang modular monolith (DDD). Chưa full microservices vì team structure chưa match.
- **PREP:** Distributed services (Laravel + Go) — gần microservices, vì 2 domains có requirements rất khác nhau.

**Bài học:**
- **Monolith first:** Cho projects mới, team nhỏ, domain chưa rõ ràng. Premature microservices = distributed monolith = worst of both worlds.
- **Microservices khi:** Clear domain boundaries, independent scaling needs, different tech stacks justified, team đủ lớn để own services riêng.
- **Modular monolith:** Sweet spot giữa 2 cái — clean boundaries nhưng deploy đơn giản. Có thể extract thành microservice sau khi boundary đã proven.

**Khó khăn với distributed services ở PREP:**
- Network latency giữa services.
- Distributed debugging.
- Data consistency across services.
- Deployment coordination.

→ Trade-offs được accept vì Go exam service cần independent scaling và different performance characteristics.

---

### Q25: Caching strategy bạn đã implement — lessons learned?

**Gợi ý trả lời:**

**Các layers đã dùng:**

| Layer | Where | Use case |
|-------|-------|----------|
| CDN (CloudFront) | Tomosia | Static assets, images |
| Application cache (Redis) | GHTK, PREP | Session, permission, exam data, schedule |
| Query cache | GHTK | Report results, aggregated data |

**Lessons learned:**

1. **Cache invalidation is the hardest problem.** Stale cache gây bug khó debug nhất. Prefer **TTL-based** cho data có thể tolerate staleness, **event-based invalidation** cho critical data.

2. **Cache stampede thực tế:** Exam data cached với TTL 5 phút. TTL expire → 1000 concurrent requests hit DB cùng lúc. Fix: **lock-based cache refresh** — 1 request refresh cache, các request khác wait hoặc serve stale data.

3. **Cache warm-up quan trọng:** Khi deploy version mới, cache trống → DB load spike. Implement **cache warming** script chạy sau deploy.

4. **Monitor cache hit ratio:** Nếu hit ratio < 80%, cache không effective. Có thể do TTL quá ngắn, key design không tốt, hoặc data quá diverse.

5. **Don't cache everything:** Caching thêm complexity. Chỉ cache data đọc nhiều ghi ít, hoặc expensive to compute.

---

### Q26: Bạn đã design database schema thế nào cho các domain phức tạp?

**Gợi ý trả lời:**

**Ví dụ: Working Schedule (GHTK)**

**Requirements:**
- Nhân viên có lịch làm việc theo ca (sáng, chiều, tối, flexible).
- Mỗi phòng ban có pattern khác nhau.
- Support nghỉ phép, đổi ca, làm thêm giờ.
- Query: "ai làm ca sáng thứ 2 tuần này?", "tổng giờ làm tháng này của phòng X?"

**Design decisions:**
- **Normalize vs Denormalize:** Schedule template (pattern) normalize riêng, schedule instances (actual daily assignments) denormalize để query nhanh.
- **Time range handling:** Dùng date range columns + exclude constraint (PostgreSQL) hoặc application-level validation — tránh overlap.
- **Composite indexes:** `(department_id, date, shift_type)` cho query pattern phổ biến nhất.

**Ví dụ: Exam System (PREP)**
- `exams` → `exam_sections` → `questions` → `answer_options` (hierarchical).
- `exam_sessions` (candidate attempt) → `session_answers` (từng câu trả lời).
- `anti_cheat_events` (time-series data, partition by date).
- **Trade-off:** exam data denormalize vào exam domain (Go) cho fast read, source of truth vẫn ở management domain (Laravel).

---

## VI. Soft Skills & Growth

---

### Q27: Dự án nào khó khăn nhất? Bạn vượt qua thế nào?

**Gợi ý trả lời:**

**Anti-cheating service ở PREP:**

- **Khó khăn 1 - Undefined problem:** Không có clear specification "cheating" là gì. Mỗi loại thi có standard khác nhau. Phải work closely với domain experts (giáo viên, exam administrators) để define rules.
- **Khó khăn 2 - Real-time constraint:** Phải detect trong seconds, không phải minutes. Force mình học sâu về Go concurrency, Redis optimization.
- **Khó khăn 3 - Testing:** Rất khó test anti-cheat system — phải simulate human behavior. Build test harness giả lập thousands candidate với different cheating patterns.
- **Vượt qua:** Iterate nhanh — build simple version first (chỉ detect tab switch), validate với real exam, rồi thêm signals dần. Không cầu toàn từ đầu.

---

### Q28: Bạn đã fail ở đâu và học được gì?

**Gợi ý trả lời:**

**Fail 1: Over-engineering early project (Tomosia)**
- Project nhỏ nhưng apply full design patterns, abstraction layers. Kết quả: code phức tạp, team khó maintain, delivery chậm.
- **Bài học:** YAGNI (You Ain't Gonna Need It). Simplicity is a feature. Start simple, refactor khi complexity thực sự cần.

**Fail 2: Không có monitoring khi launch (Tomosia)**
- Deploy xong không có monitoring. Bug ở production mất 2 ngày mới phát hiện qua user report.
- **Bài học:** Observability không phải optional. Logging, metrics, alerting phải có từ day 1.

**Fail 3: Underestimate migration complexity (GHTK)**
- Estimate DDD restructure sẽ xong trong 2 sprints. Thực tế mất 2 tháng vì phải maintain backward compatibility, migrate data, train team.
- **Bài học:** Large refactoring luôn tốn hơn estimate. Plan for incremental delivery, không big bang.

---

### Q29: Worst production incident bạn từng gặp?

**Gợi ý trả lời:**

**Context:** [Chọn 1 incident thực tế, ví dụ:]

Database connection pool exhaustion tại GHTK vào peak hour (đầu tháng, chấm công + duyệt lương).

**What happened:**
- Application log: `Connection pool exhausted, cannot acquire connection within timeout`.
- All APIs return 500.
- Duration: ~15 minutes.

**Root cause:**
- Một report query chạy hàng phút (không có timeout), hold connection.
- Nhiều users chạy report cùng lúc → hết pool.
- Cascading failure: các request bình thường cũng không lấy được connection.

**Response:**
1. Immediate: restart application để release connections.
2. Short-term: add query timeout (30s max), tăng pool size.
3. Long-term: tách report queries sang read replica, add connection pool monitoring + alert.

**Bài học:**
- Luôn set timeout cho mọi external call (DB, API, cache).
- Monitor resource utilization, không chỉ error rate.
- Một component slow có thể cascade failure toàn system.

---

### Q30: Nếu được quay lại, bạn sẽ làm khác gì trong career?

**Gợi ý trả lời:**

1. **Học system design sớm hơn:** Ban đầu focus quá nhiều vào coding, ít nghĩ về architecture. System design thinking giúp làm việc hiệu quả hơn ở mọi level.

2. **Viết test từ đầu:** Giai đoạn Tomosia ít viết test vì thấy "chậm". Sau này ở GHTK nhận ra: test saves time in the long run, đặc biệt khi refactoring.

3. **Đầu tư vào observability sớm hơn:** Monitoring, logging, tracing nên là first-class citizen, không phải afterthought.

4. **Communicate nhiều hơn:** Kỹ năng kỹ thuật quan trọng nhưng khả năng communicate technical decisions, trade-offs cho team và stakeholders cũng quan trọng không kém.

---

## VII. Deep Technical — Dựa Trên Kinh Nghiệm

---

### Q31: Distributed lock bạn đã implement thế nào với Redis?

**Gợi ý trả lời:**

**Use case tại GHTK:** Prevent double-processing khi approve leave request (multiple instances of HRM service).

**Implementation:**
```
SET lock:leave_request:{id} {unique_value} NX EX 30
```
- `NX`: chỉ set nếu key chưa tồn tại (atomic acquire).
- `EX 30`: auto-expire 30s (prevent deadlock nếu holder crash).
- `unique_value`: UUID per instance — chỉ holder mới được release.

**Release:**
- Lua script: check value match → delete (atomic check-and-delete).

**Khó khăn:**
- **Clock drift:** Nếu holder process lâu hơn TTL → lock tự release → process khác acquire → 2 processes chạy cùng lúc. Giải quyết: **fencing token** — mỗi lock có incremental token, downstream check token để reject stale operations.
- **Redis single point of failure:** Nếu Redis master down sau khi set lock nhưng trước khi replicate → lock mất. Cho use case critical, consider Redlock (multi-node) hoặc accept risk và có idempotency ở business logic.

---

### Q32: Bạn handle message ordering và exactly-once processing trong Kafka thế nào?

**Gợi ý trả lời:**

**Ordering:**
- Kafka đảm bảo ordering **within a partition**.
- Chọn **partition key** cẩn thận: employee_id cho HR events, exam_id cho exam events → events cùng entity luôn đúng thứ tự.
- Trade-off: ít partition keys → hot partition; nhiều partition keys → ordering chỉ per-key.

**Exactly-once (thực tế at-least-once + idempotency):**
- Kafka có idempotent producer (acks=all, enable.idempotence=true) → prevent duplicate produce.
- Consumer side: at-least-once delivery. Handle duplicate bằng:
  - **Idempotency key:** Mỗi event có unique ID. Consumer check "đã process event này chưa?" trước khi xử lý.
  - **Database unique constraint:** Insert with unique event_id → duplicate tự fail.
  - **Transactional outbox:** Write business data + mark event as processed trong cùng DB transaction.

**Thực tế tại PREP:**
- Scoring pipeline: `submission_id` là idempotency key. Nếu scoring worker nhận duplicate event, check scoring_results table → đã có → skip.

---

### Q33: WebSocket/real-time connection management cho exam system — challenges?

**Gợi ý trả lời:**

**Context:** Anti-cheating cần real-time bidirectional communication — server push alerts to proctor, client push events to server.

**Challenges:**
1. **Scale:** Mỗi candidate = 1 persistent connection. 5000 candidates = 5000 connections trên exam service. Go handle tốt nhờ goroutine nhẹ (~2KB stack), nhưng vẫn cần tune OS-level limits (file descriptors, TCP buffers).

2. **Reconnection handling:** Network unstable → connection drop → phải reconnect + restore state. Implement **session resumption** — server maintain session state, client reconnect với session ID, server replay missed events.

3. **Load balancing:** Sticky session cần thiết cho WebSocket (connection phải đến cùng server). Dùng **consistent hashing** trên load balancer based on session ID.

4. **Memory management:** Mỗi connection giữ buffer. 5000 connections x buffer size = significant memory. Monitor và set limits per-connection buffer.

5. **Graceful shutdown:** Khi deploy version mới, phải drain existing connections — notify clients to reconnect to new instance. Implement **connection draining** với timeout.

---

### Q34: Bạn optimize exam submission latency thế nào?

**Gợi ý trả lời:**

**Before optimization:**
- Candidate submit → validate → save to DB → score → save result → respond.
- Latency: ~2-3 seconds under load (DB bottleneck).

**After optimization:**
1. **Async scoring:** Submit → validate → save raw answer → respond immediately → score async.
   - Latency giảm xuống ~200ms.
2. **Batch writes:** Thay vì insert từng answer, buffer và batch insert mỗi 500ms.
3. **Connection pooling tuning:** Tăng pool size, giảm idle timeout.
4. **Redis cache exam data:** Đề thi cache ở Redis, validate answer không cần hit DB.
5. **Auto-save per question:** Candidate không cần submit tất cả cuối cùng. Mỗi câu trả lời auto-save → final submit chỉ là mark completion, rất nhẹ.

**Result:** p95 latency từ ~3s xuống <300ms. Candidate experience mượt hơn nhiều.

---

### Q35: Cách bạn đảm bảo data consistency giữa Laravel (management) và Go (exam) domains?

**Gợi ý trả lời:**

**Challenge:** 2 services, 2 databases riêng biệt. Exam data phải consistent.

**Strategy: Eventual consistency + verification.**

1. **Source of truth:** Management domain (Laravel) là source of truth cho exam configuration.
2. **Event-driven sync:** Management publish events → Exam domain consume và maintain local copy.
3. **Verification endpoint:** Trước khi exam bắt đầu, exam domain call management API để verify data integrity (checksum/version match).
4. **Conflict resolution:** Nếu mismatch detected → re-sync từ management domain → delay exam start nếu cần.

**Edge cases handled:**
- Event lost → periodic reconciliation job check và sync missing data.
- Event out of order → version number trên mỗi entity, chỉ apply nếu version lớn hơn.
- Management update exam đang diễn ra → exam domain reject updates cho active exams (business rule).

**Trade-off accepted:** Exam data có thể stale vài giây so với management. Acceptable vì exam config hiếm khi thay đổi sau khi đã scheduled.
