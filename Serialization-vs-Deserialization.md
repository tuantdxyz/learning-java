## Serialization là gì?
Serialization là quá trình chuyển đổi một object trong bộ nhớ thành dạng dữ liệu có thể lưu trữ hoặc truyền đi được, ví dụ như:
`*` Dạng JSON, XML, binary, YAML, v.v.
`*` Để ghi vào file, truyền qua mạng, hoặc lưu vào database.

ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(user);  // user là một Java object

user được serialize thành một chuỗi JSON như:
--> {"id": 1, "name": "Alice"}

##  Deserialization là gì?
Deserialization là quá trình ngược lại: chuyển từ dữ liệu (JSON, XML, binary...) trở về object trong chương trình.

User user = mapper.readValue(json, User.class);
Dữ liệu JSON {"id":1, "name":"Alice"} sẽ được deserialize thành đối tượng User.

## Trường hợp nên dùng
`*` JSON nhỏ, đơn giản	DataBinding (ObjectMapper)
`*` Hiệu suất quan trọng, JSON lớn, nhiều field	Streaming API (JsonParser)
`*` Xử lý realtime, stream từ mạng, log parser	(Streaming API)

## Common cấu trúc JsonUtils
public class JsonUtils {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    // Dùng cho databinding
    public static <T> T fromJson(File file, Class<T> clazz) throws IOException {
        return MAPPER.readValue(file, clazz);
    }

    public static <T> List<T> fromJsonList(File file, Class<T[]> clazz) throws IOException {
        T[] array = MAPPER.readValue(file, clazz);
        return Arrays.asList(array);
    }

    public static String toJson(Object obj) throws IOException {
        return MAPPER.writeValueAsString(obj);
    }

    // Streaming (read-only parsing)
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

## Ví dụ
DataBinding (ObjectMapper):
List<User> users = JsonUtils.fromJsonList(new File("users.json"), User[].class);
--> phù hợp khi JSON nhỏ và có cấu trúc ổn định.

Streaming (JsonParser):
JsonUtils.streamJsonArray(new File("users.json"), fieldMap -> {
    String name = fieldMap.get("name").asText();
    int age = fieldMap.get("age").asInt();
    if (age > 30) System.out.println(name + " is older than 30");
});
--> Hiệu quả với file JSON rất lớn — không cần đọc toàn bộ vào bộ nhớ.

## Benchmark Jackson: DataBinding vs Streaming
Hiệu suất giữa Jackson DataBinding (ObjectMapper) và Streaming (JsonParser) với file lớn (~100,000 records).
DataBinding	~1500–2500 ms	Toàn bộ file nạp vào bộ nhớ
Streaming + POJO	~700–1000 ms	Xử lý tuần tự từng record
Streaming + Raw	~400–700 ms	Nếu không dùng POJO
