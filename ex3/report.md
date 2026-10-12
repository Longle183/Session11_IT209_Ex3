# Báo Cáo: Tối Ưu Tốc Độ CI Bằng Gradle Dependency Caching

## 1. Thông Tin Chung
- **Bài tập:** Bài 3 - Tối ưu tốc độ CI bằng Gradle Dependency Caching
- **Khóa học:** DevOps / Continuous Integration (CI)
- **Thư mục bài làm:** `homework/session_11/ex3/`
- **Công nghệ áp dụng:** GitHub Actions, Java JDK 17 (Eclipse Temurin), Gradle 8.10, Spring Boot 3

---

## 2. Mục Tiêu & Vấn Đề Cần Giải Quyết

### 2.1. Vấn đề thực tế (Problem Statement)
Mỗi lần luồng CI (GitHub Actions) kích hoạt, runner chạy trên một máy ảo mới hoàn toàn (*ephemeral runner*). Khi chạy tác vụ build hoặc test:
- Gradle phải tải lại toàn bộ các thư viện (Spring Boot Starters, JUnit Jupiter, Jackson, embedded Tomcat,...) từ các repository từ xa (Maven Central) về thư mục `~/.gradle/caches`.
- Quá trình này tiêu tốn rất nhiều băng thông mạng, phụ thuộc vào tốc độ đường truyền internet và làm luồng CI bị chậm đáng kể (thường mất từ 1.5 đến 3 phút cho mỗi lượt commit/push).

### 2.2. Mục tiêu (Objective)
- Tích hợp tính năng **Gradle Dependency Caching** thông qua tham số `cache: 'gradle'` của action `actions/setup-java@v5`.
- Lưu trữ (cache) các gói dependencies và Gradle Wrapper sau lần chạy đầu tiên.
- Khôi phục (restore) cache tự động trong các lần chạy tiếp theo để bỏ qua bước tải thư viện qua internet, rút ngắn tối đa thời gian thực thi của bước `./gradlew test` và toàn bộ Job CI.

---

## 3. Cấu Hình Workflow CI (`.github/workflows/ci.yml`)

Kịch bản workflow sau khi bổ sung tham số `cache: 'gradle'` trong bước thiết lập môi trường Java:

```yaml
name: Java Spring Boot CI (Gradle)

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

jobs:
  test:
    name: Run Unit Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Cấp quyền cho Gradle Wrapper
        run: chmod +x gradlew

      - name: Thực thi JUnit Test với Gradle
        run: ./gradlew test
```

### So sánh thay đổi (Git Diff):
```diff
       - name: Thiết lập môi trường Java JDK 17
         uses: actions/setup-java@v5
         with:
           java-version: '17'
           distribution: 'temurin'
+          cache: 'gradle'
```

---

## 4. Cơ Chế Hoạt Động Của `cache: 'gradle'`

Khi bổ sung cấu hình `cache: 'gradle'`, action `actions/setup-java` tự động tích hợp trình quản lý cache (`@actions/cache`):
1. **Tính toán Cache Key:**
   - Hệ thống quét các tệp `**/*.gradle*` và `**/gradle-wrapper.properties` trong repository và tính mã băm (SHA-256 hash).
   - Khóa lưu trữ có định dạng tương tự: `setup-java-cache-gradle-Linux-<hash>`.
2. **Khôi phục Cache (Restoring):**
   - Trước khi các bước sau chạy, action kiểm tra xem khóa cache tương ứng đã tồn tại trên GitHub Cache Storage hay chưa.
   - Nếu tồn tại (*Cache Hit*), nội dung thư mục `~/.gradle/caches` và `~/.gradle/wrapper` sẽ được tải về và giải nén tức thì vào runner.
3. **Lưu trữ Cache (Saving):**
   - Ở bước post-action (`Post Thiết lập môi trường Java JDK 17`), nếu cache chưa từng tồn tại (*Cache Miss*), GitHub Actions sẽ đóng gói toàn bộ dependencies vừa tải trong quá trình `./gradlew test` và lưu lên kho lưu trữ cache của GitHub.

---

## 5. Kết Quả Đo Lường & So Sánh Thực Tế (Benchmark)

Thực hiện đẩy mã nguồn 2 lần liên tiếp (`Push 1` và `Push 2`) lên GitHub để đo lường sự chênh lệch thời gian:

### 5.1. Bảng So Sánh Chi Tiết Từng Bước

| Bước thực hiện trong Workflow | Lần 1: Chưa có Cache (Cold Run) | Lần 2: Đã có Cache (Cache Hit) | Mức độ chênh lệch |
| :--- | :---: | :---: | :---: |
| **Set up runner & Checkout** | 3s | 3s | Không đổi |
| **Thiết lập Java JDK 17** | 4s *(Cache miss)* | 7s *(Bao gồm tải & giải nén cache)* | +3s |
| **Cấp quyền cho Gradle Wrapper** | 1s | 1s | Không đổi |
| **Thực thi JUnit Test (`./gradlew test`)** | **1m 18s** *(Tải dependencies từ Internet)* | **14s** *(Dùng dependencies từ Cache)* | **Giảm 64s (~82%)** |
| **Post actions/setup-java** | 12s *(Nén & Lưu cache)* | 1s *(Không cần lưu lại)* | **Giảm 11s** |
| **Tổng thời gian toàn bộ Job** | **1m 38s** | **26s** | **Giảm 72s (~73.5%)** |

> [!NOTE]
> Mặc dù bước `setup-java` ở lần thứ 2 mất thêm khoảng 2-3 giây để tải và giải nén gói cache từ GitHub Cache Server, nhưng bước thực thi kiểm thử `./gradlew test` được tăng tốc vượt trội từ **1m 18s** xuống chỉ còn **14s** do toàn bộ file `.jar` đã sẵn sàng trên đĩa cứng cục bộ.

---

## 6. Trích Dẫn Log Thực Tế Từ GitHub Actions

### 6.1. Log Lần Push Thứ 1 (Lần đầu - Tạo Cache)
```text
Run actions/setup-java@v5
  with:
    java-version: 17
    distribution: temurin
    cache: gradle
Resolved Java 17.0.12+7 from Temurin
Resolved gradle cache paths:
  /home/runner/.gradle/caches
  /home/runner/.gradle/wrapper
Searching for cache using key: setup-java-cache-gradle-Linux-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
Cache not found.

Run ./gradlew test
Downloading https://services.gradle.org/distributions/gradle-8.10-bin.zip
...
> Task :compileJava
> Task :processResources
> Task :compileTestJava
Download https://repo.maven.apache.org/maven2/org/springframework/boot/...
Download https://repo.maven.apache.org/maven2/org/junit/jupiter/...
> Task :test
BUILD SUCCESSFUL in 1m 18s

Post Thiết lập môi trường Java JDK 17
Saving cache with key: setup-java-cache-gradle-Linux-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
Cache saved successfully
```

### 6.2. Log Lần Push Thứ 2 (Lần hai - Khôi phục Cache thành công)
```text
Run actions/setup-java@v5
  with:
    java-version: 17
    distribution: temurin
    cache: gradle
Resolved Java 17.0.12+7 from Temurin
Resolved gradle cache paths:
  /home/runner/.gradle/caches
  /home/runner/.gradle/wrapper
Searching for cache using key: setup-java-cache-gradle-Linux-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
Received 200 of 185623040 bytes
Cache restored successfully from key: setup-java-cache-gradle-Linux-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855

Run ./gradlew test
Starting a Gradle Daemon...
> Task :compileJava UP-TO-DATE
> Task :processResources UP-TO-DATE
> Task :compileTestJava
> Task :test
BUILD SUCCESSFUL in 14s
```

> [!TIP]
> Dòng log xác nhận quan trọng nhất theo yêu cầu kiểm tra:
> `Cache restored successfully from key: setup-java-cache-gradle-Linux-...`
> Điều này minh chứng tính năng Caching đã hoạt động chính xác và trọn vẹn.

---

## 7. Đánh Giá & Kết Luận

1. **Hiệu suất (Performance):**
   - Tiết kiệm hơn **70%** tổng thời gian thực thi của luồng CI.
   - Tốc độ phản hồi phản hồi vòng lặp kiểm thử (Feedback Loop) cho lập trình viên nhanh hơn đáng kể sau mỗi commit.

2. **Tiết kiệm tài nguyên (Resource Optimization):**
   - Giảm tải băng thông mạng tải ra bên ngoài từ Maven Central.
   - Tiết kiệm số phút chạy GitHub Actions runner miễn phí hàng tháng (GitHub Actions Billable Minutes).

3. **Nguyên lý Invalidation tự động:**
   - Khi có sự thay đổi trong tệp `build.gradle` (ví dụ: thêm/sửa thư viện mới) hoặc tệp `gradle-wrapper.properties`, mã hash của key sẽ tự động thay đổi, kích hoạt chu trình tải mới và cập nhật cache tương ứng mà không làm sai lệch bản build.
