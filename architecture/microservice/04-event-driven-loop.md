# 04 — Event-Driven & Event Loop

## Event Loop là gì?

Mô hình xử lý sự kiện bất đồng bộ trong đó **một thread duy nhất** xử lý nhiều request đồng thời mà không bị block, thông qua vòng lặp liên tục kiểm tra sự kiện.

---

## Mô hình truyền thống vs Event-Driven

**Thread-per-request (truyền thống — Apache):**
```
Request 1 ──▶ Thread 1 (block chờ DB)
Request 2 ──▶ Thread 2 (block chờ file)
Request 3 ──▶ Thread 3 (block chờ network)
...
10,000 request → 10,000 thread → OOM
```

**Event Loop (Nginx, Node.js):**
```
Request 1 ──▶ Event Loop ──▶ gửi I/O request (non-blocking) ──▶ tiếp tục
Request 2 ──▶ Event Loop ──▶ gửi I/O request (non-blocking) ──▶ tiếp tục
Request 3 ──▶ Event Loop ──▶ gửi I/O request (non-blocking) ──▶ tiếp tục
...
I/O xong ──▶ callback được gọi ──▶ trả kết quả
```

Một thread, hàng nghìn kết nối đồng thời. CPU không bị idle chờ I/O.

---

## Các thành phần

| Thành phần | Vai trò |
|-----------|---------|
| **Event** | Sự kiện xảy ra: request đến, file đọc xong, timer hết |
| **Event Queue** | Hàng đợi chứa sự kiện chờ xử lý |
| **Event Loop** | Vòng lặp liên tục lấy event từ queue và dispatch đến handler |
| **Handler/Callback** | Hàm được gọi khi event xảy ra |
| **Non-blocking I/O** | I/O không block thread: `epoll` (Linux), `kqueue` (BSD), `IOCP` (Windows) |

---

## Quy trình

```
1. Request đến → thêm vào Event Queue
2. Event Loop lấy event → gọi handler
3. Handler bắt đầu I/O (DB query, file read...) → non-blocking, trả control ngay
4. Event Loop tiếp tục xử lý event khác
5. I/O hoàn thành → OS thông báo → callback được đưa vào queue
6. Event Loop gọi callback → trả response
```

---

## Ưu và nhược điểm

**Ưu điểm:**
- Xử lý hàng nghìn kết nối đồng thời với tài nguyên thấp
- Không tốn memory cho thread-per-connection
- Phù hợp với I/O-bound workload

**Nhược điểm:**
- **Không phù hợp CPU-bound**: tính toán nặng (video encoding, ML) block Event Loop → toàn bộ kết nối bị treo
- Code phức tạp hơn: callback hell, cần async/await
- Debug khó hơn do luồng bất đồng bộ

---

## Ứng dụng thực tế

| Hệ thống | Cách dùng |
|---------|-----------|
| **Nginx** | Event loop với `epoll` → xử lý hàng chục nghìn connection đồng thời |
| **Node.js** | Single-threaded event loop với `libuv` |
| **Java NIO** | Non-blocking I/O, dùng trong Netty, Vert.x |

---

## Interview Q&A

**Q: Nginx xử lý được nhiều kết nối hơn Apache — tại sao?**

> Apache dùng thread-per-request: mỗi kết nối tốn ~1-8MB RAM cho 1 thread. 10,000 kết nối = 10-80GB RAM. Nginx dùng event loop: 1 worker process xử lý hàng nghìn kết nối với non-blocking I/O, không tốn thread per connection.

**Q: Khi nào không nên dùng Event Loop?**

> CPU-bound task (video processing, image resize, ML inference). Một tác vụ nặng sẽ block toàn bộ event loop, khiến tất cả request khác bị treo. Giải pháp: offload CPU-bound task sang worker thread/process riêng (Node.js Worker Threads, Go goroutine).
