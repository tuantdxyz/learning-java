# Best Strategies for Handling Large Datasets in Java

Tài liệu này tổng hợp các chiến lược tốt nhất để xử lý dữ liệu lớn trong Java, gồm lý thuyết, ví dụ mã cũ cần tránh và mã mới nên dùng.
Nội dung phù hợp với các dự án thực tế backend (Spring, JDBC, file xử lý lớn, microservice, hiệu năng cao).

---

## I. 9 Chiến lược chính trong xử lý dữ liệu lớn

### 1. Streaming thay vì load toàn bộ

**Lý thuyết:** Đọc file hoặc dữ liệu theo dòng/batch giúp tránh OutOfMemory.

```java
// ❌ Cũ:
List<String> lines = Files.readAllLines(Paths.get("bigfile.txt")); // OOM nếu file lớn
lines.forEach(System.out::println);

// ✅ Mới:
try (Stream<String> lines = Files.lines(Paths.get("bigfile.txt"))) {
    lines.forEach(System.out::println); // Lazy load, hiệu quả
}
```

---

### 2. Dùng primitive thay vì wrapper object

**Lý thuyết:** Integer, Long gây overhead, nên dùng int[], long[] nếu có thể.

```java
// ❌ Cũ:
List<Integer> numbers = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    numbers.add(i); // Boxing
}

// ✅ Mới:
int[] numbers = new int[1_000_000];
for (int i = 0; i < numbers.length; i++) {
    numbers[i] = i;
}
```

---

### 3. Xử lý theo batch thay vì từng bản ghi

**Lý thuyết:** Batch giúp giảm roundtrip và tránh overload CPU/memory.

```java
// ❌ Cũ:
for (User user : users) userRepository.save(user); // Gọi liên tục

// ✅ Mới:
for (int i = 0; i < users.size(); i += 1000) {
    List<User> batch = users.subList(i, Math.min(i + 1000, users.size()));
    userRepository.saveAll(batch); // Gọi theo từng 
}
```

---

### 4. Dùng Memory Mapped File cho file lớn

**Lý thuyết:** Đọc file > 2GB mà không chiếm nhiều heap memory.

```java
// ❌ Cũ:
byte[] data = Files.readAllBytes(Paths.get("large.bin")); // OOM nếu file lớn, Read all bytes

// ✅ Mới:
try (FileChannel channel = FileChannel.open(Paths.get("large.bin"), StandardOpenOption.READ)) {
    MappedByteBuffer buffer = channel.map(FileChannel.MapMode.READ_ONLY, 0, channel.size());  // Dùng MappedByteBuffer
    while (buffer.hasRemaining()) {
        byte b = buffer.get();
        // xử lý
    }
}
```

---

### 5. Tái sử dụng object

**Lý thuyết:** Hạn chế tạo object mới trong loop.

```java
// ❌ Cũ:
for (...) {
   StringBuilder sb = new StringBuilder();
}

// ✅ Mới:
StringBuilder sb = new StringBuilder();
for (...) {
    sb.setLength(0);
}
```

---

### 6. Dùng Stream hiệu quả

**Lý thuyết:** Stream lazy, không cần collect trung gian.

```java
// ❌ Cũ:
List<String> result = lines.stream().filter(...).collect(toList());
result.forEach(System.out::println);  // Stream rồi collect rồi xử lý

// ✅ Mới:
lines.stream().filter(...).forEach(System.out::println);  // Stream lazy hoàn toàn
```

---

### 7. SoftReference / Cache thông minh

**Lý thuyết:** Dùng khi muốn cache có thể bị GC khi thiếu bộ nhớ.

```java
// ❌ Cũ:
Map<String, LargeObject> cache = new HashMap<>();
cache.put("data", loadHeavy());  // Luôn giữ cache trong HashMap

// ✅ Mới:
SoftReference<LargeObject> ref = new SoftReference<>(loadHeavy());
LargeObject obj = ref.get();  // Dùng SoftReference, sẽ null nếu GC cần dọn
```

---
### 8. Tối ưu JVM & GC

**Lý thuyết:** Giảm pause, tránh OOM, GC tuning đúng giúp ổn định hơn.

```bash
// ❌ Cũ:
java -jar app.jar  // Không giới hạn heap, GC mặc định

// ✅ Mới:
java -Xms1G -Xmx4G -XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError -jar app.jar  // Tối ưu JVM flags
```

---

### 9. Giám sát và đo đạc GC

**Lý thuyết:** GC logs giúp chẩn đoán bottleneck khi xử lý nhiều dữ liệu.

```bash
// ❌ Cũ:
java -jar app.jar // Không có logging GC

// ✅ Mới:
java -Xlog:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -jar app.jar // Bật GC log và profiler
```

---

## II. 5 Hạng mục phổ biến cần cải tiến trong dự án

### 10. Không dùng `findAll()` với dữ liệu lớn

```java
// ❌ Cũ:
customerRepository.findAll();  // Không phân trang, quá tải

// ✅ Mới:
customerRepository.findAll(PageRequest.of(0, 1000));   // Dùng paging hoặc keyset pagination

// 👉 Best hơn:
List<Customer> customers = customerRepository.findByIdGreaterThanOrderByIdAsc(lastId, PageRequest.of(0, 1000));    // Dùng lastId

```

---

### 11. Log quá nhiều hoặc log mọi vòng lặp

```java
// ❌ Cũ:
for (...) log.info("Processing...");  // Log dữ liệu lớn trong vòng lặp gây IO bottleneck

// ✅ Mới:
if (i % 1000 == 0) log.info("Processed {} records", i);  // Log theo batch
```

---

### 12. Load entity full thay vì dùng projection

```java
// ❌ Cũ:
List<User> users = userRepository.findAll(); // Entity có nhiều field không dùng
List<UserDto> dtos = users.stream().map(UserDto::from).toList();  // Dùng JPA load entity full rồi map thủ công

// ✅ Mới:
List<UserDto> dtos = userRepository.findUserSummaries(); // Trả về DTO gọn nhẹ
@Query("SELECT new dto.UserDto(u.id, u.name) FROM User u")
List<UserDto> findUserSummary();
```

---

### 13. Dùng parallelStream không kiểm soát

```java
// ❌ Cũ:
files.parallelStream().forEach(this::upload); // Không kiểm soát số thread trong đa luồng

// ✅ Mới:
ExecutorService pool = Executors.newFixedThreadPool(10);  // Dùng ExecutorService hoặc Spring @Async
pool.submit(() -> process(...));
```

---

### 14. Không dùng cache cho dữ liệu ít thay đổi

```java
// ❌ Cũ:
List<Country> countries = countryRepository.findAll(); // Gọi DB liên tục

// ✅ Mới:
@Cacheable("countries")  // Cache những dữ liệu ít thay đổi
public List<Country> getCountries() {
    return repo.findAll();
}
```

---

## III. 6 lỗi phổ biến khác trong dự án

### 15. Dùng Optional sai cách

```java
// ❌ Cũ:
public class UserDto {
    private Optional<String> name; // KHÔNG nên
}
public void process(Optional<String> name) { ... } // KHÔNG nên

// ✅ Mới:
public Optional<User> findById(String id) {  // Chỉ dùng Optional trong return type của method
    // OK
}
```

### 16. N+1 query không dùng fetch join

```java
// ❌ Cũ:
List<Order> orders = orderRepository.findAll(); // Gây N+1 nếu Order → Customer là LAZY
for (Order o : orders) {
    System.out.println(o.getCustomer().getName()); // Truy vấn lặp lại
}

// ✅ Mới:
// Dùng @EntityGraph hoặc JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.customer")  
List<Order> findAllWithCustomer();

@EntityGraph(attributePaths = "customer")
List<Order> findAll();
```

### 17. Không khai báo capacity cho List

```java
// ❌ Cũ:
List<String> names = new ArrayList<>();

// ✅ Mới:
List<String> list = new ArrayList<>(expectedSize);  // xác định size = expectedSize, tránh việc ArrayList phải resize lại nhiều lần 
```

### 18. Tạo ObjectMapper liên tục

```java
// ❌ Cũ:
ObjectMapper mapper = new ObjectMapper();  // Sử dụng new ObjectMapper() nhiều lần
User user = mapper.readValue(json, User.class);

// ✅ Mới:
// Dùng @Autowired hoặc singleton cấu hình sẵn
@Bean
ObjectMapper mapper() { return new ObjectMapper(); }

@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }
}
```

### 19. Không giới hạn pageSize

```java
// ❌ Cũ:
PageRequest.of(page, size);  // size có thể gây quá tải nếu ?size=10000

// ✅ Mới:
int safeSize = Math.min(size, 100);  // Giới hạn size tối đa
PageRequest.of(page, safeSize);
```

### 20. Lưu file nhúng DB

```java
// ❌ Cũ:
@Column
private byte[] avatarImage;  // Lưu ảnh dưới dạng BLOB hoặc BASE64

// ✅ Mới:
// File nên lưu trong S3, Minio, hoặc thư mục hệ thống – giảm tải DB, dễ CDN/cache.
private String avatarPath; // thay vì byte[] avatar, lưu đường dẫn hoặc URL file
```

---

## IV. 10 tính năng Java 17–21 nên áp dụng

### 21. Record Class (Java 16+)
Tạo immutable data class gọn hơn rất nhiều so với class thông thường.

```java
public record UserDto(String name, int age) {}
```

### 22. Sealed Classes (Java 17)
Cho phép giới hạn class nào có thể kế thừa superclass → tăng tính an toàn, dễ kiểm soát trong hệ thống lớn.

```java
sealed interface Shape permits Circle, Square {}
```

### 23. Pattern Matching `instanceof` (Java 16+)
Giảm code lặp khi check và cast đối tượng. Dùng khi xử lý nhiều loại object linh hoạt, như với Object hoặc base interface.

```java
if (obj instanceof String s) System.out.println(s);
```

### 24. Switch pattern matching (Java 21)
Switch hỗ trợ nhiều dạng, dễ đọc, ít lỗi. Dùng khi thay thế if-else logic nhiều lớp.

```java
return switch(obj) {
  case String s -> "Str: " + s;
  default -> "Unknown";
};
```

### 25. Text Blocks (Java 15+)
Giúp viết multi-line string (SQL, HTML, JSON...) rõ ràng hơn. Dùng khi viết truy vấn SQL, cấu hình YAML, nội dung HTML trong code.

```java
String sql = """
 SELECT * FROM user
 WHERE age > 18
""";
```

### 26. Virtual Threads (Java 21)
Cho phép tạo hàng triệu thread nhẹ hơn mà không block OS thread. Dùng khi viết service có nhiều IO, REST call, DB call – thay vì thread pool phức tạp.

```java
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
  exec.submit(() -> fetch());
}
```

### 27. Structured Concurrency (Java 21 preview)
Tổ chức các thread con như block code – dễ quản lý lifecycle. Dùng khi viết API cần gọi song song 2–3 hệ thống khác và muốn fail-fast.

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
  Future<String> user = scope.fork(() -> fetchUser());
  scope.join();
}
```

### 28. ScopedValue thay ThreadLocal (Java 21)
Thay thế ThreadLocal an toàn hơn trong multi-thread hoặc virtual thread.  Dùng khi bạn đang dùng ThreadLocal để giữ trạng thái request, security context, v.v.

```java
ScopedValue<String> USER = ScopedValue.newInstance();
ScopedValue.where(USER, "abc").run(() -> ...);
```

### 29. Enhanced `stripIndent` và `translateEscapes`
 Dùng khi xử lý text block sạch đẹp, không cần thủ công replace \n, tab,...

```java
String text = """
 Hello\nWorld
""".stripIndent().translateEscapes();
```

### 30. Foreign Function & Memory API (Java 20+)
Thay thế JNI, truy cập native memory an toàn.

```java
try (Arena arena = Arena.ofConfined()) {
  MemorySegment segment = arena.allocate(100);
}
```

---

📘 **Ghi chú:**

- Tài liệu này dùng để học, review code hoặc định hướng refactor hệ thống lớn.
- Các ví dụ giả định Java 17+ trở lên.

