# Redis Cluster & Sentinel trong Java - Hướng dẫn tổng hợp

## 1. Mục tiêu tài liệu
Redis hỗ trợ rất nhiều kiểu cấu trúc dữ liệu như string, hash, list, set, sorted set, pub/sub,..
Đồng thời bổ sung thêm phân tích thực tế khi sử dụng Redis trong dự án Java/Spring Boot.

### Bảng tổng hợp cách sử dụng các kiểu dữ liệu phổ biến của Redis trong Java (Spring Data Redis)

| Kiểu dữ liệu Redis   | Mục đích sử dụng                           | Annotation trong Spring Data Redis     | Cách thao tác (Spring)                                   | Ghi chú                              |
|---------------------|-------------------------------------------|----------------------------------------|----------------------------------------------------------|-------------------------------------|
| **String**          | Lưu key-value đơn giản                     | Không                                  | `redisTemplate.opsForValue()`                            | Dữ liệu dạng chuỗi, phổ biến nhất   |
| **Hash**            | Lưu object dạng map key-field-value       | `@RedisHash` (trên class entity)       | `redisTemplate.opsForHash()` hoặc dùng Spring Data Repository | Thường dùng lưu entity, map trường  |
| **List**            | Danh sách có thứ tự, truy cập đầu/cuối    | Không                                  | `redisTemplate.opsForList()`                             | Push/pop đầu/cuối, duyệt tuần tự    |
| **Set**             | Tập hợp không trùng lặp                   | Không                                  | `redisTemplate.opsForSet()`                              | Không theo thứ tự, kiểm tra thành viên nhanh |
| **Sorted Set (ZSet)** | Tập hợp có thứ tự theo điểm số (score)   | Không                                  | `redisTemplate.opsForZSet()`                             | Thứ tự theo score, dùng cho ranking |
| **Pub/Sub**          | Gửi nhận thông điệp bất đồng bộ            | Không                                  | `redisTemplate.convertAndSend(channel, message)` và message listener | Không lưu trữ dữ liệu, chỉ truyền message |

---

### Annotation thường dùng trong Spring Data Redis (chủ yếu cho entity Hash)

| Annotation             | Mục đích                                       |
|------------------------|------------------------------------------------|
| `@RedisHash`           | Đánh dấu class sẽ được lưu dưới dạng Hash      |
| `@Id`                  | Đánh dấu trường khóa chính (key trong Redis)   |
| `@Indexed`             | Đánh dấu trường để tạo chỉ mục (cho tìm kiếm)  |
| `@TimeToLive`          | Thiết lập thời gian sống cho key (TTL)          |
| `@Reference`           | Đánh dấu tham chiếu đến entity khác             |
| `@PersistenceConstructor` | Xác định constructor dùng khi deserialize     |

---
### Cache trong Spring:

- Cần `@EnableCaching` trong config để bật cache.
- Annotation dùng:
  - `@Cacheable` → Cache kết quả nếu chưa có
  - `@CachePut` → Luôn ghi đè vào cache
  - `@CacheEvict` → Xoá cache

### Bảng tổng hợp annotation cache trong Spring

| Annotation | Hành vi | Ví dụ sử dụng | Ghi chú |
|------------|--------|----------------|--------|
| `@Cacheable` | Lấy từ cache nếu có, nếu không thì gọi method và lưu | `@Cacheable(value = "users", key = "#id")` | Tốt cho dữ liệu ít thay đổi |
| `@CachePut` | Luôn gọi method và lưu vào cache | `@CachePut(value = "users", key = "#user.id")` | Dùng khi cần cập nhật cache |
| `@CacheEvict` | Xoá cache | `@CacheEvict(value = "users", key = "#id")` | Dùng sau update/delete |
| `@EnableCaching` | Bật tính năng cache toàn ứng dụng | Thêm vào class `@Configuration` | Cần có để sử dụng annotation trên |

### Tóm tắt từ Redis.io Lesson 9

- `@Cacheable("book-search")`: Dùng để cache kết quả tìm kiếm sách
- Redis hoạt động như L2 cache
- Dùng Redis giảm tải DB, tăng hiệu suất truy vấn dữ liệu lặp lại
- TTL có thể cấu hình để dữ liệu tự hết hạn

### Khi nào nên dùng cache Redis trong Spring

| Trường hợp | Ghi chú |
|------------|---------|
| Truy vấn dữ liệu lặp lại nhiều lần | Tăng tốc độ, giảm tải DB |
| Dữ liệu ít thay đổi hoặc cho phép delay cập nhật | Như lookup bảng mã, quyền |
| Kết quả tốn chi phí tính toán | Như thống kê, report |
| Dữ liệu cần TTL tự xoá sau thời gian | Session, OTP, token |

---
### Ví dụ thao tác cơ bản với Redis List, Set, Hash:

```java
// List
redisTemplate.opsForList().leftPush("myList", "value1");
List<String> list = redisTemplate.opsForList().range("myList", 0, -1);

// Set
redisTemplate.opsForSet().add("mySet", "value1", "value2");
Set<String> set = redisTemplate.opsForSet().members("mySet");

// Hash (thường cho object)
redisTemplate.opsForHash().put("user:1", "name", "Tuan");
String name = (String) redisTemplate.opsForHash().get("user:1", "name");
```
---

## 2. Phân biệt Redis Sentinel vs Redis Cluster

| Tiêu chí                  | Redis Sentinel                                 | Redis Cluster                                 |
| ------------------------- | ---------------------------------------------- | --------------------------------------------- |
| Mục đích chính            | High Availability (HA) + Auto Failover         | Sharding + HA + Horizontal Scaling            |
| Số lượng master           | 1 master, nhiều replica                        | Nhiều master (sharded), mỗi master có replica |
| Phân mảnh dữ liệu (shard) | Không                                          | Có - theo slot hash                           |
| Failover tự động          | Có - Sentinel phát hiện master chết và bầu lại | Có - Cluster tự chuyển replica thành master   |
| Client hỗ trợ             | Cần client hỗ trợ Sentinel protocol            | Cần client hỗ trợ cluster protocol            |

---

## 3. Sharding tự động là gì?

Sharding là kỹ thuật chia dữ liệu thành nhiều phần (mảnh) lưu ở các node khác nhau. Redis Cluster hỗ trợ tự động chia dữ liệu dựa trên hash slot (từ 0 đến 16383):

- Key sẽ được băm ra slot
- Slot được phân bổ đều trên các node master
- Khi thêm node mới, Redis Cluster sẽ tự động chia lại slot

**Lợi ích:** Không dồn toàn bộ dữ liệu vào 1 Redis, giúp mở rộng theo chiều ngang.

---

## 4. Tự động failover trong Redis

- **Redis Sentinel:** phát hiện khi master chết, bầu replica thành master mới, cập nhật cấu hình cho client
- **Redis Cluster:** nếu 1 master chết và có quorum, Redis tự động chuyển replica thành master để đảm bảo hoạt động liên tục

---

## 5. Dữ liệu Redis có mất khi server chết không?

### Tùy vào cấu hình persistence:

#### a. RDB (snapshot)

- Mặc định Redis bật RDB
- Redis sẽ ghi snapshot sau mỗi khoảng thời gian nếu có thay đổi
- Ví dụ:
  ```
  save 900 1
  save 300 10
  save 60 10000
  ```
- **Khi server chết và restart:** dữ liệu sẽ được khôi phục lại từ file dump.rdb gần nhất

#### b. AOF (Append Only File)

- Ghi mọi command ghi dữ liệu vào file
- Khôi phục dữ liệu gần như đầy đủ nếu bật

#### c. Kết luận:

- Nếu Redis bị chết nhưng **RDB hoặc AOF đã bật**, dữ liệu vẫn tồn tại khi khởi động lại
- Với Redis Cluster hoặc Sentinel, việc failover không làm mất dữ liệu nếu persistence bật và các replica đồng bộ tốt

---

## 6. Bao nhiêu dữ liệu là "lớn"? Cần xóa khi nào?

Redis là in-memory nên giới hạn phụ thuộc vào RAM:

| RAM Redis | Tổng số key an toàn ước lượng |
| --------- | ----------------------------- |
| 1 GB      | \~500,000 key (small object)  |
| 4 GB      | \~2 triệu key                 |

**Gợi ý:**

- Nếu 1 Redis server vượt \~75-80% RAM → nên scale ra (sharding)
- Dùng `maxmemory` và `maxmemory-policy` để tự động xóa dữ liệu cũ

**Ví dụ:**

```conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

---

## 7. Thực tế dùng Redis trong Spring Boot

### 1. Cấu hình Redis với YAML (single server):

```yaml
singleServerConfig:
  address: "redis://your-host:6379"
  connectionPoolSize: 64
  connectionMinimumIdleSize: 24
  timeout: 3000
  retryAttempts: 3
  retryInterval: 1500
  database: 0
codec: !<org.redisson.codec.MarshallingCodec> {}
```

> ❗ YAML này không quy định persistence (RDB/AOF), mà được cấu hình ở Redis server.

### 2. Mặc định Redis có bật RDB snapshot không?

- **Có**: Redis mặc định bật RDB với các dòng `save` trong redis.conf
- AOF thì **tắt mặc định**, có thể bật thủ công nếu cần durability cao hơn

---

## 8. Spring Boot: Fallback DB khi Redis lỗi (load role)

Kịch bản phổ biến:

- Lưu `userRole` vào Redis để giảm tải DB
- Khi Redis lỗi → fallback DB
- Khi Redis sống lại → ghi đè lại Redis

### VD lưu Role user trong Redis:

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final RedisUserService redisUserService;
    private final UserRepository userRepository;

    public CustomUserDetailsService(RedisUserService redisUserService, UserRepository userRepository) {
        this.redisUserService = redisUserService;
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        UserDetails userDetails = null;

        try {
            userDetails = redisUserService.getUserDetailsFromRedis(username);
        } catch (Exception e) {
            // Redis chết hoặc lỗi
            log.warn("Redis unavailable, fallback DB: {}", e.getMessage());
        }

        if (userDetails != null) {
            return userDetails;
        }

        // Fallback DB
        User user = userRepository.findByUsernameWithRoles(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found"));

        // Chuyển thành Spring Security UserDetails
        userDetails = new org.springframework.security.core.userdetails.User(
                user.getUsername(),
                user.getPassword(),
                user.getRoles().stream()
                        .map(role -> new SimpleGrantedAuthority(role.getName()))
                        .collect(Collectors.toList())
        );

        // Ghi lại Redis nếu có thể
        try {
            redisUserService.saveUserDetailsToRedis(username, userDetails);
        } catch (Exception e) {
            log.warn("Cannot cache to Redis: {}", e.getMessage());
        }

        return userDetails;
    }
}
```
---

## 9. Tình huống đặc biệt: Redis chết lâu

- Nếu Redis chết 24h, logic `try-catch` Redis vẫn chạy → fallback DB
- Redis sống lại, cache lại từ DB
- Spring Boot sẽ không chờ Redis sống mà vẫn hoạt động được nếu bạn code tốt phần fallback

---

## 10. Kết luận

- **Redis Sentinel** phù hợp nếu bạn cần HA đơn giản, không cần sharding
- **Redis Cluster** dành cho hệ thống lớn, yêu cầu scale ngang, lưu nhiều dữ liệu
- Nên dùng cả RDB + AOF nếu cần khôi phục tốt
- Trong Spring Boot, cần chuẩn bị fallback logic khi Redis lỗi để đảm bảo hệ thống ổn định

---

## 11. Tài liệu tham khảo
Tài liệu:
- [https://medium.com/@rajatmtr30/redis-cluster-and-redis-sentinel-with-java-a-comprehensive-guide-8ce1bc485f6f](https://medium.com/@rajatmtr30/redis-cluster-and-redis-sentinel-with-java-a-comprehensive-guide-8ce1bc485f6f)
- [https://redis.io/learn/develop/java/redis-and-spring-course/lesson_4](https://redis.io/learn/develop/java/redis-and-spring-course/lesson_4)
- [https://redis.io/learn/develop/java/redis-and-spring-course/lesson_9](https://redis.io/learn/develop/java/redis-and-spring-course/lesson_9)

---

