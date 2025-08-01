# Keyset Pagination trong Java: Giải pháp hiệu quả cho phân trang dữ liệu lớn

## 1. Giới thiệu

Khi làm việc với cơ sở dữ liệu lớn trong Java (Spring Boot, JPA, MyBatis...), việc phân trang dữ liệu là rất quan trọng để tránh tải toàn bộ dữ liệu vào bộ nhớ. Tuy nhiên, kỹ thuật phân trang phổ biến dùng `OFFSET` trở nên kém hiệu quả khi dữ liệu tăng cao. Bài viết này trình bày **Keyset Pagination (truy vấn theo Last ID)** — một giải pháp tối ưu hóa hiệu suất phân trang trong hệ thống lớn.

---

## 2. Lý thuyết

### 2.1 Vấn đề với OFFSET Pagination

```sql
SELECT * FROM orders ORDER BY id LIMIT 50 OFFSET 1000;
```

- Database phải quét qua 1000 bản ghi trước đó và **bỏ qua** → tốn thời gian.
- Không ổn định nếu dữ liệu thay đổi trong lúc phân trang (insert/delete).

### 2.2 Keyset Pagination là gì?

- Truy vấn dựa trên **giá trị của bản ghi cuối cùng** ở trang trước (`lastId`).
- Không cần OFFSET → DB truy vấn nhanh hơn và ổn định hơn.
- Không hỗ trợ truy cập ngẫu nhiên đến trang cụ thể.

---

## 3. So sánh code cũ vs mới

### 3.1 Code cũ: OFFSET Pagination (MyBatis / JPA native)

```sql
SELECT * FROM orders
WHERE status = 'DONE'
ORDER BY id
LIMIT 50 OFFSET 1000;
```

### 3.2 Code mới: Keyset Pagination

```sql
-- Trang đầu tiên
SELECT * FROM orders
WHERE status = 'DONE'
ORDER BY id
LIMIT 50;

-- Trang tiếp theo (lastId = 1050)
SELECT * FROM orders
WHERE status = 'DONE'
  AND id > 1050
ORDER BY id
LIMIT 50;
```

---

## 4. Áp dụng trong Java

### 4.1 Spring Boot JPA (native query / querydsl)

```java
@Query(value = "SELECT * FROM orders WHERE status = :status AND id > :lastId ORDER BY id ASC LIMIT :limit", nativeQuery = true)
List<Order> findNextPage(@Param("status") String status, @Param("lastId") Long lastId, @Param("limit") int limit);
```

### 4.2 Truyền tham số từ controller

```java
@GetMapping("/orders")
public List<Order> getOrders(
    @RequestParam(required = false) Long lastId,
    @RequestParam(defaultValue = "50") int limit) {
    if (lastId == null) {
        return orderRepository.findFirstPage(limit);
    }
    return orderRepository.findNextPage("DONE", lastId, limit);
}
```

### 4.3 Thiết kế phản hồi API

```json
{
  "items": [ ... ],
  "nextCursor": 1050,
  "hasNext": true
}
```

---

## 5. Mở rộng với nhiều cột sort (Composite Keyset)

```sql
SELECT * FROM events
WHERE (event_date < '2023-12-01')
   OR (event_date = '2023-12-01' AND id < 150)
ORDER BY event_date DESC, id DESC
LIMIT 50;
```

### Yêu cầu tạo index phù hợp:

```sql
CREATE INDEX idx_event_date_id ON events (event_date DESC, id DESC);
```

---

## 6. Ưu và nhược điểm

| Hạng mục              | Keyset Pagination                               |
| --------------------- | ----------------------------------------------- |
| Hiệu suất             | Cao, ổn định kể cả với hàng triệu bản ghi       |
| Truy cập trang cụ thể | Không hỗ trợ jump đến page bất kỳ               |
| Xử lý next            | Dễ dàng với lastId                              |
| Xử lý prev            | Cần lưu lại lịch sử cursor hoặc dùng reverse    |
| Ổn định dữ liệu       | Tốt hơn OFFSET khi có thêm/xóa dữ liệu liên tục |

---

## 7. Best Practices

- Luôn `ORDER BY` theo key duy nhất (`id` hoặc `(timestamp, id)`)
- Tạo index theo thứ tự sort để đảm bảo hiệu suất
- Hạn chế dùng OFFSET trong hệ thống có dữ liệu lớn hoặc tăng liên tục
- Với frontend infinite scroll, Keyset cực kỳ phù hợp

---

## 8. Kết luận

Keyset Pagination là giải pháp mạnh mẽ giúp nâng cao hiệu suất hệ thống Java khi truy vấn dữ liệu lớn. Việc áp dụng đúng cách giúp giảm tải hệ thống, đảm bảo trải nghiệm người dùng mượt mà và ổn định.

> Áp dụng tốt nhất trong hệ thống sử dụng Spring Boot, JPA, hoặc MyBatis có dữ liệu trên 1 triệu bản ghi trở lên.

