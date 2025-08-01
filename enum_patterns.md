## 5 Powerful Enum-Based Patterns in Java

---

### 1. Enum-based Strategy Pattern

**Lý thuyết:** Mỗi hằng số enum biểu diễn một chiến lược khác nhau. Enum implement một interface hoặc override method để thực thi hành vi cụ thể cho từng strategy.

**Code cũ (dùng if/else hoặc switch):**

```java
public class Calculator {
    public double calculate(String operation, double a, double b) {
        switch (operation) {
            case "ADD": return a + b;
            case "SUB": return a - b;
            case "MUL": return a * b;
            case "DIV": return a / b;
            default: throw new IllegalArgumentException("Unknown op");
        }
    }
}
```

**Code mới (enum strategy):**

```java
public enum Operation {
    ADD { public double apply(double a, double b) { return a + b; } },
    SUB { public double apply(double a, double b) { return a - b; } },
    MUL { public double apply(double a, double b) { return a * b; } },
    DIV { public double apply(double a, double b) { return a / b; } };

    public abstract double apply(double a, double b);
}

// Sử dụng
Operation op = Operation.ADD;
double result = op.apply(10, 5);
```

---

### 2. Enum-based Command Pattern

**Lý thuyết:** Enum đóng vai trò như command object. Mỗi constant override phương thức `execute()`.

**Code cũ (dùng lớp Command riêng):**

```java
interface Command {
    void execute();
}
class SaveCommand implements Command {
    public void execute() { System.out.println("Saving..."); }
}
class DeleteCommand implements Command {
    public void execute() { System.out.println("Deleting..."); }
}
```

**Code mới (enum command):**

```java
public enum Command {
    SAVE {
        public void execute() { System.out.println("Saving..."); }
    },
    DELETE {
        public void execute() { System.out.println("Deleting..."); }
    };

    public abstract void execute();
}

// Sử dụng
Command.SAVE.execute();
```

---

### 3. Enum as Factory

**Lý thuyết:** Enum constant có thể đóng vai trò là factory tạo ra đối tượng tương ứng.

**Code cũ (dùng switch để tạo object):**

```java
public class NotificationFactory {
    public Notification create(String type) {
        switch (type) {
            case "EMAIL": return new EmailNotification();
            case "SMS": return new SmsNotification();
            default: throw new IllegalArgumentException();
        }
    }
}
```

**Code mới (enum factory):**

```java
public interface Notification {}
class EmailNotification implements Notification {}
class SmsNotification implements Notification {}

public enum NotificationType {
    EMAIL {
        public Notification create() { return new EmailNotification(); }
    },
    SMS {
        public Notification create() { return new SmsNotification(); }
    };

    public abstract Notification create();
}

// Sử dụng
Notification notif = NotificationType.EMAIL.create();
```

---

### 4. Enum-based State Machine

**Lý thuyết:** Mỗi trạng thái trong enum biểu diễn một bước của state machine. Có thể có phương thức `next()` để xác định chuyển trạng thái.

**Code cũ (dùng biến trạng thái + if):**

```java
String state = "START";
if (state.equals("START")) {
    state = "PROCESSING";
} else if (state.equals("PROCESSING")) {
    state = "DONE";
}
```

**Code mới (enum state):**

```java
public enum State {
    START {
        public State next() { return PROCESSING; }
    },
    PROCESSING {
        public State next() { return DONE; }
    },
    DONE {
        public State next() { return DONE; }
    };

    public abstract State next();
}

// Sử dụng
State state = State.START;
state = state.next();
```

---

### 5. Enum-based Validators

**Lý thuyết:** Enum đóng vai trò các validator khác nhau cho từng kiểu dữ liệu. Dùng abstract method `validate()` để thực thi.

**Code cũ (dùng if để xác định validator):**

```java
public boolean validate(String type, String input) {
    if (type.equals("EMAIL")) return input.contains("@");
    if (type.equals("PHONE")) return input.matches("\\d{10}");
    return false;
}
```

**Code mới (enum validator):**

```java
public enum Validator {
    EMAIL {
        public boolean validate(String input) {
            return input.contains("@");
        }
    },
    PHONE {
        public boolean validate(String input) {
            return input.matches("\\d{10}");
        }
    };

    public abstract boolean validate(String input);
}

// Sử dụng
boolean isValid = Validator.EMAIL.validate("test@example.com");
```

---

## Tổng kết

- Enum có thể đóng nhiều vai trò: Strategy, Command, Factory, State, Validator.
- Giúp code ngắn gọn, dễ mở rộng, giảm lỗi switch-case.
- Phù hợp với logic nhẹ, số lượng trạng thái/chiến lược cố định.
- Không nên lạm dụng nếu logic phức tạp hoặc thay đổi thường xuyên.

