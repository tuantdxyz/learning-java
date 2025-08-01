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

**Ghi chú:**  
- Có thể mở rộng file này theo hướng: hướng dẫn cho controller/service, các mẫu exception handler Spring, response body chuẩn hóa lỗi.
