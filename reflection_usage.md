# Reflection trong Java - Ứng dụng thực tế

## I. Tổng quan về Reflection

Reflection là cơ chế cho phép Java chương trình kiểm tra hoặc sửa đổi hành vi của class, interface, field, và method tại runtime. Dù mạnh mẽ, Reflection tiềm ẩn rủi ro như giảm hiệu suất, mất an toàn type-checking và khó maintain nếu lạm dụng.

---

## II. Các ứng dụng Reflection trong logic nghiệp vụ thực tế (Step-by-step)

### 1. Mapping dữ liệu động (DTO ↔ Entity)

**📌 Mục tiêu:** Chuyển dữ liệu giữa DTO ↔ Entity dựa theo cấu hình động.

**✔️ Khi dùng:** Khi các field thay đổi theo config YAML, JSON hoặc không cố định.

**Step-by-step:**

1. Input: `source` (VD: DTO từ request).
2. Output: `target` (VD: Entity để lưu DB).
3. Cách thực hiện:
   - Duyệt field trong `source` bằng Reflection.
   - Nếu `target` có field trùng tên:
     - Gán giá trị từ `source` → `target`.

**Code:**

```java
public void copyProperties(Object source, Object target) throws Exception {
    for (Field field : source.getClass().getDeclaredFields()) {
        field.setAccessible(true);
        Object value = field.get(source);

        try {
            Field targetField = target.getClass().getDeclaredField(field.getName());
            targetField.setAccessible(true);
            targetField.set(target, value);
        } catch (NoSuchFieldException ignore) {
            // Field không tồn tại trong target → bỏ qua
        }
    }
}
```

**Ví dụ Input/Output:**

```java
class UserDTO {
    private String name;
    private int age;
}

class UserEntity {
    private String name;
    private int age;
}

// Gọi: copyProperties(dto, entity);
```

---

### 2. Workflow / Rule Engine cấu hình động

**📌 Mục tiêu:** Cho phép hệ thống chạy rule theo config mà không cần hard-code.

**✔️ Khi dùng:** Logic thay đổi theo file/database như YAML, Redis, DB.

**Step-by-step:**

1. Input:
   - Order đầu vào
   - List tên các Rule class từ config ngoài.
2. Output: Order sau khi áp dụng các rule
3. Cách thực hiện:
   - Lấy danh sách rule từ Redis hoặc file
   - Duyệt từng ruleClass:
     - Dùng `Class.forName(...)` để load class
     - Tạo instance qua `getDeclaredConstructor().newInstance()`
     - Lấy method `apply(Order)` bằng `getDeclaredMethod(...)`
     - Gọi `method.invoke(ruleObj, order)`

**Code:**

```java
public void applyRulesFromConfig(List<String> ruleClassNames, Order order) throws Exception {
    for (String ruleClass : ruleClassNames) {
        Class<?> clazz = Class.forName(ruleClass);
        Object ruleObj = clazz.getDeclaredConstructor().newInstance();
        Method method = clazz.getDeclaredMethod("apply", Order.class);
        method.invoke(ruleObj, order);
    }
}
```

**Ví dụ Rule:**

```java
public class ShippingRule {
    public void apply(Order order) {
        if (order.getAmount() > 100) {
            order.setShippingCost(0);
        } else {
            order.setShippingCost(10);
        }
    }
}
```

---

### 3. Dynamic Plugin / Module Loader

**📌 Mục tiêu:** Cho phép load module plugin từ JAR bên ngoài runtime.

**✔️ Khi dùng:** Hệ thống CMS/ERP cần plugin mở rộng theo từng khách hàng.

**Step-by-step:**

1. Input:
   - Đường dẫn JAR
   - Tên class plugin
   - Context truyền vào
2. Output: Kết quả xử lý plugin
3. Cách thực hiện:
   - Dùng `URLClassLoader` để load JAR
   - Load class plugin từ JAR
   - Gọi method `run(context)` bằng Reflection

**Code:**

```java
URLClassLoader loader = new URLClassLoader(new URL[]{pluginPath.toURI().toURL()});
Class<?> plugin = loader.loadClass("com.plugin.EntryPoint");
Method execute = plugin.getMethod("run", Context.class);
execute.invoke(plugin.getDeclaredConstructor().newInstance(), context);
```

---

### 4. Object Inspection / Audit / Logging

**📌 Mục tiêu:** In/log toàn bộ thông tin object, kể cả private field.

**✔️ Khi dùng:** Debug, audit hoặc tracking request

**Step-by-step:**

1. Input: Bất kỳ object
2. Output: Thông tin log tên field + giá trị
3. Logic:
   - Duyệt field → setAccessible → lấy giá trị → log

**Code:**

```java
for (Field field : object.getClass().getDeclaredFields()) {
    field.setAccessible(true);
    System.out.println(field.getName() + " = " + field.get(object));
}
```

---

### 5. Unit Test private method

**📌 Mục tiêu:** Gọi method private trong test mà không expose ra public.

**✔️ Khi dùng:** Không muốn phá vỡ encapsulation nhưng cần test logic nội bộ.

**Step-by-step:**

1. Input: instance của class cần test
2. Output: Giá trị trả về từ method private
3. Cách làm:
   - Lấy method bằng `getDeclaredMethod`
   - `setAccessible(true)` để truy cập private
   - `invoke(...)` để thực thi

**Code:**

```java
Method method = MyService.class.getDeclaredMethod("calculateTax", BigDecimal.class);
method.setAccessible(true);
BigDecimal result = (BigDecimal) method.invoke(service, BigDecimal.TEN);
```

---

### 6. Custom Validator kiểm tra duplicate String[]

**📌 Mục tiêu:** Dùng annotation để validate không có duplicate trong List (ví dụ: String[] field).

**✔️ Khi dùng:** Dữ liệu từ nhiều DTO class nhưng có chung field (VD: `getLangCd`) cần validate.

**Step-by-step:**

1. Tạo annotation `@NoDuplicateLangCd`
2. Tạo `ConstraintValidator` sử dụng Reflection để gọi `getLangCd()`
3. Áp dụng annotation lên List input DTO trong controller

**Annotation:**

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = LangCdDuplicateValidator.class)
public @interface NoDuplicateLangCd {
    String message() default "Duplicated langCd";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    Class<?> clazz(); // class để lấy method chung
}
```

**Validator:**

```java
public class LangCdDuplicateValidator implements ConstraintValidator<NoDuplicateLangCd, List<?>> {
    private Class<?> clazz;

    @Override
    public void initialize(NoDuplicateLangCd constraintAnnotation) {
        this.clazz = constraintAnnotation.clazz();
    }

    @Override
    public boolean isValid(List<?> objects, ConstraintValidatorContext context) {
        try {
            Method method = clazz.getMethod("getLangCd");
            Set<String> set = new HashSet<>();
            for (Object o : objects) {
                String val = (String) method.invoke(o);
                if (!set.add(val)) return false;
            }
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

**Input DTO + Cách dùng:**

```java
public class LangItem {
    private String langCd;
    public String getLangCd() { return langCd; }
}

public class LangRequest {
    @NoDuplicateLangCd(clazz = LangItem.class)
    private List<LangItem> langs;
}
```

**Controller:**

```java
@PostMapping("/api/lang")
public ResponseEntity<?> save(@Valid @RequestBody LangRequest req) {
    // xử lý bình thường nếu không lỗi
    return ResponseEntity.ok("OK");
}
```

**Output:**

- Trường hợp duplicate `langCd` → trả về lỗi 400 với message "Duplicated langCd"

---

## III. Khi không nên dùng Reflection

| Trường hợp                    | Giải pháp thay thế           |
| ----------------------------- | ---------------------------- |
| Mapping DTO ↔ Entity đơn giản | MapStruct, ModelMapper       |
| Gọi method trong cùng class   | Gọi trực tiếp (code rõ ràng) |
| Cấu hình ít thay đổi          | Strategy pattern, Factory    |
| Chạy logic cố định            | Interface + generic          |

---

## IV. Tổng hợp nhanh

| Use case                                | Dùng Reflection? | Lý do sử dụng                                        |
| --------------------------------------- | ---------------- | ---------------------------------------------------- |
| DTO ↔ Entity mapping động               | ✅                | Field linh hoạt, đọc từ config                       |
| Workflow cấu hình động (tax, policy...) | ✅                | Rule runtime khó xác định từ trước                   |
| Dynamic Plugin hệ thống mở rộng         | ✅                | Load module theo jar                                 |
| Logging / Auditing nội dung object      | ✅                | Ghi log field private / động                         |
| Validate/Inject bằng custom annotation  | ✅                | Dùng mini framework, không dùng Spring               |
| Gọi private method trong test           | ✅ (test-only)    | Đảm bảo encapsulation trong production               |
| Logic nghiệp vụ rõ ràng, static         | 🚫               | Dễ maintain hơn nếu dùng interface / pattern rõ ràng |

---

## V. Kết luận

Reflection là công cụ mạnh trong Java nhưng nên sử dụng có kiểm soát. Phù hợp nhất trong những ngữ cảnh:

- Tính năng cấu hình runtime
- Debug/Logging sâu
- Plugin/Modular architecture

Không nên dùng nếu logic có thể được biểu diễn rõ ràng bằng class/interface/design pattern thông thường.

