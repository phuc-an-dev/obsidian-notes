---
created: 2026-04-17
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/performance"
related:
  - "[[criteria-api]]"
---

# Spring Data JPA Specification

## 1. What

`Specification<T>` là một interface trong Spring Data JPA, đóng gói một điều kiện lọc (predicate) thành một object có thể kết hợp, tái sử dụng và truyền đi như tham số. Nó là một wrapper mỏng trên JPA Criteria API, giúp xây dựng các câu query động (dynamic query) một cách type-safe mà không phải nối chuỗi SQL. Repository cần implement `JpaSpecificationExecutor<T>` để sử dụng Specification.

## 2. Why

Trước khi có Specification, việc xử lý tìm kiếm động (form có 10 ô filter, user có thể điền 0 đến 10 ô) thường dẫn đến:

- **Query method bùng nổ**: phải viết `findByFirstName`, `findByLastName`, `findByFirstNameAndLastName`, `findByFirstNameAndLastNameAndCity`... — số lượng method tăng theo hàm mũ của số lượng trường lọc.
- **JPQL string manipulation**: tự cộng chuỗi JPQL với `if-else` — dễ gây SQL injection nếu không cẩn thận, code cực kỳ khó bảo trì.
- **@Query cứng nhắc**: mỗi sự thay đổi điều kiện yêu cầu viết thêm annotation mới, không tái sử dụng được logic lọc.
- **Lack of composability**: không thể ghép điều kiện "giống như Lego" — `isAdult().and(isFromHanoi())` — buộc phải copy-paste logic.

Specification giải quyết bằng cách biến mỗi điều kiện lọc thành một object độc lập, có thể test riêng và kết hợp tự do.

## 3. Mental Model

Hãy nghĩ đến hệ thống lọc nước nhiều tầng trong nhà máy xử lý nước.

- **Mỗi Specification** là một **module lọc** — có thể là lọc phù sa, lọc vi khuẩn, lọc kim loại nặng, hoặc lọc hóa chất. Mỗi module làm đúng một việc.
- **Specification.where().and().or()** là hệ thống **ống nối** — bạn lắp ráp các module lọc theo thứ tự và cách kết hợp tùy ý.
- **JpaSpecificationExecutor.findAll(spec)** là **máy bơm** — nó đẩy nước (data) qua toàn bộ hệ thống ống đã lắp ráp.
- **Database** là **nguồn nước thực** — dữ liệu đi qua hệ thống lọc và ra đầu kia là kết quả đã lọc.

Khi user không điền ô filter nào, bạn nối tắt các module lọc đó lại (trả về null predicate = không lọc gì). Khi user điền một số ô, bạn lắp ráp đúng những module đó. Thay đổi hợp lý đơn giản vì mỗi module hoàn toàn độc lập.

## 4. Where It Fits

```
HTTP Request (search params)
    |
    v
Controller  ->  SearchCriteria POJO
    |
    v
Service  ->  Spec.where(hasName(criteria.name))
             .and(hasDepartment(criteria.dept))
             .and(hasSalaryBetween(criteria.minSalary, criteria.maxSalary))
    |
    v
Repository (JpaSpecificationExecutor)
    |
    v
Spring Data JPA  ->  CriteriaQuery / CriteriaBuilder
    |
    v
JPA Provider (Hibernate)  ->  SQL
    |
    v
Database
```

## 5. When to Use

- Form tìm kiếm có nhiều trường lọc tùy chọn (optional filters) — user có thể điền bất kỳ tổ hợp nào.
- Khi cần tái sử dụng logic lọc ở nhiều chỗ khác nhau (cùng Specification dùng cho cả tìm kiếm và export CSV).
- Khi muốn test logic lọc độc lập với database (Specification là POJO thuần túy, test không cần Spring context).
- Khi cần kết hợp điều kiện phức tạp: `(isActive AND isFromHanoi) OR (isVip AND hasBalance > 1000)`.
- Khi team muốn có một nơi duy nhất quản lý tất cả business rules về lọc dữ liệu — "single source of truth" cho filter logic.

## 6. When NOT to Use

- Câu query đơn giản và cố định (static) — dùng `findByName(String name)` query method hoặc `@Query("SELECT u FROM User u WHERE u.name = :name")` — ngắn gọn hơn nhiều.
- Khi performance là tối quan trọng và cần fine-tune SQL cụ thể — dùng Native Query (`@Query(nativeQuery = true)`) để viết SQL tay tối ưu.
- Khi project đã dùng QueryDSL — QueryDSL mạnh hơn và có cú pháp đẹp hơn (strongly-typed path expressions), không nên dùng cả hai để tránh chồng chéo.
- Khi query quá nhiều bảng và có nhiều GROUP BY, HAVING, subquery phức tạp — Criteria API bắt đầu trở nên vô dụng ở đó, Native Query rõ ràng hơn.

## 7. Trade-offs

| Pros | Cons |
|---|---|
| Dynamic query không cần viết nhiều method | Criteria API verbose và khó đọc cho người mới |
| Type-safe — lỗi được phát hiện lúc compile | Tạo thêm "boilerplate" class ProductSpecs, EmployeeSpecs |
| Composable — and/or/not như Boolean algebra | Debug SQL phát sinh khó khi quá nhiều tầng abstraction |
| Tái sử dụng logic lọc giữa nhiều service | Fetch join trong Specification có thể bị conflict với count query khi dùng Pageable |
| Dễ unit test từng Specification độc lập | N+1 problem vẫn xảy ra nếu không cấu hình kỹ fetch strategy |
| Tích hợp với Pageable và Sort | Học thêm JPA Criteria API để viết Specification phức tạp |

## 8. Alternatives

| Option | Phù hợp khi | Hạn chế so với Specification |
|---|---|---|
| Query Methods (findByX) | Query đơn giản, ít trường lọc, không đổi | Bùng nổ khi có nhiều tổ hợp |
| @Query JPQL | Query trung bình, cần viết JPQL tay | Không có khả năng compose, khó tái sử dụng |
| @Query Native SQL | Performance quan trọng, dùng đặc thù DB | Phụ thuộc database, khó test |
| QueryDSL | Project lớn, cần type-safe mạnh hơn | Cần code generation plugin, setup phức tạp hơn |
| jOOQ | Muốn SQL-first approach, full SQL control | Không phải ORM, khó migrate DB |
| Criteria API thuần túy | Muốn full control mà không cần Spring Data | Rất verbose, không có khả năng compose dễ |

## 9. How

### Bước 1: Thêm dependency (nếu chưa có)

```xml
<!-- Spring Data JPA đã bao gồm sẵn trong spring-boot-starter-data-jpa -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

### Bước 2: Entity và Repository

```java
// Employee.java
@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String firstName;
    private String lastName;
    private Double salary;
    private Boolean active;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;

    // getters, setters, constructors
}

// Department.java
@Entity
@Table(name = "departments")
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
}

// EmployeeRepository.java
public interface EmployeeRepository
    extends JpaRepository<Employee, Long>,
            JpaSpecificationExecutor<Employee> {
    // Thêm JpaSpecificationExecutor là đủ — Spring Data tự generate implementation
}
```

### Bước 3: Tạo Specification helper class

```java
// EmployeeSpecs.java
public final class EmployeeSpecs {

    // Private constructor: utility class, không được instantiate
    private EmployeeSpecs() {}

    // Lọc theo firstName (partial match, case-insensitive)
    public static Specification<Employee> hasFirstName(String firstName) {
        return (root, query, cb) -> {
            if (firstName == null || firstName.isBlank()) return null;
            return cb.like(cb.lower(root.get("firstName")), "%" + firstName.toLowerCase() + "%");
        };
    }

    // Lọc theo trạng thái active
    public static Specification<Employee> isActive(Boolean active) {
        return (root, query, cb) -> {
            if (active == null) return null;
            return cb.equal(root.get("active"), active);
        };
    }

    // Lọc theo salary range
    public static Specification<Employee> hasSalaryBetween(Double min, Double max) {
        return (root, query, cb) -> {
            if (min == null && max == null) return null;
            if (min == null) return cb.lessThanOrEqualTo(root.get("salary"), max);
            if (max == null) return cb.greaterThanOrEqualTo(root.get("salary"), min);
            return cb.between(root.get("salary"), min, max);
        };
    }

    // Lọc theo tên phòng ban — cần JOIN
    public static Specification<Employee> hasDepartmentName(String deptName) {
        return (root, query, cb) -> {
            if (deptName == null || deptName.isBlank()) return null;
            // root.join() tạo INNER JOIN với bảng departments
            Join<Employee, Department> dept = root.join("department", JoinType.INNER);
            return cb.equal(dept.get("name"), deptName);
        };
    }

    // Fetch join — dùng khi cần load eager cho List result (tránh N+1)
    // QUAN TRỌNG: chỉ dùng fetch join khi query không phải count query (Pageable)
    public static Specification<Employee> withDepartmentFetched() {
        return (root, query, cb) -> {
            // Chỉ fetch nếu đây là SELECT query, không phải COUNT query
            if (query.getResultType() != Long.class && query.getResultType() != long.class) {
                root.fetch("department", JoinType.LEFT);
            }
            return null; // không thêm predicate, chỉ thay đổi fetch strategy
        };
    }
}
```

### Bước 4: Dùng Specification trong Service

```java
// EmployeeSearchCriteria.java (DTO cho search params)
public class EmployeeSearchCriteria {
    private String firstName;
    private String departmentName;
    private Double minSalary;
    private Double maxSalary;
    private Boolean active;
    // getters, setters
}

// EmployeeService.java
@Service
@RequiredArgsConstructor
public class EmployeeService {

    private final EmployeeRepository employeeRepository;

    public Page<Employee> searchEmployees(EmployeeSearchCriteria criteria, Pageable pageable) {
        Specification<Employee> spec = Specification
            .where(EmployeeSpecs.hasFirstName(criteria.getFirstName()))
            .and(EmployeeSpecs.hasDepartmentName(criteria.getDepartmentName()))
            .and(EmployeeSpecs.hasSalaryBetween(criteria.getMinSalary(), criteria.getMaxSalary()))
            .and(EmployeeSpecs.isActive(criteria.getActive()));

        return employeeRepository.findAll(spec, pageable);
    }

    // Kết hợp điều kiện OR
    public List<Employee> findSeniorOrVip(Double seniorSalaryThreshold, Boolean vipFlag) {
        Specification<Employee> seniorSpec =
            EmployeeSpecs.hasSalaryBetween(seniorSalaryThreshold, null);
        Specification<Employee> vipSpec =
            EmployeeSpecs.isActive(vipFlag);

        return employeeRepository.findAll(seniorSpec.or(vipSpec));
    }
}
```

### Bước 5: Kiểm tra Specification bằng Unit Test

```java
// EmployeeSpecsTest.java — test KHÔNG cần Spring context
class EmployeeSpecsTest {

    @Test
    void hasFirstName_shouldReturnNull_whenInputIsBlank() {
        Specification<Employee> spec = EmployeeSpecs.hasFirstName("  ");
        // Setup mock root, query, cb
        Root<Employee> root = mock(Root.class);
        CriteriaQuery<?> query = mock(CriteriaQuery.class);
        CriteriaBuilder cb = mock(CriteriaBuilder.class);

        Predicate result = spec.toPredicate(root, query, cb);

        assertThat(result).isNull();
        verifyNoInteractions(cb); // cb.like() không được gọi
    }
}
```

### Bước 6: Pagination với Specification

```java
// Controller
@GetMapping("/employees")
public ResponseEntity<Page<EmployeeDto>> searchEmployees(
        @ModelAttribute EmployeeSearchCriteria criteria,
        @PageableDefault(size = 20, sort = "lastName") Pageable pageable) {

    Page<Employee> page = employeeService.searchEmployees(criteria, pageable);
    Page<EmployeeDto> dtoPage = page.map(EmployeeDto::fromEntity);
    return ResponseEntity.ok(dtoPage);
}
```

## 10. Production Concerns

### Scaling

- Với bảng lớn (> 1 triệu bản ghi), luôn đảm bảo các trường dùng trong Specification có database index. Specification chỉ sinh SQL — optimizer vẫn cần index để chạy nhanh.
- Trang đầu của paginated result (`OFFSET 0`) nhanh. Trang cuối (`OFFSET 999000`) rất chậm. Với dataset lớn, xem xét cursor-based pagination thay vì offset.
- `Specification.where(null)` trả về tất cả bản ghi — cần có default limit ở tầng Repository hoặc Service để tránh query không giới hạn.

### Failure / N+1 Problem

Câu chuyện cần cẩn thận nhất với Specification là N+1 query problem:

```java
// NGUY HIỂM: Nếu Employee.department là LAZY, việc truy cập dept.name cho N employees
// sẽ tạo N+1 SELECT query
List<Employee> employees = repo.findAll(spec);
employees.forEach(e -> System.out.println(e.getDepartment().getName())); // N queries!

// FIX OPTION 1: Dùng fetch join trong Specification
Specification<Employee> withDept = (root, query, cb) -> {
    if (!isCountQuery(query)) {
        root.fetch("department", JoinType.LEFT);
    }
    return null;
};

// FIX OPTION 2: Dùng @EntityGraph trên Repository method
@EntityGraph(attributePaths = {"department"})
Page<Employee> findAll(Specification<Employee> spec, Pageable pageable);

// FIX OPTION 3: Projection DTO trong Query (tránh load toàn bộ entity)
```

Kiểm tra N+1: bật `spring.jpa.show-sql=true` và `logging.level.org.hibernate.SQL=DEBUG` trong dev. Đếm số câu SELECT trong log.

### Fetch Join và Count Query Conflict

Khi dùng `Pageable`, Spring Data JPA chạy 2 query: một SELECT để lấy dữ liệu, một COUNT(*) để tính tổng. Fetch join trong SELECT query sẽ làm COUNT query fail:

```
org.hibernate.QueryException: query specified join fetching, but the owner of the fetched association was not present in the select list
```

Giải pháp: Kiểm tra `query.getResultType()` trong Specification để chỉ áp dụng fetch khi không phải count query (xem ví dụ ở Bước 3 trên).

### Monitoring

- Log SQL trong dev: `spring.jpa.show-sql=true` và `spring.jpa.properties.hibernate.format_sql=true`.
- Production: dùng Hibernate statistics hoặc p6spy để theo dõi slow query.
- Monitor query count per request — nếu > 5 SELECT cho một API call, có khả năng có N+1.

## 11. Common Mistakes

- Mistake: Quên implement `JpaSpecificationExecutor<T>` trong Repository interface — Spring Data không cung cấp các method `findAll(Specification)` và compile sẽ lỗi hoặc Runtime exception.
  Fix: Repository phải extend cả `JpaRepository<T, ID>` và `JpaSpecificationExecutor<T>`. Đây là mixin interface — không tăng thêm dependency, chỉ mở khóa các generated method.

- Mistake: Đặt logic nghiệp vụ phức tạp (ví dụ: tính toán discount, check business rule) bên trong lambda của Specification — làm Specification khó test và vi phạm SRP.
  Fix: Specification chỉ chứa logic so sánh dữ liệu (tạo Predicate). Logic nghiệp vụ nên được xử lý trước ở tầng Service, truyền kết quả đã tính toán vào Specification như tham số.

- Mistake: Dùng `root.join()` thay vì `root.fetch()` cho quan hệ LAZY khi muốn eager load, dẫn đến N+1 — `join()` chỉ tạo JOIN trong SQL nhưng Hibernate vẫn load lazy khi truy cập.
  Fix: Dùng `root.fetch()` để chỉ thị Hibernate eager load kết quả join vào entity. Nhớ kiểm tra kiểu query để tránh conflict với COUNT query khi dùng Pageable.

- Mistake: Không xử lý null an toàn bên trong Specification — nếu tham số filter là null mà không kiểm tra, `cb.like()` hoặc `cb.equal()` sẽ throw NullPointerException lúc runtime.
  Fix: Mỗi Specification helper method đều nên kiểm tra null/blank ở đầu tiên và trả về `null` (no predicate) nếu input rỗng. Spring Data JPA xử lý `null` predicate bằng cách bỏ qua nó trong mệnh đề WHERE.

## 12. Sample Project

Xây dựng **Employee Directory** với ràng buộc cứng sau:

**Constraint:** Hệ thống PHẢI hỗ trợ tìm kiếm nhân viên với các điều kiện kết hợp bất kỳ:
1. Tìm theo tên (firstName + lastName, partial match, case-insensitive).
2. Lọc theo phòng ban (JOIN bảng `departments`, exact match).
3. Lọc theo mức lương (range: minSalary đến maxSalary).
4. Lọc theo trạng thái (active/inactive/tất cả).
5. Kết quả PHẢI hỗ trợ phân trang và sắp xếp.

**Hard Constraint:**
- `EmployeeRepository` không được có bất kỳ method nào ngoài kế thừa từ `JpaRepository` và `JpaSpecificationExecutor`.
- Tất cả logic xây dựng query PHẢI nằm trong class `EmployeeSpecs` độc lập.
- Không được dùng `@Query` annotation.
- Phải có unit test cho `EmployeeSpecs` (mock Root, CriteriaQuery, CriteriaBuilder) và integration test với H2 in-memory database.

**Structure gợi ý:**
```
employees/
    Employee.java              // Entity
    Department.java            // Entity
    EmployeeRepository.java    // JpaRepository + JpaSpecificationExecutor
    EmployeeSpecs.java         // Tất cả Specification factories
    EmployeeSearchCriteria.java // DTO
    EmployeeService.java       // Build và execute Spec
    EmployeeController.java    // REST endpoint với Pageable
tests/
    EmployeeSpecsTest.java     // Unit test (no Spring context)
    EmployeeSearchIntegrationTest.java  // @DataJpaTest với H2
```

## 13. Interview

### Core Q&A

**Q: Interface `Specification<T>` có method gì và các tham số là gì?**
A: Chỉ có một method functional: `Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder)`. `Root<T>` đại diện cho entity đang query (FROM clause). `CriteriaQuery<?>` là câu query hoàn chỉnh (dùng để kiểm tra có phải count query hay không). `CriteriaBuilder` là factory tạo Predicate (WHERE conditions).

**Q: `JpaSpecificationExecutor` cung cấp những method nào?**
A: Các method chính: `findOne(Spec)`, `findAll(Spec)`, `findAll(Spec, Sort)`, `findAll(Spec, Pageable)`, `count(Spec)`, `exists(Spec)`. Tất cả đều nhận `Specification<T>` làm tham số để lọc.

**Q: Làm thế nào để kết hợp nhiều Specification?**
A: Dùng các static method trên interface Specification: `Specification.where(s1).and(s2).or(s3).not()`. Tương tự Boolean algebra. `where(null)` trả về tất cả bản ghi.

**Q: Tại sao Specification trả về `null` thay vì `cb.conjunction()` khi không có điều kiện?**
A: Cả hai đều được. `null` có nghĩa là "không có predicate nào, bỏ qua điều kiện này." Spring Data JPA xử lý null predicate bằng cách bỏ qua nó. `cb.conjunction()` là `TRUE` literal (tương đương KHÔNG lọc gì). Trả `null` ngắn gọn hơn và được recommend trong doc chính thức.

**Q: Sự khác biệt giữa `root.join()` và `root.fetch()` trong Specification?**
A: `root.join()` tạo SQL JOIN nhưng Hibernate vẫn load associated entity theo fetch strategy được cấu hình (LAZY hoặc EAGER) — có thể dẫn đến N+1. `root.fetch()` tạo SQL JOIN và chỉ thị Hibernate load associated entity vào bộ nhớ cùng với query đó — tránh N+1. Dùng `fetch()` khi muốn eager load, nhưng phải kiểm tra kiểu query để tránh conflict với count query.

### Scenario

**Scenario 1:** Form tìm kiếm có 20 trường lọc, bạn sẽ tổ chức code như thế nào?

Trả lời: Tạo class `SearchCriteria` là POJO chứa 20 trường. Tạo class `EntitySpecs` với 20 static method, mỗi method nhận một tham số và kiểm tra null/blank trước khi tạo predicate. Trong Service, dùng `Specification.where(null)` rồi nối từng spec với `.and()` — các spec trả `null` tự động bị bỏ qua. Kết quả là code sạch, dễ test, dễ mở rộng.

**Scenario 2:** Bạn phải tìm tất cả Order có giá trị > 1000 USD HOẶC được đặt bởi khách VIP. Cấu hình Specification như thế nào?

Trả lời: `Specification<Order> highValueOrVip = OrderSpecs.hasValueGreaterThan(1000.0).or(OrderSpecs.isFromVipCustomer(true));` Sau đó truyền vào `orderRepository.findAll(highValueOrVip)`. Specification compose rất tự nhiên cho `OR` condition — đây là điểm mạnh so với JPQL viết tay.

**Scenario 3:** Sau khi deploy, bạn thấy SQL log có hàng trăm query nhỏ cho mỗi request tìm kiếm. Nguyên nhân là gì và fix thế nào?

Trả lời: Đây là N+1 problem — mỗi Employee trong kết quả đang tạo thêm một SELECT để load entity liên quan (ví dụ Department). Fix: (1) Thêm `root.fetch("department", JoinType.LEFT)` vào Specification để eager load trong cùng một JOIN query, nhớ kiểm tra kiểu query để tránh conflict với count query khi dùng Pageable. (2) Hoặc thêm `@EntityGraph(attributePaths = {"department"})` trên method `findAll` trong Repository. (3) Xem xét dùng Projection DTO để chỉ SELECT trường cần thiết thay vì toàn bộ entity.

**Scenario 4:** Specification của bạn hoạt động tốt khi test với findAll(spec), nhưng khi thêm Pageable thì ném exception. Tại sao?

Trả lời: Nếu Specification chứa `root.fetch()` mà query là COUNT query (Pageable chạy 2 query), Hibernate sẽ throw `QueryException: query specified join fetching, but the owner of the fetched association was not present in the select list`. Fix: Kiểm tra `query.getResultType() != Long.class` trước khi gọi `root.fetch()` — chỉ fetch khi là SELECT query, không phải COUNT query.

## 14. References

- Spring Data JPA Docs - Specifications: https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#specifications
- JPA Criteria API Tutorial (Oracle): https://docs.oracle.com/javaee/7/tutorial/persistence-criteria.htm
- JavaDoc - JpaSpecificationExecutor: https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/repository/JpaSpecificationExecutor.html
- JavaDoc - Specification: https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/domain/Specification.html
- Spring Data JPA source - Specification interface: https://github.com/spring-projects/spring-data-jpa/blob/main/spring-data-jpa/src/main/java/org/springframework/data/jpa/domain/Specification.java

## 15. Real-world Code

- Spring Data JPA integration tests (official): https://github.com/spring-projects/spring-data-jpa/tree/main/spring-data-jpa/src/test/java/org/springframework/data/jpa
- Baeldung tutorial repo - Specification examples: https://github.com/eugenp/tutorials/tree/master/persistence-modules/spring-data-jpa-query-3
- Vlad Mihalcea blog examples (N+1, fetch joins): https://github.com/vladmihalcea/high-performance-java-persistence
- Spring PetClinic REST (uses Specifications for filtering): https://github.com/spring-petclinic/spring-petclinic-rest

## 16. Community

- Stack Overflow - "Spring Data JPA Specification with Join": https://stackoverflow.com/questions/18707129/using-specification-and-querydsl-with-spring-data-jpa
- Stack Overflow - "fetch join in specification causes CountQuery to fail": https://stackoverflow.com/questions/21549460/spring-data-jpa-specification-with-fetch-joins-causing-issue
- Baeldung - REST Query Language with Spring Data JPA Specifications: https://www.baeldung.com/rest-api-search-language-spring-data-specifications
- Vlad Mihalcea - N+1 problem explained: https://vladmihalcea.com/n-plus-1-query-problem/
- Reddit r/java - discussion on QueryDSL vs Specification: https://www.reddit.com/r/java/comments/specification-vs-querydsl
