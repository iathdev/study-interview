# Service Registry & Discovery

## 1. Khái niệm

### Service Registry

Là một hệ thống lưu trữ trung tâm, nơi các dịch vụ đăng ký thông tin (tên, IP, port, metadata) để các dịch vụ khác có thể truy xuất.

### Service Discovery

Là quá trình tìm kiếm (lookup) thông tin dịch vụ khác trong hệ thống để giao tiếp.

Có hai loại chính:
- Client-side discovery: Ứng dụng khách trực tiếp truy vấn registry để lấy thông tin dịch vụ và tự quyết định cách liên lạc.
- Server-side discovery: Một thành phần trung gian (như API Gateway hoặc Load Balancer) truy vấn registry và chuyển tiếp yêu cầu đến dịch vụ phù hợp.

---

## 2. Mục tiêu sử dụng

* Loại bỏ cấu hình IP cứng (hard-coded address)
* Hỗ trợ scale động
* Tự động phát hiện lỗi / failover
* Hỗ trợ load balancing

---

## 3. Các loại Service Discovery

### 3.1 Client-side Discovery

```plaintext
+-----------+           +--------------------+          +-------------+
|           |  Lookup   |                    |          |             |
|  ServiceA +--------->+   Service Registry  +--------->+   ServiceB  |
|  (Client) |           |                    |          |             |
+-----------+           +--------------------+          +-------------+
      |                                                        ^
      |               Health Check (optional)                  |
      +--------------------------------------------------------+
```

* Service A hỏi registry để lấy danh sách các instance của Service B
* A chọn 1 instance để gọi
* Logic chọn (round robin, random...) nằm ở phía client

### 3.2 Server-side Discovery

```plaintext
+-----------+          +-------------+
|           |          |             |
|  ServiceA +--------->+  Load Balancer / Gateway
|  (Client) |          |             |
+-----------+          +-------------+
                            |
         +------------------+------------------+
         |                  |                  |
   +-------------+    +-------------+    +-------------+
   |  ServiceB-1  |    |  ServiceB-2  |    |  ServiceB-3  |
   +-------------+    +-------------+    +-------------+
```

* Client chỉ gọi 1 endpoint trung gian (gateway / API Gateway / service mesh)
* Gateway chịu trách nhiệm query registry, chọn instance phù hợp

---

## 4. Các công cụ phổ biến

| Tên       | Kiểu discovery            | Mô tả                                           |
| --------- | ------------------------- | ----------------------------------------------- |
| Consul    | Client-side / Server-side | HashiCorp, hỗ trợ health check, key-value store |
| Eureka    | Client-side               | Netflix OSS, Spring Cloud ecosystem             |
| Zookeeper | Server-side               | Dùng nhiều trong Kafka, Hadoop                  |
| Etcd      | Server-side               | Nhẹ, dùng trong Kubernetes                      |
| K8s DNS   | Server-side               | Built-in trong Kubernetes (ClusterIP, DNS)      |

---

## 5. Kết luận

* Service Discovery là trụ cột trong kiến trúc microservices
* Cho phép hệ thống giao tiếp động, scale dễ dàng, giảm thiểu downtime
* Nên lựa chọn công cụ phù hợp với công nghệ nền (Spring Boot, Kubernetes, etc.)
