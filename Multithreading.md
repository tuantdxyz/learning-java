# 🧵 Tổng hợp cơ chế đa luồng trong Java & ShedLock

## 1. ThreadPool
- **Khái niệm:** Gom nhóm số lượng thread cố định để xử lý nhiều task thay vì tạo mới liên tục.
- **Ưu điểm:** Tái sử dụng thread, giới hạn tài nguyên, dễ quản lý.
- **Nhược điểm:** Nếu task chặn IO → dễ nghẽn, cần cấu hình pool size hợp lý.

**Bài toán áp dụng:**  
- Batch xử lý file log lớn: chia file thành nhiều chunk, mỗi thread trong pool đọc và phân tích một phần.

---

## 2. CompletableFuture
- **Khái niệm:** Lập trình bất đồng bộ với callback, chaining, combine nhiều task song song.
- **Ưu điểm:** Code async dễ đọc, tránh callback hell, hỗ trợ pipeline nhiều bước.
- **Nhược điểm:** Vẫn dựa trên ThreadPool, debug callback phức tạp.

**Bài toán áp dụng:**  
- Gọi nhiều microservice song song: một request cần dữ liệu từ 3 service, dùng `CompletableFuture.allOf()` để gọi song song và combine kết quả.

---

## 3. Virtual Threads (Java 21 – Project Loom)
- **Khái niệm:** Thread nhẹ, có thể tạo hàng triệu thread mà không tốn nhiều RAM.
- **Ưu điểm:** Code đa luồng viết như tuần tự, scale tốt cho IO-bound.
- **Nhược điểm:** Công nghệ mới, một số thư viện chưa tương thích hoàn toàn.

**Bài toán áp dụng:**  
- Web server nhiều kết nối IO: mỗi request chạy trên một virtual thread riêng, dễ scale lên hàng chục nghìn request/giây.

---

## 4. Tranh chấp tài nguyên
- Dù dùng ThreadPool, CompletableFuture hay Virtual Threads, **tranh chấp tài nguyên vẫn xảy ra** nếu nhiều thread cùng truy cập một resource.
- **Trong cùng JVM:** dùng `synchronized`, `ReentrantLock`, hoặc `Atomic*` để đảm bảo an toàn.
- **Trong hệ thống phân tán:** cần distributed lock để tránh nhiều node chạy cùng một job.

---

## 5. ShedLock
- **Khái niệm:** Thư viện giúp đảm bảo chỉ một instance trong cluster chạy job định kỳ tại một thời điểm.
- **Cách hoạt động:** Dùng DB/Redis để lock job, tránh duplicate execution.
- **Best practice:**  
  - Dùng ShedLock cho scheduled tasks trong cluster.  
  - Kết hợp với ThreadPool/Virtual Threads để xử lý song song bên trong một node.  

**Ví dụ Spring Boot với ShedLock:**
```java
@Scheduled(cron = "0 0 * * * ?")
@SchedulerLock(name = "processLogJob", lockAtMostFor = "10m", lockAtLeastFor = "1m")
public void processLogJob() {
    // chỉ một node trong cluster chạy job này
    logService.processLogs();
}
```

## 6. Volatile & AtomicInteger
volatile: đảm bảo visibility (các thread thấy cùng một giá trị biến), nhưng không giải quyết race condition.

AtomicInteger: cung cấp các phép toán nguyên tử (atomic operations) như incrementAndGet(), tránh race condition khi nhiều thread cùng cập nhật biến.

Best practice:

Dùng volatile cho flag điều khiển (ví dụ: isRunning).

Dùng AtomicInteger hoặc LongAdder cho counter.

Dùng synchronized hoặc lock cho logic phức tạp.

## 7. Chống OutOfMemory & Lưu ý
ThreadPool size hợp lý: tránh tạo quá nhiều thread gây OOM.

Virtual Threads: giảm nguy cơ OOM vì mỗi thread nhẹ hơn.

Sử dụng queue: giới hạn số lượng task chờ.

Dọn dẹp tài nguyên: đóng file, socket, connection sau khi dùng.

Monitoring: theo dõi heap, GC, thread count để phát hiện sớm vấn đề.

Redis TTL: khi revoke JWT, dùng TTL để tránh phình to bộ nhớ.

📂 Bài toán: Đọc và xử lý file 500MB
File lớn (500MB) cần được phân tích hoặc chuyển đổi dữ liệu.

Nếu đọc tuần tự → mất nhiều thời gian.

Giải pháp: chia file thành nhiều chunk (ví dụ 50MB mỗi chunk), mỗi thread xử lý một phần song song.


⚙️ Cách triển khai
1. Dùng ThreadPool
```java
ExecutorService pool = Executors.newFixedThreadPool(10);
Path path = Paths.get("data.bin");
long fileSize = Files.size(path);
long chunkSize = 50 * 1024 * 1024; // 50MB

for (long offset = 0; offset < fileSize; offset += chunkSize) {
    final long start = offset;
    final long end = Math.min(offset + chunkSize, fileSize);

    pool.execute(() -> {
        try (RandomAccessFile raf = new RandomAccessFile(path.toFile(), "r")) {
            raf.seek(start);
            byte[] buffer = new byte[(int)(end - start)];
            raf.readFully(buffer);
            // xử lý dữ liệu buffer
        } catch (IOException e) {
            e.printStackTrace();
        }
    });
}
pool.shutdown();
```
Ưu điểm: tận dụng CPU đa lõi.

Nhược điểm: cần tính toán chunk size hợp lý, tránh tạo quá nhiều thread.

2. Dùng CompletableFuture
```java
List<CompletableFuture<Void>> futures = new ArrayList<>();

for (long offset = 0; offset < fileSize; offset += chunkSize) {
    final long start = offset;
    final long end = Math.min(offset + chunkSize, fileSize);

    futures.add(CompletableFuture.runAsync(() -> {
        try (RandomAccessFile raf = new RandomAccessFile(path.toFile(), "r")) {
            raf.seek(start);
            byte[] buffer = new byte[(int)(end - start)];
            raf.readFully(buffer);
            // xử lý dữ liệu buffer
        } catch (IOException e) {
            e.printStackTrace();
        }
    }));
}

CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();

```
Ưu điểm: dễ quản lý nhiều bước xử lý (pipeline).

Nhược điểm: vẫn dựa trên ThreadPool, cần cấu hình hợp lý.

3. Dùng Virtual Threads (Java 21)
```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (long offset = 0; offset < fileSize; offset += chunkSize) {
        final long start = offset;
        final long end = Math.min(offset + chunkSize, fileSize);

        executor.submit(() -> {
            try (RandomAccessFile raf = new RandomAccessFile(path.toFile(), "r")) {
                raf.seek(start);
                byte[] buffer = new byte[(int)(end - start)];
                raf.readFully(buffer);
                // xử lý dữ liệu buffer
            }
            return null;
        });
    }
}

```
Ưu điểm: có thể tạo hàng nghìn virtual thread để xử lý song song mà không lo OutOfMemory.

Nhược điểm: công nghệ mới, cần Java 21+.

---

# 📂 Bài toán xử lý file nặng (Excel, Word, PDF…)

## 1. Đặc thù từng loại file
- **Excel (XLSX):** thực chất là file ZIP chứa nhiều XML (mỗi sheet là một file riêng).  
  → Không nên chia chunk raw 50MB, mà dùng **Apache POI SXSSF** hoặc streaming API để đọc sheet theo row.  
- **Word (DOCX):** cũng là file ZIP chứa XML.  
  → Dùng thư viện (Apache POI, docx4j) để parse nội dung, không đọc raw chunk.  
- **PDF:** định dạng phức tạp, có cấu trúc trang, font, hình ảnh.  
  → Dùng thư viện (PDFBox, iText) để đọc theo trang hoặc stream nội dung.

---

## 2. Chiến lược xử lý đa luồng
- **Excel:**  
  - Đọc từng sheet bằng thread riêng.  
  - Nếu sheet quá lớn → đọc row theo streaming, đẩy vào queue, nhiều worker thread xử lý song song.  

- **Word:**  
  - Tách từng section hoặc chapter, mỗi thread xử lý một phần.  
  - Với file lớn, nên stream text thay vì load toàn bộ vào RAM.  

- **PDF:**  
  - Tách theo trang, mỗi thread xử lý một page.  
  - Phù hợp cho phân tích nội dung hoặc trích xuất dữ liệu song song.  

---

## 3. Công nghệ áp dụng
- **ThreadPool:** giới hạn số thread, xử lý song song theo sheet/trang/section.  
- **CompletableFuture:** khi cần pipeline nhiều bước (đọc → phân tích → ghi DB).  
- **Virtual Threads (Java 21+):** phù hợp cho IO-bound, nhiều request đọc file lớn, code đơn giản như tuần tự.  
- **ShedLock:** nếu job chạy trên nhiều node trong cluster, đảm bảo chỉ một node xử lý file tại một thời điểm.

---

## 4. Lưu ý quan trọng
- **Không chia chunk raw (50MB) cho Excel/Word/PDF**: vì sẽ phá vỡ cấu trúc ZIP/XML/PDF.  
- **Đọc theo đơn vị logic**: sheet, row, section, page.  
- **Đồng bộ khi ghi**: nhiều thread ghi vào cùng file → cần lock hoặc queue.  
- **Chống OutOfMemory:**  
  - Dùng streaming API (SXSSF cho Excel, PDFBox cho PDF).  
  - Giới hạn thread pool size.  
  - Dọn dẹp tài nguyên (close file, stream).  
- **Monitoring:** theo dõi heap, GC, thread count để phát hiện sớm vấn đề.

---

## 📌 Tổng kết
- **Excel:** đọc theo sheet/row bằng streaming API.  
- **Word:** đọc theo section/chapter, tránh load toàn bộ.  
- **PDF:** đọc theo page, xử lý song song.  
- **ThreadPool/CompletableFuture/Virtual Threads:** chọn tùy bài toán (batch, pipeline, IO-bound).  
- **ShedLock:** cần thiết trong cluster để tránh duplicate job.  
- **Best practice:** xử lý song song theo đơn vị logic của file, không chia chunk raw, kết hợp lock và streaming để đảm bảo hiệu năng và an toàn.


** Lưu ý:
* Đọc tuần tự: mất vài giây đến vài chục giây tùy tốc độ ổ đĩa.

* Đọc đa luồng: nhanh hơn nếu ổ đĩa hỗ trợ truy cập song song (SSD, NVMe).

* Ghi file: thường phải tuần tự hoặc đồng bộ để tránh ghi đè, nên tốc độ cải thiện ít hơn đọc.

* Thực tế: với file 500MB trên SSD, đọc đa luồng có thể giảm thời gian từ ~5–10 giây xuống còn ~2–3 giây, tùy cấu hình hệ thống.

* ThreadPool: phù hợp cho batch job, xử lý file lớn.

* CompletableFuture: phù hợp cho async pipeline, nhiều bước xử lý song song.

* Virtual Threads: phù hợp cho hệ thống IO-bound, nhiều request, cần scale lớn.

* ShedLock: cần thiết trong hệ thống phân tán để tránh duplicate job.

* volatile & AtomicInteger: giải quyết visibility và race condition trong JVM.

* Best practice: kết hợp cơ chế đồng bộ trong JVM + ShedLock/distributed lock trong cluster, cấu hình thread hợp lý để tránh OOM.
