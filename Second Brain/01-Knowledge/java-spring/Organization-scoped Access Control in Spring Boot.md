---
created: 2026-04-20
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/security"
related:
  - "[[Spring Security]]"
  - "[[ConfigurationProperties in Spring Boot]]"
  - "[[Validated in Spring Boot]]"
---

# Organization-scoped Access Control in Spring Boot

## 1. What

Organization-scoped access control (hay multi-tenant row-level isolation) là pattern đảm bảo mỗi user chỉ có thể đọc/ghi dữ liệu thuộc **tổ chức (organization) mà họ là thành viên** — không phải dữ liệu của tổ chức khác. Trong Spring Boot, pattern này được thực thi ở **service layer** bằng cách tự động gán `organizationId` hoặc `userId` vào mọi query, thay vì để client tự khai báo scope trong request. Hệ thống lấy scope từ `UserPrincipal` (authenticated context), resolve sang điều kiện lọc tương ứng, rồi kiểm tra hợp lệ trước khi thực hiện bất kỳ truy vấn nào.

---

## 2. Why

Trong các hệ thống SaaS nhiều khách hàng (multi-tenant), dữ liệu của các tổ chức khác nhau nằm chung trên một database. Nếu không có cơ chế isolation ở tầng ứng dụng:

- Một user có thể chỉnh sửa `organizationId` trong request body/query param để lấy dữ liệu của org khác — **Insecure Direct Object Reference (IDOR)**.
- Mỗi developer viết thêm query phải tự nhớ thêm điều kiện `WHERE organization_id = ?` — dễ quên, khó audit, khó enforce qua code review.
- Khó mở rộng: khi thêm role mới (super-admin, auditor), phải sửa nhiều chỗ trong codebase thay vì sửa một chỗ duy nhất.
- Không có validation tập trung: một số endpoint có thể thiếu `organizationId`, một số khác dùng `userId`, gây ra hành vi không nhất quán.

Pattern này giải quyết bằng cách **tập trung logic scope vào service layer**: mọi request đi qua một bước `applyScope` (lấy scope từ token) và `validateScope` (kiểm tra scope hợp lệ) trước khi gọi repository.

---

## 3. Mental Model

Hãy hình dung tòa nhà văn phòng nhiều công ty thuê chung (co-working space):

- Mỗi phòng (organization) có khóa riêng.
- Khi bạn vào tòa nhà, bảo vệ (Spring Security) kiểm tra thẻ (JWT token) để biết bạn là ai và bạn thuộc phòng nào.
- Mỗi khi bạn mở cửa phòng nào, hệ thống không hỏi "bạn muốn vào phòng nào?" mà tự quét thẻ của bạn và chỉ mở đúng phòng của bạn.
- Manager của một công ty có thể vào mọi phòng của công ty đó, nhưng không thể vào phòng của công ty khác.
- Nếu bạn cố gắng khai báo "tôi thuộc phòng 5" trong khi thẻ của bạn ghi phòng 3, bảo vệ từ chối ngay — scope đến từ thẻ, không đến từ lời nói của bạn.

Tương tự: `organizationId` đến từ `UserPrincipal` trong JWT, không bao giờ đến từ request body hay query param do client gửi lên.

---

## 4. Where it fits

```
HTTP Request
    |
    v
[Spring Security Filter]     <-- xác thực JWT, tạo UserPrincipal
    |
    v
[Controller]                 <-- nhận request DTO, gọi service
    |
    v
[Service - applyScope()]     <-- lấy organizationId/userId từ UserPrincipal
    |
    v
[Service - validateScope()]  <-- đảm bảo scope không rỗng
    |
    v
[BaseQueryRequest.build()]   <-- tạo Specification với scope predicate
    |
    v
[Repository - findAll(spec, pageable)]  <-- chỉ trả về dữ liệu của org/user đó
    |
    v
[Controller]                 <-- trả về PageResponse
```

Tầng service là nơi duy nhất biết về `UserPrincipal` và role. Tầng repository không biết về authentication — nó chỉ nhận `Specification` đã được đóng gói sẵn.

---

## 5. When to use

- Hệ thống SaaS có nhiều khách hàng (tenant) dùng chung một database.
- API trả về danh sách entity có thể thuộc nhiều tổ chức khác nhau (listing, order, employee, invoice...).
- Role có sự phân biệt: manager thấy tất cả dữ liệu của org, employee chỉ thấy dữ liệu của chính mình.
- Cần audit trail rõ ràng: ai query cái gì, scope nào được áp dụng.
- Cần enforce isolation mà không phụ thuộc vào developer tự nhớ thêm điều kiện WHERE.

---

## 6. When NOT to use

- Ứng dụng đơn khách hàng (single-tenant): không có khái niệm "org khác" — thêm scope chỉ gây phức tạp thừa.
- Dữ liệu hoàn toàn public, không cần phân quyền (ví dụ: catalog sản phẩm công khai).
- Hệ thống dùng Row-Level Security (RLS) trực tiếp ở database (PostgreSQL RLS, CockroachDB): tầng DB đã xử lý isolation, tầng application không cần lặp lại.
- Microservice có dedicated database riêng cho từng tenant (schema-per-tenant hoặc database-per-tenant): isolation ở cấp infra, không cần scope predicate.

Lưu ý: nếu dùng RLS ở DB kết hợp với pattern này thì thành duplicate logic — chọn một trong hai, không nên dùng cả hai.

---

## 7. Trade-offs

| Khía cạnh | Ưu điểm | Nhược điểm |
|-----------|---------|-----------|
| Bảo mật | Scope đến từ token, không thể bị client ghi đè | Nếu JWT bị đánh cắp, attacker có scope đó |
| Nhất quán | Mọi query đều có scope predicate, enforce qua code | Phải đảm bảo mọi service method gọi applyScope trước build() |
| Khả năng bảo trì | Logic scope tập trung một chỗ, dễ sửa | Thêm bước bắt buộc trong workflow — dễ quên nếu không có code review |
| Hiệu năng | Index trên `organization_id` làm query nhanh | Mỗi query thêm một điều kiện JOIN/WHERE — nhỏ nhưng có thực |
| Test | Dễ mock `UserPrincipal` với org cụ thể | Phải test cả trường hợp scope sai (no org, no user) |

---

## 8. Alternatives

| Phương án | Khi nào dùng | Nhược điểm so với pattern này |
|-----------|--------------|-------------------------------|
| Row-Level Security (PostgreSQL RLS) | Khi cần isolation ở tầng DB, không tin tầng app | Phức tạp hơn để debug; Spring Data không native support |
| Schema-per-tenant | Isolation mạnh, mỗi tenant có schema riêng | Chi phí infra cao; migration phức tạp |
| Database-per-tenant | Isolation mạnh nhất | Rất đắt; khó scale số lượng tenant lớn |
| @PreAuthorize + SpEL | Bảo vệ từng method bằng annotation | Khó enforce scope động (orgId lấy từ token); verbose khi nhiều method |
| Filter trong Controller | Đơn giản, rõ ràng | Logic lặp lại mỗi controller; dễ bỏ qua; khó test tập trung |

---

## 9. How

### 9.1 UserPrincipal

```java
@Getter
@AllArgsConstructor
public class UserPrincipal implements UserDetails {
    private Long id;
    private String organizationId;
    private String role; // "MANAGER" hoặc "EMPLOYEE"

    public boolean isManager() {
        return "MANAGER".equals(this.role);
    }
}
```

### 9.2 BaseQueryRequest

```java
@Getter
@Setter
public abstract class BaseQueryRequest {

    private int page = 0;
    private int pageSize = 20;
    private String organizationId;
    private String userId;

    public abstract Specification<?> build();
    public abstract String getSort();
    public abstract String getOrder();

    public final void validateScope() {
        boolean hasOrg  = organizationId != null && !organizationId.isBlank();
        boolean hasUser = userId != null && userId.matches("\\d+") && Long.parseLong(userId) > 0;
        if (!hasOrg && !hasUser) {
            throw new IllegalArgumentException("Either organizationId or userId is required");
        }
    }

    protected final Predicate buildScopePredicate(Root<?> root, CriteriaBuilder cb) {
        if (organizationId != null && !organizationId.isBlank()) {
            return cb.equal(root.get("organizationId"), organizationId);
        }
        return cb.equal(root.get("user").get("id"), Long.parseLong(userId));
    }

    public Pageable getPageable() {
        Sort sort = Sort.by("ASC".equalsIgnoreCase(getOrder())
            ? Sort.Order.asc(getSort())
            : Sort.Order.desc(getSort()));
        return PageRequest.of(page, pageSize, sort);
    }
}
```

### 9.3 Concrete QueryRequest

```java
@Getter
@Setter
public class ListingQueryRequest extends BaseQueryRequest {

    private String query;
    private List<String> status;

    @Override
    public String getSort() { return "createdAt"; }

    @Override
    public String getOrder() { return "DESC"; }

    @Override
    public Specification<Listing> build() {
        return (root, q, cb) -> {
            List<Predicate> predicates = new ArrayList<>();

            predicates.add(buildScopePredicate(root, cb)); // luôn đầu tiên

            if (query != null && query.trim().length() >= 2) {
                predicates.add(cb.like(
                    cb.lower(root.get("title")),
                    "%" + query.trim().toLowerCase() + "%"
                ));
            }

            if (status != null && !status.isEmpty()) {
                List<ListingStatus> valid = status.stream()
                    .map(s -> parseEnum(s, ListingStatus.class))
                    .filter(Objects::nonNull)
                    .toList();
                if (!valid.isEmpty()) {
                    predicates.add(root.get("status").in(valid));
                }
            }

            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }
}
```

### 9.4 Service — với applyScope tập trung

```java
@Service
@RequiredArgsConstructor
public class ListingService {

    private final ListingRepository listingRepository;

    private void applyScope(BaseQueryRequest request, UserPrincipal principal) {
        if (principal.isManager()) {
            request.setOrganizationId(principal.getOrganizationId());
            request.setUserId(null);
        } else {
            request.setUserId(principal.getId().toString());
            request.setOrganizationId(null);
        }
    }

    public PageResponse<ListingDto> getListings(
            ListingQueryRequest request,
            UserPrincipal principal) {

        applyScope(request, principal);   // 1. dịch role -> scope
        request.validateScope();          // 2. fail fast nếu scope rỗng
        Page<Listing> page = listingRepository
            .findAll(request.build(), request.getPageable()); // 3. query

        return PageResponse.of(page.map(ListingDto::from));
    }
}
```

### 9.5 Repository

```java
public interface ListingRepository
        extends JpaRepository<Listing, Long>,
                JpaSpecificationExecutor<Listing> {

    @Override
    Page<Listing> findAll(Specification<Listing> spec, Pageable pageable);
}
```

---

## 10. Production concerns

**Indexing**: Bắt buộc phải có index trên `organization_id` (hoặc composite index `organization_id + created_at` nếu có filter ngày). Thiếu index, mọi query sẽ full scan toàn bảng — chết performance khi table lớn.

```sql
CREATE INDEX idx_listing_org_created
    ON listing (organization_id, created_at DESC);
```

**Token tampering**: `organizationId` phải đến từ JWT claim đã được verify bởi Spring Security, không phải từ request param. Ký JWT bằng RS256 hoặc HS256 với secret mạnh; validate `exp`, `iss`, `aud`.

**Audit logging**: Log mọi truy cập scope (org nào query entity nào, bởi user nào) vào audit trail. Dùng Spring AOP hoặc `@EntityListeners` để không làm bẩn service code.

**Failure modes**:
- `validateScope()` ném exception nhưng không có handler: trả về 500 thay vì 400. Thêm `@ExceptionHandler` cho `IllegalArgumentException` trả về 400.
- Index bị drop: query chậm đột ngột — đặt cảnh báo trên slow query log (> 500ms).
- JWT secret bị lộ: tất cả token của hệ thống bị giả mạo — cần rotation cơ chế và revocation list (Redis blacklist).

**Caching**: Nếu cùng một scope và cùng tham số được gọi nhiều lần trong một request cycle, xem xét `@Cacheable` ở service method. Nhưng cần invalidate cache khi dữ liệu org thay đổi.

---

## 11. Common mistakes

Lỗi 1: Tin vào `organizationId` do client gửi trong request body.

```java
// SAI: client có thể gửi bất kỳ organizationId nào
public Page<Listing> getListings(ListingQueryRequest request) {
    return listingRepository.findAll(request.build(), request.getPageable());
}

// ĐÚNG: luôn lấy từ UserPrincipal, bỏ qua giá trị client gửi
public Page<Listing> getListings(ListingQueryRequest request, UserPrincipal principal) {
    applyScope(request, principal);
    request.validateScope();
    return listingRepository.findAll(request.build(), request.getPageable());
}
```

Lý do: IDOR là lỗ hổng nghiêm trọng. Nếu client tự khai báo orgId, họ có thể lấy dữ liệu của bất kỳ org nào.

---

Lỗi 2: Gọi `build()` trước khi gọi `validateScope()`.

```java
// SAI: nếu cả organizationId và userId đều null,
// build() tạo predicate null -> NullPointerException hoặc query sai
Specification spec = request.build();
request.validateScope();
repository.findAll(spec, pageable);

// ĐÚNG: validate trước, build() sau
applyScope(request, principal);
request.validateScope();
repository.findAll(request.build(), pageable);
```

Lý do: `build()` giả định scope đã được set hợp lệ. Nếu không validate trước, `buildScopePredicate` có thể throw `NumberFormatException` (khi `userId` là null mà code gọi `Long.parseLong(null)`) hoặc tạo predicate rỗng.

---

Lỗi 3: Dùng `.in()` với list rỗng.

```java
// SAI: in(emptyList) tạo invalid SQL -> RuntimeException từ Hibernate
predicates.add(root.get("status").in(List.of()));

// ĐÚNG: guard trước khi add predicate
if (!validStatuses.isEmpty()) {
    predicates.add(root.get("status").in(validStatuses));
}
```

---

Lỗi 4: Quên tạo index trên `organization_id`.

Fix: Kiểm tra explain plan (`EXPLAIN ANALYZE` trong PostgreSQL) sau khi deploy. Nếu thấy "Seq Scan" trên bảng lớn, thêm index ngay.

---

## 12. Sample project

Xây dựng API quản lý công việc (task management) có organization isolation:

- Entity: `Task` (id, title, status, organizationId, assigneeId, createdAt)
- Role: `MANAGER` thấy tất cả task của org; `EMPLOYEE` chỉ thấy task được gán cho mình
- Endpoint: `GET /tasks?status=OPEN&query=bug&page=0&pageSize=10`
- Ràng buộc cứng: `organizationId` và `userId` không được phép xuất hiện trong request body hay query param của client

Bước thực hiện:
1. Tạo `Task` entity với `organizationId` và `User assignee`.
2. Tạo `TaskQueryRequest extends BaseQueryRequest`, implement `build()` với scope predicate và filter `status`, `query`.
3. Tạo `TaskService.getTasks(request, principal)` theo 3 bước: `applyScope -> validateScope -> findAll`.
4. Tạo `TaskController` nhận `@AuthenticationPrincipal UserPrincipal`.
5. Viết integration test kiểm tra: manager thấy đủ task của org, employee chỉ thấy task của mình, user gửi orgId tay trong param bị ignore.

---

## 13. Interview

### Core Q&A

**Q: Tại sao không để client tự khai báo `organizationId` trong request?**
A: Vì đây là Insecure Direct Object Reference (IDOR) — client có thể thay đổi giá trị để truy cập dữ liệu của org khác. `organizationId` phải đến từ JWT token đã được server ký và xác thực, không bao giờ từ input của client.

**Q: Scope predicate nên đặt ở tầng nào — controller, service hay repository?**
A: Service layer. Controller không nên biết logic business. Repository không nên biết về authentication. Service là nơi duy nhất biết cả `UserPrincipal` (từ security context) lẫn `Specification` (cho repository).

**Q: Nếu có nhiều entity khác nhau (Listing, Order, Employee), scope logic có bị lặp lại không?**
A: Không, nếu dùng `BaseQueryRequest`. `applyScope()` và `validateScope()` được định nghĩa một lần ở base class hoặc service base method; từng `XxxQueryRequest` chỉ override `build()` để thêm filter riêng.

**Q: Pattern này xử lý thế nào nếu một user có thể thuộc nhiều organization?**
A: Cần sửa `UserPrincipal` để chứa list `organizationIds`. `applyScope` set scope là tập con được phép, `buildScopePredicate` dùng `root.get("organizationId").in(allowedOrgs)` thay vì `cb.equal`.

**Q: Ưu điểm của JPA Specification so với `@Query` trong trường hợp này?**
A: Specification là composable và type-safe: từng điều kiện filter là một object riêng, có thể kết hợp linh hoạt mà không cần viết N câu JPQL khác nhau. `@Query` với optional filter thì phải dùng JPQL string nối hoặc `@Query` với `WHERE 1=1 AND (:orgId IS NULL OR ...)` — khó đọc, khó bảo trì, dễ sai.

### Scenarios

**Scenario 1**: Một employee gửi request `GET /listings?organizationId=org-999` để thử lấy dữ liệu của org khác. Hệ thống xử lý thế nào?
Trả lời: `applyScope()` bỏ qua `organizationId` trong request, thay vào đó set `userId = principal.getId()`. Query chỉ trả về listing của user đó. `org-999` hoàn toàn bị ignore.

**Scenario 2**: `organizationId` trong JWT token bị null (lỗi ở auth service). Hệ thống xử lý thế nào?
Trả lời: `applyScope()` set cả `organizationId` và `userId` dựa trên role. Nếu `principal.getOrganizationId()` là null, `request.setOrganizationId(null)`. `validateScope()` kiểm tra cả hai đều null/empty, ném `IllegalArgumentException`, tầng controller trả về 400 Bad Request.

**Scenario 3**: Một manager cần xem tất cả listing của org, bao gồm listing của tất cả employee. Làm thế nào?
Trả lời: `applyScope()` detect `principal.isManager()` -> set `organizationId = principal.getOrganizationId()`, `userId = null`. `buildScopePredicate` trả về `cb.equal(root.get("organizationId"), orgId)` — không lọc theo user, trả về tất cả listing của org.

---

## 14. References

- Spring Data JPA — Specifications: https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#specifications
- OWASP — Broken Access Control (IDOR): https://owasp.org/Top10/A01_2021-Broken_Access_Control/
- Spring Security — Authentication Principal: https://docs.spring.io/spring-security/reference/servlet/integrations/mvc.html#mvc-authentication-principal
- JPA Criteria API — Jakarta EE Spec: https://jakarta.ee/specifications/persistence/3.1/apidocs/jakarta.persistence/jakarta/persistence/criteria/package-summary.html
- Spring Data Commons Changelog: https://github.com/spring-projects/spring-data-commons/blob/main/changelog.txt

---

## 15. Real-world Code

- **jhipster/generator-jhipster** (GitHub): JHipster tạo Spring Boot app với multi-tenancy support; xem cách họ generate `UserRepository` và scope filtering theo từng user.
- **OpenSaas** hoặc **Wasp** (GitHub): SaaS boilerplate hiển thị isolation pattern ở tầng application.
- **keycloak/keycloak** (GitHub): Xem `OrganizationProvider` và cách Keycloak quản lý member của từng organization ở tầng identity.
- **spring-projects/spring-authorization-server** (GitHub): Xem cách `JwtDecoder` extract claims để build `UserPrincipal`.

---

## 16. Community

- Reddit r/java — "Multi-tenancy in Spring Boot: schema vs row-level": thảo luận về các approach khác nhau, kinh nghiệm thực tế.
- Stack Overflow tag `[spring-data-jpa]` + `[multitenancy]`: nhiều câu hỏi về Hibernate filter vs Specification vs RLS.
- Baeldung — "Introduction to Spring Data Specifications": https://www.baeldung.com/rest-api-search-language-spring-data-specifications
- Blog "Vlad Mihalcea" — Hibernate Multi-Tenancy guide: chi tiết về schema-per-tenant và discriminator column approach.
- Spring I/O Conference talks (YouTube): tìm "Spring Security multi-tenant" để xem demo thực tế với JWT và organization claim.
