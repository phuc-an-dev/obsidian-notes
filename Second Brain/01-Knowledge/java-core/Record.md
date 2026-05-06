---
created: 2026-04-20
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/immutability"
related:
  - "[[Optional.md]]"
  - "[[Pattern Matching for Switch.md]]"
  - "[[Compact Constructor.md]]"
  - "[[Transient.md]]"
---

# Record

## 1. What

`record` là một loại class đặc biệt được giới thiệu chính thức trong Java 16 (preview từ Java 14), được thiết kế để mô hình hóa **dữ liệu thuần túy bất biến (immutable data carrier)**. Khi khai báo `record Point(int x, int y)`, compiler tự động sinh ra: constructor, `getter` theo tên field (`x()`, `y()`), `equals()`, `hashCode()`, và `toString()` — loại bỏ toàn bộ boilerplate. Record là **implicitly final** (không thể extend), tất cả fields là **private final**, và không thể thêm instance field ngoài những field đã khai báo trong header.

---

## 2. Why

Trước Java 16, để tạo một class đơn giản chứa dữ liệu bất biến (ví dụ một cặp tọa độ `(x, y)`), developer phải viết:

```java
public final class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int x() { return x; }
    public int y() { return y; }

    @Override
    public boolean equals(Object o) { ... }

    @Override
    public int hashCode() { ... }

    @Override
    public String toString() { ... }
}
```

Khoảng 30 dòng cho một data class 2 field. Với Lombok `@Value` có thể rút gọn, nhưng vẫn là annotation processing bên ngoài, không phải ngôn ngữ. Record giải quyết:

- Loại bỏ boilerplate: 30 dòng còn 1 dòng.
- Immutability là mặc định, không cần nhớ thêm `final`.
- `equals/hashCode` chính xác dựa trên tất cả fields — không quên implement.
- Tăng tính biểu đạt: nhìn vào `record Point(int x, int y)` hiểu ngay đây là data, không phải service hay entity.

---

## 3. Mental Model

Hãy tưởng tượng sự khác biệt giữa **hợp đồng có thể sửa đổi** và **văn bằng chứng chỉ đã đóng dấu**:

Class thông thường giống hợp đồng: có thể thêm điều khoản mới, sửa nội dung, thêm chữ ký phụ, và ai đó có thể kế thừa template của nó để tạo hợp đồng mới.

`record` giống văn bằng đã đóng dấu: nội dung cố định ngay từ khi phát hành (`final fields`), không ai có thể thêm thông tin vào sau (`no extra instance fields`), không thể photocopy và ghi đè lên (`final class, no extension`), và hai văn bằng giống nhau hoàn toàn về nội dung thì được coi là tương đương (`equals/hashCode` by fields).

Khi nhìn thấy `record`, bộ não lập tức phân loại: "đây là **dữ liệu**, không phải **hành vi**".

---

## 4. Where it fits

```
[Domain Layer]
  record Money(BigDecimal amount, Currency currency)    <- Value Object
  record Coordinates(double lat, double lng)           <- DTO nội bộ

[API Layer]
  record CreateUserRequest(String name, String email)  <- Request DTO
  record UserResponse(Long id, String name)            <- Response DTO

[Application Layer]
  record PageResult<T>(List<T> items, int total)       <- Generic wrapper

[Pattern Matching (Java 21+)]
  switch (shape) {
    case Circle(double r) -> ...
    case Rectangle(double w, double h) -> ...
  }
```

Record phù hợp nhất ở tầng biên giới (API, event) và các Value Object trong domain. Không phù hợp cho JPA entity (cần mutable) hay service class (cần hành vi phức tạp).

---

## 5. When to use

- **DTO (Data Transfer Object)**: request/response body của REST API — cần bất biến, không cần method nghiệp vụ.
- **Value Object trong Domain-Driven Design**: `Money`, `Email`, `PhoneNumber` — định danh bằng giá trị, không phải identity.
- **Tuple / compound return**: method cần trả về nhiều giá trị mà không muốn tạo class riêng — `record Range(int min, int max)`.
- **Event object**: `record OrderCreatedEvent(String orderId, Instant occurredAt)` — event bất biến sau khi phát sinh.
- **Configuration holder**: nhóm các config liên quan lại thay vì dùng `Map<String, Object>`.
- **Pattern matching với sealed interface** (Java 17+): `record Circle(double radius) implements Shape`.

---

## 6. When NOT to use

- **JPA/Hibernate entity**: entity cần no-arg constructor, mutable fields (`set` methods), và lazy loading proxy — tất cả đều incompatible với record. Dùng `@Entity class` thông thường.
- **Class cần kế thừa**: record không thể extend class khác (ngoài `Object`) và không thể bị extend. Nếu cần inheritance hierarchy, dùng class thường hoặc sealed class.
- **Class có hành vi phức tạp**: nếu class có nhiều method nghiệp vụ hơn là data, record không phải đúng abstraction. Record nên "nói data nhiều hơn hành vi".
- **Khi cần mutability**: ví dụ builder pattern từng bước hoặc entity tracking thay đổi. Record không có setter.
- **Framework cần reflection-based no-arg constructor**: một số serialization framework cũ (không hỗ trợ record) cần no-arg constructor — record không có.

---

## 7. Trade-offs

| Ưu điểm | Nhược điểm |
|---------|-----------|
| Loại bỏ boilerplate cực lớn — 1 dòng thay 30 dòng | Không thể extend hoặc bị extend — inflexible nếu cần hierarchy |
| Immutability mặc định — thread-safe by design | Không compatible với JPA entity, Hibernate proxy |
| `equals/hashCode/toString` tự động và chính xác | Không có setter — không dùng được cho mutable workflow |
| Tăng tính biểu đạt — đọc code biết ngay đây là data class | No-arg constructor không có — một số framework cũ không support |
| Hỗ trợ Deconstruction Patterns trong Java 21+ | Tất cả component fields luôn public — không ẩn được field |
| Compact constructor cho validation gọn | Khó add computed field phức tạp phụ thuộc lẫn nhau |

---

## 8. Alternatives

| Giải pháp | Ưu điểm | Nhược điểm | Khi nào chọn |
|-----------|---------|-----------|-------------|
| Lombok `@Value` | Flexible hơn, có thể extend, tương thích nhiều framework | Annotation processing bên ngoài, không phải ngôn ngữ, IDE cần plugin | Legacy Java (<16), cần flexibility |
| Lombok `@Data` | Có setter, builder sẵn | Mutable, dễ vi phạm encapsulation | Khi cần mutability + ít boilerplate |
| Thủ công (final class + constructor) | Full control | Verbose, dễ quên implement `equals/hashCode` | Cần tùy chỉnh sâu mà record không cho phép |
| Kotlin `data class` | Tương tự record nhưng có `copy()`, destructuring | Không phải Java | Nếu dùng Kotlin trong cùng project |

---

## 9. How

**Khai báo cơ bản:**

```java
record Point(int x, int y) {}

// Compiler tự sinh:
// - public Point(int x, int y) { this.x = x; this.y = y; }
// - public int x() { return x; }
// - public int y() { return y; }
// - public boolean equals(Object o) { ... }
// - public int hashCode() { ... }
// - public String toString() { return "Point[x=1, y=2]"; }

Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);
p1.x();          // 1
p1.equals(p2);   // true
```

**Compact constructor — validate hoặc normalize input:**

```java
record Email(String value) {

    // Compact constructor: không khai báo tham số, gán tự động sau khi body chạy
    Email {
        Objects.requireNonNull(value, "email must not be null");
        value = value.trim().toLowerCase();  // normalize trước khi gán
        if (!value.contains("@")) {
            throw new IllegalArgumentException("Invalid email: " + value);
        }
    }
}

Email email = new Email("  User@Example.COM  ");
email.value();  // "user@example.com"
```

**Thêm method tùy chỉnh:**

```java
record Money(BigDecimal amount, Currency currency) {

    // Static factory method
    public static Money of(String amount, String currencyCode) {
        return new Money(new BigDecimal(amount), Currency.getInstance(currencyCode));
    }

    // Instance method
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public boolean isPositive() {
        return amount.compareTo(BigDecimal.ZERO) > 0;
    }
}

Money price = Money.of("100.00", "USD");
Money tax   = Money.of("10.00", "USD");
Money total = price.add(tax);  // Money[amount=110.00, currency=USD]
```

**Record với Generic:**

```java
record Page<T>(List<T> content, int pageNumber, int totalPages) {

    public boolean hasNext() {
        return pageNumber < totalPages - 1;
    }
}

Page<UserResponse> page = new Page<>(users, 0, 5);
```

**Record implement interface:**

```java
public sealed interface Shape permits Circle, Rectangle {}

record Circle(double radius) implements Shape {
    public double area() {
        return Math.PI * radius * radius;
    }
}

record Rectangle(double width, double height) implements Shape {
    public double area() {
        return width * height;
    }
}
```

**Pattern matching với record (Java 21+):**

```java
double area = switch (shape) {
    case Circle(double r)            -> Math.PI * r * r;
    case Rectangle(double w, double h) -> w * h;
};
```

**Record làm DTO với Jackson:**

```java
// Jackson hỗ trợ record từ 2.12+, không cần annotation thêm
record CreateUserRequest(
    @NotBlank String name,
    @Email String email,
    @Min(18) int age
) {}

// Deserialize JSON -> record tự động
// {"name":"Alice","email":"alice@example.com","age":25}
```

---

## 10. Production concerns

**Serialization:**
Jackson hỗ trợ record từ phiên bản 2.12 (Spring Boot 2.5+). Nếu dùng Jackson cũ hơn hoặc thư viện serialization khác, kiểm tra compatibility trước. Record không có no-arg constructor — một số framework (ví dụ Gson < 2.10, Kryo mặc định) cần config thêm.

**JPA không dùng record cho entity:**
Hibernate cần no-arg constructor, mutable field, và CGLIB proxy cho lazy loading — tất cả incompatible với record. Tuy nhiên, record dùng được cho JPA **Projection** (trả về subset field từ query):

```java
// Spring Data JPA Projection với record
interface UserProjection {
    record Summary(String name, String email) {}
}

List<UserProjection.Summary> findBy();
```

**`equals/hashCode` based on all components:**
Record tự động so sánh tất cả fields. Nếu field là mutable object (ví dụ `List`, `Date`), hai record "bằng nhau" tại thời điểm tạo có thể không bằng nhau sau này nếu mutable field bị thay đổi từ bên ngoài. Tránh dùng mutable field trong record.

**Thread safety:**
Record với tất cả primitive hoặc immutable fields là thread-safe by default. Nhưng nếu component là mutable object (`List`, `Map`, `StringBuilder`), thread safety không được đảm bảo — wrap bằng `Collections.unmodifiableList()` hoặc dùng immutable collection.

**Compact constructor và validation:**
Compact constructor là nơi lý tưởng để validate và normalize. Tuy nhiên, tránh logic phức tạp hoặc I/O — constructor phải nhẹ và không throw checked exception.

---

## 11. Common mistakes

**Lỗi 1: Dùng record làm JPA entity**

```java
@Entity
record User(Long id, String name) {}  // SAI — compile error hoặc runtime error
```

JPA yêu cầu: no-arg constructor (record không có), mutable fields (record field là final), CGLIB proxy (record là final class). Spring Data sẽ báo lỗi khi khởi động.

Fix: Dùng `@Entity class User` thông thường cho JPA entity. Dùng record riêng cho DTO/response object và map thủ công hoặc dùng MapStruct.

**Lỗi 2: Expose mutable collection trực tiếp qua component**

```java
record Order(List<String> items) {}

Order order = new Order(new ArrayList<>(List.of("A", "B")));
order.items().add("C");  // SAI — đột biến internal state của record!
System.out.println(order.items());  // [A, B, C] — record bị biến đổi
```

Fix: Dùng compact constructor để wrap bằng immutable collection:

```java
record Order(List<String> items) {
    Order {
        items = List.copyOf(items);  // tạo bản copy bất biến
    }
}
```

**Lỗi 3: Nhầm tên getter — dùng `getX()` thay vì `x()`**

Record tạo accessor theo tên field, không theo JavaBean convention. Accessor là `x()`, không phải `getX()`.

```java
record Point(int x, int y) {}

Point p = new Point(1, 2);
p.getX();  // SAI — CompileError: method getX() not found
p.x();     // ĐÚNG
```

Một số framework cũ dựa vào JavaBean convention (`getX()`) sẽ không tìm được field. Fix: configure framework để nhận accessor không theo JavaBean, hoặc thêm explicit getter.

**Lỗi 4: Định nghĩa instance field ngoài component header**

```java
record User(String name) {
    private String nickname;  // SAI — compile error
    // Record không cho phép khai báo instance field ngoài header
}
```

Fix: Nếu cần field bổ sung, dùng class thông thường. Nếu cần field tính toán từ component, dùng method:

```java
record User(String firstName, String lastName) {
    public String fullName() {  // method, không phải field
        return firstName + " " + lastName;
    }
}
```

---

## 12. Sample project

**Bài tập: Money Value Object với validation và arithmetic**

Xây dựng `record Money(BigDecimal amount, Currency currency)` với:
- Compact constructor validate: `amount` không null, không âm; `currency` không null.
- Method `add(Money other)` — cộng hai Money cùng currency, throw `CurrencyMismatchException` nếu khác.
- Method `multiply(int factor)` — nhân với hệ số nguyên dương.
- Static factory `Money.of(String amount, String currencyCode)`.
- Method `isZero()`, `isPositive()`.

Hard constraint: Tất cả method trả về `Money` mới (không mutate), toàn bộ test dùng `assertEquals` so sánh bằng value (tận dụng `equals` auto-generated của record). Viết tối thiểu 10 unit test bao phủ: normal case, currency mismatch, negative amount, null input.

---

## 13. Interview

**Core Q&A:**

Q: Record là gì và nó tự sinh ra những gì?
A: Record là class đặc biệt trong Java 16 dùng để mô hình hóa immutable data. Compiler tự sinh: canonical constructor (nhận tất cả component), accessor method theo tên field (không theo JavaBean — `x()` không phải `getX()`), `equals()` so sánh tất cả component, `hashCode()` dựa trên tất cả component, và `toString()` dạng `ClassName[field=value, ...]`. Record là implicitly `final` và không thể extend class khác.

Q: Record khác final class thông thường như thế nào?
A: Cả hai đều immutable và không thể extend. Nhưng record tự động sinh `equals/hashCode/toString/constructor/accessor`, còn class thủ công phải viết tay. Ngoài ra, record không thể thêm instance field ngoài header — đảm bảo tất cả state đều visible từ constructor. Record cũng hỗ trợ Deconstruction Pattern trong Java 21+.

Q: Compact constructor là gì?
A: Compact constructor là cú pháp đặc biệt của record cho phép thêm validation hay normalization mà không cần khai báo tham số và gán field — compiler tự làm. Dùng compact constructor để validate input (throw exception nếu sai) hoặc normalize giá trị (trim, toLowerCase) trước khi gán vào final field.

Q: Tại sao không dùng record làm JPA entity?
A: Ba lý do kỹ thuật: (1) JPA yêu cầu no-arg constructor — record không có. (2) JPA/Hibernate proxy yêu cầu class không phải `final` để CGLIB có thể subclass — record là `final`. (3) Lazy loading cần mutable field để inject proxy object — record field là `final`.

Q: Record có thread-safe không?
A: Với component là primitive hoặc immutable object (String, Integer, LocalDate, v.v.), record thread-safe by default vì không có mutable state. Nếu component là mutable object (ArrayList, HashMap), record KHÔNG thread-safe — cần wrap bằng immutable collection trong compact constructor.

**Scenarios:**

Q: Bạn cần trả về cả `User` object và `token` string từ một service method. Thiết kế thế nào?
A: Dùng record làm compound return type: `record AuthResult(User user, String token) {}`. Gọn hơn tạo class riêng, tự động có `equals/hashCode`, và rõ ràng hơn trả về `Object[]` hay `Map`.

Q: Team muốn dùng record cho DTO của API nhưng cần validation với `@NotBlank`, `@Email`. Có được không?
A: Được. Bean Validation annotations (`@NotBlank`, `@Email`, v.v.) hoạt động bình thường trên record component. Jackson 2.12+ deserialize JSON sang record tự động. Spring MVC nhận `@RequestBody record CreateUserRequest(...)` và trigger validation với `@Valid` hay `@Validated` như class thường.

Q: Bạn cần tạo một record nhưng muốn một field là list và không cho phép caller modify list đó. Làm thế nào?
A: Dùng compact constructor với `List.copyOf()`: `record Order(List<Item> items) { Order { items = List.copyOf(items); } }`. `List.copyOf` tạo ra unmodifiable list, nên `order.items().add(...)` sẽ throw `UnsupportedOperationException`. Caller truyền vào list nào cũng không ảnh hưởng được internal state của record.

---

## 14. References

- Java SE 16 — JEP 395 Records (final): https://openjdk.org/jeps/395
- Java SE 14 — JEP 359 Records (preview): https://openjdk.org/jeps/359
- Java Language Specification — Record Classes: https://docs.oracle.com/javase/specs/jls/se21/html/jls-8.html#jls-8.10
- JEP 440 — Record Patterns (Java 21): https://openjdk.org/jeps/440
- Baeldung — Java Records: https://www.baeldung.com/java-record-keyword

---

## 15. Real-world Code

- Spring Framework source — nhiều record được dùng nội bộ từ Spring 6: https://github.com/spring-projects/spring-framework
- Tìm `record` usage trong Spring Boot source: https://github.com/spring-projects/spring-boot/search?q=record&type=code
- `resilience4j` — dùng record cho immutable configuration objects: https://github.com/resilience4j/resilience4j
- OpenJDK JDK source — xem các record được dùng trong java.base module: https://github.com/openjdk/jdk/search?q=record&type=code

---

## 16. Community

- Reddit: r/java — nhiều thread về "records vs lombok", "record best practices"
- Stack Overflow tag: `java-record`: https://stackoverflow.com/questions/tagged/java-record
- Blog: Inside Java — "Records Come to Java": https://inside.java/2020/02/17/records-come-to-java/
- Blog: Nicolai Parlog (nipafx) — chuyên sâu về modern Java, nhiều bài về record: https://nipafx.dev/java-record-semantics/
- Talk: "Java Records" tại JEP Cafe trên YouTube (Oracle channel): tìm "JEP Cafe Records"
- Talk: Venkat Subramaniam — "Exploring Java Records" tại GOTO hoặc Devoxx: tìm trên YouTube
