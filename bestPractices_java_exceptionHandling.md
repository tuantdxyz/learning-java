# Best Practices and Pitfalls in Java Exception Handling

Tổng hợp các thực hành tốt (Best Practices) và sai lầm phổ biến (Pitfalls) khi xử lý ngoại lệ trong Java, áp dụng cho các dự án thực tế, đặc biệt phù hợp với hệ thống Spring Boot, Java 8+ trở lên.

---

## 1. Dùng exception cụ thể thay vì Exception hoặc Throwable chung chung
**Lý thuyết:** Tránh `catch (Exception)` hoặc `catch (Throwable)` vì không rõ loại lỗi gì, gây khó debug và xử lý không chính xác.

**Code cũ:**
```java
try {
    doSomething();
} catch (Exception e) {
    // xử lý mơ hồ
}
```

**Code mới:**
```java
try {
    doSomething();
} catch (IOException e) {
    // xử lý lỗi IO cụ thể
}
```

---

## 2. Chỉ catch exception khi thật sự cần thiết, tránh empty catch block
**Lý thuyết:** Không xử lý hoặc ghi log khi bắt lỗi khiến lỗi bị nuốt mất, khó trace khi gặp sự cố.

**Code cũ:**
```java
try {
    readFile();
} catch (IOException e) {
    // bỏ qua
}
```

**Code mới:**
```java
try {
    readFile();
} catch (IOException e) {
    log.error("Lỗi đọc file", e);
}
```

---

## 3. Sử dụng logging framework (Logback, Log4j, SLF4J), không dùng System.out.println
**Lý thuyết:** Hệ thống logging cho phép điều chỉnh log level, ghi file, gửi qua remote... tiện lợi và dễ quản trị hơn.

**Code cũ:**
```java
System.out.println("Lỗi: " + e);
```

**Code mới:**
```java
log.error("Có lỗi xảy ra", e);
```

---

## 4. Không dùng exception để điều khiển luồng chính
**Lý thuyết:** Gây ảnh hưởng performance và khiến code khó đọc, rối logic.

**Code cũ:**
```java
try {
    Integer.parseInt(value);
} catch (NumberFormatException e) {
    return 0;
}
```

**Code mới:**
```java
if (value.matches("\d+")) {
    return Integer.parseInt(value);
} else {
    return 0;
}
```

---

## 5. Dùng try-with-resources để tự động đóng tài nguyên
**Lý thuyết:** Tránh rò rỉ tài nguyên khi làm việc với stream, reader, connection...

**Code cũ:**
```java
BufferedReader reader = new BufferedReader(new FileReader("file.txt"));
try {
    return reader.readLine();
} finally {
    reader.close();
}
```

**Code mới:**
```java
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    return reader.readLine();
}
```

---

## 6. Sử dụng exception chaining để giữ stack trace và thêm context
**Lý thuyết:** Bao lại exception gốc bên trong custom exception, giúp giữ thông tin nguyên nhân ban đầu.

**Code cũ:**
```java
hrow new BusinessException("Lỗi xử lý user"); // mất trace gốc (e)
```

**Code mới:**
```java
catch (SQLException e) {
    throw new DataAccessException("Không thể truy vấn DB", e);
}
```

---

## 7. Khai báo checked exception cụ thể thay vì throws Exception chung chung
**Lý thuyết:** Giúp người gọi biết rõ phương thức có thể ném lỗi gì để xử lý đúng.

**Code cũ:**
```java
public void doTask() throws Exception
```

**Code mới:**
```java
public void doTask() throws IOException, SQLException
```

---

## 8. Tránh vừa log vừa throw exception
**Lý thuyết:** Gây trùng log nếu phía trên cũng log lại. Chỉ log hoặc throw tùy theo cấp xử lý.

**Code cũ:**
```java
log.error("Lỗi", e);
throw e;
```

**Code mới:**
```java
throw e; // hoặc log nếu không throw tiếp
```

---

## 9. Không throw exception từ trong finally block
**Lý thuyết:** Có thể làm mất trace exception ban đầu nếu ghi đè trong finally.

**Code cũ:**
```java
try {
    doSomething();
} finally {
    throw new RuntimeException("Fail ở cuối");
}
```

**Code mới:**
```java
try {
    doSomething();
} finally {
    cleanup();
}
```

---

## 10. Viết thông điệp exception rõ ràng và có context
**Lý thuyết:** Giúp dễ hiểu nguyên nhân lỗi và cải thiện log/debug.

**Code cũ:**
```java
throw new IllegalArgumentException("Sai");
```

**Code mới:**
```java
throw new IllegalArgumentException("Giá trị userId không được null");
```

---

## Các nguyên tắc bổ sung:

- Không bắt exception chung trong tầng thấp nếu không cần.
- Không xử lý logic nghiệp vụ trong catch.
- Không dùng exception cho luồng bình thường.
- Đóng tài nguyên đúng cách.
- Không swallow lỗi.
- Không catch exception không được throw từ block đó.
- Tránh `catch(Throwable t)` trừ khi thực sự cần chặn toàn bộ hệ thống.
- Tránh `throws Exception` trong signature nếu không cần.
- Exception nên được gom nhóm theo module để phân loại rõ ràng.
- Custom Exception nên theo chuẩn (có constructors truyền message + cause).
- Logging nên theo từng lớp (per class logger).
- Global Exception Handler nên tách biệt rõ ràng controller/service.
- Nếu dùng Spring Boot: dùng @ControllerAdvice + ResponseEntityExceptionHandler.
- Không dùng Exception cho validate đầu vào nếu có thể kiểm tra được logic trước.
- Chỉ throw RuntimeException nếu thật sự là lỗi nghiêm trọng.
- Exception dùng trong API nên có mã lỗi (error code) và thông điệp cụ thể.

---

## Lý thuết:

1. Phân biệt Exception trong Java
2. `throw` vs `throws`
3. `Checked` vs `Unchecked` Exception
4. Custom Exception
5. GlobalExceptionHandler trong Spring Boot
6. Chuẩn hóa mã lỗi (code, message)

---

## 1. Phân biệt Exception trong Java

| Loại | Mô tả |
| `Checked`   | Bắt buộc phải xử lý (compile-time). Ví dụ: `IOException`, `SQLException`                       |
| ------------- | ---------------------------------------------------------------------------------------------- |
| `Unchecked` | Không bắt buộc phải xử lý (runtime). Ví dụ: `NullPointerException`, `IllegalArgumentException` |

📌 **Nguyên tắc**:

- Dùng **checked exception** khi: caller có thể xử lý lỗi được.
- Dùng **unchecked exception** khi: lỗi là do bug hoặc logic sai mà caller **không xử lý được**.
- Khái niệm: Compile-time (Checked Exception): Xảy ra khi biên dịch. Trình biên dịch sẽ báo lỗi nếu không xử lý. Đối tượng exception này thường do ngoại cảnh (I/O, DB...).
- Khái niệm: Runtime (Unchecked Exception): Xảy ra khi chương trình đang chạy. Thường do bug, sai logic.

---

## 2. `throw` vs `throws`

| Từ khóa  | Ý nghĩa                                               |
| -------- | ----------------------------------------------------- |
| `throw`  | Dùng để ném một instance của Exception                |
| `throws` | Dùng để khai báo method có thể ném loại exception nào |

🔍 **Ví dụ:**

```java
public void readFile(String path) throws IOException {
    if (path == null) {
        throw new IllegalArgumentException("Path must not be null"); // unchecked
    }
    Files.readAllLines(Path.of(path)); // checked -> phải throws IOException
}
```

---

## 3. Checked vs Unchecked Exception - Tình huống sử dụng

| Tình huống                                    | Exception nên dùng                   | Giải thích                   |
| --------------------------------------------- | ------------------------------------ | ---------------------------- |
| Gọi đến API, DB, file system                  | Checked (IOException, etc)           | Caller nên biết để retry/log |
| Tham số đầu vào sai (null, invalid enum, ...) | Unchecked (IllegalArgumentException) | Do bug nghiệp vụ             |
| Tính toán nội bộ lỗi (divide by 0)            | Unchecked (ArithmeticException)      | Không recover được           |

---

## 4. Custom Exception
Dùng khi muốn tạo thông báo lỗi có ngữ nghĩa hơn với context cụ thể.

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

public class BusinessException extends RuntimeException {
    private final String errorCode;

    public BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public String getErrorCode() {
        return errorCode;
    }
}
```

---

## 5. GlobalExceptionHandler trong Spring Boot

📁 Package structure:

```
com.example
├── controller
├── service
├── exception
│   ├── GlobalExceptionHandler.java
│   ├── ApiError.java
│   ├── ResourceNotFoundException.java
│   └── BusinessException.java
```
✅ Dễ mở rộng: thêm các BusinessException, ValidationException tùy logic.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404).body(new ApiError("NOT_FOUND", ex.getMessage()));    // Mã lỗi 404 được hiểu là thiếu resource
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleGeneric(Exception ex) {
        return ResponseEntity.status(500).body(new ApiError("INTERNAL_ERROR", "Unexpected error"));
    }

    public record ApiError(String code, String message) {}
}
```

---

## 6. Chuẩn hóa trả mã lỗi (Error Code + Message)

| Code             | Message                       | HTTP Status |
| ---------------- | ----------------------------- | ----------- |
| `INVALID_INPUT`  | Dữ liệu nhập vào không hợp lệ | `400`       |
| `NOT_FOUND`      | Không tìm thấy resource       | `404`       |
| `INTERNAL_ERROR` | Lỗi hệ thống                  | `500`       |


