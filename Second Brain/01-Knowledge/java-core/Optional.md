---
created: 2026-04-17
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/null-safety"
related: "[[Pattern Matching for Switch]]"
---

# Optional

## 1. What

`Optional<T>` là một container object trong Java 8+ có thể chứa hoặc không chứa một giá trị non-null. Nó được thiết kế để làm rõ ý định của API — khi một method có thể không trả về giá trị thay vì trả về `null`. Thay vì trả về `null` và hy vọng caller nhớ kiểm tra, method trả về `Optional<T>` để compiler buộc caller phải đối mặt với khả năng không có giá trị.

## 2. Why

Trước `Optional`, Java developer phải đối mặt với `NullPointerException` (NPE) — lỗi runtime khó debug nhất. Vấn đề là `null` không mang ngữ nghĩa: khi một method trả về `null`, caller không biết đây là lỗi, là "không tìm thấy", hay là giá trị mặc định hợp lệ. Code phòng thủ với `null` check tràn lan khắp nơi, làm giảm readability và dễ bỏ sót. `Optional` giải quyết vấn đề này bằng cách đưa khả năng "không có giá trị" vào type system, buộc developer phải xử lý tường minh.

## 3. Mental Model

Hãy nghĩ `Optional<T>` như một chiếc hộp quà có nắp. Người tặng (method trả về) đặt quà vào hộp hoặc để hộp trống rồi giao cho bạn (caller). Bạn **không được** mở hộp ra mà không kiểm tra trước — đó là `get()` không có `isPresent()`, tương đương mở hộp mà không biết có quà không. Thay vào đó, bạn nói: "Nếu có quà thì mở ra dùng (`ifPresent`), nếu không có thì dùng quà dự phòng (`orElse`), hoặc ném exception nếu nhất quyết phải có quà (`orElseThrow`)." Hộp chỉ là phương tiện vận chuyển — bạn không nên cất hộp vào tủ (dùng làm field) hay bọc hộp vào hộp khác (Optional trong collection).

## 4. Where it fits

```
Business Logic Layer
       |
       v
Service method returns Optional<User>
       |
       v
Controller / Caller unpacks Optional
       |
   [present] -> use value
   [empty]   -> return 404 / default value / throw exception
```

Trong kiến trúc layered, `Optional` thường xuất hiện tại tầng Repository và Service khi query có thể không tìm thấy kết quả.

## 5. When to use

- Return type của method mà kết quả có thể không tồn tại: `findById()`, `findFirst()`, `getConfig()`.
- Khi muốn buộc caller phải xử lý trường hợp "không có giá trị" ngay tại compile time.
- Khi viết fluent pipeline xử lý dữ liệu nullable mà không muốn lồng `if-null` check.
- Thay thế pattern `null` + Javadoc "có thể trả về null" bằng type tường minh hơn.

## 6. When NOT to use

- **Không dùng làm field của class**: làm tăng overhead serialization, phá vỡ các framework như JPA/Hibernate, và không phải ý định thiết kế của tác giả.
- **Không dùng làm method parameter**: caller sẽ phải bọc argument vào `Optional.of(x)` trước khi gọi — vô nghĩa và cồng kềnh. Dùng overloading hoặc `@Nullable` thay thế.
- **Không dùng trong Collection**: `List<Optional<T>>` là anti-pattern. Nếu muốn biểu diễn "có thể không có", hãy dùng `List<T>` và lọc null, hoặc dùng collection trống.
- **Không gọi `get()` trực tiếp**: nếu Optional rỗng sẽ ném `NoSuchElementException`. Luôn dùng `orElse`, `orElseGet`, hoặc `orElseThrow`.

## 7. Trade-offs

| Pros | Cons |
|------|------|
| Loại bỏ NPE ở tầng API design | Overhead object allocation cho mỗi lần gọi |
| Ý định rõ ràng: "method này có thể không có kết quả" | Verbose hơn null check đơn giản |
| Hỗ trợ functional pipeline với `map`, `flatMap`, `filter` | Không serializable (không dùng được với JPA field) |
| Tương thích tốt với Stream API | Dễ bị lạm dụng sai mục đích |
| Compiler cảnh báo khi bạn không xử lý giá trị | Hiệu năng thấp hơn null check thuần túy trong hot path |

## 8. Alternatives

| Approach | Ưu điểm | Nhược điểm |
|----------|---------|------------|
| `Optional<T>` | Type-safe, fluent API, rõ ý định | Overhead allocation |
| `null` + Javadoc | Không overhead | Không được enforce, dễ quên kiểm tra |
| `@Nullable` / `@NonNull` (JSR-305) | IDE và static analysis hỗ trợ | Chỉ là annotation, không enforce runtime |
| Throw exception | Rõ ràng khi không tìm thấy là lỗi thực sự | Sai khi "không tìm thấy" là trạng thái bình thường |
| Return empty collection | Phù hợp khi kết quả là tập hợp | Không phù hợp khi kết quả là single object |
| `Try<T>` (Vavr) | Bắt cả exception lẫn absent value | Phụ thuộc thư viện ngoài |

## 9. How

```java
import java.util.Optional;

// --- Tạo Optional ---
Optional<String> present  = Optional.of("hello");           // NPE nếu argument null
Optional<String> nullable = Optional.ofNullable(getUserName()); // safe với null
Optional<String> empty    = Optional.empty();

// --- Terminal operations ---
// NGUY HIỂM: get() ném NoSuchElementException nếu empty
// String val1 = present.get();

String val2 = nullable.orElse("default");                   // luôn evaluate "default"
String val3 = nullable.orElseGet(() -> computeDefault());   // lazy evaluate — chỉ gọi khi empty
String val4 = nullable.orElseThrow(
    () -> new UserNotFoundException("user not found")
);

// ifPresent: chỉ thực thi khi có giá trị
nullable.ifPresent(name -> log.info("User: {}", name));     // Java 8

// ifPresentOrElse: xử lý cả hai nhánh — Java 9+
nullable.ifPresentOrElse(
    name -> log.info("User: {}", name),
    ()   -> log.warn("No user found")
);

// --- Chaining: map, flatMap, filter ---
Optional<Integer> nameLength = nullable
    .filter(name -> !name.isBlank())   // bỏ qua nếu blank
    .map(String::length);              // biến đổi nếu có giá trị

// flatMap khi hàm bên trong cũng trả về Optional
Optional<String> city = findUser(userId)       // -> Optional<User>
    .flatMap(user -> findAddress(user.getId())) // -> Optional<Address>
    .map(Address::getCity);                     // -> Optional<String>

// Java 9: or() — cung cấp Optional thay thế khi empty
Optional<User> user = findInCache(id)
    .or(() -> findInDatabase(id));

// Java 9: stream() — chuyển Optional thành Stream (0 hoặc 1 phần tử)
List<String> names = List.of(
        Optional.of("Alice"),
        Optional.empty(),
        Optional.of("Bob"))
    .stream()
    .flatMap(Optional::stream)  // lọc empty, unwrap present
    .collect(Collectors.toList()); // ["Alice", "Bob"]

// --- Spring Data JPA pattern ---
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    Optional<User> findFirstByStatusOrderByCreatedAtDesc(UserStatus status);
}

// --- Service layer ---
public UserDto getUser(Long id) {
    return userRepository.findById(id)
        .map(userMapper::toDto)
        .orElseThrow(() -> new UserNotFoundException(
            "User %d not found".formatted(id)));
}

// --- REST Controller ---
@GetMapping("/{id}")
public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
    return userService.findById(id)
        .map(userMapper::toDto)
        .map(ResponseEntity::ok)
        .orElse(ResponseEntity.notFound().build());
}
```

## 10. Production concerns

**Scaling**

`Optional` tạo thêm 1 object allocation cho mỗi lần wrap. Trong hot path (hàng triệu lần/giây), overhead này có thể đáng kể. JVM Escape Analysis thường optimize được nếu `Optional` không escape ra ngoài method, nhưng đừng giả định. Với Java 21+ và Virtual Threads, cần chú ý khi Optional chain gọi blocking I/O — đảm bảo dùng `orElseGet` với lazy supplier thay vì `orElse` evaluate eager.

**Failure modes**

- `NoSuchElementException` khi gọi `get()` trên empty Optional — đây là runtime exception, không bị compiler bắt.
- Stack trace từ `orElseThrow` cần message đủ thông tin để debug (bao gồm ID, context).
- Serialization: `Optional` không implement `Serializable`. Nếu dùng trong DTO được serialize bởi Jackson, cần thêm `jackson-datatype-jdk8` module và đăng ký `Jdk8Module`.

```java
// Jackson configuration
ObjectMapper mapper = new ObjectMapper()
    .registerModule(new Jdk8Module());
```

**Monitoring**

- Nếu `orElseThrow` ném exception thường xuyên, đây là signal có vấn đề ở tầng data consistency. Track exception rate.
- Dùng `ifPresent` kết hợp metrics: `optional.ifPresent(v -> counter.increment("cache.hit"))`.
- Log và alert khi tỷ lệ "not found" (empty Optional) tăng đột biến.

## 11. Common mistakes

- Mistake: Gọi `optional.get()` trực tiếp mà không check `isPresent()` trước.
  Fix: Dùng `orElseThrow()` với exception message rõ ràng, hoặc `orElse(defaultValue)`. `get()` chỉ nên dùng sau khi đã gọi `isPresent()` và kết quả là `true` — nhưng ngay cả khi đó, `orElseThrow` vẫn rõ ràng hơn và bảo vệ code tốt hơn khi có refactor.

- Mistake: Dùng `Optional` làm field của JPA Entity hoặc DTO.
  Fix: Giữ field là `String name` (nullable). Chỉ wrap vào Optional ở return type của method: `public Optional<String> getName() { return Optional.ofNullable(name); }`. JPA, Jackson, và các framework khác làm việc trực tiếp với field, không qua getter Optional.

- Mistake: Dùng `orElse(expensiveComputation())` khi muốn lazy evaluation.
  Fix: Dùng `orElseGet(() -> expensiveComputation())`. `orElse` luôn evaluate argument ngay cả khi Optional có giá trị. Với side-effect operations như database call, đây là bug nghiêm trọng — bạn đang gọi DB cho dù Optional đã có giá trị.

- Mistake: Lồng Optional: `Optional<Optional<User>>` do gọi `.map()` với function trả về `Optional`.
  Fix: Dùng `flatMap()` thay vì `map()` khi function trả về `Optional<T>`. `flatMap` tự động unwrap lớp Optional bên trong.

## 12. Sample project

**Constraint cứng: Xây dựng UserProfileService không được có một dòng `if (x == null)` nào trong source code.**

```java
// Domain objects
public record Address(String street, String city, String country) {}
public record User(Long id, String name, String email, Address address) {}

// Repository interface
public interface UserRepository {
    Optional<User> findById(Long id);
    Optional<User> findByEmail(String email);
}

// External service
public interface GeoService {
    Optional<String> getTimezone(String country, String city);
}

// Service — không một dòng null check nào
@Service
public class UserProfileService {

    private final UserRepository userRepository;
    private final GeoService geoService;
    private final UserCache cache;

    public UserProfileService(
            UserRepository userRepository,
            GeoService geoService,
            UserCache cache) {
        this.userRepository = userRepository;
        this.geoService = geoService;
        this.cache = cache;
    }

    // Lấy city của user, fallback về "Unknown"
    public String getUserCity(Long userId) {
        return userRepository.findById(userId)
            .map(User::address)
            .map(Address::city)
            .filter(city -> !city.isBlank())
            .orElse("Unknown");
    }

    // Tìm user theo email, nếu không có thì tạo guest user
    public User findOrCreateGuest(String email) {
        return userRepository.findByEmail(email)
            .orElseGet(() -> createGuestUser(email));
    }

    // Lấy timezone từ address của user
    public Optional<String> getUserTimezone(Long userId) {
        return userRepository.findById(userId)
            .map(User::address)
            .flatMap(addr -> geoService.getTimezone(addr.country(), addr.city()));
    }

    // Cập nhật email, throw nếu user không tồn tại
    public User updateEmail(Long userId, String newEmail) {
        return userRepository.findById(userId)
            .map(u -> new User(u.id(), u.name(), newEmail, u.address()))
            .orElseThrow(() -> new UserNotFoundException(
                "Cannot update email: user %d not found".formatted(userId)));
    }

    // Java 9+: tìm trong cache trước, rồi mới hit database
    public Optional<User> findUser(Long userId) {
        return cache.get(userId)
            .or(() -> userRepository.findById(userId));
    }

    // Log warning nếu user không tìm thấy
    public void sendWelcomeEmail(Long userId) {
        userRepository.findById(userId)
            .map(User::email)
            .filter(email -> !email.isBlank())
            .ifPresentOrElse(
                email -> emailService.sendWelcome(email),
                () -> log.warn("Cannot send welcome: user {} not found", userId)
            );
    }

    private User createGuestUser(String email) {
        return new User(null, "Guest", email, null);
    }
}
```

## 13. Interview

### Core Q&A

**Q1: Sự khác biệt giữa `Optional.of()` và `Optional.ofNullable()` là gì?**
A: `Optional.of(value)` ném `NullPointerException` ngay lập tức nếu `value` là null — dùng khi bạn chắc chắn 100% giá trị không null. `Optional.ofNullable(value)` wrap null thành `Optional.empty()` một cách an toàn — dùng khi value có thể null. Trong hầu hết trường hợp production, `ofNullable` là lựa chọn an toàn hơn.

**Q2: Tại sao không nên dùng `Optional` làm parameter của method?**
A: Caller sẽ phải viết `method(Optional.of(value))` hoặc `method(Optional.empty())` — cồng kềnh và không tự nhiên. `Optional` được thiết kế để signal "kết quả có thể không có" ở return type. Nếu muốn parameter optional, dùng overloading (`method(String s)` và `method()`) hoặc dùng `@Nullable` annotation. Effective Java của Joshua Bloch cũng khuyến nghị điều này.

**Q3: `orElse()` vs `orElseGet()` — khi nào dùng cái nào?**
A: `orElse(T other)` evaluate `other` **ngay lập tức** bất kể Optional có giá trị hay không. `orElseGet(Supplier<T> supplier)` chỉ gọi supplier khi Optional empty (lazy). Nếu default value là constant hoặc cheap, dùng `orElse`. Nếu computing default value có cost (DB call, object construction, I/O), luôn dùng `orElseGet`. Đây là lỗi thường gặp nhất trong production với Optional.

**Q4: `map()` vs `flatMap()` trong Optional?**
A: `map(Function<T, R> f)` dùng khi `f` trả về giá trị thông thường (non-Optional). `flatMap(Function<T, Optional<R>> f)` dùng khi `f` đã trả về `Optional<R>` — tránh lồng `Optional<Optional<R>>`. Tương tự Stream API, quy tắc này áp dụng nhất quán.

**Q5: Java 9 thêm gì vào Optional?**
A: Ba method quan trọng: `ifPresentOrElse(Consumer, Runnable)` — xử lý cả hai nhánh present và empty; `or(Supplier<Optional<T>>)` — cung cấp Optional thay thế khi empty (thay vì value như `orElseGet`); `stream()` — chuyển Optional thành Stream 0 hoặc 1 phần tử, hữu ích khi flatMap trong Stream pipeline để lọc null.

**Q6: Optional có Serializable không? Hệ quả là gì?**
A: `Optional` không implement `Serializable`. Hệ quả: không thể dùng làm field của Serializable class, không dùng được với Java serialization, với một số cache framework. Jackson mặc định không biết serialize `Optional` — cần thêm `jackson-datatype-jdk8` và đăng ký `Jdk8Module`.

**Q7: Khi nào nên throw exception thay vì trả về `Optional.empty()`?**
A: Trả về `Optional.empty()` khi "không tìm thấy" là trạng thái hợp lệ và bình thường của business logic (ví dụ: search không có kết quả). Throw exception khi "không tìm thấy" là vi phạm invariant hoặc precondition (ví dụ: `getById` trong update operation — nếu không tìm thấy là data inconsistency). Nguyên tắc: nếu caller có thể xử lý "không có" một cách hợp lý, dùng Optional; nếu không có là bug, dùng exception.

**Q8: Optional có ảnh hưởng đến hiệu năng không?**
A: Có nhưng thường không đáng kể. Mỗi `Optional.of()` tạo một object allocation nhỏ trên heap. Với JVM hiện đại, các object sống ngắn như vậy thường được GC nhanh (young generation). JVM Escape Analysis đôi khi có thể optimize away allocation nếu Optional không escape method. Chỉ cần lo ở hot path thực sự (>10 triệu lần/giây) — lúc đó dùng null check trực tiếp hoặc `@Nullable`.

### Scenario

**S1: Bạn có chain: `userRepo.findById(id).map(User::getProfile).map(Profile::getAvatar).get()`. Code này có vấn đề gì?**
A: Gọi `get()` ở cuối chain là nguy hiểm — nếu bất kỳ bước nào trả về empty, sẽ ném `NoSuchElementException`. Fix: thay `get()` bằng `orElse(defaultAvatar)` hoặc `orElseThrow(() -> new ProfileNotFoundException("..."))`.

**S2: Teammate viết `Optional.of(user).map(User::getName).orElse(null)`. Bạn review code này như thế nào?**
A: Đây là anti-pattern — bạn tạo Optional chỉ để unwrap về null. Kết quả tương đương `user.getName()` nhưng với overhead. Optional nên xuất hiện ở return type của repository/service, không nên tạo thủ công để rồi lại về null. Nếu `user` có thể null, dùng `Optional.ofNullable(user).map(User::getName).orElse(null)` — nhưng lúc này nên dùng `@Nullable` hoặc null check thông thường cho đơn giản.

**S3: Làm thế nào để filter một `List<User>` lấy những user có city là "Hanoi" khi `getCity()` trả về `Optional<String>`?**
A:
```java
List<User> hanoiUsers = users.stream()
    .filter(u -> u.getCity()
        .map("Hanoi"::equals)
        .orElse(false))
    .collect(Collectors.toList());
```

**S4: orElse và orElseGet có hậu quả gì khác nhau với code sau?**
```java
// Code A
User user = findUser(id).orElse(createDefaultUser());
// Code B
User user = findUser(id).orElseGet(() -> createDefaultUser());
```
A: Nếu `findUser(id)` trả về present Optional, code A vẫn gọi `createDefaultUser()` (lãng phí). Code B chỉ gọi `createDefaultUser()` khi Optional empty. Nếu `createDefaultUser()` có side effect (ghi DB, gọi API), code A là bug nghiêm trọng.

**S5: Bạn đang đọc code cũ thấy pattern này: `if (user != null && user.getEmail() != null && !user.getEmail().isEmpty())`. Refactor bằng Optional?**
A:
```java
Optional.ofNullable(user)
    .map(User::getEmail)
    .filter(email -> !email.isEmpty())
    .ifPresent(email -> sendWelcomeEmail(email));
```

## 14. References

- Java SE 21 API Docs — Optional: https://docs.oracle.com/en/java/docs/api/java.base/java/util/Optional.html
- JDK Enhancement Proposal — Optional design: https://bugs.openjdk.org/browse/JDK-8050820
- Effective Java, 3rd Edition — Item 55: Return optionals judiciously (Joshua Bloch)
- Stuart Marks — "Optional: The Mother of All Bikesheds" (Devoxx 2016): https://www.youtube.com/watch?v=Ej0sss6cq14
- Oracle Java Magazine — 12 recipes for using Optional: https://blogs.oracle.com/javamagazine/post/12-recipes-for-using-the-optional-class-as-its-meant-to-be-used
- jackson-datatype-jdk8 module: https://github.com/FasterXML/jackson-modules-java8

## 15. Real-world Code

- Spring Data Commons — `CrudRepository.findById()` returns `Optional`: https://github.com/spring-projects/spring-data-commons/blob/main/src/main/java/org/springframework/data/repository/CrudRepository.java
- Spring Security — `Authentication` optional handling: https://github.com/spring-projects/spring-security/search?q=Optional
- Hibernate ORM — Optional in TypedQuery: https://github.com/hibernate/hibernate-orm/search?q=Optional
- Quarkus Panache — `findByIdOptional()`: https://github.com/quarkusio/quarkus/search?q=findByIdOptional

## 16. Community

- Stack Overflow — "Should Java 8 getters return Optional type?": https://stackoverflow.com/questions/26327957/should-java-8-getters-return-optional-type
- Stack Overflow — "Optional.of vs Optional.ofNullable": https://stackoverflow.com/questions/31696485/why-use-optional-of-over-optional-ofnullable
- Reddit r/java — Optional misuse patterns: https://www.reddit.com/r/java/search/?q=optional+misuse
- Baeldung — Guide to Java 8 Optional: https://www.baeldung.com/java-optional
- DZone — Using Optional correctly is not optional: https://dzone.com/articles/using-optional-correctly-is-not-optional
- InfoQ — Tired of Null Pointer Exceptions: https://www.infoq.com/articles/java-optional-class/
