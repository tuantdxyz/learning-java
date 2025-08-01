# Tổng hợp Best Practices xử lý file - Part 01

---

## 1. Xử lý file trong Java

### 1.1. Đọc/Ghi file theo kích thước và loại dữ liệu

| Kích thước file    | Loại dữ liệu            | Giải pháp đề xuất                            | Ưu điểm chính                  |
|--------------------|------------------------|---------------------------------------------|-------------------------------|
| Nhỏ (<10MB)        | Text hoặc Binary        | `Files.readAllBytes()` hoặc `Files.readString()` | Dễ dùng, code ngắn gọn        |
| Trung bình (10-100MB) | Text                  | `BufferedReader` + `BufferedWriter`         | Đọc ghi dòng hiệu quả, dễ xử lý chuỗi |
| Trung bình (10-100MB) | Binary                | `BufferedInputStream` + `BufferedOutputStream` | Đơn giản, hiệu quả, ổn định   |
| Lớn (>100MB)       | Text hoặc Binary        | Java NIO `FileChannel` + `ByteBuffer`       | Hiệu suất cao, tối ưu bộ nhớ  |

-  ví dụ như ảnh, video, file zip,.. là thuộc loại Binary (file nhị phân)
-  ví dụ như file log, file config, file CSV, JSON, TXT,.. là thuộc loại Text
---

### 1.2. Xử lý ZIP/UNZIP

- **File nhỏ đến trung bình:**  
  + Dùng thư viện chuẩn Java `java.util.zip` (ZipInputStream, ZipOutputStream).  
  + Xử lý stream từng entry, buffer 8KB - 64KB.

- **File lớn:**  
  + Không giải nén toàn bộ vào RAM.  
  + Stream giải nén từng phần, buffer lớn, đa luồng nếu cần.

- **Thư viện bên thứ 3:**  
  + Zip4j (hỗ trợ password).  
  + Apache Commons Compress (đa định dạng).

---

## 2. Hiển thị file trên frontend

### 2.1. File nhỏ và vừa (< vài chục MB)

- Dùng `fetch` lấy file dạng Blob hoặc ArrayBuffer.  
- Hiển thị trực tiếp ảnh, video, audio, PDF qua tag HTML (`<img>`, `<video>`, `<audio>`, `<iframe>`).  
- Tạo link tải qua `URL.createObjectURL(blob)`.

### 2.2. File lớn (> 100MB)

- Dùng kỹ thuật streaming, tải file theo chunk (Range HTTP Header).  
- Hiển thị dần (như video streaming).  
- Dùng API hỗ trợ streaming như Media Source Extensions (MSE) cho video/audio.

### 2.3. Hiển thị file ZIP trên frontend

- Thường backend giải nén, gửi nội dung cụ thể.  
- Nếu cần giải nén client, dùng thư viện JS như [JSZip](https://stuk.github.io/jszip/).

---

## 3. Ví dụ bài toán thực tế ứng dụng

| Trường hợp       | Bài toán minh họa                                   | Công nghệ & Lý do sử dụng                                  |
|------------------|----------------------------------------------------|------------------------------------------------------------|
| File nhỏ         | Sửa file config JSON, đọc toàn bộ, chỉnh sửa giá trị | `Files.readString()` + Jackson JSON                        |
| File trung bình  | Phân tích log lớn, đếm lỗi, lọc dòng lỗi ra file riêng | `BufferedReader` + `BufferedWriter` (text)                |
| File trung bình  | Copy file ảnh/video, file nhị phân                  | `BufferedInputStream` + `BufferedOutputStream` (binary)    |
| File lớn         | Copy video lớn đồng thời đếm marker codec phức tạp | Java NIO `FileChannel` + `ByteBuffer` + buffer lớn         |

---

## 4. Giải thích về ký tự đặc biệt thường gặp

| Ký tự/Byte       | Ý nghĩa                          | Ứng dụng thực tế                                      |
|------------------|---------------------------------|------------------------------------------------------|
| `0x0A`           | Newline (xuống dòng) Unix/Linux | Đếm số dòng file log, phân tách dữ liệu text         |
| `0xFF`           | Byte đặc biệt trong file nhị phân | Marker trong ảnh JPEG, dấu hiệu protocol phần cứng    |
| `0x00 0x00 0x01` | Sequence marker trong video codec | Dò marker trong file video MPEG, H264                  |

---

## 5. Mẫu code xử lý file thực tế

### 5.1. File nhỏ (<10MB) — Đọc, sửa file config JSON

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.io.IOException;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;

public class SmallFileJsonConfig {
    public static void main(String[] args) {
        Path configPath = Paths.get("config.json");
        Path outputPath = Paths.get("config-updated.json");
        ObjectMapper mapper = new ObjectMapper();

        try {
            // Đọc toàn bộ JSON thành Map
            Map<String, Object> config = mapper.readValue(configPath.toFile(), Map.class);

            // Thay đổi giá trị cấu hình
            config.put("maxThreads", 20);

            // Ghi lại file mới
            mapper.writerWithDefaultPrettyPrinter().writeValue(outputPath.toFile(), config);

            System.out.println("Cập nhật config.json thành công!");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 5.2. File trung bình (10MB - 100MB) — Phân tích log lớn, đếm lỗi, lọc lỗi ra file riêng

```java
public class MediumFileLogAnalyzer {
    public static void main(String[] args) {
        File inputLog = new File("app.log");
        File errorLog = new File("error.log");

        int errorCount = 0;

        try (BufferedReader br = new BufferedReader(new FileReader(inputLog));
             BufferedWriter bw = new BufferedWriter(new FileWriter(errorLog))) {

            String line;
            while ((line = br.readLine()) != null) {
                if (line.contains("ERROR")) {
                    errorCount++;
                    bw.write(line);
                    bw.newLine();
                }
            }

            System.out.println("Tổng số lỗi: " + errorCount);
            System.out.println("File lỗi lưu tại: " + errorLog.getAbsolutePath());

        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 5.3. File lớn (>100MB) — Copy video lớn, đếm marker codec

```java
public class LargeFileVideoProcessor {
    public static void main(String[] args) {
        try (RandomAccessFile srcFile = new RandomAccessFile("video-large.mp4", "r");
             RandomAccessFile destFile = new RandomAccessFile("video-copy.mp4", "rw");
             FileChannel srcChannel = srcFile.getChannel();
             FileChannel destChannel = destFile.getChannel()) {

            ByteBuffer buffer = ByteBuffer.allocateDirect(64 * 1024); // 64KB buffer
            long markerCount = 0;

            byte[] prevBytes = new byte[2];
            int prevLen = 0;

            while (srcChannel.read(buffer) != -1) {
                buffer.flip();

                byte[] data = new byte[buffer.limit()];
                buffer.get(data);

                for (int i = 0; i < data.length; i++) {
                    if (prevLen == 2 && data[i] == 0x01 &&
                        prevBytes[0] == 0x00 && prevBytes[1] == 0x00) {
                        markerCount++;
                    }
                    if (i < data.length - 2) {
                        prevBytes[0] = data[i];
                        prevBytes[1] = data[i + 1];
                        i++;
                    }
                }

                buffer.clear();
                buffer.put(data);
                buffer.flip();
                destChannel.write(buffer);
                buffer.clear();
            }

            System.out.println("Copy video xong.");
            System.out.println("Số marker 0x000001 trong file video: " + markerCount);

        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
