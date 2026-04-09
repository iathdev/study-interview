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

Thường xảy ra khi:
+ Mới khởi tạo hoặc restart cache
+ Khi truy cập dữ liệu mới

Giải quyết:
+ Giải quyết vấn đề cache miss ít nhất có thể: Exeternal computation hoặc Probalilistic
early expiration
+ Dùng lock hoặc promise

### 5.1 Exeternal computation
Tạo worker hay trigger theo event expired từ cache để chủ động cập nhật lại cache khi
hết hạn hoặc gần hết hạn
Nhưng cache sẽ chứa nhiều dữ liệu rác

### 5.2 Probalilistic early expiration
currentTime - ( timeToCompute * beta * log(rand()) ) > expiry
Kết hợp việc đọc dữ liệu và sử dụng công thức trên để tính toàn thời điểm restart cache
Có thể nói là dựa trên xác xuất để biết thời điểm restart cache

### 5.3 Lock
Lock lại key cần lấy dữ liệu
Khi có cache miss thì lấy dữ liệu từ DB và đồng thời update vào cache => release lock
Sau khi release lock sẽ xử lý những request có cùng key

### 5.4 Promise
Thay vì bắt request phải xếp hàng đợi như Lock thì
Gắn các request vào callback của promise
Khi một promise chạy xong thì tất cả các promise đều có kết quả
==> Rất phù hợp với nodejs
PHP thì :))

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
