---
created: 2026-04-17
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/async"
related:
  - "[[Spring HTTP Core Classes]]"
---

# EventListener in Spring Boot

## 1. What

Spring Event System là cơ chế giao tiếp nội bộ giữa các component trong cùng một application context, hoạt động theo mô hình publish-subscribe. Thay vì gọi trực tiếp method của một bean khác, bạn publish một event và để Spring định tuyến đến tất cả listener đã đăng ký. Hệ thống này được xây dựng trên nền `ApplicationEventPublisher` và hỗ trợ cả đồng bộ lẫn bất đồng bộ.

## 2. Why

Trước khi có event system, khi một service cần thông báo cho nhiều service khác (ví dụ: sau khi đăng ký user thành công cần gửi email, ghi audit log, cấp điểm thưởng), bạn phải inject tất cả các service đó vào `UserService` và gọi chúng theo thứ tự. Điều này dẫn đến:

- `UserService` bị phụ thuộc vào `EmailService`, `AuditService`, `RewardService` — vi phạm Single Responsibility Principle.
- Mỗi khi thêm một hành động mới, bạn phải sửa `UserService` — vi phạm Open/Closed Principle.
- Unit test trở nên phức tạp vì phải mock nhiều dependency.
- Circular dependency dễ xảy ra khi các service bắt đầu phụ thuộc lẫn nhau.

Event system giải quyết bằng cách đảo chiều phụ thuộc: `UserService` chỉ cần biết về event, không cần biết ai sẽ xử lý nó.

## 3. Mental Model

Hãy nghĩ đến hệ thống loa phát thanh trong một tòa nhà văn phòng.

- **Publisher** là người cầm micro và đọc thông báo: "Có người giao hàng ở sảnh lễ tân."
- **Event** là nội dung thông báo đó — một gói thông tin được truyền đi.
- **ApplicationContext** là hệ thống loa — nó phát thông báo đến tất cả các phòng.
- **Listener** là những người ngồi trong các phòng lắng nghe và quyết định có cần hành động không.

Người đọc thông báo (publisher) không biết và không quan tâm ai sẽ nghe. Phòng kế toán nghe xong bỏ qua, phòng IT nghe xong chạy xuống nhận máy tính, phòng lễ tân nghe xong ghi vào sổ. Mỗi bên độc lập xử lý theo logic riêng.

Điểm quan trọng: mặc định hệ thống loa này là **đồng bộ** — người đọc thông báo phải chờ tất cả mọi người phản hồi xong mới tiếp tục làm việc. Để làm bất đồng bộ, cần dùng `@Async`.

## 4. Where It Fits

```
HTTP Request
    |
    v
Controller
    |
    v
UserService.registerUser()
    |
    +-- publish(UserRegisteredEvent)
                |
                v
        ApplicationEventMulticaster (Spring internals)
                |
          +-----+-----+-----+
          |           |     |
          v           v     v
   EmailListener  AuditListener  RewardListener
```

Event system nằm ở tầng Service, không phải Controller hay Repository. Nó là công cụ để tách logic nghiệp vụ phụ trợ (side effects) ra khỏi luồng nghiệp vụ chính.

## 5. When to Use

- Sau khi hoàn thành một hành động chính, cần trigger nhiều hành động phụ không liên quan trực tiếp đến kết quả trả về.
- Khi các hành động phụ có thể thay đổi hoặc mở rộng trong tương lai mà không muốn sửa code của hành động chính.
- Audit logging: ghi lại mọi hành động của user mà không làm ô nhiễm business logic.
- Gửi notification (email, push notification, SMS) sau khi có sự kiện quan trọng.
- Cache invalidation: khi entity thay đổi, publish event để các cache liên quan tự làm mới.
- Khi cần đảm bảo một hành động chỉ xảy ra sau khi transaction commit thành công (`@TransactionalEventListener`).

## 6. When NOT to Use

- Khi bạn cần kết quả trả về từ listener để tiếp tục xử lý — event là fire-and-forget, không trả về giá trị.
- Khi thứ tự xử lý giữa các listener là quan trọng và nghiêm ngặt — tuy có `@Order` nhưng phụ thuộc vào thứ tự là dấu hiệu thiết kế không tốt.
- Khi communication xảy ra giữa các microservice khác nhau — dùng message broker (Kafka, RabbitMQ) thay vì Spring event.
- Khi logic quá đơn giản và chỉ có một listener duy nhất — gọi trực tiếp dễ đọc hơn.
- Khi dùng với `@Async` mà không hiểu rõ: nếu listener throw exception, nó sẽ bị nuốt nếu không có `AsyncUncaughtExceptionHandler`.

## 7. Trade-offs

| Pros | Cons |
|---|---|
| Loose coupling giữa các component | Luồng code khó trace hơn — phải tìm kiếm listener |
| Dễ mở rộng không cần sửa publisher | Debug khó hơn, stack trace bị gián đoạn với @Async |
| Single Responsibility rõ ràng hơn | Mặc định đồng bộ, có thể làm chậm request nếu listener nặng |
| Dễ test từng listener độc lập | Không có return value từ listener |
| Transaction-aware với @TransactionalEventListener | Phải cấu hình thêm để dùng @Async đúng cách |

## 8. Alternatives

| Approach | Phù hợp khi | Hạn chế |
|---|---|---|
| Direct method call | Logic đơn giản, 1-2 side effects | Tight coupling, khó mở rộng |
| Spring Event (@EventListener) | Side effects trong cùng JVM | Chỉ trong application context |
| Spring Integration | Cần pipeline xử lý phức tạp | Nặng hơn, learning curve cao |
| Kafka / RabbitMQ | Cross-service communication | Infrastructure overhead |
| @Scheduled polling | Periodic batch processing | Không phải event-driven |
| Spring WebFlux reactive streams | Reactive entire stack | Phải chuyển toàn bộ sang reactive |

## 9. How

### Bước 1: Tạo Custom Event

```java
// UserRegisteredEvent.java
public class UserRegisteredEvent {

    private final String userId;
    private final String email;
    private final Instant occurredAt;

    public UserRegisteredEvent(String userId, String email) {
        this.userId = userId;
        this.email = email;
        this.occurredAt = Instant.now();
    }

    public String getUserId() { return userId; }
    public String getEmail() { return email; }
    public Instant getOccurredAt() { return occurredAt; }
}
```

Từ Spring 4.2 trở đi, không cần extend `ApplicationEvent`. Class POJO thuần túy là đủ.

### Bước 2: Publish Event

```java
// UserService.java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public User registerUser(RegisterUserRequest request) {
        User user = User.builder()
            .id(UUID.randomUUID().toString())
            .email(request.getEmail())
            .passwordHash(hashPassword(request.getPassword()))
            .build();

        userRepository.save(user);

        // Publish event sau khi save thành công trong transaction
        eventPublisher.publishEvent(new UserRegisteredEvent(user.getId(), user.getEmail()));

        return user;
    }
}
```

### Bước 3: Tạo Listener đồng bộ

```java
// AuditListener.java
@Component
@Slf4j
public class AuditListener {

    private final AuditLogRepository auditLogRepository;

    public AuditListener(AuditLogRepository auditLogRepository) {
        this.auditLogRepository = auditLogRepository;
    }

    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        log.info("Audit: user {} registered at {}", event.getUserId(), event.getOccurredAt());
        auditLogRepository.save(new AuditLog(
            "USER_REGISTERED",
            event.getUserId(),
            event.getOccurredAt()
        ));
    }
}
```

### Bước 4: Listener bất đồng bộ với @Async

```java
// EmailNotificationListener.java
@Component
@Slf4j
public class EmailNotificationListener {

    private final EmailService emailService;

    public EmailNotificationListener(EmailService emailService) {
        this.emailService = emailService;
    }

    @Async
    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        log.info("Sending welcome email to {}", event.getEmail());
        emailService.sendWelcomeEmail(event.getEmail());
    }
}
```

Để `@Async` hoạt động, phải thêm `@EnableAsync` vào một `@Configuration` class:

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("event-listener-");
        executor.initialize();
        return executor;
    }
}
```

### Bước 5: @TransactionalEventListener — chỉ chạy sau khi transaction commit

```java
// RewardListener.java
@Component
@Slf4j
public class RewardListener {

    private final RewardService rewardService;

    public RewardListener(RewardService rewardService) {
        this.rewardService = rewardService;
    }

    // Chỉ chạy sau khi transaction của registerUser() COMMIT thành công
    // Nếu transaction rollback, listener này KHÔNG chạy
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleUserRegistered(UserRegisteredEvent event) {
        rewardService.grantSignupBonus(event.getUserId());
    }
}
```

### Bước 6: Conditional Listener với SpEL

```java
@EventListener(condition = "#event.email.endsWith('@vip.com')")
public void handleVipUserRegistered(UserRegisteredEvent event) {
    // Chỉ chạy nếu email kết thúc bằng @vip.com
    vipOnboardingService.startVipOnboarding(event.getUserId());
}
```

### Bước 7: Xử lý exception trong @Async listener

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (throwable, method, objects) ->
            log.error("Async event listener error in method: {}", method.getName(), throwable);
    }
}
```

## 10. Production Concerns

### Scaling

- Mặc định `ApplicationEventMulticaster` là `SimpleApplicationEventMulticaster` — đồng bộ, single-threaded.
- Với `@Async`, cần cấu hình `ThreadPoolTaskExecutor` phù hợp với load thực tế.
- Nếu listener xử lý lâu (gửi email, gọi external API), phải dùng `@Async` để không block request thread.
- Event system trong Spring chỉ hoạt động trong **một JVM instance** — không scale ngang qua cluster.

### Failure

- Với listener đồng bộ: nếu một listener throw exception, toàn bộ chuỗi listener bị dừng và exception propagate ngược về publisher. Transaction có thể bị rollback.
- Với listener `@Async`: exception bị nuốt hoàn toàn nếu không cấu hình `AsyncUncaughtExceptionHandler`.
- Với `@TransactionalEventListener`: nếu database không có transaction active, listener sẽ bị bỏ qua theo mặc định. Cần set `fallbackExecution = true` nếu muốn chạy ngay cả khi không có transaction.

### Monitoring

- Dùng Spring Boot Actuator để expose metrics.
- Log event type và processing time trong mỗi listener.
- Với `@Async`, monitor thread pool usage qua Actuator endpoint `/actuator/metrics/executor.pool.size`.
- Xem xét thêm tracing (Micrometer + Zipkin) để trace event qua các listener.

## 11. Common Mistakes

- Mistake: Publish event bên trong một method KHÔNG có `@Transactional` nhưng dùng `@TransactionalEventListener`, dẫn đến listener không bao giờ được gọi.
  Fix: Đảm bảo method publish event được bao bọc trong `@Transactional`, hoặc set `fallbackExecution = true` trên listener nếu muốn chạy khi không có transaction.

- Mistake: Dùng `@Async` trên listener nhưng quên thêm `@EnableAsync`, khiến `@Async` bị bỏ qua hoàn toàn và listener chạy đồng bộ mà không có warning.
  Fix: Luôn thêm `@EnableAsync` vào một `@Configuration` class. Viết integration test kiểm tra hành vi async.

- Mistake: Mutate object event sau khi publish — vì listener đồng bộ chạy trong cùng thread, nếu bạn thay đổi state của event object, các listener sau có thể thấy dữ liệu đã bị thay đổi.
  Fix: Thiết kế event class là immutable — chỉ có final fields và getter, không có setter.

- Mistake: Inject quá nhiều dependency vào một listener class, biến nó thành "god listener" xử lý mọi thứ.
  Fix: Mỗi listener class chỉ nên xử lý một loại concern — email listener, audit listener, reward listener riêng biệt.

## 12. Sample Project

Xây dựng hệ thống Order Management với các ràng buộc cứng sau:

**Constraint:** Khi một order được đặt thành công, hệ thống PHẢI:
1. Gửi email xác nhận cho customer (async, không block response).
2. Ghi audit log vào database (đồng bộ, trong cùng transaction).
3. Cập nhật inventory (chỉ chạy SAU KHI transaction commit, dùng `@TransactionalEventListener`).
4. Nếu customer là VIP (spent > $1000), trigger thêm VIP notification (conditional với SpEL).

**Hard Constraint:** `OrderService` không được phép import bất kỳ class nào của `EmailService`, `AuditService`, `InventoryService`, hoặc `VipService`. Tất cả coupling phải đi qua event.

**Structure gợi ý:**
```
events/
    OrderPlacedEvent.java
services/
    OrderService.java           // chỉ publish event
listeners/
    OrderEmailListener.java     // @Async
    OrderAuditListener.java     // @EventListener
    InventoryUpdateListener.java // @TransactionalEventListener
    VipNotificationListener.java // @EventListener + condition
```

## 13. Interview

### Core Q&A

**Q: Spring Event System mặc định là synchronous hay asynchronous?**
A: Mặc định là **synchronous**. Publisher publish event và chờ tất cả listener hoàn thành xử lý trong cùng một thread trước khi tiếp tục. Để chuyển sang async, cần dùng `@Async` trên listener method và `@EnableAsync` trên configuration class.

**Q: Sự khác biệt giữa `@EventListener` và `@TransactionalEventListener` là gì?**
A: `@EventListener` xử lý event ngay khi được publish. `@TransactionalEventListener` bind việc xử lý vào một transaction phase cụ thể — mặc định là `AFTER_COMMIT`. Điều này đảm bảo listener chỉ chạy khi transaction đã commit thành công, tránh trường hợp side effects (gửi email, cập nhật inventory) xảy ra nhưng transaction chính lại rollback sau đó.

**Q: Nếu một @Async listener throw exception, điều gì xảy ra?**
A: Exception bị nuốt hoàn toàn và không propagate về publisher. Để xử lý, cần implement `AsyncUncaughtExceptionHandler` trong `AsyncConfigurer` và đăng ký với Spring.

**Q: Có thể có nhiều listener cùng lắng nghe một event không? Thứ tự chạy được xác định thế nào?**
A: Có. Mặc định thứ tự không được đảm bảo. Dùng `@Order(n)` để kiểm soát thứ tự — giá trị nhỏ hơn chạy trước. Tuy nhiên, phụ thuộc vào thứ tự giữa các listener thường là dấu hiệu của thiết kế cần được xem xét lại.

**Q: Custom event có cần extend ApplicationEvent không?**
A: Không, từ Spring 4.2 trở đi. Bất kỳ POJO nào cũng có thể là event. Extend `ApplicationEvent` chỉ cần thiết nếu bạn cần truy cập `source` object hoặc cần tương thích với code cũ.

**Q: @TransactionalEventListener có hoạt động khi không có transaction active không?**
A: Mặc định không — event sẽ bị bỏ qua. Cần set `fallbackExecution = true` để listener chạy ngay cả khi không có transaction.

### Scenario

**Scenario 1:** Bạn có `UserService.registerUser()` annotated với `@Transactional`. Bên trong, bạn save user và publish `UserRegisteredEvent`. Bạn có một listener dùng `@TransactionalEventListener`. Trong test, bạn thấy listener không bao giờ được gọi. Tại sao?

Trả lời: Có thể đang dùng `@DataJpaTest` hoặc một test configuration không wrap method trong real transaction, hoặc method được gọi trực tiếp mà không qua Spring proxy (self-invocation). Cần đảm bảo transaction thực sự active và commit. Dùng `@SpringBootTest` với transaction thật để kiểm tra.

**Scenario 2:** Hệ thống đang chạy trên 3 server instances (horizontal scaling). Bạn publish một event trên server 1. Listener trên server 2 và 3 có nhận được event không?

Trả lời: Không. Spring Application Events chỉ hoạt động trong cùng một JVM / application context. Để broadcast event qua nhiều instances, cần dùng message broker như Kafka hoặc RabbitMQ với Spring Cloud Bus hoặc Spring Integration.

**Scenario 3:** Bạn cần gửi email sau khi user đăng ký. EmailService gọi SMTP server và có thể mất 2-3 giây. Làm thế nào để không làm chậm API response?

Trả lời: Annotate listener method với `@Async` và đảm bảo `@EnableAsync` được thêm vào configuration. Cấu hình một `ThreadPoolTaskExecutor` riêng với thread pool size phù hợp. Thêm `AsyncUncaughtExceptionHandler` để log lỗi khi gửi email thất bại mà không làm crash thread pool.

**Scenario 4:** Sau khi ghi audit log trong listener, bạn nhận ra nếu business transaction rollback thì audit log vẫn được ghi — tạo ra audit record "phantom". Cách fix?

Trả lời: Có hai hướng. Thứ nhất, dùng `@TransactionalEventListener(phase = AFTER_COMMIT)` để đảm bảo listener chỉ chạy sau khi transaction commit thành công. Thứ hai, nếu cần audit ngay cả khi rollback, lưu audit log vào một database connection khác không tham gia vào main transaction (dùng `REQUIRES_NEW` propagation hoặc một `DataSource` riêng).

**Scenario 5:** Bạn có 5 listener cho cùng một event, một listener trong số đó làm fail (throw exception). Các listener còn lại có tiếp tục chạy không?

Trả lời: Với listener đồng bộ mặc định — không. Exception từ listener đầu tiên propagate lên và các listener sau bị skip. Với `@Async` listener — mỗi listener chạy trong thread riêng nên độc lập với nhau. Để giải quyết cho trường hợp đồng bộ, có thể wrap logic trong listener vào try-catch hoặc customize `SimpleApplicationEventMulticaster` để ignore exception.

## 14. References

- Spring Framework Docs - Application Events and Listeners: https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-events
- Spring Framework Docs - @EventListener: https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-events-annotation
- Spring Framework Docs - @TransactionalEventListener: https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html
- Spring Framework Docs - @Async: https://docs.spring.io/spring-framework/reference/integration/scheduling.html#scheduling-annotation-support-async
- Spring Boot Reference - Auto-configured Task Execution: https://docs.spring.io/spring-boot/reference/features/task-execution-and-scheduling.html
- JavaDoc - ApplicationEventPublisher: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/ApplicationEventPublisher.html
- JavaDoc - TransactionalEventListener: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionalEventListener.html

## 15. Real-world Code

- Spring PetClinic (event usage example): https://github.com/spring-projects/spring-petclinic
- Spring Data Commons uses @TransactionalEventListener internally: https://github.com/spring-projects/spring-data-commons/search?q=TransactionalEventListener
- Baeldung example repo - spring-boot-basic-customization: https://github.com/eugenp/tutorials/tree/master/spring-boot-modules/spring-boot-basic-customization
- Spring Integration examples (event-driven patterns): https://github.com/spring-projects/spring-integration-samples

## 16. Community

- Stack Overflow - "@TransactionalEventListener not called": https://stackoverflow.com/questions/45327839/transactionaleventlistener-not-triggered
- Stack Overflow - "Spring Events async vs sync": https://stackoverflow.com/questions/30431776/using-async-event-listener-in-spring-4-2
- Baeldung - Spring Events guide: https://www.baeldung.com/spring-events
- Baeldung - @TransactionalEventListener deep dive: https://www.baeldung.com/spring-transactional-event-listener
- Stack Overflow - "Exception in @Async @EventListener being swallowed": https://stackoverflow.com/questions/35459599/spring-async-eventlistener-exception-handling
