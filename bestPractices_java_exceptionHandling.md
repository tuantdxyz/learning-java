# Best Practices and Pitfalls in Java Exception Handling

Tổng hợp 34 mục về xử lý ngoại lệ (exception handling) trong Java áp dụng cho dự án thực tế. Mỗi mục gồm lý thuyết, ví dụ code cũ nên tránh và code mới nên dùng.

---

## 1. Dùng exception cụ thể thay vì Exception hoặc Throwable chung chung
**Lý thuyết:** Tránh `catch (Exception)` hay `catch (Throwable)` vì không rõ loại lỗi gì, khó debug.

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
    // xử lý IO cụ thể
}
```

## 2. Chỉ catch exception khi cần thiết, tránh empty catch block
**Lý thuyết:** Catch nhưng không xử lý gì dễ gây mất dấu lỗi.

**Code cũ:**
```java
try {
    readFile();
} catch (IOException e) {
    // bỏ qua lỗi
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

## 3. Sử dụng logging framework thay cho System.out.println
**Lý thuyết:** Logging tốt hơn, có log level, dễ redirect log.

**Code cũ:**
```java
System.out.println("Lỗi: " + e);
```

**Code mới:**
```java
log.error("Có lỗi xảy ra", e);
```

## 4. Không dùng exception để điều khiển luồng
**Lý thuyết:** Gây chậm và rối logic nếu dùng exception như if-else.

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

## 5. Dùng try-with-resources để tự đóng tài nguyên
**Lý thuyết:** Tránh quên đóng stream, connection.

**Code cũ:**
```java
BufferedReader br = new BufferedReader(new FileReader("file.txt"));
try {
    return br.readLine();
} finally {
    br.close();
}
```

**Code mới:**
```java
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    return br.readLine();
}
```

...

## 34. Chỉ catch những exception mà phương thức throws
**Lý thuyết:** Tránh bắt exception không rõ nguồn gốc.

**Code cũ:**
```java
try {
    risky();
} catch (Exception e) {
    // catch chung
}
```

**Code mới:**
```java
try {
    risky();
} catch (IOException e) {
    // xử lý rõ ràng
} catch (SQLException e) {
    // xử lý SQL
}
```

---

**Chú thích:**  
- Các ví dụ phù hợp với Java 8+ và áp dụng được trong Spring Boot, hệ thống backend enterprise.
- Có thể mở rộng cho controller advice, global exception handler, logging AOP nếu dùng Spring.
