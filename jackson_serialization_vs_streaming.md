# 📦 Serialization vs Deserialization trong Java (Jackson)

## 🔄 Serialization là gì?

Serialization là quá trình chuyển đổi một object trong bộ nhớ thành dạng dữ liệu có thể lưu trữ hoặc truyền đi được, ví dụ như:

- Dạng JSON, XML, binary, YAML, v.v.
- Để ghi vào file, truyền qua mạng, hoặc lưu vào database.

Ví dụ:

```java
ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(user);  // user là một Java object
```

Kết quả:

```json
{"id": 1, "name": "Alice"}
```

---

## 🔁 Deserialization là gì?

Deserialization là quá trình ngược lại: chuyển từ dữ liệu (JSON, XML, binary...) trở về object trong chương trình.

```java
User user = mapper.readValue(json, User.class);
```

Dữ liệu JSON:

```json
{"id": 1, "name": "Alice"}
```

→ sẽ được deserialize thành đối tượng `User`.

---

## 📌 Khi nào nên dùng?

| Trường hợp | Nên dùng |
|-----------|----------|
| JSON nhỏ, đơn giản | ✅ DataBinding (`ObjectMapper`) |
| JSON lớn, hiệu suất quan trọng | ✅ Streaming API (`JsonParser`) |
| Dữ liệu stream, log, realtime | ✅ Streaming API |

---

## 🔧 `JsonUtils` – class dùng chung

```java
public class JsonUtils {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    // Databinding: đọc object
    public static <T> T fromJson(File file, Class<T> clazz) throws IOException {
        return MAPPER.readValue(file, clazz);
    }

    // Databinding: đọc danh sách
    public static <T> List<T> fromJsonList(File file, Class<T[]> clazz) throws IOException {
        T[] array = MAPPER.readValue(file, clazz);
        return Arrays.asList(array);
    }

    // Serialize object thành JSON
    public static String toJson(Object obj) throws IOException {
        return MAPPER.writeValueAsString(obj);
    }

    // Streaming API: đọc từng object từ mảng JSON lớn
    public static void streamJsonArray(File file, Consumer<Map<String, JsonNode>> handler) throws IOException {
        JsonFactory factory = new JsonFactory();
        JsonParser parser = factory.createParser(file);
        ObjectMapper tempMapper = new ObjectMapper();

        if (parser.nextToken() != JsonToken.START_ARRAY) return;

        while (parser.nextToken() == JsonToken.START_OBJECT) {
            JsonNode node = tempMapper.readTree(parser);
            Map<String, JsonNode> fieldMap = new HashMap<>();
            node.fields().forEachRemaining(entry -> fieldMap.put(entry.getKey(), entry.getValue()));
            handler.accept(fieldMap);
        }

        parser.close();
    }
}
```

---

## 🧪 Ví dụ sử dụng

### ✅ Databinding (`ObjectMapper`)

```java
List<User> users = JsonUtils.fromJsonList(new File("users.json"), User[].class);
```

- Thích hợp với JSON nhỏ, cấu trúc ổn định
- Dễ đọc, dễ dùng

---

### ✅ Streaming (`JsonParser`)

```java
JsonUtils.streamJsonArray(new File("users.json"), fieldMap -> {
    String name = fieldMap.get("name").asText();
    int age = fieldMap.get("age").asInt();
    if (age > 30) {
        System.out.println(name + " is older than 30");
    }
});
```

- Không cần đọc toàn bộ file vào bộ nhớ
- Tốt khi xử lý file JSON lớn (log, dữ liệu nhiều record)

---

## ⚙️ Benchmark: Jackson DataBinding vs Streaming

| Phương pháp        | Thời gian xử lý | Ghi chú |
|--------------------|------------------|--------|
| DataBinding         | ~1500–2500 ms    | Toàn bộ file được deserialize vào memory |
| Streaming + POJO   | ~700–1000 ms     | Đọc từng record vào object |
| Streaming + Raw    | ~400–700 ms      | Chỉ đọc các trường cần, không tạo POJO |

---
