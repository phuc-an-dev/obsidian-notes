---
created: 2026-04-17
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/pattern-matching"
related: "[[Optional]]"
---

# Pattern Matching for Switch

## 1. What

Pattern Matching for Switch (JEP 441, finalized trong Java 21) là tính năng cho phép dùng các "mẫu" (patterns) trong mệnh đề `case` của `switch` expression và `switch` statement. Thay vì chỉ kiểm tra giá trị hằng số, `switch` này có thể kiểm tra kiểu dữ liệu (type patterns), kèm theo điều kiện logic bổ sung (guarded patterns), và xử lý `null` trực tiếp. Đây là sự phát triển tự nhiên sau khi Java 14 giới thiệu switch expression và Java 16 hoàn thiện Pattern Matching cho `instanceof`.

## 2. Why

Trước Java 21, để xử lý một Object có thể là nhiều kiểu khác nhau, developer phải viết chuỗi `if-else` dài kết hợp `instanceof` và ép kiểu thủ công:

```java
// Cách cũ — verbose, dễ lỗi, khó mở rộng
if (obj instanceof Integer i) {
    return "int: " + i;
} else if (obj instanceof String s && s.length() > 5) {
    return "long string: " + s;
} else if (obj instanceof Double d) {
    return "double: " + d;
} else {
    return "unknown";
}
```

Vấn đề:
- **Ép kiểu thủ công**: Mỗi nhánh `if` phải cast riêng, dễ lỗi và cồng kềnh.
- **Thiếu exhaustiveness**: Compiler không biết bạn đã xử lý hết các trường hợp chưa — đặc biệt nguy hiểm khi thêm subclass mới.
- **Xử lý null không tập trung**: Phải check null bên ngoài `switch`, tạo nguy cơ NPE.
- **Khó đọc**: Logic kiểm tra và logic xử lý bị trộn lẫn.

## 3. Mental Model

Hãy tưởng tượng bạn là một **Nhân viên Hải quan ở cửa khẩu**. Trước đây (switch cũ), bạn chỉ được phép kiểm tra theo quốc tịch trong hộ chiếu (giá trị hằng số). Giờ đây (Pattern Matching switch), bạn có thể kiểm tra nhiều thứ hơn một lúc: "Nếu là Thương nhân (kiểu) và số tiền khai báo trên 10,000 USD (guard), đưa sang phòng kiểm tra đặc biệt. Nếu là Nhà ngoại giao (kiểu), cho qua ngay. Nếu hồ sơ bị thiếu (null), giữ lại ở quầy đầu tiên." Bạn quan sát người đi qua một lần, áp dụng nhiều quy tắc cùng lúc, và compiler đảm bảo bạn đã có quy tắc cho tất cả trường hợp có thể xảy ra — không thể để sót ai đi qua mà không kiểm tra.

## 4. Where it fits

```
Input: Object / Sealed Type
            |
            v
   switch (input) {
       case TypeA a when condition -> handleA(a)
       case TypeB b                -> handleB(b)
       case null                   -> handleNull()
       default                     -> handleDefault()
   }
            |
            v
   Output: giá trị / hành động tương ứng
```

Pattern Matching for Switch dùng tốt nhất ở tầng Service/Domain khi có hierarchy phức tạp (Sealed Classes, algebraic data types) hoặc khi cần dispatch command/event theo kiểu.

## 5. When to use

- Khi xử lý Object có nhiều kiểu khác nhau, thay thế chuỗi `if-else instanceof` dài.
- Khi kết hợp với **Sealed Classes** để compiler enforce exhaustiveness — đảm bảo không bỏ sót subclass nào.
- Khi cần logic dispatch phụ thuộc cả kiểu và giá trị: `case Integer i when i > 0`.
- Thay thế Visitor Pattern trong nhiều trường hợp đơn giản hơn.
- Khi parsing JSON, command dispatch, xử lý AST, xử lý message queue events.

## 6. When NOT to use

- Với các kiểu đơn giản như `int`, `enum`, `String` hằng số — `switch` truyền thống đơn giản hơn.
- Khi logic trong `when` guard quá phức tạp (phụ thuộc trạng thái bên ngoài, có side effect) — khó đọc và khó test.
- Trong dự án sử dụng Java < 21 (JEP 441 chưa phát hành chính thức, chỉ ở preview từ Java 17-20).
- Khi các nhánh xử lý quá khác biệt về logic và nên tách thành các method riêng — `switch` lớn không thay thế được good OOP design.

## 7. Trade-offs

| Pros | Cons |
|------|------|
| Loại bỏ ép kiểu thủ công, code gọn hơn | Cần làm quen cú pháp mới (`case Type t when cond`) |
| Compiler enforce exhaustiveness với Sealed Classes | Guarded pattern quá phức tạp làm khó đọc |
| Xử lý `null` tập trung, không bị NPE bất ngờ | Yêu cầu Java 21+ (LTS) — dự án cũ không dùng được |
| Kết hợp tốt với Records và Sealed Classes | Nhiều trường hợp có thể giải quyết bằng OOP tốt hơn |
| Rõ ràng: kiểu và điều kiện đọc cùng một chỗ | Dominance rule lúc đầu dễ nhầm (specific trước generic) |
| Biểu thức (expression) có thể return giá trị trực tiếp | |

## 8. Alternatives

| Approach | Ưu điểm | Nhược điểm |
|----------|---------|------------|
| `if-else` + `instanceof` (Java 16+) | Quen thuộc, không cần Java 21 | Dài dòng, không enforce exhaustiveness |
| Visitor Pattern | Tốt cho hierarchy lớn, OCP-compliant | Boilerplate nhiều, khó thêm case mới |
| Strategy Pattern | Tách logic thành class riêng, dễ test | Phức tạp hơn cho logic đơn giản |
| Method dispatch qua polymorphism | Clean OOP, không cần switch | Phải thêm method vào tất cả subclass |
| Map<Class, Handler> | Linh hoạt runtime | Không type-safe, mất exhaustiveness check |

## 9. How

```java
// --- Ví dụ căn bản: type pattern ---
public String describe(Object obj) {
    return switch (obj) {
        case null          -> "null value";
        case Integer i     -> "Integer: " + i;
        case Long l        -> "Long: " + l;
        case Double d      -> "Double: %.2f".formatted(d);
        case String s      -> "String of length " + s.length();
        default            -> "Unknown: " + obj.getClass().getSimpleName();
    };
}

// --- Guarded pattern: case Type t when condition ---
public String classifyNumber(Number n) {
    return switch (n) {
        case null              -> "null";
        case Integer i when i < 0  -> "negative int: " + i;
        case Integer i when i == 0 -> "zero";
        case Integer i             -> "positive int: " + i;
        case Double d when d.isNaN() -> "NaN";
        case Double d              -> "double: " + d;
        default                    -> "other number: " + n;
    };
}

// --- Sealed Classes + exhaustive switch (không cần default) ---
public sealed interface Shape permits Circle, Rectangle, Triangle {}
public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public record Triangle(double base, double height) implements Shape {}

public double area(Shape shape) {
    return switch (shape) {
        case Circle c      -> Math.PI * c.radius() * c.radius();
        case Rectangle r   -> r.width() * r.height();
        case Triangle t    -> 0.5 * t.base() * t.height();
        // Không cần default: compiler biết Shape chỉ có 3 subtype
        // Nếu thêm SubType mới mà quên update switch, compiler báo lỗi compile-time
    };
}

// --- Sử dụng với Records để destructure ---
public sealed interface JsonValue permits JsonString, JsonNumber, JsonBool, JsonNull {}
public record JsonString(String value) implements JsonValue {}
public record JsonNumber(double value) implements JsonValue {}
public record JsonBool(boolean value) implements JsonValue {}
public record JsonNull() implements JsonValue {}

public String toJavaString(JsonValue json) {
    return switch (json) {
        case JsonString s  -> "\"" + s.value() + "\"";
        case JsonNumber n when n.value() == Math.floor(n.value())
                           -> String.valueOf((long) n.value());
        case JsonNumber n  -> String.valueOf(n.value());
        case JsonBool b    -> String.valueOf(b.value());
        case JsonNull ignored -> "null";
    };
}

// --- Event dispatch: thay thế Visitor Pattern ---
public sealed interface AppEvent permits UserCreated, UserDeleted, OrderPlaced {}
public record UserCreated(String userId, String email) implements AppEvent {}
public record UserDeleted(String userId, String reason) implements AppEvent {}
public record OrderPlaced(String orderId, double amount) implements AppEvent {}

public void handleEvent(AppEvent event) {
    switch (event) {
        case UserCreated uc -> {
            emailService.sendWelcome(uc.email());
            auditLog.record("user.created", uc.userId());
        }
        case UserDeleted ud when ud.reason().equals("GDPR") -> {
            dataEraser.eraseAllData(ud.userId());
            auditLog.record("user.gdpr.deleted", ud.userId());
        }
        case UserDeleted ud -> {
            softDelete(ud.userId());
            auditLog.record("user.deleted", ud.userId());
        }
        case OrderPlaced op when op.amount() > 10_000 -> {
            fraudDetection.flag(op.orderId());
            processOrder(op);
        }
        case OrderPlaced op -> processOrder(op);
    }
}

// --- Pattern matching kết hợp với instanceof (Java 16+) so sánh ---
// TRƯỚC (Java 15 trở xuống):
if (obj instanceof String) {
    String s = (String) obj;  // phải cast thủ công
    System.out.println(s.length());
}

// SAU (Java 16+ instanceof pattern):
if (obj instanceof String s) {
    System.out.println(s.length());  // s đã được bind, không cần cast
}

// TỐT NHẤT (Java 21 switch pattern khi có nhiều kiểu):
String result = switch (obj) {
    case String s  -> "string: " + s.length();
    case Integer i -> "int: " + i;
    default        -> "other";
};
```

## 10. Production concerns

**Scaling**

- Pattern matching switch được compile thành bytecode hiệu quả — the JVM optimizer (HotSpot) có thể optimize thành jump table tương tự switch truyền thống khi có đủ trường hợp. Không có overhead đáng kể so với `if-else instanceof` tương đương.
- Với Sealed Classes, exhaustiveness được kiểm tra ở compile time, giảm bug runtime. Đây là lợi ích lớn nhất ở scale: khi team mở rộng domain model (thêm subclass), compiler báo lỗi ngay, không phải bug production.

**Failure modes**

- Nếu `switch` trên Object không có `case null` và nhận được null input, sẽ ném `NullPointerException`. Luôn thêm `case null ->` nếu input có thể null.
- Dominance error: nếu `case Object o` đặt trước `case String s`, compiler báo compile-time error ("case label is dominated"). Sắp xếp specific trước generic.
- Với non-sealed hierarchy (interface thông thường), cần có `default` — nếu không compiler sẽ báo error. Chỉ sealed hierarchy mới được hưởng lợi từ exhaustiveness check không cần default.

**Monitoring**

- `default` case trong switch Sealed Class thường là "dead code". Nếu bạn monitor và thấy default case được gọi, đây là signal ai đó đã break sealed hierarchy theo cách bất thường (reflection, deserialization).
- Dùng metrics để track distribution của các case: bao nhiêu `UserCreated`, bao nhiêu `OrderPlaced` — giúp phát hiện bất thường trong event stream.

## 11. Common mistakes

- Mistake: Đặt `case Object o` hoặc `case CharSequence s` trước `case String s` trong cùng một switch.
  Fix: Các mẫu cụ thể hơn (more specific type) phải nằm trước mẫu tổng quát hơn. Java compiler sẽ báo lỗi "this case label is dominated by a preceding case label" — hãy đọc lỗi này và sắp xếp lại thứ tự. Nguyên tắc: specific -> general.

- Mistake: Quên `case null` khi input có thể null, dẫn đến NPE ở runtime.
  Fix: Luôn thêm `case null -> handleNull()` nếu có bất kỳ khả năng nào input là null. Hoặc validate null trước khi vào switch: `Objects.requireNonNull(input, "input must not be null")`.

- Mistake: Viết guard quá phức tạp: `case Order o when o.getItems().stream().anyMatch(i -> i.getPrice() > 100 && !i.isDiscounted())`.
  Fix: Trích xuất điều kiện phức tạp thành method riêng: `case Order o when hasExpensiveNonDiscountedItems(o)`. Code switch chỉ nên chứa logic dispatch, không chứa business logic phức tạp.

- Mistake: Dùng Pattern Matching switch thay cho polymorphism trong OOP hierarchy vững chắc.
  Fix: Nếu mỗi Shape biết cách tính area của nó (`circle.area()`), dùng polymorphism. Pattern Matching switch phù hợp khi bạn không muốn hoặc không thể thêm method vào các type (ví dụ: type từ thư viện ngoài, hoặc type dữ liệu thuần túy).

## 12. Sample project

**Constraint cứng: Xây dựng JSON-to-SQL query builder. Tất cả logic dispatch phải dùng Pattern Matching switch. Không được dùng `if-else` hoặc `instanceof` nào trong toàn bộ QueryBuilder class. Phải compile được mà không có warning.**

```java
// Domain: JSON query DSL
public sealed interface QueryNode permits
    AndNode, OrNode, NotNode,
    EqualsNode, GreaterThanNode, LessThanNode,
    IsNullNode, InNode {}

public record AndNode(List<QueryNode> children) implements QueryNode {}
public record OrNode(List<QueryNode> children) implements QueryNode {}
public record NotNode(QueryNode child) implements QueryNode {}
public record EqualsNode(String field, Object value) implements QueryNode {}
public record GreaterThanNode(String field, Number value) implements QueryNode {}
public record LessThanNode(String field, Number value) implements QueryNode {}
public record IsNullNode(String field) implements QueryNode {}
public record InNode(String field, List<Object> values) implements QueryNode {}

// Builder: không có if-else hay instanceof
public class SqlQueryBuilder {

    public String build(QueryNode node) {
        return switch (node) {
            case AndNode and -> and.children().stream()
                .map(this::build)
                .collect(Collectors.joining(" AND ", "(", ")"));

            case OrNode or -> or.children().stream()
                .map(this::build)
                .collect(Collectors.joining(" OR ", "(", ")"));

            case NotNode not -> "NOT (" + build(not.child()) + ")";

            case EqualsNode eq when eq.value() == null ->
                eq.field() + " IS NULL";

            case EqualsNode eq when eq.value() instanceof String s ->
                eq.field() + " = '" + escape(s) + "'";

            case EqualsNode eq ->
                eq.field() + " = " + eq.value();

            case GreaterThanNode gt when gt.value() instanceof Integer i ->
                gt.field() + " > " + i;

            case GreaterThanNode gt ->
                gt.field() + " > " + gt.value().doubleValue();

            case LessThanNode lt ->
                lt.field() + " < " + lt.value();

            case IsNullNode isNull ->
                isNull.field() + " IS NULL";

            case InNode in -> {
                String values = in.values().stream()
                    .map(v -> switch (v) {
                        case String s  -> "'" + escape(s) + "'";
                        case Number n  -> n.toString();
                        case null      -> "NULL";
                        default        -> v.toString();
                    })
                    .collect(Collectors.joining(", "));
                yield in.field() + " IN (" + values + ")";
            }
        };
        // Không cần default: QueryNode là sealed, compiler biết tất cả trường hợp
    }

    private String escape(String s) {
        return s.replace("'", "''");
    }
}

// Sử dụng
QueryNode query = new AndNode(List.of(
    new EqualsNode("status", "ACTIVE"),
    new GreaterThanNode("age", 18),
    new OrNode(List.of(
        new EqualsNode("city", "Hanoi"),
        new EqualsNode("city", "HCMC")
    ))
));

String sql = new SqlQueryBuilder().build(query);
// Kết quả: (status = 'ACTIVE' AND age > 18 AND (city = 'Hanoi' OR city = 'HCMC'))
```

## 13. Interview

### Core Q&A

**Q1: Pattern Matching for Switch là gì và tại sao nó tốt hơn chuỗi if-else instanceof?**
A: Là khả năng dùng type patterns và guarded patterns trong `case` của `switch`. Tốt hơn if-else vì: (1) không cần ép kiểu thủ công, biến được bind tự động; (2) compiler enforce exhaustiveness với Sealed Classes — nếu quên case, báo compile-time error; (3) xử lý null tập trung với `case null`; (4) code dễ đọc hơn khi có nhiều nhánh.

**Q2: "Guarded Pattern" là gì?**
A: Là kết hợp type pattern với điều kiện logic bổ sung bằng từ khóa `when`: `case Integer i when i > 0 -> ...`. Guarded pattern chỉ match khi cả kiểu lẫn điều kiện đều đúng. Điều kiện trong `when` có thể là bất kỳ boolean expression nào.

**Q3: Sealed Class hỗ trợ Pattern Matching switch như thế nào?**
A: Khi `switch` trên một Sealed type, compiler biết tập hợp đầy đủ tất cả subtype. Do đó không cần `default` — compiler sẽ báo lỗi nếu bạn bỏ sót bất kỳ subtype nào. Đây là exhaustiveness check: thêm subtype mới mà quên update switch sẽ bị báo lỗi compile-time thay vì bug runtime.

**Q4: Quy tắc "dominance" trong Pattern Matching switch là gì?**
A: Một case "dominates" case khác nếu nó match mọi thứ mà case kia match. Ví dụ `case Object o` dominates `case String s` vì mọi String cũng là Object. Java compiler bắt lỗi dominance — case specific hơn phải đặt trước case tổng quát hơn.

**Q5: Khi nào cần `default` trong Pattern Matching switch?**
A: Cần `default` khi switch trên type không phải Sealed (ví dụ `Object`, interface thường) — compiler không thể biết hết tất cả subtype nên yêu cầu có nhánh mặc định. Với Sealed type và tất cả permitted subtypes đã được xử lý, `default` là tùy chọn (nhưng có thể thêm để xử lý trường hợp "impossible").

**Q6: Sự khác biệt giữa Pattern Matching for `instanceof` (Java 16) và Pattern Matching for `switch` (Java 21)?**
A: `instanceof` pattern chỉ xử lý một kiểu trong một biểu thức điều kiện: `if (obj instanceof String s)`. Switch pattern xử lý nhiều kiểu cùng lúc trong một cấu trúc tập trung, hỗ trợ guard, hỗ trợ null, và có exhaustiveness check với Sealed types. Switch là công cụ mạnh hơn cho dispatch đa kiểu.

**Q7: Pattern Matching switch có ảnh hưởng đến performance không?**
A: Không đáng kể. JVM compile switch pattern tương tự `if-else` ở bytecode level. HotSpot JIT có thể optimize các pattern switch thành jump table ở runtime. Benchmark thực tế cho thấy tương đương hoặc nhanh hơn if-else tương đương vì compiler có thêm thông tin để optimize.

**Q8: Java 21 có Record Patterns không? Liên hệ với switch như thế nào?**
A: Có, JEP 440 (finalized Java 21) cho phép destructure Record trực tiếp trong pattern: `case Point(int x, int y) when x > 0 -> ...`. Kết hợp với switch, bạn có thể vừa kiểm tra kiểu, vừa lấy field của Record trong cùng một bước — rất mạnh cho DDD value objects.

### Scenario

**S1: Bạn có hierarchy `Animal` với `Dog`, `Cat`, `Bird`. Mỗi con vật có âm thanh khác nhau. Nên dùng polymorphism hay Pattern Matching switch?**
A: Nên dùng polymorphism — thêm method `sound()` vào interface `Animal` và mỗi subclass implement. Pattern Matching switch phù hợp hơn khi bạn không thể hoặc không muốn thêm method vào các type (ví dụ type từ library ngoài, hoặc logic quá phức tạp không nên nằm trong domain object).

**S2: Code này có vấn đề gì?**
```java
Object obj = getInput();
String result = switch (obj) {
    case CharSequence cs -> "CharSequence: " + cs.length();
    case String s        -> "String: " + s;
    default              -> "other";
};
```
A: `case CharSequence cs` dominates `case String s` vì `String` là subtype của `CharSequence`. Case `String s` sẽ không bao giờ được reach. Compiler sẽ báo compile-time error. Fix: đảo ngược thứ tự — `case String s` trước `case CharSequence cs`.

**S3: Làm sao viết switch exhaustive mà không có `default` khi `Shape` là sealed interface?**
A: Đảm bảo tất cả permitted subtypes của sealed interface đều được xử lý:
```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}
// Switch không cần default:
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t  -> 0.5 * t.base() * t.height();
};
// Nếu thêm `Square` vào permits mà quên update switch -> compile error ngay
```

**S4: Trong microservice nhận event từ Kafka, làm sao dùng Pattern Matching switch để xử lý nhiều loại event?**
A:
```java
public sealed interface DomainEvent permits OrderCreated, OrderCancelled, PaymentReceived {}

public void processEvent(DomainEvent event) {
    switch (event) {
        case OrderCreated oc when oc.totalAmount() > 5_000_000 ->
            handleHighValueOrder(oc);
        case OrderCreated oc ->
            handleStandardOrder(oc);
        case OrderCancelled oc ->
            refundService.process(oc.orderId());
        case PaymentReceived pr ->
            orderService.markPaid(pr.orderId(), pr.amount());
    }
    // Compiler đảm bảo tất cả event types được xử lý
}
```

**S5: Bạn được yêu cầu implement `toString()` cho cây biểu thức toán học (AST). Cách nào tốt nhất?**
A: Dùng Sealed Interface + Records + Pattern Matching switch:
```java
sealed interface Expr permits Num, Add, Mul, Neg {}
record Num(double value) implements Expr {}
record Add(Expr left, Expr right) implements Expr {}
record Mul(Expr left, Expr right) implements Expr {}
record Neg(Expr expr) implements Expr {}

String prettyPrint(Expr e) {
    return switch (e) {
        case Num n       -> String.valueOf(n.value());
        case Neg neg     -> "-(" + prettyPrint(neg.expr()) + ")";
        case Add a       -> prettyPrint(a.left()) + " + " + prettyPrint(a.right());
        case Mul m       -> "(" + prettyPrint(m.left()) + " * " + prettyPrint(m.right()) + ")";
    };
}
```

## 14. References

- JEP 441 — Pattern Matching for switch (Java 21, final): https://openjdk.org/jeps/441
- JEP 440 — Record Patterns (Java 21, final): https://openjdk.org/jeps/440
- JEP 409 — Sealed Classes (Java 17, final): https://openjdk.org/jeps/409
- Java Language Specification — Pattern Matching: https://docs.oracle.com/javase/specs/jls/se21/html/jls-14.html#jls-14.11
- Brian Goetz — Data-oriented programming in Java: https://www.infoq.com/articles/data-oriented-programming-java/
- Oracle Java 21 release notes: https://www.oracle.com/java/technologies/javase/21-relnote-issues.html

## 15. Real-world Code

- Quarkus — uses sealed types + switch patterns in internal event processing: https://github.com/quarkusio/quarkus/search?q=sealed
- Spring Framework — exploring records and patterns in 6.x: https://github.com/spring-projects/spring-framework/search?q=instanceof+pattern
- Hibernate — type dispatch using instanceof patterns: https://github.com/hibernate/hibernate-orm/search?q=instanceof
- Error Prone (Google) — checks for pattern matching correctness: https://github.com/google/error-prone

## 16. Community

- Stack Overflow — "Java 21 Pattern Matching for switch examples": https://stackoverflow.com/questions/tagged/java-21+switch
- Reddit r/java — Pattern matching discussions: https://www.reddit.com/r/java/search/?q=pattern+matching+switch
- Baeldung — Pattern Matching for switch in Java 21: https://www.baeldung.com/java-switch-pattern-matching
- InfoQ — Java 21: The new features and what they mean: https://www.infoq.com/news/2023/09/java21-released/
- Inside Java — Pattern Matching deep dive: https://inside.java/2023/09/19/switch-pattern-matching/
- Dev.to — Sealed classes and pattern matching practical guide: https://dev.to/search?q=java+sealed+pattern+matching
