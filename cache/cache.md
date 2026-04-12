# 1. Caching Strategie
Cache lưu ở memory
Bộ nhớ đệm là thành phần phần cứng hoặc phần mềm lưu trữ dữ liệu tạm thời
Giúp việc truy vấn dữ liệu nhanh hơn
Dữ liệu trong bộ nhớ đệm:
Bản sao dữ liệu từ nguồn dữ liệu
Kết quả của phép tính trước đó
Bộ nhớ đệm là lá chắn cho DB, giúp DB không bị quá tải

## 1. Read Strategies
### 1.1 Read-Through:
Application lấy trực tiếp dữ liệu từ cache mà không liên quan gì đến DB. Cache sẽ tự lấy
dữ liệu từ DB
Cache được coi là database chính
Cache die => ứng dụng die
==> Không nên dùng. Chỉ dùng trong trường hợp đọc dữ liệu nhiều, thay đổi dữ liệu ít
Nhược điểm:
- Khó control time to live
- Có nhiều dữ liệu rác, không đồng nhất với DB

### 1.2 Read-Aside
Application sẽ giao tiếp trực tiếp giữa cache & DB
Cơ chế:
+ Application kiểm tra trong cache có dữ liệu hay không. Có thì trả ra kết quả, không có
thì sang bước 2
+ Cache không có dữ liệu thì application sẽ lấy dữ liệu trong DB
+ Đồng thời khi lấy dữ liệu cũng sẽ update dữ liệu vào cache để sử dụng cho lần truy
cập tiếp théo
Ưu điểm:
+ Khi cache die vẫn có thể lấy dữ liệu từ DB
+ Chỉ lưu trữ giá trị cần thiết. Tiết kiệm chi phí và resource của cache server
+ Kết hợp được nhiều nguồn dữ liệu vào trong cache
Nhược điểm
+ Hay xảy ra cache miss khi truy vấn lần đầu tiên hoặc dữ liệu hết hạn cache. Để giảm
thiểu có thể load dữ liệu thủ công định kì vào cache
+ Khi cache miss thì thời gian load vẫn lâu (cache stampede)
+ Data inconsistency (dữ liệu không nhất quán) => Cần giải quyết bài toán cache
invalidation

Dùng khi nào:
+ Dữ liệu sử dụng nhiều, dữ liệu sử dụng lặp đi lặp lại
+ Dữ liệu tốn thời gian / resource xử lý, tính toán

## 2. Write strategy
### 2.1 Write through
Data được lưu vào cache, cache sẽ gửi yêu cầu lưu dữ liệu vào DB ngay lập tức

### 2.2 Write Back (computer architecture)
Data được lưu vào cache, cache sẽ gửi yêu cầu lưu dữ liệu vào DB định kỳ

### 2.3 Write Around
Lưu trực tiếp dữ liệu từ application vào DB
Sau đó sử dụng
Read aside hoặc read through để đọc
Thường sẽ kết hợp Write around + Read aside

## 3. Cache invalidation
Là xoá dữ liệu không hợp lệ khỏi cache
Tại sao cache invalidation lại khó:
- TTL: cần tìm giá trị phù hợp
- Concurrency: Race condition (sai dữ liệu)
- Data relationship: Cache A&B, A&B có quan hệ, Khi update A thì phải update B. Đối với
hệ thống phức tạp sẽ khó control
Các phương pháp:
+ Time-based: chọn time to live hợp lý - xoá bị động. Không nên chọn thời gian ttl quá
lớn
+ Command-based: Viết command để chủ động xoá cache
+ Event-based: Khi update dữ liệu trong DB sẽ bắn event để xoá cache (có thể kết hợp
với queue để ko ảnh hưởng đến luồng hiện tại)
+ Group-based: Nhóm key và xoá theo group

## 4. Cache & DB
Cache:
+ Lưu ở memory
+ Nhanh hơn DB
+ Độ tin cậy thấp hơn. Do lưu ở memory thì dễ bị thay đổi
+ Cache có thể ở mọi nơi: FE, BE, CDN, Proxy, ... => Khó invalidation cache

DB:
+ Lưu ở disk
+ Chậm hơn cache
+ Độ tin cậy cao hơn

## 5. Cache stampede - quá tải DB
Là vấn đề nhiều thread (user - BE) truy cập cùng lúc vào thời điểm cache miss dẫn đến
quá tải DB

**Kịch bản điển hình:**
```
T=0: Cache key "product:123" expire
T=0: 1000 request cùng lúc đến, tất cả cache miss
T=0: 1000 request cùng lúc query DB
T=0: DB bị overload => chậm hoặc crash
```

Thường xảy ra khi:
+ Mới khởi tạo hoặc restart cache (cold start)
+ Khi TTL của một key hot (được truy cập nhiều) hết hạn đồng loạt
+ Khi truy cập dữ liệu mới chưa có trong cache

**Tại sao nguy hiểm:**
- Key càng hot (traffic càng lớn) thì stampede càng mạnh
- DB nhận N request thay vì 1, dễ dẫn đến cascade failure (DB chết => service chết)
- Thời gian xử lý càng lâu (query phức tạp) thì window stampede càng rộng, càng nhiều
request dồn vào

Giải quyết:
+ Giải quyết vấn đề cache miss ít nhất có thể: External computation hoặc Probabilistic
early expiration
+ Dùng lock hoặc promise để đảm bảo chỉ 1 request query DB, còn lại chờ

### 5.1 External computation
Tạo worker hay trigger theo event expired từ cache để chủ động cập nhật lại cache khi
hết hạn hoặc gần hết hạn

```
Flow:
Cache worker giám sát TTL
=> Trước khi key expire: worker tự query DB và set lại cache
=> Request của user luôn hit cache, không bao giờ gặp cache miss
```

Ưu điểm:
- Loại bỏ hoàn toàn cache miss với key hot
- User không bao giờ phải chờ do cache miss

Nhược điểm:
- Cache sẽ chứa nhiều dữ liệu rác (key không còn được dùng nữa vẫn bị refresh)
- Tốn resource để chạy worker liên tục
- Phức tạp hơn trong triển khai

### 5.2 Probabilistic early expiration (XFetch algorithm)
```
currentTime - ( timeToCompute * beta * log(rand()) ) > expiry
```

Giải thích công thức:
- `timeToCompute`: thời gian cần để query DB và set lại cache (đo trước)
- `beta`: hệ số điều chỉnh mức độ tích cực refresh sớm (thường = 1)
- `log(rand())`: giá trị âm ngẫu nhiên, tạo ra tính xác suất
- Kết quả: mỗi request tự tính xác suất để quyết định có nên refresh cache sớm không

```
Flow:
Request đọc cache => còn TTL nhưng chạy công thức tính xác suất
=> Nếu công thức trả về true: request này tự refresh cache (recompute sớm)
=> Key càng gần expire thì xác suất refresh càng cao
=> Stampede được trải đều ra thay vì dồn vào 1 thời điểm
```

**Liên quan đến quá tải DB:**
Thay vì 1000 request đồng loạt miss cache rồi cùng query DB, XFetch khiến một vài
request tự nguyện refresh trước khi expire. Mỗi lần refresh thành công là set lại TTL
=> key tiếp tục sống, không bao giờ có thời điểm expire thật sự.
DB chỉ nhận lác đác vài query rải đều, thay vì stampede tập trung.

Ưu điểm:
- Không cần worker riêng, logic nằm trong application
- Không cần lock hay coordination giữa các request
- Phân tán việc refresh tự nhiên nhờ tính xác suất

Nhược điểm:
- Vẫn có thể nhiều request refresh cùng lúc (giảm xác suất, không loại bỏ hoàn toàn)
- Cần đo `timeToCompute` chính xác để công thức hoạt động đúng

### 5.3 Lock (Mutex/Distributed Lock)
Lock lại key cần lấy dữ liệu, đảm bảo chỉ 1 request được phép query DB

```
Flow:
Cache miss => Request 1 acquire lock thành công
=> Request 1 query DB, set cache, release lock
=> Request 2,3,...N thấy đang lock => chờ (wait/retry)
=> Sau khi Request 1 release lock: Request 2 check cache => hit => trả kết quả
=> Các request sau không cần query DB nữa
```

Ưu điểm:
- Đảm bảo chỉ 1 request query DB tại một thời điểm
- Đơn giản để hiểu

Nhược điểm:
- Request phải xếp hàng chờ => tăng latency
- Nếu request giữ lock bị crash => deadlock (cần set timeout cho lock)
- Trong distributed system cần dùng distributed lock (Redis SET NX + TTL)

Ví dụ với Redis:
```
SET lock:product:123 "1" NX EX 5   # Lock với TTL 5s để tránh deadlock
```

### 5.4 Promise / In-flight deduplication
Thay vì bắt request phải xếp hàng đợi như Lock thì gắn các request vào callback của
cùng 1 promise đang chạy

```
Flow:
Cache miss => Request 1 tạo Promise, bắt đầu query DB
=> Request 2,3,...N đến: phát hiện đã có Promise đang chạy
=> Request 2,3,...N subscribe vào Promise đó (không tạo query mới)
=> Promise hoàn thành: tất cả request nhận được kết quả cùng lúc
```

```javascript
// Ví dụ pattern trong Node.js
const inFlight = new Map();

async function getWithDedup(key) {
  const cached = await cache.get(key);
  if (cached) return cached;

  if (inFlight.has(key)) {
    return inFlight.get(key); // subscribe vào promise đang chạy
  }

  const promise = db.query(key).then(data => {
    cache.set(key, data);
    inFlight.delete(key);
    return data;
  });

  inFlight.set(key, promise);
  return promise;
}
```

Ưu điểm:
- Không có request nào phải xếp hàng, tất cả nhận kết quả cùng lúc khi DB trả về
- Hiệu quả hơn Lock về latency
- Rất phù hợp với Node.js (single-threaded, event loop)

Nhược điểm:
- Chỉ hiệu quả trong phạm vi 1 process/instance. Nếu có nhiều BE instance thì mỗi
instance vẫn query DB 1 lần => cần kết hợp với distributed lock
- PHP không có shared memory giữa các request => không áp dụng được pattern này

## 6. Data inconsistency
Write-around:
Update cache trước hay update cache sau

Update cache trước => xảy ra race condition
Update cache sau => xảy ra race condition
Kết hợp Write-around + Read-aside: nên chọn xoá cache sau
Vì
Hạn chế ảnh hưởng của data inconsistency
Set time to live ngắn

### 8. Khó khăn
8.1 Cập nhật cache lỗi: retry lại

8.2 Cache down do nhiều vấn đề: lượng lớn request, vấn đề cơ sở hạ tầng
Trước sự cố: Có cache cluster
Trong sự cố: Rate limit: giới hạn lượng truy cập vào application. Để request vào DB ít đi.
Đảm bảo DB sống
Sau sự cố: Sau khi giải quyết sự cố thì bỏ giới hạn rate-limit

8.3 Nhiều request và cache miss nhiều, traffic đổ vào DB nhiều. Tương ứng với mục 5 -
Cache stampede
- Phân bố đề thời gian hết hạn của key

8.4 DB miss
- Set default value
- Validate đầu vào

8.4 Hight traffic
