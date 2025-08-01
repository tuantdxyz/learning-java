
# Tổng hợp Best Practices về Builder Pattern & DTO trong dự án Java hiện đại

---

## 1. Builder Pattern là gì? Khi nào nên dùng?

- **Mục đích:** Tạo đối tượng phức tạp với nhiều tham số, đặc biệt có tham số tùy chọn.
- **Khi dùng:**
  - Lớp có nhiều thuộc tính (>=4).
  - Tránh constructor dài dòng.
  - Giữ đối tượng bất biến (immutable).
  - Cần dễ đọc, dễ bảo trì code tạo object.

---

## 2. Các kiểu Builder phổ biến hiện nay

| Kiểu Builder                   | Mô tả                                  | Ưu điểm                           | Nhược điểm                     | Khi dùng phù hợp                  |
|-------------------------------|---------------------------------------|----------------------------------|--------------------------------|---------------------------------|
| Classic Builder (Inner Static) | Lớp static inner builder, method set  | Rõ ràng, kiểm soát tốt, immutable| Code dài, boilerplate           | Dự án Java chuẩn, codebase lớn   |
| Lombok Builder                | Dùng annotation `@Builder`             | Giảm boilerplate, dễ dùng         | Phụ thuộc Lombok                | Dự án dùng Lombok                |
| Step Builder                  | Builder theo bước, interface ép thứ tự | Ép buộc tham số bắt buộc          | Code phức tạp                  | Khi cần kiểm soát thứ tự tham số |
| Java Record + withers         | Record bất biến + method copy          | Ngắn gọn, bất biến mặc định       | Cần tự tạo withers             | Java 16+, dữ liệu đơn giản       |
| Fluent Interface Builder     | Chain method, không tách Builder       | Dễ viết, linh hoạt                | Dễ gây lỗi nếu không immutable  | Dự án nhỏ, prototype             |

---

## 3. Best Practices triển khai Builder Pattern

- Giữ đối tượng immutable (`final` fields, không setter).
- Sử dụng method chaining trả về `this`.
- Thêm validation trong method `build()`.
- Dùng giá trị mặc định cho tham số tùy chọn trong Builder.
- Giữ Builder làm `public static class` inner class.
- Tách riêng Builder class nếu cần reusable hoặc test.

---

## 4. DTO (Data Transfer Object) và vai trò

- DTO dùng để truyền dữ liệu giữa các tầng trong ứng dụng, đặc biệt trong nhận request và trả response API.
- Tách biệt tầng domain (entity) với dữ liệu bên ngoài.
- Giúp validate, bảo mật và giảm coupling.
- Thường kết hợp với Builder để tạo DTO dễ dàng, tránh constructor dài.

---

## 5. Kết hợp DTO và Builder Pattern

- Builder dùng để tạo DTO, giúp code rõ ràng, maintainable.
- Giúp validate tham số khi tạo DTO.
- Hỗ trợ mở rộng thêm trường mới dễ dàng.
- Ví dụ:

```java
public class UserDTO {
    private final String name;
    private final String email;
    private final int age;

    private UserDTO(Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.age = builder.age;
    }

    public static class Builder {
        private String name;
        private String email;
        private int age;

        public Builder name(String name) {
            this.name = name; return this;
        }

        public Builder email(String email) {
            this.email = email; return this;
        }

        public Builder age(int age) {
            this.age = age; return this;
        }

        public UserDTO build() {
            // validate nếu cần
            return new UserDTO(this);
        }
    }
}
```

---

## 6. Khi nào nên dùng Step Builder Pattern?

- Khi cần ép thứ tự bắt buộc tham số.
- Khi dữ liệu bắt buộc phải đầy đủ, tránh tạo object thiếu.
- Code phức tạp, nên dùng khi thật sự cần.

---

## 7. Lời khuyên cho dự án Java hiện đại

- Dùng Classic Builder hoặc Lombok Builder cho hầu hết trường hợp.
- Dùng DTO tách biệt với Entity, tạo DTO bằng Builder.
- Thêm validation trong `build()` hoặc tầng service.
- Dùng Java Record + withers nếu đơn giản, Java 16+.
- Chuẩn hóa builder trong team, tránh tạo code không đồng nhất.
- Tránh builder nếu class quá đơn giản (1-2 trường).

---

## 8. Ví dụ Classic Builder với validation

### 8.1. Khai báo classclass
```java
public class Product {
    private final String name;
    private final double price;
    private final int quantity;

    private Product(Builder builder) {
        this.name = builder.name;
        this.price = builder.price;
        this.quantity = builder.quantity;
    }

    public static class Builder {
        private String name;
        private double price = 0.0;
        private int quantity = 1;

        public Builder name(String name) {
            if (name == null || name.isEmpty()) {
                throw new IllegalArgumentException("name must not be empty");
            }
            this.name = name;
            return this;
        }

        public Builder price(double price) {
            if (price < 0) {
                throw new IllegalArgumentException("price must be positive");
            }
            this.price = price;
            return this;
        }

        public Builder quantity(int quantity) {
            if (quantity <= 0) {
                throw new IllegalArgumentException("quantity must be > 0");
            }
            this.quantity = quantity;
            return this;
        }

        public Product build() {
            if (name == null) {
                throw new IllegalStateException("name is required");
            }
            return new Product(this);
        }
    }
}

```
### 8.2. Sử dụng
```java
public class Main {
    public static void main(String[] args) {
        // Tạo product hợp lệ
        Product product = new Product.Builder()
                .name("Laptop")
                .price(1500.0)
                .quantity(2)
                .build();

        System.out.println("Product created:");
        System.out.println("Name: " + product.name);
        System.out.println("Price: " + product.price);
        System.out.println("Quantity: " + product.quantity);

        // Tạo product với giá mặc định, số lượng mặc định
        Product defaultProduct = new Product.Builder()
                .name("Mouse")
                .build();

        System.out.println("Default product:");
        System.out.println("Name: " + defaultProduct.name);
        System.out.println("Price: " + defaultProduct.price);
        System.out.println("Quantity: " + defaultProduct.quantity);

        // Thử tạo product không đặt name (bắt buộc)
        try {
            Product invalidProduct = new Product.Builder()
                    .price(100.0)
                    .quantity(1)
                    .build();  // Sẽ ném IllegalStateException
        } catch (IllegalStateException e) {
            System.out.println("Error: " + e.getMessage());
        }

        // Thử đặt giá âm, sẽ ném IllegalArgumentException
        try {
            Product invalidProduct2 = new Product.Builder()
                    .name("Invalid Product")
                    .price(-50)
                    .build();
        } catch (IllegalArgumentException e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
}
```
### 8.3. Output
```java
Product created:
Name: Laptop
Price: 1500.0
Quantity: 2

Default product:
Name: Mouse
Price: 0.0
Quantity: 1

Error: name is required
Error: price must be positive
```
---
