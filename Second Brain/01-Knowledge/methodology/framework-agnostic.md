---
created: 2026-04-17
tags:
  - type/pattern
  - status/draft
  - lang/java
related: []
---

# Framework-Agnostic Design

## 1. What

Framework-agnostic design (thiết kế không phụ thuộc framework) là nguyên tắc kiến trúc phần mềm trong đó logic nghiệp vụ (business logic) được tách biệt hoàn toàn khỏi các concern của framework cụ thể như Spring, Jakarta EE, hay Micronaut. Business logic được viết bằng Java thuần, không có annotation hay kiểu dữ liệu của framework xâm nhập vào lớp domain. Framework đóng vai trò là "delivery mechanism" phục vụ domain, không phải ngược lại. Nguyên tắc này là trái tim của Hexagonal Architecture (Ports and Adapters) do Alistair Cockburn định nghĩa.

---

## 2. Why

Khi phát triển theo phong cách "framework-first", annotation của framework lan dần vào khắp nơi:

```java
// Anti-pattern: Spring annotation trong domain model
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private Long id;

    @Transactional
    public void confirm() {
        // business logic tron lan voi persistence concern
        this.status = Status.CONFIRMED;
    }
}
```

Hậu quả của cách này:
- Không thể unit test `Order.confirm()` mà không spin up Spring context và Hibernate session.
- Khi đổi từ Spring sang Quarkus hoặc Micronaut, tất cả các annotation phải đổi theo.
- Domain model trở nên "heavy" — nó biết quá nhiều về cách nó được lưu trữ, cách nó được expose qua HTTP.
- Các rule nghiệp vụ bị chôn vùi giữa các layer của framework, khó đọc, khó test, khó tìm.

Framework-agnostic design giải quyết những vấn đề này bằng cách đặt ra ranh giới rõ ràng: domain không biết gì về framework; framework biết về domain nhưng chỉ thông qua các interface (ports) mà domain định nghĩa.

---

## 3. Mental Model

Hãy nghĩ domain của bạn như **bộ máy chiếu phim (film projector) bên trong rạp chiếu bóng**.

Bộ máy chiếu phim (domain / business logic) chỉ biết một việc: chiếu phim. Nó không biết rạp chiếu được xây bằng vật liệu gì, hệ thống điều hòa là của hãng nào, hay ghế ngồi có đệm bọc hay không. Nó có các cổng kết nối chuẩn (ports): cổng điện, cổng cấp tín hiệu — bất kỳ thiết bị nào tuân thủ chuẩn đó đều có thể kết nối.

Framework (Spring, Hibernate, REST API, message queue) là **cơ sở vật chất của rạp chiếu** — tường, dây điện, ghế. Chúng phục vụ bộ máy chiếu, cung cấp nguồn điện và nơi phát âm thanh, nhưng chúng không thay đổi cách bộ máy chiếu hoạt động bên trong.

Nếu bạn muốn chuyển từ rạp chiếu cũ sang rạp chiếu mới (đổi framework), bạn vẫn dùng nguyên bộ máy chiếu — chỉ cần kết nối lại các dây.

**Adapter** là **đầu chuyển đổi phích cắm**: nó biết cả hai phía — một phía là port chuẩn của bộ máy (interface), một phía là đặc điểm cụ thể của cơ sở vật chất (Spring Bean, JPA Repository).

---

## 4. Where it fits

```
HTTP / gRPC / Message Queue / CLI
         |
         v
[Adapter Layer: Controllers, MessageListeners, CLI handlers]
  -- Spring @Controller, @KafkaListener, etc. --
         |
         v (goi qua Port interface)
[Application Layer: Use Cases / Application Services]
  -- Pure Java, no framework annotations --
         |
         v (goi qua Port interface)
[Domain Layer: Entities, Value Objects, Domain Services]
  -- Pure Java, no framework, no JPA, no Spring --
         |
         ^ (implements Port interface)
[Adapter Layer: Repositories, External Services]
  -- Spring @Repository, JPA, RestTemplate, etc. --
```

Domain và Application layer không biết gì về layer bên ngoài. Các mũi tên phụ thuộc chỉ hướng vào trong (Dependency Inversion Principle).

---

## 5. When to use

- Ứng dụng có vòng đời kép (long-lived): khi bạn dự kiến maintain sản phẩm trong 3 năm trở lên, framework-agnostic giúp giảm technical debt.
- Business logic phức tạp: khi domain có nhiều rule, workflow, và invariant quan trọng — tách biệt giúp cho phép test từng use case một cách độc lập.
- Yêu cầu testability cao: khi team theo TDD hoặc có yêu cầu code coverage cao — unit test domain logic chạy trong microseconds, không cần Spring context.
- Khả năng migration framework: khi có khả năng chuyển đổi framework trong tương lai (Spring Boot -> Quarkus, hoặc thêm CLI adapter bên cạnh REST adapter).
- Nhiều delivery mechanism: khi cùng một business logic cần được expose qua cả REST API, gRPC, message consumer, và scheduled job.
- Micro-service cùng team lớn: mỗi bounded context có domain riêng, giao tiếp qua event — framework-agnostic giúp mỗi team chọn framework phù hợp mà không ảnh hưởng nhau.

---

## 6. When NOT to use

- CRUD đơn giản, ít logic: khi ứng dụng chủ yếu là wrapper cho database (read/write entity), thêm layer trung gian chỉ là boilerplate không có giá trị.
- MVP / prototype: khi cần delivery nhanh để validate business, ưu tiên tốc độ hơn kiến trúc.
- Team nhỏ, project ngắn hạn: overhead viết port interface và adapter lớn hơn lợi ích nếu project chỉ sống 6 tháng và team chỉ có 2-3 người.
- Framework-specific feature là trái tim của ứng dụng: ví dụ, ứng dụng chủ yếu là Spring Batch pipeline hoặc Spring Integration flow — các framework này là essence, không phải delivery mechanism.

Hậu quả nếu dùng sai: thêm 2-3 lớp abstraction không cần thiết, code base bị "over-engineered", team mới khó hiểu dự án, velocity chậm hơn trong giai đoạn đầu mà không có lợi ích rõ ràng.

---

## 7. Trade-offs

| Pros | Cons |
|------|------|
| Unit test business logic cực nhanh, không cần Spring context | Thêm nhiều interface, class, package — code base lớn hơn |
| Domain có thể được test độc lập hoàn toàn bằng JUnit thuần | Indirection: phải trace qua nhiều lớp để hiểu một tính năng |
| Dễ swap framework (Spring -> Quarkus) mà không đổi domain | Điều kiện bên tiện ích (Utilities) đôi khi khó phân loại |
| Các quy tắc nghiệp vụ tập trung một chỗ, dễ đọc và sửa sau khi quen | Cần discipline cao từ toàn team — dễ "leak" annotation vào domain |
| Adapter có thể được mock/stub đơn giản trong test | Overhead khi setup: POM, package structure, interface naming convention |
| Separation of concern giúp nhiều dev làm việc song song | Một số tính năng Spring (Spring Events, `@Scheduled`) khó dùng mà không để "lộ" |

---

## 8. Alternatives

| Alternative | Mô tả | Khi nào phù hợp |
|---|---|---|
| Framework-first (Anemic Domain) | Entity chỉ là POJO, logic nằm trong @Service | CRUD app, team nhỏ, MVP |
| Layered Architecture (3-tier) | Controller -> Service -> Repository, dùng Spring annotation tự do | Ứng dụng trung bình, team quen Spring |
| Hexagonal Architecture | Framework-agnostic đầy đủ, ports & adapters chính thức | Complex domain, long-lived app |
| Clean Architecture (Uncle Bob) | Rings model: Entities, Use Cases, Interface Adapters, Frameworks | Large-scale, nhiều delivery mechanism |
| CQRS + Event Sourcing | Tách read/write, lưu sự kiện thay vì state | Audit trail, event-driven system |
| Modular Monolith | Modules có well-defined interface, vẫn dùng Spring bên trong | Migration path từ monolith sang microservices |

---

## 9. How

Đây là ví dụ thực tế: tính năng "Đặt đơn hàng" (Place Order) trong hệ thống e-commerce.

### Package structure

```
src/main/java/com/example/
    domain/                         -- Pure Java, zero framework
        model/
            Order.java              -- Domain Entity (no @Entity)
            OrderItem.java
            Money.java              -- Value Object
        port/
            in/
                PlaceOrderUseCase.java   -- Input Port (interface)
            out/
                OrderRepository.java     -- Output Port (interface)
                PaymentGateway.java      -- Output Port (interface)
    application/                    -- Pure Java, no Spring annotation
        usecase/
            PlaceOrderService.java  -- Implements PlaceOrderUseCase
    adapter/                        -- Framework code lives here
        in/
            web/
                OrderController.java     -- Spring @RestController
        out/
            persistence/
                JpaOrderRepository.java  -- Spring @Repository
                OrderJpaEntity.java      -- @Entity (JPA concern)
                OrderMapper.java         -- domain <-> jpa entity
            payment/
                StripePaymentGateway.java
```

### Domain model (pure Java, không Spring, không JPA)

```java
// domain/model/Order.java
public class Order {

    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private OrderStatus status;

    // Constructor dung factory method -- khong de framework tao object
    public static Order create(CustomerId customerId, List<OrderItem> items) {
        if (items == null || items.isEmpty()) {
            throw new DomainException("Order must have at least one item");
        }
        return new Order(OrderId.generate(), customerId, items, OrderStatus.PENDING);
    }

    // Business rule nam trong domain
    public void confirm(Money availableBalance) {
        if (status != OrderStatus.PENDING) {
            throw new DomainException("Only PENDING orders can be confirmed");
        }
        Money total = calculateTotal();
        if (availableBalance.isLessThan(total)) {
            throw new InsufficientFundsException(total, availableBalance);
        }
        this.status = OrderStatus.CONFIRMED;
    }

    public Money calculateTotal() {
        return items.stream()
                    .map(OrderItem::subtotal)
                    .reduce(Money.ZERO, Money::add);
    }

    // Getters only -- no setters to enforce invariants
    public OrderId getId() { return id; }
    public OrderStatus getStatus() { return status; }
    public List<OrderItem> getItems() { return Collections.unmodifiableList(items); }
    // ...
}
```

### Input Port (interface)

```java
// domain/port/in/PlaceOrderUseCase.java
public interface PlaceOrderUseCase {

    OrderId placeOrder(PlaceOrderCommand command);

    // Command object -- pure data, no framework dependency
    record PlaceOrderCommand(
        CustomerId customerId,
        List<OrderItemRequest> items
    ) {}
}
```

### Output Ports (interfaces)

```java
// domain/port/out/OrderRepository.java
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

// domain/port/out/PaymentGateway.java
public interface PaymentGateway {
    PaymentResult charge(CustomerId customerId, Money amount);
}
```

### Application Service (pure Java, no Spring annotation)

```java
// application/usecase/PlaceOrderService.java
public class PlaceOrderService implements PlaceOrderUseCase {

    private final OrderRepository orderRepository;
    private final PaymentGateway  paymentGateway;

    // Constructor injection -- khong co @Autowired trong class nay
    public PlaceOrderService(OrderRepository orderRepository,
                              PaymentGateway  paymentGateway) {
        this.orderRepository = orderRepository;
        this.paymentGateway  = paymentGateway;
    }

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        List<OrderItem> items = mapToOrderItems(command.items());
        Order order = Order.create(command.customerId(), items);

        Money total  = order.calculateTotal();
        PaymentResult payment = paymentGateway.charge(command.customerId(), total);

        if (!payment.isSuccessful()) {
            throw new PaymentFailedException(payment.errorCode());
        }

        order.confirm(payment.authorizedAmount());
        orderRepository.save(order);

        return order.getId();
    }

    private List<OrderItem> mapToOrderItems(List<OrderItemRequest> requests) {
        return requests.stream()
            .map(r -> new OrderItem(r.productId(), r.quantity(), r.unitPrice()))
            .toList();
    }
}
```

### Adapter: Spring Configuration (framework code nằm đây)

```java
// adapter/config/UseCaseConfig.java
@Configuration
public class UseCaseConfig {

    // Spring biet ve PlaceOrderService -- domain khong biet ve Spring
    @Bean
    public PlaceOrderUseCase placeOrderUseCase(
            OrderRepository orderRepository,
            PaymentGateway  paymentGateway) {
        return new PlaceOrderService(orderRepository, paymentGateway);
    }
}
```

### Adapter: JPA Repository (framework concern nằm đây)

```java
// adapter/out/persistence/JpaOrderRepository.java
@Repository
public class JpaOrderRepository implements OrderRepository {

    private final SpringDataOrderRepository springRepo;
    private final OrderMapper mapper;

    public JpaOrderRepository(SpringDataOrderRepository springRepo,
                               OrderMapper mapper) {
        this.springRepo = springRepo;
        this.mapper     = mapper;
    }

    @Override
    public void save(Order order) {
        OrderJpaEntity entity = mapper.toJpaEntity(order);
        springRepo.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return springRepo.findById(id.value())
                         .map(mapper::toDomainOrder);
    }
}

// Spring Data interface -- framework specific
interface SpringDataOrderRepository extends JpaRepository<OrderJpaEntity, Long> {}
```

### Adapter: REST Controller

```java
// adapter/in/web/OrderController.java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final PlaceOrderUseCase placeOrderUseCase;

    public OrderController(PlaceOrderUseCase placeOrderUseCase) {
        this.placeOrderUseCase = placeOrderUseCase;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public PlaceOrderResponse placeOrder(@RequestBody @Valid PlaceOrderRequest request) {
        PlaceOrderCommand command = new PlaceOrderCommand(
            new CustomerId(request.customerId()),
            request.items().stream()
                   .map(i -> new OrderItemRequest(i.productId(), i.quantity(), i.unitPrice()))
                   .toList()
        );
        OrderId orderId = placeOrderUseCase.placeOrder(command);
        return new PlaceOrderResponse(orderId.value());
    }
}
```

### @Transactional dùng ở đâu?

```java
// @Transactional thuoc ve adapter/persistence, KHONG phai domain
// Cach 1: dat tren JpaOrderRepository.save() -- fine
// Cach 2 (khuyen nghi hon): tao TransactionalPlaceOrderService wrapper

@Service
public class TransactionalPlaceOrderService implements PlaceOrderUseCase {

    private final PlaceOrderService delegate;

    public TransactionalPlaceOrderService(PlaceOrderService delegate) {
        this.delegate = delegate;
    }

    @Override
    @Transactional
    public OrderId placeOrder(PlaceOrderCommand command) {
        return delegate.placeOrder(command);
    }
}
```

### Unit test domain — không cần Spring, chạy trong < 1ms

```java
class PlaceOrderServiceTest {

    // Mock output ports bang Mockito -- khong can Spring context
    private final OrderRepository  orderRepository  = mock(OrderRepository.class);
    private final PaymentGateway   paymentGateway   = mock(PaymentGateway.class);
    private final PlaceOrderService service =
        new PlaceOrderService(orderRepository, paymentGateway);

    @Test
    void placeOrder_whenPaymentSucceeds_savesConfirmedOrder() {
        // Arrange
        var command  = buildValidCommand();
        var payment  = PaymentResult.successful(Money.of(100, "USD"));
        when(paymentGateway.charge(any(), any())).thenReturn(payment);

        // Act
        OrderId orderId = service.placeOrder(command);

        // Assert
        assertNotNull(orderId);
        var orderCaptor = ArgumentCaptor.forClass(Order.class);
        verify(orderRepository).save(orderCaptor.capture());
        assertEquals(OrderStatus.CONFIRMED, orderCaptor.getValue().getStatus());
    }

    @Test
    void placeOrder_whenPaymentFails_throwsPaymentFailedException() {
        var command = buildValidCommand();
        when(paymentGateway.charge(any(), any()))
            .thenReturn(PaymentResult.failed("CARD_DECLINED"));

        assertThrows(PaymentFailedException.class,
            () -> service.placeOrder(command));

        verify(orderRepository, never()).save(any());
    }

    @Test
    void placeOrder_whenNoItems_throwsDomainException() {
        var command = new PlaceOrderCommand(new CustomerId(1L), List.of());
        assertThrows(DomainException.class,
            () -> service.placeOrder(command));
    }
}
```

---

## 10. Production Concerns

### Scaling

- Framework-agnostic design không ảnh hưởng đến throughput hay latency theo cách trực tiếp. Nhưng nó cải thiện scalability của team: nhiều dev có thể làm việc độc lập trên domain, adapter, và application layer mà không conflict.
- Với microservices, mỗi service có bounded context riêng. Framework-agnostic giúp mỗi service có thể chọn framework khác nhau (Spring Boot, Micronaut, Quarkus) mà vẫn dùng chung các domain library qua Maven module.
- Không có overhead runtime đáng kể: interface dispatch trong JVM là virtual method call, O(1) và cực nhanh.

### Failure

- Lỗi hay gặp nhất là "port leakage": developer thêm `@Transactional` hoặc `@Component` vào class trong `application/` hoặc `domain/`, làm cho chúng phụ thuộc Spring. Giải pháp: dùng ArchUnit để enforce rằng `domain.*` và `application.*` không được import bất kỳ class nào từ `org.springframework.*`:
  ```java
  @AnalyzeClasses(packages = "com.example")
  class ArchitectureTest {
      @ArchTest
      ArchRule domainMustNotDependOnSpring =
          noClasses().that().resideInAPackage("..domain..")
                     .should().dependOnClassesThat()
                              .resideInAPackage("org.springframework..");
  }
  ```
- Mapper bao giờ cũng nằm trong adapter layer, không bao giờ trong domain. Nếu để mapper trong domain, domain sẽ biết về JPA entity — vi phạm nguyên tắc.

### Monitoring

- Các use case là điểm đo lường lý tưởng: bọc mỗi `UseCase.execute()` trong Micrometer `Timer` để track latency và lỗi theo từng business operation.
- Dùng Spring AOP để thêm metrics/logging ở adapter layer mà không modify domain code:
  ```java
  @Aspect
  @Component
  public class UseCaseMetricsAspect {
      private final MeterRegistry registry;
      @Around("@within(com.example.application.annotation.UseCase)")
      public Object measureUseCase(ProceedingJoinPoint pjp) throws Throwable {
          String name = pjp.getSignature().getDeclaringTypeName();
          return registry.timer("usecase." + name).recordCallable(pjp::proceed);
      }
  }
  ```

---

## 11. Common Mistakes

- Mistake: Đặt `@Transactional` trên domain service hoặc application service, buộc chúng phụ thuộc Spring.
  Fix: `@Transactional` thuộc về adapter layer. Tạo một `Transactional*` wrapper class trong adapter package implements cùng interface, ủy quyền cho application service thuần, và chỉ wrapper mới có `@Transactional`.

- Mistake: Domain entity extend JPA entity hoặc implement `Serializable` vì Spring data. Khi đó domain biết về persistence concern.
  Fix: Tạo hai loại object riêng biệt — `Order` (domain entity) và `OrderJpaEntity` (@Entity). Dùng `OrderMapper` trong adapter/persistence để chuyển đổi. Thêm ArchUnit test để bảo đảm domain chưa bao giờ import `javax.persistence` hoặc `jakarta.persistence`.

- Mistake: Trả về JPA entity trực tiếp từ repository adapter (`List<OrderJpaEntity>` thay vì `List<Order>`), để lộ persistence detail ra ngoài.
  Fix: Output port luôn trả về domain object. Mapper là trách nhiệm của adapter, không bao giờ expose ra ngoài adapter package.

- Mistake: Đặt logic nghiệp vụ phức tạp bên trong Spring `@Service` thay vì domain entity — "Anemic Domain Model".
  Fix: Domain entity nên chứa các methods thể hiện hành vi (`order.confirm()`, `order.cancel()`). `@Service` chỉ orchestrate — gọi port để load data, gọi domain method, gọi port để save. Kiểm tra: nếu có thể xóa domain entity và chỉ giữ @Service mà logic vẫn đầy đủ, đây là anemic domain.

---

## 12. Sample Project

**Dự án: Billing System với nhiều delivery mechanism**

Constraint cứng:
- Business logic "Generate Invoice" phải hoạt động y hệt thông qua cả ba entry point: REST API (`POST /invoices`), Kafka consumer (event `OrderCompleted`), và scheduled job (cuối tháng).
- `PlaceInvoiceUseCase` không được chứa bất kỳ annotation Spring nào (`@Service`, `@Transactional`, `@Autowired`, `@KafkaListener`, v.v.).
- Domain entity `Invoice` không được chứa `@Entity`, `@JsonIgnore`, hoặc bất kỳ annotation framework nào.
- Phải có unit test cho `InvoiceService` chạy không cần Spring context, test time < 50ms.
- Phải có ArchUnit test tự động fail build nếu ai đó thêm Spring annotation vào `domain.*` hoặc `application.*`.
- `@Transactional` chỉ được phép xuất hiện trong `adapter.*` package.

Các bước triển khai:
1. Định nghĩa `GenerateInvoiceUseCase` interface trong `domain/port/in/`.
2. Định nghĩa `InvoiceRepository`, `CustomerRepository`, `EmailGateway` interfaces trong `domain/port/out/`.
3. Implement `InvoiceService` trong `application/usecase/` — pure Java, no annotation.
4. Tạo ba adapter in: `InvoiceController` (@RestController), `OrderCompletedKafkaListener` (@KafkaListener), `MonthlyInvoiceScheduler` (@Scheduled).
5. Tạo adapter out: `JpaInvoiceRepository`, `JpaCustomerRepository`, `SmtpEmailGateway`.
6. Cấu hình Spring bean trong `@Configuration` class trong `adapter/config/`.
7. Viết ArchUnit test trong `src/test/java/` để enforce architecture rules.

---

## 13. Interview

### Core Q&A

**Q: Framework-agnostic design là gì và lợi ích chính là gì?**
A: Là cách tổ chức code sao cho business logic nằm trong các class Java thuần, không import framework API. Lợi ích chính: unit test business logic cực nhanh (không cần Spring context), dễ swap framework, các rule nghiệp vụ tập trung một chỗ để tìm và sửa, nhiều developer làm việc song song mà ít conflict.

**Q: Port là gì, Adapter là gì trong Hexagonal Architecture?**
A: Port là interface Java được domain định nghĩa — thể hiện nhu cầu của domain (CanSaveOrder, CanSendEmail). Adapter là class có Spring/JPA/HTTP annotation implement interface đó. Domain biết về Port, không biết về Adapter. Đây là Dependency Inversion Principle: high-level policy (domain) không phụ thuộc low-level detail (framework).

**Q: Domain entity và JPA entity khác nhau thế nào, và tại sao nên tách ra?**
A: Domain entity là class thể hiện khái niệm nghiệp vụ và chứa business rules — nó không biết về database. JPA entity là class ánh xạ vào bảng database, có `@Entity`, `@Column`, `@ManyToOne`. Tách ra vì: (1) domain entity có thể có final fields, immutable state, business methods — JPA entity cần no-arg constructor và setters. (2) Schema database có thể khác model domain (de-normalized, extra columns). (3) Domain không nên biết về persistence concern.

**Q: @Transactional nên đặt ở đâu trong clean architecture?**
A: `@Transactional` thuộc adapter layer, không phải domain hay application. Lý do: `@Transactional` là Spring annotation — đặt nó vào domain/application làm chúng phụ thuộc Spring. Cách tốt nhất: tạo `Transactional*` wrapper class trong adapter/config package, nó implements cùng use case interface và ủy quyền cho application service thuần, chỉ add `@Transactional`.

**Q: Framework-agnostic có ảnh hưởng đến performance không?**
A: Không đáng kể. Thêm một lớp interface dispatch là virtual method call, O(1) và được JVM JIT optimize. Overhead thực sự của approach này là dev time cho việc viết thêm mapper code và interface, không phải runtime latency.

**Q: Làm sao enforce framework-agnostic boundary tự động trong CI/CD?**
A: Dùng ArchUnit — thư viện Java cho phép viết architecture rules như unit test. Ví dụ:
```java
noClasses()
    .that().resideInAPackage("..domain..")
    .should().dependOnClassesThat()
             .resideInAPackage("org.springframework..")
    .check(importedClasses);
```
Test này fail build ngay khi ai đó vi phạm quy tắc.

**Q: Anemic Domain Model là gì và nó trái ngược với framework-agnostic thế nào?**
A: Anemic Domain Model (Martin Fowler) là khi entity chỉ là container data (getters/setters) và toàn bộ logic nằm trong @Service. Đây là anti-pattern vì domain không có behavior, logic bị scattered ở @Service layer, không encapsulated. Framework-agnostic design khuyến khích Rich Domain Model — entity chứa behavior (`order.confirm()`, `invoice.void()`), service chỉ orchestrate.

**Q: Spring Data JPA Specification có vi phạm framework-agnostic principle không?**
A: Có một phần. `Specification<T>` phụ thuộc vào `CriteriaBuilder` là JPA API. Nếu đặt Specification trong domain layer, domain biết về JPA. Giải pháp: Specification thuộc về adapter/persistence layer; domain định nghĩa search criteria bằng POJO thuần (`OrderSearchCriteria`); adapter/persistence chuyển `OrderSearchCriteria` sang `Specification`.

### Scenario

**Scenario 1**: Bạn nhập dự án, thấy business logic nằm trong `@Service` có 500 dòng, full annotation Spring. Làm sao cải thiện dần dần mà không break production?

Trả lời: Dùng Strangler Fig pattern — refactor từng bước nhỏ, không rewrite toàn bộ một lúc. Bước 1: extract domain logic ra thành pure Java class (chưa test, chưa gọi đến repository). Bước 2: viết unit test cho class mới. Bước 3: thay thế logic trong @Service bằng gọi đến class mới. Bước 4: tách interface (Port) ra và di chuyển class vào đúng package. Mỗi bước kèm theo test để đảm bảo behavior không thay đổi.

**Scenario 2**: Sau khi áp dụng framework-agnostic, test suite chậm vì phải load Spring context cho integration test. Làm sao cải thiện?

Trả lời: Tách thành hai loại test: (1) Unit test cho domain và application layer — chạy không cần Spring, dùng Mockito mock các Port interface, chạy rất nhanh (< 100ms toàn bộ). (2) Integration test cho adapter layer (Spring slice: `@DataJpaTest`, `@WebMvcTest`) — chỉ test adapter, không test business logic. Tỷ lệ hướng đến: 70% unit test, 20% integration test, 10% E2E test. Với cách này, đa số test chạy nhanh.

**Scenario 3**: Team muốn thêm Kafka consumer cho cùng business logic "Place Order" hiện đang chỉ có REST adapter. Viết thêm bao nhiêu code?

Trả lời: Rất ít — chỉ thêm một adapter class. `PlaceOrderUseCase` interface và `PlaceOrderService` không thay đổi gì. Chỉ cần tạo `OrderEventKafkaListener` trong `adapter/in/messaging/`:
```java
@KafkaListener(topics = "order-requests")
public class OrderEventKafkaListener {
    private final PlaceOrderUseCase useCase;

    public void handleOrderRequest(OrderRequestEvent event) {
        PlaceOrderCommand command = map(event);
        useCase.placeOrder(command);
    }
}
```
Đây là lợi ích lớn nhất của framework-agnostic: thêm delivery mechanism mới không ảnh hưởng đến domain hay application layer.

**Scenario 4**: Đồng nghiệp nói "chúng ta đang viết e-commerce CRUD app, áp dụng Hexagonal Architecture là over-engineering". Bạn phản hồi thế nào?

Trả lời: Đồng nghiệp có một phần đúng. Với app CRUD đơn giản, full Hexagonal Architecture với domain entity riêng, JPA entity riêng, mapper riêng, port riêng có thể quá nhiều. Nhưng có một số nuance: (1) App CRUD thường phát triển thêm business logic theo thời gian — refactor sau khó hơn lúc đầu. (2) Có thể dùng phiên bản nhẹ hơn: vẫn dùng @Service / @Repository nhưng tách business rule ra khỏi controller và tránh logic trong @Entity JPA. (3) Rồi swap sang đầy đủ port & adapter khi complexity thực sự cần. Lựa chọn approach phù hợp với trạng thái hiện tại và dự kiến tương lai của project.

**Scenario 5**: Domain entity `Order` cần biết IP address của request (vì audit log). Làm sao lấy thông tin này mà không đưa `HttpServletRequest` vào domain?

Trả lời: Domain không được biết về HTTP. Giải pháp: adapter layer (Controller) lấy IP từ `HttpServletRequest`, thêm vào Command object:
```java
// Command chua du information adapter thu thap
record PlaceOrderCommand(
    CustomerId customerId,
    List<OrderItemRequest> items,
    String requestIpAddress   // adapter thêm trước khi gọi use case
) {}

// Controller:
String ip = request.getRemoteAddr();
PlaceOrderCommand command = new PlaceOrderCommand(customerId, items, ip);
useCase.placeOrder(command);
```
Domain nhận IP như một giá trị thuần túy, không biết nó đến từ HTTP.

---

## 14. References

- Alistair Cockburn — Hexagonal Architecture (bài báo gốc): https://alistair.cockburn.us/hexagonal-architecture/
- Robert C. Martin — Clean Architecture (sách): https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/
- Martin Fowler — Anemic Domain Model: https://martinfowler.com/bliki/AnemicDomainModel.html
- Martin Fowler — DomainModel: https://martinfowler.com/eaaCatalog/domainModel.html
- ArchUnit documentation — architecture testing in Java: https://www.archunit.org/userguide/html/000_Index.html
- Baeldung — Hexagonal Architecture with Spring Boot: https://www.baeldung.com/hexagonal-architecture-ddd-spring
- Tom Hombergs — Get Your Hands Dirty on Clean Architecture (sách, rất thực tế): https://leanpub.com/get-your-hands-dirty-on-clean-architecture
- Vlad Mihalcea — Why you should not use Spring @Transactional on domain entities: https://vladmihalcea.com/spring-transactional-annotation/

---

## 15. Real-world Code

- Buckpal — reference implementation cho cuốn sách "Get Your Hands Dirty on Clean Architecture" của Tom Hombergs: https://github.com/thombergs/buckpal
- ArchUnit examples — ví dụ enforce architecture rules trong Java: https://github.com/TNG/ArchUnit-Examples
- Spring Modulith — Spring's own approach to modular, framework-aware but loosely coupled architecture: https://github.com/spring-projects/spring-modulith
- EventStorming + Hexagonal Architecture example: https://github.com/ddd-by-examples/library
- Netflix Conductor — workflow engine với clean separation của domain và infrastructure: https://github.com/Netflix/conductor

---

## 16. Community

- Stack Overflow — "Where should @Transactional be placed in Clean Architecture?": https://stackoverflow.com/questions/54652836/where-should-transactional-be-in-clean-architecture
- Stack Overflow — "Hexagonal architecture: should domain entities know about JPA?": https://stackoverflow.com/questions/34941823/ddd-should-i-use-jpa-annotations-in-domain-model
- Reddit r/java — "Is Clean Architecture / Hexagonal Architecture worth it for Spring Boot apps?": https://www.reddit.com/r/java/comments/clean_architecture_spring_boot
- Reddit r/softwarearchitecture — ongoing discussions on ports & adapters: https://www.reddit.com/r/softwarearchitecture/
- Tom Hombergs blog — "Hexagonal Architecture with Java and Spring": https://reflectoring.io/spring-hexagonal/
- Philipp Hauer blog — "Clean Architecture with Spring Boot": https://phauer.com/2020/spring-boot-clean-architecture/
- Baeldung — "Intro to the ArchUnit library": https://www.baeldung.com/java-archunit-intro
