
# Design Patterns và Kiến trúc Sử dụng trong Dự Án

## 1. Dependency Injection (DI)
### Bài toán thực tế: Bóng đèn - đui đèn
- Đại diện cho **Composition over Inheritance**
- Một đối tượng nên được tạo thành từ các thành phần bên ngoài thay vì tự khởi tạo bên trong.

### Ưu điểm:
- Dễ viết unit test thông qua mocking
- Giảm sự phụ thuộc và ràng buộc chặt chẽ
- Dễ dàng thay đổi các dependencies (implement)

### Ứng dụng:
- Áp dụng nhiều trong các controller/service để inject service qua constructor.

---

## 2. Factory Design Pattern
Dự án có nhiều loại item:
- Vé bình thường, vé năm, vé tháng, vé cổng vào, vé chỗ ngồi
- Vé bình thường cũng có nhiều trường hợp khởi tạo khác nhau

### Ứng dụng:
- Sử dụng Factory để khởi tạo các đối tượng tương ứng với từng loại vé.

---

## 3. Builder Design Pattern
Tránh việc khởi tạo một đối tượng phức tạp, builder tạo ra từng phần cụ thể.

### Ứng dụng:
- Sử dụng cho OpenSearch
- Viết tương tự như Eloquent ORM
- Các hàm sử dụng: khởi tạo, query, where, get, pagination

---

## 4. Observer Design Pattern
### Ứng dụng:
- Sử dụng Event - Listener để lắng nghe và xử lý logic.

---

## 5. Singleton Design Pattern
### Ứng dụng:
- Kết nối API bên thứ 3
- Chỉ khởi tạo một class duy nhất chứa các cấu hình cần thiết để gọi API.
- Áp dụng thêm Builder để gọi các hàm cần thiết.

---

## 6. Command Design Pattern
- Áp dụng command để đóng gói hành động và tách logic xử lý.

---

## 7. Data Transfer Object (DTO)
### Ưu điểm:
- Đảm bảo tính đúng đắn của dữ liệu
- Type hint giúp code rõ ràng hơn
- Dễ dàng tái sử dụng code

---

## 8. Repositories (Interface vs Implement)
Là lớp trung gian giữa lớp business logic và lớp hạ tầng (Infrastructure)

### Ưu điểm:
- Nghiệp vụ rõ ràng, tách biệt với logic kỹ thuật
- Linh hoạt trong triển khai: gọi API, đọc DB khác, master/slave, ...
- Tái sử dụng code, linh hoạt thay đổi DB (ít khi dùng)

### Lưu ý:
- Nếu dự án quá đơn giản thì không nên áp dụng.

---

## 9. SOLID Principles

> **Cách nhớ nhanh:** **S-O-L-I-D** = **S**ingle - **O**pen - **L**iskov - **I**nterface - **D**ependency

### 1. Single Responsibility Principle (SRP) — Một class, một việc

> **Một class chỉ nên có MỘT lý do để thay đổi.**

**Cách nhớ:** Giống nhân viên trong công ty: kế toán chỉ lo sổ sách, marketing chỉ lo quảng cáo. Nếu 1 người làm cả 2 → khi đổi quy trình kế toán, ảnh hưởng luôn việc marketing.

**Ví dụ trong Spring — DispatcherServlet:**

Spring MVC xử lý 1 HTTP request qua rất nhiều bước, nhưng KHÔNG gom vào 1 class. Mỗi class chỉ làm đúng 1 việc:

```
HTTP Request vào
    │
    ▼
DispatcherServlet          → Điều phối (không xử lý logic gì)
    │
    ├── HandlerMapping     → Tìm controller nào xử lý URL này
    │
    ├── HandlerAdapter     → Gọi method trong controller
    │
    ├── ViewResolver       → Tìm view (template) để render
    │
    └── HttpMessageConverter → Chuyển đổi JSON ↔ Object
```

| Class trong Spring | Trách nhiệm duy nhất |
|---|---|
| `DispatcherServlet` | Điều phối request, KHÔNG xử lý logic |
| `HandlerMapping` | Chỉ mapping URL → Controller |
| `HandlerAdapter` | Chỉ gọi method của controller |
| `ViewResolver` | Chỉ tìm và resolve view template |
| `HttpMessageConverter` | Chỉ convert JSON ↔ Java Object |

Nếu Spring gom hết vào `DispatcherServlet` → khi đổi cách resolve view, phải sửa cả class xử lý routing. **Vi phạm SRP.**

---

### 2. Open/Closed Principle (OCP) — Mở để mở rộng, đóng để sửa đổi

> **Muốn thêm tính năng mới → tạo class mới, KHÔNG sửa class cũ.**

**Cách nhớ:** Giống ổ cắm điện: muốn dùng thiết bị mới → cắm vào (mở rộng), không cần đập tường sửa dây điện (sửa đổi).

**Ví dụ trong Spring — `HttpMessageConverter`:**

Spring cần convert nhiều định dạng: JSON, XML, Protobuf... Nếu dùng if-else:

```java
// SAI — Spring KHÔNG làm thế này
public class MessageConverter {
    public Object convert(HttpInputMessage input, String contentType) {
        if ("application/json".equals(contentType)) {
            // parse JSON
        } else if ("application/xml".equals(contentType)) {
            // parse XML
        }
        // Thêm protobuf? → sửa class này
    }
}
```

Spring làm đúng OCP — dùng interface `HttpMessageConverter<T>`:

```java
// Interface mở rộng — package: org.springframework.http.converter
public interface HttpMessageConverter<T> {
    boolean canRead(Class<?> clazz, MediaType mediaType);
    boolean canWrite(Class<?> clazz, MediaType mediaType);
    T read(Class<? extends T> clazz, HttpInputMessage inputMessage);
    void write(T t, MediaType contentType, HttpOutputMessage outputMessage);
}

// Mỗi định dạng = 1 class riêng, Spring đã có sẵn:
MappingJackson2HttpMessageConverter   // JSON (Jackson)
GsonHttpMessageConverter              // JSON (Gson)
MappingJackson2XmlHttpMessageConverter // XML
ProtobufHttpMessageConverter          // Protobuf
StringHttpMessageConverter            // Plain text
ByteArrayHttpMessageConverter         // byte[]
```

**Thêm định dạng mới?** → Chỉ cần tạo class mới `implements HttpMessageConverter` rồi đăng ký:

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new MyCustomMessageConverter()); // cắm vào, không sửa gì
    }
}
```

**Không sửa bất kỳ class cũ nào trong Spring.**

---

### 3. Liskov Substitution Principle (LSP) — Class con thay thế được class cha

> **Ở đâu dùng class cha, thay bằng class con vào phải chạy đúng y như cũ.**

**Cách nhớ:** Con kế thừa cha → con phải giữ "lời hứa" của cha. Cha hứa trả về `List` → con cũng phải trả về `List`, không được trả `null` hay throw exception bất ngờ.

**Ví dụ trong Spring — chuỗi kế thừa Repository:**

```java
// Spring Data JPA thiết kế chuỗi kế thừa chuẩn LSP:
Repository                        // interface gốc (marker)
    └── CrudRepository            // thêm: save, findById, delete...
        └── ListCrudRepository    // thêm: findAll() trả List thay Iterable
            └── JpaRepository     // thêm: flush, saveAllAndFlush...
```

```java
// Ở đâu dùng CrudRepository, thay bằng JpaRepository vẫn chạy đúng
@Service
@RequiredArgsConstructor
public class UserService {

    private final CrudRepository<User, Long> repo;

    public User getUser(Long id) {
        return repo.findById(id).orElseThrow();
    }
}

// Inject JpaRepository thay CrudRepository → chạy y chang, không lỗi
// Vì JpaRepository giữ đúng mọi "lời hứa" của CrudRepository:
//   - findById() vẫn trả Optional<T>
//   - save() vẫn trả T
//   - deleteById() vẫn trả void
```

**Ví dụ VI PHẠM LSP** nếu Spring thiết kế sai:

```java
// Giả sử có ReadOnlyRepository extends CrudRepository
// nhưng override save() và throw exception:
public class ReadOnlyRepository extends CrudRepository {
    @Override
    public <S> S save(S entity) {
        throw new UnsupportedOperationException("Read only!");
        // Cha hứa save() luôn hoạt động → con phá vỡ → VI PHẠM LSP
    }
}
```

→ Đó là lý do Spring **KHÔNG** thiết kế thế. Thay vào đó Spring tách ra interface riêng (xem ISP bên dưới).

---

### 4. Interface Segregation Principle (ISP) — Interface nhỏ, chuyên biệt

> **Không ép class implement những method mà nó không cần.**

**Cách nhớ:** Giống remote TV: remote cơ bản chỉ cần nút tắt/mở/chuyển kênh. Đừng ép remote cơ bản phải có nút karaoke.

**Ví dụ trong Spring — tách Repository interface:**

Nếu Spring gom tất cả vào 1 interface:

```java
// SAI — Spring KHÔNG làm thế này
public interface EverythingRepository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    T save(T entity);
    void delete(T entity);
    void flush();
    Page<T> findAll(Pageable pageable);  // phân trang
    List<T> findAll(Sort sort);          // sắp xếp
    T saveAndFlush(T entity);
}
// → Class chỉ cần đọc cũng bị ép implement flush(), save(), delete()
```

Spring tách nhỏ, mỗi interface đúng 1 nhóm chức năng:

```java
// Chỉ CRUD cơ bản
public interface CrudRepository<T, ID> extends Repository<T, ID> {
    <S extends T> S save(S entity);
    Optional<T> findById(ID id);
    Iterable<T> findAll();
    void deleteById(ID id);
    // ...
}

// Chỉ phân trang + sắp xếp
public interface PagingAndSortingRepository<T, ID> extends Repository<T, ID> {
    Iterable<T> findAll(Sort sort);
    Page<T> findAll(Pageable pageable);
}

// Tổng hợp cho JPA (cần tất cả)
public interface JpaRepository<T, ID>
    extends ListCrudRepository<T, ID>,
            ListPagingAndSortingRepository<T, ID>,
            QueryByExampleExecutor<T> {
    void flush();
    <S extends T> S saveAndFlush(S entity);
    // ...
}
```

**Khi dùng:**

```java
// Chỉ cần đọc → dùng CrudRepository, không bị ép implement flush/paging
public interface UserReadRepo extends CrudRepository<User, Long> {}

// Cần phân trang → dùng PagingAndSortingRepository
public interface ProductRepo extends PagingAndSortingRepository<Product, Long> {}

// Cần tất cả → dùng JpaRepository
public interface OrderRepo extends JpaRepository<Order, Long> {}
```

---

### 5. Dependency Inversion Principle (DIP) — Phụ thuộc abstraction, không phụ thuộc implementation

> **Module cấp cao không phụ thuộc module cấp thấp. Cả hai phụ thuộc vào interface.**

**Cách nhớ:** Giống ổ cắm USB: laptop (cấp cao) không cần biết chuột loại gì (cấp thấp). Cả hai tuân theo chuẩn USB (interface).

**Ví dụ trong Spring — `JdbcTemplate` và `DataSource`:**

```java
// SAI — nếu Spring phụ thuộc trực tiếp vào implementation
public class JdbcTemplate {
    private final HikariDataSource dataSource = new HikariDataSource(); // cứng HikariCP
    // Muốn đổi sang Tomcat DBCP? → sửa JdbcTemplate
}
```

Spring làm đúng DIP — `JdbcTemplate` chỉ biết interface `javax.sql.DataSource`:

```java
// JdbcTemplate (module cấp cao) — chỉ phụ thuộc interface
public class JdbcTemplate {
    private final DataSource dataSource; // interface, không biết implementation

    public JdbcTemplate(DataSource dataSource) {
        this.dataSource = dataSource;
    }
}

// DataSource (interface/abstraction) — từ javax.sql
public interface DataSource {
    Connection getConnection() throws SQLException;
}

// Implementations (module cấp thấp) — có thể thay bất kỳ lúc nào
HikariDataSource implements DataSource      // HikariCP
TomcatDataSource implements DataSource      // Tomcat DBCP
DriverManagerDataSource implements DataSource // Spring built-in (dev/test)
```

```java
// Đổi connection pool chỉ cần đổi config, KHÔNG sửa code:
// application.yml
spring:
  datasource:
    type: com.zaxxer.hikari.HikariDataSource    # đổi thành Tomcat? sửa dòng này
    url: jdbc:mysql://localhost:3306/mydb
```

**Toàn bộ Spring IoC Container** chính là hiện thân của DIP:
- `@Controller` không biết `@Service` cụ thể nào → inject qua interface
- `@Service` không biết `@Repository` cụ thể nào → inject qua interface
- Spring Container đóng vai trò "người kết nối" giữa interface và implementation

---

### Tổng kết nhanh — Ví dụ từ chính Spring

| Nguyên tắc | Một câu để nhớ | Ví dụ trong Spring |
|---|---|---|
| **S** — Single Responsibility | 1 class = 1 việc | `DispatcherServlet` chỉ điều phối, `ViewResolver` chỉ resolve view |
| **O** — Open/Closed | Thêm mới = tạo class mới | `HttpMessageConverter` — thêm định dạng mới không sửa code cũ |
| **L** — Liskov Substitution | Con thay cha, chạy y chang | `JpaRepository` thay `CrudRepository` chạy đúng y hệt |
| **I** — Interface Segregation | Interface nhỏ, đúng việc | `CrudRepository` / `PagingAndSortingRepository` / `JpaRepository` tách riêng |
| **D** — Dependency Inversion | Depend interface, không depend class | `JdbcTemplate` depend `DataSource` interface, không depend `HikariDataSource` |
