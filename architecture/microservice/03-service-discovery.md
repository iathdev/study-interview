# 03 — Service Registry & Discovery

## Vấn đề

Trong Microservice, service instance có thể tăng/giảm động (auto-scaling), IP thay đổi liên tục. Hard-code IP là không thể → cần cơ chế tìm địa chỉ service tự động.

---

## Service Registry

Là nơi lưu trữ trung tâm thông tin các service đang chạy: tên, IP, port, health status.

- Service khởi động → **tự đăng ký** (self-registration) vào registry
- Service tắt → tự hủy đăng ký hoặc bị phát hiện qua health check

---

## Service Discovery

Quá trình tìm kiếm địa chỉ service khác để giao tiếp. Có 2 loại:

### Client-side Discovery

```
Service A ──lookup "order-service"──▶ Registry
          ◀────── [IP:port list] ────
          ──chọn instance──▶ Service B (192.168.1.5:8080)
```

- Client tự query registry, tự chọn instance (round-robin, random)
- Logic load balancing nằm ở client
- Ví dụ: Netflix Eureka + Ribbon

### Server-side Discovery

```
Service A ──▶ Load Balancer / API Gateway
                    │ (query registry, chọn instance)
                    └──▶ Service B-1 | Service B-2 | Service B-3
```

- Client chỉ biết một endpoint (gateway)
- Gateway chịu trách nhiệm lookup + load balance
- Ví dụ: AWS ALB + ECS, Kubernetes Service

---

## So sánh

| Tiêu chí | Client-side | Server-side |
|---------|------------|------------|
| Logic LB | Ở client | Ở gateway/LB |
| Complexity | Client phức tạp hơn | Client đơn giản |
| Dependency | Client biết về registry | Client chỉ biết 1 endpoint |
| Ví dụ | Eureka + Ribbon | K8s Service, AWS ALB |

---

## Công cụ phổ biến

| Tool | Loại | Đặc điểm |
|------|------|----------|
| **Consul** | Client + Server | Health check, key-value store, service mesh |
| **Eureka** | Client-side | Netflix OSS, Spring Cloud ecosystem |
| **Etcd** | Server-side | Nhẹ, dùng trong Kubernetes |
| **K8s DNS** | Server-side | Built-in K8s, ClusterIP + DNS tự động |
| **Zookeeper** | Server-side | Dùng nhiều trong Kafka, Hadoop |

**Thực tế hiện đại**: Kubernetes tự giải quyết service discovery qua DNS (`order-service.default.svc.cluster.local`) — không cần Eureka hay Consul nếu đã dùng K8s.

---

## Interview Q&A

**Q: Tại sao cần Service Discovery?**

> IP của service thay đổi liên tục trong môi trường cloud/container (scale up/down, restart, rolling deploy). Hard-code IP không khả thi. Service Discovery tự động track địa chỉ mới nhất và loại bỏ instance unhealthy.

**Q: Client-side vs Server-side — chọn gì?**

> Kubernetes → dùng built-in DNS (server-side). Spring Boot on VMs → Eureka (client-side). Cần multi-datacenter, multi-cloud → Consul. Với K8s, service discovery gần như được giải quyết hoàn toàn mà không cần tool thêm.
