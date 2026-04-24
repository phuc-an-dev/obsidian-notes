---
created: 2026-04-20
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/datetime"
related:
  - "[[Record]]"
  - "[[Optional]]"
---

# Instant

## 1. What

`java.time.Instant` là class trong Java Date/Time API (java.time, Java 8+) đại diện cho một **điểm cụ thể trên trục thời gian** — được đo bằng số giây và nanoseconds tính từ **Unix epoch** (1970-01-01T00:00:00Z theo UTC). `Instant` là immutable, thread-safe, và không mang thông tin về timezone hay lịch — nó chỉ là một con số offset từ epoch. `Instant` thường được dùng để **ghi lại thời điểm xảy ra sự kiện** (timestamp) trong log, database, audit trail, hoặc tính toán khoảng cách thời gian giữa hai sự kiện.

---

## 2. Why

Trước Java 8, `java.util.Date` và `java.util.Calendar` có nhiều vấn đề cơ bản:

- `Date` vừa đại diện cho một instant, vừa có method định dạng theo timezone — trách nhiệm không rõ ràng.
- `Date` là mutable — truyền qua nhiều layer có thể bị modify.
- API không nhất quán: tháng đánh số từ 0, năm tính từ 1900 — dễ off-by-one.
- Không thread-safe: `SimpleDateFormat` gây bug khó phát hiện trong môi trường concurrent.
- Không có khái niệm duration, period, hay timezone-aware datetime rõ ràng.

`java.time.Instant` giải quyết vai trò cốt lõi nhất: **lưu một thời điểm trên trục thời gian** — không hơn, không kém. Kết hợp với `ZonedDateTime`, `LocalDateTime`, `Duration`, `ZoneId` từ cùng package, developer có bộ công cụ đầy đủ, rõ ràng trách nhiệm, immutable, và thread-safe.

---

## 3. Mental Model

Hãy tưởng tượng trục thời gian như một **thước đo tuyến tính dài vô tận**:

- Điểm gốc `0` là **Unix epoch**: 1970-01-01 00:00:00 UTC — một thỏa thuận quốc tế.
- `Instant` là một **vạch đánh dấu** trên thước đó — một con số chính xác đến nanosecond.
- Con số này **giống nhau ở mọi nơi trên thế giới**: khi máy chủ ở Hà Nội và máy chủ ở New York đều ghi `Instant.now()`, cả hai ghi cùng một vạch trên thước — dù đồng hồ tường của họ hiển thị giờ khác nhau.

`ZonedDateTime` mới là "hiển thị đồng hồ tường" — nó nói "vạch này, nhìn từ múi giờ Hà Nội, là 14:30 ngày 20/4/2026". Nhưng vạch gốc trên thước — `Instant` — vẫn là một con số duy nhất, không thay đổi.

Khi cần **lưu trữ hay so sánh thời điểm**, dùng `Instant`. Khi cần **hiển thị cho người dùng theo timezone của họ**, convert sang `ZonedDateTime`.

---

## 4. Where it fits

```
[Trục thời gian vật lý]
  ... 1969 | EPOCH (1970-01-01T00:00:00Z) | 2026 ...
                        |
                   Instant = số giây + nanos từ epoch

[Khi cần lưu/truyền — dùng Instant]
  Database  : TIMESTAMP WITH TIME ZONE hoặc BIGINT (epoch millis)
  JSON API  : "2026-04-20T07:30:00Z"  (ISO-8601 UTC string)
  Log/audit : Instant.now()

[Khi cần hiển thị — convert sang ZonedDateTime]
  Instant.atZone(ZoneId.of("Asia/Ho_Chi_Minh"))
    -> ZonedDateTime: 2026-04-20T14:30:00+07:00[Asia/Ho_Chi_Minh]

[Khi cần tính khoảng cách — dùng Duration]
  Duration.between(start, end)
    -> Duration: PT5M30S (5 phút 30 giây)
```

`Instant` là đơn vị lưu trữ và truyền tin. `ZonedDateTime` là đơn vị hiển thị. `Duration` là khoảng cách giữa hai `Instant`.

---

## 5. When to use

- **Audit timestamp**: `createdAt`, `updatedAt`, `deletedAt` trong entity — ghi lại thời điểm tuyệt đối, không phụ thuộc timezone.
- **Event timestamp**: `occurredAt` trong domain event, message queue event — thời điểm sự kiện xảy ra.
- **Log entry**: mọi log line cần một `Instant` để có thể correlate log từ nhiều service khác nhau.
- **TTL / expiry**: `expiresAt = Instant.now().plus(Duration.ofHours(24))` — tính thời điểm hết hạn.
- **Performance measurement**: `Instant start = Instant.now(); ...; Duration.between(start, Instant.now())`.
- **API request/response timestamp**: JSON field `"timestamp": "2026-04-20T07:30:00Z"` — ISO-8601 UTC.
- **So sánh thứ tự sự kiện**: `event1.occurredAt().isBefore(event2.occurredAt())`.

---

## 6. When NOT to use

- **Hiển thị thời gian cho người dùng**: `Instant` không có timezone — không thể hiển thị "14:30 ngày 20/4" theo giờ địa phương. Phải convert sang `ZonedDateTime` trước: `instant.atZone(userZone)`.
- **Biểu diễn ngày tháng thuần (không có giờ)**: dùng `LocalDate` — ví dụ ngày sinh, ngày hết hạn hợp đồng.
- **Biểu diễn giờ trong ngày không gắn timezone**: dùng `LocalTime` — ví dụ giờ mở cửa "09:00".
- **Lịch hay timezone-aware datetime cho business logic**: dùng `ZonedDateTime` — ví dụ "lịch họp lúc 14:00 múi giờ Tokyo", cần tính đúng DST (Daylight Saving Time).
- **Khoảng thời gian theo ngày/tháng/năm**: dùng `Period` — ví dụ "hợp đồng 2 năm". `Duration` chỉ tốt cho khoảng thời gian cố định (giây, phút, giờ), không xử lý được DST hay tháng có số ngày khác nhau.

---

## 7. Trade-offs

| Ưu điểm | Nhược điểm |
|---------|-----------|
| Immutable và thread-safe — không bao giờ bị modify ngoài ý muốn | Không có timezone — không thể format trực tiếp thành "14:30 Hà Nội" |
| Unambiguous — một con số duy nhất, không nhập nhằng timezone | `toString()` luôn là UTC — dễ nhầm khi debug nếu không quen |
| Độ chính xác đến nanosecond (JVM thực tế thường millisecond hay microsecond) | Không hỗ trợ leap second — hành xử theo UTC-SLS (smeared) |
| Serialization đơn giản: lưu dạng `long` (epoch millis) hoặc ISO-8601 string | Không thể cộng "1 tháng" — phải convert sang `ZonedDateTime` |
| Tích hợp tốt với Java 8 Streams, Optional, CompletableFuture | Một số ORM cũ (Hibernate < 5) cần converter thủ công |
| Tính toán khoảng cách chính xác qua `Duration.between()` | Khó đọc raw value — `1745151600` không nói lên gì trực tiếp |

---

## 8. Alternatives

| Giải pháp | Ưu điểm | Nhược điểm | Khi nào chọn |
|-----------|---------|-----------|-------------|
| `ZonedDateTime` | Có timezone, dễ format hiển thị | Phức tạp hơn, DST có thể gây bug nếu xử lý sai | Hiển thị cho user, lịch, scheduling theo múi giờ |
| `LocalDateTime` | Đơn giản, không có timezone overhead | Không đại diện cho một thời điểm tuyệt đối — ambiguous | Thời gian cục bộ không cần timezone (ví dụ: giờ hẹn trong app offline) |
| `OffsetDateTime` | Có UTC offset, ít ambiguous hơn `LocalDateTime` | Không xử lý DST | Lưu trong database yêu cầu offset, giao tiếp với hệ thống không dùng zone ID |
| `java.util.Date` (legacy) | Đã tồn tại trước Java 8, nhiều framework cũ dùng | Mutable, API kém, không thread-safe | Chỉ khi bắt buộc tương thích framework cũ |
| `long` (epoch millis) | Nhỏ gọn, dễ serialize | Không có type safety, không có API tính toán | Lưu trong cache, binary protocol cần compact size |

---

## 9. How

**Tạo Instant:**

```java
// Thời điểm hiện tại
Instant now = Instant.now();

// Từ epoch millis (thường dùng khi đọc từ DB hay JSON)
Instant fromMillis = Instant.ofEpochMilli(1745151600000L);

// Từ epoch second
Instant fromSeconds = Instant.ofEpochSecond(1745151600L);

// Từ epoch second + nanosecond adjustment
Instant precise = Instant.ofEpochSecond(1745151600L, 500_000_000L);  // + 0.5s

// Từ String ISO-8601
Instant parsed = Instant.parse("2026-04-20T07:30:00Z");

// Epoch — điểm gốc
Instant epoch = Instant.EPOCH;  // 1970-01-01T00:00:00Z

// Min và Max
Instant min = Instant.MIN;  // -1000000000-01-01T00:00:00Z
Instant max = Instant.MAX;  // 1000000000-12-31T23:59:59.999999999Z
```

**Lấy giá trị số:**

```java
Instant instant = Instant.parse("2026-04-20T07:30:00Z");

long epochMilli   = instant.toEpochMilli();       // 1745133000000
long epochSecond  = instant.getEpochSecond();     // 1745133000
int  nanoAdjust   = instant.getNano();            // nanoseconds trong giây hiện tại (0–999999999)
```

**Arithmetic — cộng và trừ:**

```java
Instant now = Instant.now();

// Cộng thêm thời gian
Instant in24h      = now.plus(Duration.ofHours(24));
Instant in30min    = now.plus(30, ChronoUnit.MINUTES);
Instant nextSecond = now.plusSeconds(1);
Instant nextMilli  = now.plusMillis(100);

// Trừ đi thời gian
Instant yesterday  = now.minus(Duration.ofDays(1));
Instant ago5min    = now.minus(5, ChronoUnit.MINUTES);

// Không thể cộng theo tháng hay năm — phải convert sang ZonedDateTime
Instant nextMonth = now
    .atZone(ZoneId.of("UTC"))
    .plusMonths(1)
    .toInstant();
```

**So sánh:**

```java
Instant a = Instant.parse("2026-01-01T00:00:00Z");
Instant b = Instant.parse("2026-06-01T00:00:00Z");

boolean before    = a.isBefore(b);      // true
boolean after     = a.isAfter(b);       // false
boolean equal     = a.equals(b);        // false
int     compared  = a.compareTo(b);     // âm — a trước b

// Lấy Instant xa hơn hoặc gần hơn
Instant later   = a.isAfter(b)  ? a : b;  // b
Instant earlier = a.isBefore(b) ? a : b;  // a
```

**Duration giữa hai Instant:**

```java
Instant start = Instant.now();
// ... thực hiện tác vụ ...
Instant end = Instant.now();

Duration elapsed = Duration.between(start, end);

long millis  = elapsed.toMillis();
long seconds = elapsed.toSeconds();
long minutes = elapsed.toMinutes();

// Format đẹp hơn
System.out.println(elapsed);  // PT0.123S (ISO-8601 duration)
```

**Convert sang ZonedDateTime để hiển thị:**

```java
Instant instant = Instant.parse("2026-04-20T07:30:00Z");

ZonedDateTime hcm  = instant.atZone(ZoneId.of("Asia/Ho_Chi_Minh"));
// 2026-04-20T14:30:00+07:00[Asia/Ho_Chi_Minh]

ZonedDateTime tokyo = instant.atZone(ZoneId.of("Asia/Tokyo"));
// 2026-04-20T16:30:00+09:00[Asia/Tokyo]

ZonedDateTime utc   = instant.atZone(ZoneOffset.UTC);
// 2026-04-20T07:30:00Z

// Format để hiển thị
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm")
    .withZone(ZoneId.of("Asia/Ho_Chi_Minh"));
String display = fmt.format(instant);  // "20/04/2026 14:30"
```

**Dùng trong JPA entity:**

```java
@Entity
public class Order {

    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @Column
    private Instant updatedAt;

    @Column
    private Instant deletedAt;

    @PrePersist
    void onPrePersist() {
        createdAt = Instant.now();
        updatedAt = Instant.now();
    }

    @PreUpdate
    void onPreUpdate() {
        updatedAt = Instant.now();
    }

    public boolean isDeleted() {
        return deletedAt != null;
    }

    public boolean isExpired(Instant expiresAt) {
        return Instant.now().isAfter(expiresAt);
    }
}
```

**Jackson serialization (Spring Boot mặc định serialize sang ISO-8601):**

```java
// application.properties
// spring.jackson.serialization.write-dates-as-timestamps=false

// JSON output: "createdAt": "2026-04-20T07:30:00Z"
// JSON input:  "createdAt": "2026-04-20T07:30:00Z"  <- tự parse thành Instant
```

**Dùng trong record:**

```java
record OrderCreatedEvent(
    String orderId,
    BigDecimal amount,
    Instant occurredAt
) {
    OrderCreatedEvent {
        Objects.requireNonNull(orderId);
        Objects.requireNonNull(occurredAt);
    }

    static OrderCreatedEvent now(String orderId, BigDecimal amount) {
        return new OrderCreatedEvent(orderId, amount, Instant.now());
    }
}
```

---

## 10. Production concerns

**Clock injection cho testability:**
Code gọi `Instant.now()` trực tiếp không thể test với thời gian giả. Inject `Clock` bean để control trong test:

```java
// Thay vì:
Instant now = Instant.now();

// Dùng:
@Component
@RequiredArgsConstructor
public class OrderService {
    private final Clock clock;

    public Order create(OrderRequest request) {
        return new Order(request, Instant.now(clock));
    }
}

// Config:
@Bean
public Clock clock() {
    return Clock.systemUTC();
}

// Trong test:
Clock fixedClock = Clock.fixed(
    Instant.parse("2026-04-20T07:30:00Z"),
    ZoneOffset.UTC
);
// Inject fixedClock vào service — Instant.now(fixedClock) luôn trả về cùng một thời điểm
```

**Database mapping:**
- PostgreSQL: `TIMESTAMP WITH TIME ZONE` — Hibernate tự map sang `Instant` từ Hibernate 5+.
- MySQL: `DATETIME(6)` — lưu UTC, không có timezone info trong column. Cần config `serverTimezone=UTC` trong JDBC URL.
- Tránh `TIMESTAMP` không có timezone trên MySQL — có thể bị convert sai timezone theo server config.

**JSON serialization:**
Jackson mặc định serialize `Instant` thành epoch seconds dạng số nếu không config. Thêm `JavaTimeModule` và tắt `WRITE_DATES_AS_TIMESTAMPS`:

```java
// Spring Boot tự config nếu có jackson-datatype-jsr310 trong classpath
// Thêm vào application.properties:
// spring.jackson.serialization.write-dates-as-timestamps=false
```

Kết quả: `"occurredAt": "2026-04-20T07:30:00Z"` — ISO-8601 UTC, dễ đọc và unambiguous.

**Clock drift trong distributed system:**
`Instant.now()` đọc từ system clock của máy chủ — máy chủ khác nhau có thể lệch vài milliseconds hay seconds. Dùng NTP để sync clock. Khi cần ordering tuyệt đối trong distributed system, cân nhắc logical clock (Lamport timestamp) thay vì wall clock.

**Precision thực tế của JVM:**
`Instant` hỗ trợ nanosecond precision về mặt API, nhưng `System.currentTimeMillis()` (mà `Instant.now()` dùng bên dưới) chỉ có độ chính xác millisecond trên hầu hết OS. Java 9+ dùng `ProcessHandle` và `VarHandle` để đạt microsecond trên một số OS. Không nên rely vào nanosecond precision trong code production.

---

## 11. Common mistakes

**Lỗi 1: Dùng `LocalDateTime` để lưu timestamp trong database — mất thông tin timezone**

```java
// SAI — LocalDateTime không đại diện cho một thời điểm tuyệt đối
@Column
private LocalDateTime createdAt = LocalDateTime.now();
// "2026-04-20T14:30:00" — là giờ HCM hay UTC? Không rõ.
// Khi deploy server sang timezone khác, data bị đọc sai
```

Fix: Dùng `Instant` cho audit timestamp:

```java
@Column
private Instant createdAt = Instant.now();
// Lưu xuống DB là UTC, đọc lên đúng bất kể timezone của server
```

**Lỗi 2: Format `Instant` trực tiếp không chỉ định timezone — dùng timezone JVM mặc định**

```java
// SAI — timezone không rõ ràng, kết quả phụ thuộc JVM timezone
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm");
String display = fmt.format(instant);  // IllegalArgumentException hoặc dùng JVM default timezone
```

Fix: Luôn chỉ định timezone khi format:

```java
DateTimeFormatter fmt = DateTimeFormatter
    .ofPattern("dd/MM/yyyy HH:mm")
    .withZone(ZoneId.of("Asia/Ho_Chi_Minh"));
String display = fmt.format(instant);  // "20/04/2026 14:30" — rõ ràng
```

**Lỗi 3: Cộng tháng vào `Instant` bằng `plus(30, ChronoUnit.DAYS)` thay vì `plusMonths(1)`**

```java
// SAI — 30 ngày không phải lúc nào cũng là 1 tháng
Instant nextMonth = now.plus(30, ChronoUnit.DAYS);
// Tháng 2 chỉ có 28/29 ngày, tháng 7 có 31 ngày

// ĐÚNG — chuyển sang ZonedDateTime để cộng tháng đúng
Instant nextMonth = now
    .atZone(ZoneId.of("Asia/Ho_Chi_Minh"))
    .plusMonths(1)
    .toInstant();
```

**Lỗi 4: So sánh `Instant` bằng `==` thay vì `equals()` hoặc `compareTo()`**

```java
Instant a = Instant.ofEpochSecond(1000L);
Instant b = Instant.ofEpochSecond(1000L);

a == b;          // false — hai object khác nhau trên heap
a.equals(b);     // true — so sánh giá trị
a.compareTo(b);  // 0 — bằng nhau
a.isBefore(b);   // false
```

`Instant` là object, không phải primitive — luôn dùng `equals()`, `isBefore()`, `isAfter()`, `compareTo()`.

---

## 12. Sample project

**Bài tập: Event Sourcing Audit Log**

Xây dựng `AuditLog` service với:
1. `record AuditEntry(String entityType, String entityId, String action, String userId, Instant occurredAt)` — dùng compact constructor validate không null.
2. `AuditLogService.record(...)` — tạo `AuditEntry` với `occurredAt = Instant.now(clock)`.
3. `AuditLogService.findBetween(Instant from, Instant to)` — lọc entries trong khoảng thời gian.
4. `AuditLogService.findRecent(Duration window)` — lấy entries trong `window` gần nhất tính từ `Instant.now(clock)`.
5. REST endpoint `GET /audit?from=2026-04-01T00:00:00Z&to=2026-04-30T23:59:59Z` — nhận ISO-8601 UTC string, parse thành `Instant`.

Hard constraints:
- Inject `Clock` bean — không được gọi `Instant.now()` trực tiếp trong service.
- Unit test dùng `Clock.fixed(...)` để kiểm soát thời gian — không có `Thread.sleep()`.
- JSON response phải serialize `Instant` thành ISO-8601 string, không phải epoch number.
- `findBetween` phải validate `from.isBefore(to)`, throw `IllegalArgumentException` nếu không.

---

## 13. Interview

**Core Q&A:**

Q: `Instant` là gì và khác `LocalDateTime` như thế nào?
A: `Instant` đại diện cho một thời điểm tuyệt đối trên trục thời gian — đo bằng số giây từ Unix epoch (1970-01-01T00:00:00Z), không có timezone. `LocalDateTime` đại diện cho "ngày giờ trên đồng hồ tường" không gắn với timezone hay offset — không phải một thời điểm tuyệt đối. "2026-04-20T14:30:00" không nói lên là 14:30 ở đâu. Dùng `Instant` để lưu timestamp (audit, event). Dùng `LocalDateTime` khi thời gian cục bộ không cần timezone (form input, schedule nội bộ không cần cross-timezone).

Q: Tại sao không dùng `LocalDateTime` để lưu `createdAt` trong database?
A: `LocalDateTime` không có timezone — khi lưu xuống và đọc lên, kết quả phụ thuộc vào timezone của JVM và database server. Nếu deploy server sang timezone khác hoặc thay đổi timezone config, timestamp bị đọc sai. `Instant` luôn là UTC — lưu xuống UTC, đọc lên UTC, hiển thị thì convert theo timezone của user. Không bao giờ bị lệch do config.

Q: Clock injection là gì và tại sao quan trọng trong testing?
A: Thay vì gọi `Instant.now()` trực tiếp (hardcode wall clock), inject `Clock` bean vào service và gọi `Instant.now(clock)`. Trong production, bean là `Clock.systemUTC()`. Trong test, inject `Clock.fixed(specificInstant, UTC)` để kiểm soát "hiện tại" là lúc nào. Không cần `Thread.sleep()` để test time-sensitive logic — test chạy nhanh và deterministic.

Q: `Duration` khác `Period` như thế nào?
A: `Duration` đo khoảng thời gian theo giây và nanoseconds — phù hợp cho khoảng thời gian ngắn có độ chính xác cao (milliseconds, seconds, hours). `Duration.between(Instant, Instant)` là phép tính chính xác tuyệt đối. `Period` đo theo ngày, tháng, năm trong lịch — phù hợp cho khoảng cách "2 năm 3 tháng" mà không quan tâm đến số giây chính xác. `Period` không thể dùng với `Instant` — phải dùng với `LocalDate`.

Q: Serialize `Instant` sang JSON và database như thế nào là đúng?
A: JSON: serialize thành ISO-8601 UTC string — `"2026-04-20T07:30:00Z"`. Cần `JavaTimeModule` trong Jackson và tắt `WRITE_DATES_AS_TIMESTAMPS` (Spring Boot tự config nếu có `jackson-datatype-jsr310`). Database: dùng `TIMESTAMP WITH TIME ZONE` (PostgreSQL) hoặc `DATETIME(6)` với UTC timezone (MySQL). Tránh lưu dạng `long` epoch millis trong column database — mất readability, khó query.

**Scenarios:**

Q: Bạn cần hiển thị thời điểm order được tạo theo timezone của user. `createdAt` lưu là `Instant`. Làm thế nào?
A: Convert sang `ZonedDateTime` theo timezone của user: `createdAt.atZone(ZoneId.of(userTimezone))`. Sau đó format: `DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm").withZone(userZoneId).format(createdAt)`. Timezone của user lấy từ profile hoặc request header `Accept-Timezone`. Không bao giờ format `Instant` trực tiếp mà không chỉ định timezone.

Q: Service A ghi log `Instant.now()` lúc 14:30:00.100 (HCM time) và service B ghi `Instant.now()` lúc 14:30:00.050 (HCM time) cho cùng một request. Khi correlate log, order event nào trước?
A: Nhìn vào epoch milliseconds: B (050ms) trước A (100ms). `Instant` so sánh qua `isBefore()/isAfter()` hoặc `compareTo()` — không liên quan đến timezone. Tuy nhiên, trong distributed system, clock của hai server có thể drift vài milliseconds — không thể tin tuyệt đối vào ordering dựa trên wall clock. Cần trace ID để correlate đúng, và NTP để giảm clock drift.

---

## 14. References

- Java SE 21 API — `java.time.Instant`: https://docs.oracle.com/en/java/api/java.base/java/time/Instant.html
- Java SE 21 API — `java.time.Clock`: https://docs.oracle.com/en/java/api/java.base/java/time/Clock.html
- JEP 150 — Date and Time API (Java 8): https://openjdk.org/jeps/150
- Joda-Time migration guide (nền tảng của java.time): https://www.joda.org/joda-time/
- Baeldung — Introduction to the Java 8 Date/Time API: https://www.baeldung.com/java-8-date-time-intro
- Baeldung — Working with Instant: https://www.baeldung.com/java-instant-vs-localdatetime

---

## 15. Real-world Code

- Spring Framework source — `Instant` dùng trong audit, cache, scheduling: https://github.com/spring-projects/spring-framework/search?q=Instant.now&type=code
- Spring Security — `Instant` dùng cho token expiration: https://github.com/spring-projects/spring-security/search?q=Instant&type=code
- Hibernate ORM — `InstantJavaType` mapper: https://github.com/hibernate/hibernate-orm/blob/main/hibernate-core/src/main/java/org/hibernate/type/descriptor/java/InstantJavaType.java
- Tìm `Clock.fixed` trong Spring Boot test samples để xem pattern Clock injection: https://github.com/spring-projects/spring-boot/search?q=Clock.fixed&type=code

---

## 16. Community

- Reddit: r/java — tìm "java.time Instant vs LocalDateTime" và "java time best practices"
- Stack Overflow tag: `java-time`: https://stackoverflow.com/questions/tagged/java-time
- Stack Overflow — "Should I use Instant or LocalDateTime to store timestamps?": https://stackoverflow.com/questions/32437550
- Blog: Baeldung — "Guide to Java 8 Date-Time API": https://www.baeldung.com/java-8-date-time-intro
- Blog: Nicolai Parlog (nipafx) — "Java Time": https://nipafx.dev/tags/java-time/
- Talk: "Modern Java Date and Time" tại Devoxx — tìm trên YouTube với keyword "Java time Devoxx"
- Book: "Modern Java in Action" (Manning) — Chapter 12: New Date and Time API
