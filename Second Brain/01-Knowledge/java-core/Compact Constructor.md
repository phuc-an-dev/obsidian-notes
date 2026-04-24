---
created: 2026-04-20
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/immutability"
related:
  - "[[Record]]"
  - "[[Validated in Spring Boot]]"
---

# Compact Constructor

## 1. What

Compact constructor là cú pháp constructor đặc biệt chỉ có trong Java `record`, được giới thiệu cùng với record trong Java 16. Khác với canonical constructor thông thường, compact constructor **không khai báo danh sách tham số** và **không cần gán field thủ công** — compiler tự thêm phần gán `this.field = field` sau khi body của compact constructor chạy xong. Mục đích duy nhất của compact constructor là **validate** hoặc **normalize** các component trước khi chúng được gán vào final field của record.

---

## 2. Why

Record tự động sinh canonical constructor gán trực tiếp tham số vào field. Nhưng đôi khi cần kiểm soát giá trị trước khi gán:

- Đảm bảo `name` không blank, `amount` không âm, `email` đúng format.
- Normalize: trim whitespace, chuyển sang lowercase, tạo defensive copy của List.
- Throw exception sớm nếu dữ liệu không hợp lệ — thay vì để lỗi xuất hiện muộn ở nơi khác.

Trước khi có compact constructor, cách duy nhất là viết lại toàn bộ canonical constructor:

```java
record Email(String value) {
    Email(String value) {          // canonical constructor — phải khai báo tham số
        Objects.requireNonNull(value);
        value = value.trim();
        this.value = value;        // phải tự gán
    }
}
```

Compact constructor loại bỏ phần lặp lại — khai báo tham số và gán field:

```java
record Email(String value) {
    Email {                        // compact constructor — không có danh sách tham số
        Objects.requireNonNull(value);
        value = value.trim();      // modify tham số trước khi compiler tự gán
    }
    // compiler tự thêm: this.value = value;
}
```

---

## 3. Mental Model

Hãy tưởng tượng record là một **két sắt** — sau khi khóa lại, không ai thay đổi được nội dung bên trong. Canonical constructor là thời điểm **nhân viên bỏ tài liệu vào két** và khóa lại.

Compact constructor là **bàn kiểm tra** ngay trước miệng két: trước khi nhân viên bỏ tài liệu vào và khóa, bàn kiểm tra làm hai việc:
1. **Kiểm soát chất lượng** (validate): nếu tài liệu thiếu chữ ký hay điền sai, ném trả lại ngay — két sắt không bao giờ nhận tài liệu lỗi.
2. **Chuẩn hóa** (normalize): tự động chỉnh sửa nhỏ như bỏ khoảng trắng thừa, đóng dấu ngày tháng — trước khi bỏ vào và khóa.

Sau khi qua bàn kiểm tra, **nhân viên tự bỏ vào két và khóa** — bạn không cần viết bước đó.

---

## 4. Where it fits

```
[Caller]
  new Email("  User@EXAMPLE.COM  ")
          |
          v
[Compact Constructor body chạy]
  - validate: giá trị có null không?
  - normalize: trim() và toLowerCase()
  - tham số "value" = "user@example.com"
          |
          v
[Compiler tự thêm: this.value = value]
          |
          v
[Record được tạo — field final, bất biến]
  Email { value = "user@example.com" }
```

Compact constructor nằm giữa lời gọi `new` và thời điểm field được sealed. Không có cách nào khác để chạy code tại thời điểm này trong record.

---

## 5. When to use

- **Validate đầu vào bắt buộc**: null check, range check, format check — đảm bảo record không bao giờ tồn tại ở trạng thái không hợp lệ.
- **Normalize giá trị**: trim whitespace, lowercase, uppercase, format chuẩn hóa — đảm bảo record luôn chứa dữ liệu đã được chuẩn hóa.
- **Defensive copy cho mutable collection**: bọc `List`, `Set`, `Map` bằng `List.copyOf()` trước khi gán — đảm bảo caller không thể modify internal state sau khi tạo.
- **Computed validation cross-field**: kiểm tra ràng buộc giữa nhiều component, ví dụ `startDate` phải trước `endDate`.

---

## 6. When NOT to use

- **Logic nghiệp vụ phức tạp**: compact constructor nên nhẹ — chỉ validate và normalize. Logic nghiệp vụ như tính toán, gọi service, hay kết nối database không thuộc về constructor.
- **Checked exception**: compact constructor không thể khai báo `throws CheckedException`. Chỉ có thể throw `RuntimeException` hoặc subclass. Nếu cần checked exception, cần dùng static factory method thay thế.
- **Async hoặc I/O**: không bao giờ gọi I/O, database, hay network call trong constructor — nguyên tắc chung, không chỉ riêng compact constructor.
- **Khi record không cần validation hay normalization**: nếu record chỉ là data holder đơn giản và data đã được validate trước khi truyền vào, compact constructor là thừa — không cần thêm.

---

## 7. Trade-offs

| Ưu điểm | Nhược điểm |
|---------|-----------|
| Loại bỏ boilerplate khai báo tham số và gán field | Chỉ dùng được trong `record`, không dùng được trong class thường |
| Đảm bảo record invariant ngay tại điểm tạo — fail fast | Không thể throw checked exception — hạn chế error handling |
| Dễ đọc — body tập trung vào validate/normalize, không bị che khuất bởi assignment boilerplate | Thứ tự thực thi không rõ ràng với developer mới: tham số được modify trong body, compiler tự gán sau |
| Không thể bypass bằng cách gọi setter (record không có setter) | Không hợp cho logic phức tạp — dễ bị lạm dụng |
| Kết hợp tốt với `Objects.requireNonNull` và custom exception | Khó debug nếu compact constructor ném exception mà message không rõ ràng |

---

## 8. Alternatives

| Giải pháp | Ưu điểm | Nhược điểm | Khi nào chọn |
|-----------|---------|-----------|-------------|
| Canonical constructor (đầy đủ) | Rõ ràng tường minh, có thể throw checked exception | Phải viết tham số và gán field thủ công — verbose | Cần checked exception hoặc cần control hoàn toàn |
| Static factory method | Tên gợi nhớ, có thể throw checked exception, có thể return null hay Optional | Caller phải dùng factory, không dùng `new` trực tiếp | Validation phức tạp, cần return type khác, hoặc cần checked exception |
| `@Valid` + Bean Validation (Spring) | Tích hợp framework, constraint annotation tái sử dụng | Phụ thuộc Spring/Jakarta Validation, không standalone | DTO trong Spring app, cần tích hợp với framework validation |
| Builder pattern | Step-by-step construction, optional field dễ handle | Mutable trong quá trình build, phức tạp hơn | Object có nhiều optional field, cần flexible construction |

---

## 9. How

**Validate null và throw exception:**

```java
record UserId(Long value) {
    UserId {
        Objects.requireNonNull(value, "UserId must not be null");
        if (value <= 0) {
            throw new IllegalArgumentException("UserId must be positive, got: " + value);
        }
    }
}

new UserId(1L);    // OK
new UserId(null);  // NullPointerException: UserId must not be null
new UserId(-5L);   // IllegalArgumentException: UserId must be positive, got: -5
```

**Normalize String — trim và lowercase:**

```java
record Email(String value) {
    Email {
        Objects.requireNonNull(value, "Email must not be null");
        value = value.strip().toLowerCase();
        if (!value.contains("@")) {
            throw new IllegalArgumentException("Invalid email format: " + value);
        }
    }
}

Email e = new Email("  Admin@Example.COM  ");
e.value();  // "admin@example.com"
```

**Defensive copy cho mutable collection:**

```java
record Order(String id, List<String> items) {
    Order {
        Objects.requireNonNull(id, "Order id must not be null");
        Objects.requireNonNull(items, "Items must not be null");
        items = List.copyOf(items);  // unmodifiable copy — caller không thể mutate
    }
}

List<String> mutableList = new ArrayList<>(List.of("A", "B"));
Order order = new Order("ORD-001", mutableList);
mutableList.add("C");          // thêm vào original list
order.items();                 // vẫn là ["A", "B"] — unaffected
order.items().add("D");        // UnsupportedOperationException
```

**Cross-field validation:**

```java
record DateRange(LocalDate start, LocalDate end) {
    DateRange {
        Objects.requireNonNull(start, "Start date must not be null");
        Objects.requireNonNull(end, "End date must not be null");
        if (end.isBefore(start)) {
            throw new IllegalArgumentException(
                "End date %s must not be before start date %s".formatted(end, start)
            );
        }
    }
}

new DateRange(LocalDate.of(2026, 1, 1), LocalDate.of(2026, 12, 31));  // OK
new DateRange(LocalDate.of(2026, 12, 31), LocalDate.of(2026, 1, 1));  // IllegalArgumentException
```

**Normalize với range clamp:**

```java
record Percentage(int value) {
    Percentage {
        value = Math.clamp(value, 0, 100);  // Java 21 — clamp thay vì throw
    }
}

new Percentage(150).value();  // 100
new Percentage(-10).value();  // 0
new Percentage(75).value();   // 75
```

**Compact constructor và canonical constructor cùng tồn tại — không được:**

```java
// SAI — record không thể có cả compact và canonical constructor
record Point(int x, int y) {
    Point { ... }              // compact
    Point(int x, int y) { ... } // canonical — compile error
}
```

Một record chỉ có thể có một trong hai: compact constructor hoặc canonical constructor, không phải cả hai.

**Static factory method kết hợp với compact constructor:**

```java
record Temperature(double celsius) {

    Temperature {
        if (celsius < -273.15) {
            throw new IllegalArgumentException(
                "Temperature below absolute zero: " + celsius
            );
        }
    }

    // Static factory cho unit conversion
    public static Temperature fromFahrenheit(double fahrenheit) {
        return new Temperature((fahrenheit - 32) * 5.0 / 9.0);
    }

    public static Temperature fromKelvin(double kelvin) {
        return new Temperature(kelvin - 273.15);
    }

    public double toFahrenheit() {
        return celsius * 9.0 / 5.0 + 32;
    }
}

Temperature t = Temperature.fromFahrenheit(98.6);
t.celsius();       // 37.0
t.toFahrenheit();  // 98.6
```

---

## 10. Production concerns

**Immutability contract:**
Compact constructor là tuyến phòng thủ duy nhất cho invariant của record. Một khi record được tạo thành công (compact constructor không throw), invariant được đảm bảo mãi mãi — không có setter, không có cách nào thay đổi field sau đó. Đây là lý do validate đầy đủ trong compact constructor quan trọng hơn nhiều so với class thông thường.

**Message lỗi rõ ràng:**
Exception từ compact constructor thường là điểm đầu tiên phát hiện data xấu. Luôn thêm context vào message:

```java
// Tệ — không biết giá trị nào gây lỗi
throw new IllegalArgumentException("Invalid value");

// Tốt — biết ngay giá trị nào và tại sao
throw new IllegalArgumentException(
    "Price must be positive, got: " + amount + " " + currency
);
```

**`List.copyOf()` vs `Collections.unmodifiableList()`:**
`List.copyOf(items)` tạo bản copy hoàn toàn mới — caller thay đổi list gốc sau đó không ảnh hưởng record. `Collections.unmodifiableList(items)` chỉ wrap list gốc — nếu caller giữ reference đến list gốc và thay đổi, record bị ảnh hưởng. Trong compact constructor, luôn dùng `List.copyOf()`.

**Không gọi instance method của record trong compact constructor:**
Trong thân compact constructor, field chưa được gán (gán sau khi body chạy xong). Gọi `this.someMethod()` nếu method đó truy cập field sẽ gặp uninitialized field. Chỉ làm việc với tham số, không phải field thông qua `this`.

**Performance:**
Compact constructor chạy mỗi lần tạo record. Với record được tạo nhiều lần trong hot path (hàng triệu lần/giây), tránh tạo object thừa trong compact constructor. `List.copyOf()` tạo bản copy — không sao với kích thước nhỏ, nhưng cần cân nhắc với list lớn.

---

## 11. Common mistakes

**Lỗi 1: Cố gán `this.field` trong compact constructor**

```java
record Email(String value) {
    Email {
        value = value.strip();
        this.value = value;  // SAI — compile error
        // Compact constructor không cho phép gán this.field
        // Compiler tự làm sau khi body chạy xong
    }
}
```

Fix: Chỉ modify tham số (local variable), không gán `this.field`:

```java
record Email(String value) {
    Email {
        value = value.strip();  // modify tham số — compiler tự gán this.value = value
    }
}
```

**Lỗi 2: Dùng `Collections.unmodifiableList()` thay vì `List.copyOf()` — vẫn bị mutate từ bên ngoài**

```java
record Order(List<String> items) {
    Order {
        items = Collections.unmodifiableList(items);  // SAI — chỉ wrap, không copy
    }
}

List<String> original = new ArrayList<>(List.of("A", "B"));
Order order = new Order(original);
original.add("C");         // mutate original list
order.items();             // ["A", "B", "C"] — record bị ảnh hưởng!
```

Fix: Dùng `List.copyOf()` để tạo bản copy độc lập:

```java
record Order(List<String> items) {
    Order {
        items = List.copyOf(items);  // tạo bản copy — caller không ảnh hưởng được
    }
}
```

**Lỗi 3: Validate sau khi đã normalize — thứ tự sai gây bug**

```java
record Username(String value) {
    Username {
        if (value.isBlank()) {                 // kiểm tra trước khi strip
            throw new IllegalArgumentException("Username must not be blank");
        }
        value = value.strip();                 // normalize sau
        // "  a  " qua được kiểm tra blank nhưng sau khi strip thành "a" — không nhất quán
        // "   " qua được kiểm tra blank? Không — isBlank() trả về true cho chuỗi chỉ có space
        // Thực ra trường hợp này OK, nhưng thứ tự logic không rõ ràng
    }
}
```

Fix: Normalize trước, validate sau — logic rõ ràng hơn:

```java
record Username(String value) {
    Username {
        value = value.strip();                 // normalize trước
        if (value.isBlank()) {                 // validate sau — trên giá trị đã normalized
            throw new IllegalArgumentException("Username must not be blank");
        }
        if (value.length() < 3) {
            throw new IllegalArgumentException("Username too short: " + value);
        }
    }
}
```

**Lỗi 4: Gọi instance method của record trong compact constructor — field chưa được gán**

```java
record Range(int min, int max) {

    Range {
        validate();  // SAI — gọi instance method khi this.min và this.max chưa được gán
    }

    private void validate() {
        if (this.max < this.min) {  // this.min và this.max là 0 (default) lúc này
            throw new IllegalArgumentException("max must be >= min");
        }
    }
}
```

Fix: Validate trực tiếp trên tham số trong compact constructor, không gọi instance method:

```java
record Range(int min, int max) {
    Range {
        if (max < min) {  // dùng tham số min, max — không phải this.min, this.max
            throw new IllegalArgumentException(
                "max (%d) must be >= min (%d)".formatted(max, min)
            );
        }
    }
}
```

---

## 12. Sample project

**Bài tập: Value Objects cho domain e-commerce**

Xây dựng các Value Object dùng record với compact constructor:

1. `Money(BigDecimal amount, String currencyCode)` — amount không âm, currencyCode là ISO 4217 (3 chữ cái hoa), normalize currencyCode về uppercase.
2. `ProductName(String value)` — strip whitespace, không blank, độ dài 2–100 ký tự.
3. `Quantity(int value)` — phải dương (`> 0`), tối đa 9999.
4. `OrderLineItem(ProductName name, Money price, Quantity quantity)` — validate tất cả component không null, thêm method `Money totalPrice()`.

Hard constraints:
- Tất cả compact constructor normalize trước, validate sau.
- Exception message phải chứa giá trị vi phạm để dễ debug.
- Viết tối thiểu 3 unit test cho mỗi record: một happy path, một null input, một boundary violation.
- Không dùng Spring hay bất kỳ framework nào — pure Java.

---

## 13. Interview

**Core Q&A:**

Q: Compact constructor là gì và khác canonical constructor thế nào?
A: Compact constructor là cú pháp đặc biệt trong record không khai báo tham số và không cần gán field thủ công. Compiler tự thêm phần gán `this.field = param` sau khi body chạy xong. Dùng để validate hoặc normalize tham số trước khi gán. Canonical constructor thông thường phải khai báo đầy đủ tham số và tự gán từng field — verbose hơn nhưng cho phép throw checked exception.

Q: Compiler tự làm gì sau khi compact constructor body chạy xong?
A: Compiler tự sinh phần gán cho mỗi component của record: `this.field1 = field1; this.field2 = field2; ...`. Tham số trong compact constructor trùng tên với component — nếu bạn gán lại tham số trong body (`value = value.strip()`), giá trị đã được modify sẽ là giá trị được gán vào field.

Q: Tại sao phải dùng `List.copyOf()` thay vì `Collections.unmodifiableList()` trong compact constructor?
A: `List.copyOf()` tạo bản copy hoàn toàn mới và độc lập — caller thay đổi list gốc sau đó không ảnh hưởng record. `Collections.unmodifiableList()` chỉ bọc list gốc — nếu caller giữ reference đến list gốc và mutate, record bị ảnh hưởng dù field là `final`. Record cần invariant bất biến hoàn toàn, nên phải dùng `List.copyOf()`.

Q: Có thể throw checked exception trong compact constructor không?
A: Không. Compact constructor không thể khai báo `throws CheckedException` vì không có chỗ để khai báo (không có signature). Chỉ có thể throw `RuntimeException` hoặc subclass. Nếu cần throw checked exception, dùng static factory method bọc ngoài và khai báo `throws` ở đó.

Q: Compact constructor và canonical constructor có thể cùng tồn tại trong một record không?
A: Không. Một record chỉ được có một constructor dạng "full" — hoặc compact, hoặc canonical, không phải cả hai. Tuy nhiên, record có thể có thêm các non-canonical constructor (constructor với tham số khác) miễn là chúng delegate về canonical constructor.

**Scenarios:**

Q: Bạn có record `record Price(BigDecimal value, String currency)`. Caller truyền vào `new ArrayList<>()` rỗng cho một field List ở record khác. Compact constructor nên xử lý thế nào?
A: Trong compact constructor, validate không null trước, sau đó tạo defensive copy: `items = List.copyOf(items)`. Nếu List rỗng là hợp lệ theo domain thì chấp nhận — `List.copyOf()` của list rỗng trả về empty unmodifiable list. Nếu domain yêu cầu ít nhất một item: `if (items.isEmpty()) throw new IllegalArgumentException("Items must not be empty")` — validate sau khi copy.

Q: Record `Email` của bạn normalize về lowercase trong compact constructor. Sau này team muốn thêm kiểm tra domain whitelist (cần query database). Cách tiếp cận đúng?
A: Không đưa database query vào compact constructor — constructor phải nhẹ, không có I/O. Domain whitelist là business rule, không phải structural invariant của data. Đưa validation đó vào service layer: `emailValidator.validateDomain(email)` trước khi tạo record, hoặc dùng static factory method trong domain service có access đến repository.

---

## 14. References

- JEP 395 — Records (Java 16, compact constructor spec): https://openjdk.org/jeps/395
- Java Language Specification — Compact Constructor: https://docs.oracle.com/javase/specs/jls/se21/html/jls-8.html#jls-8.10.4
- Java SE API — java.lang.Record: https://docs.oracle.com/en/java/api/java.base/java/lang/Record.html
- Baeldung — Java Record Compact Constructors: https://www.baeldung.com/java-record-compact-constructor
- Inside Java — "Records: Compact Constructors": https://inside.java/2021/04/05/compact-constructors/

---

## 15. Real-world Code

- Spring Framework 6 — tìm record với compact constructor trong source để xem cách team Spring dùng: https://github.com/spring-projects/spring-framework/search?q=record&type=code
- OpenJDK source — `java.lang` package có nhiều record mới dùng compact constructor: https://github.com/openjdk/jdk/tree/master/src/java.base/share/classes/java/lang
- Quarkus — nhiều Value Object dùng record + compact constructor: https://github.com/quarkusio/quarkus/search?q=record&type=code

---

## 16. Community

- Reddit: r/java — tìm "record compact constructor" hoặc "java record validation"
- Stack Overflow: https://stackoverflow.com/questions/tagged/java-record — nhiều Q&A về compact constructor behavior
- Blog: Nicolai Parlog (nipafx) — "Java Records — Compact Constructors": https://nipafx.dev/java-record-semantics/
- Blog: Inside Java Newsletter — nhiều bài về record patterns và compact constructor: https://inside.java
- Talk: "Java Records Deep Dive" tại JEP Cafe (Oracle YouTube): tìm "JEP Cafe episode 14"
- Talk: Venkat Subramaniam tại Devoxx — "Records and Sealed Types": tìm trên YouTube Devoxx channel
