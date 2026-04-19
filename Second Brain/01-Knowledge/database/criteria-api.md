---
created: 2026-04-17
tags:
  - type/concept
  - status/draft
  - lang/java
  - topic/performance
related:
  - "[[specification-spring-data-jpa]]"
---

# JPA Criteria API

## 1. What

JPA Criteria API là một bộ API trong JPA 2.0 cho phép xây dựng câu truy vấn cơ sở dữ liệu theo kiểu lập trình (programmatic) và type-safe, không cần viết chuỗi JPQL hoặc SQL thủ công. Thay vì ghép chuỗi query, lập trình viên dùng các đối tượng Java (`CriteriaBuilder`, `CriteriaQuery`, `Root`, `Predicate`) để mô tả cấu trúc câu truy vấn. Compiler có thể bắt lỗi sai tên field hoặc kiểu dữ liệu ngay lúc biên dịch, không phải lúc runtime.

---

## 2. Why

Trước khi có Criteria API, cách phổ biến nhất để xây dựng query động trong JPA là nối chuỗi JPQL hoặc native SQL:

```java
// Anti-pattern: ghep chuoi query
String jpql = "SELECT u FROM User u WHERE 1=1";
if (name != null) jpql += " AND u.name = :name";
if (age  != null) jpql += " AND u.age > :age";
```

Cách này có hàng loạt vấn đề: lỗi sai tên field chỉ xuất hiện lúc runtime, refactor đổi tên entity không được IDE hỗ trợ, SQL injection tiềm ẩn khi ghép tham số sai cách, và không có compile-time type checking. Criteria API ra đời để giải quyết tất cả những vấn đề đó bằng cách đưa cấu trúc query vào thế giới đối tượng Java có kiểu dữ liệu rõ ràng.

---

## 3. Mental Model

Hãy nghĩ Criteria API như **bộ lắp ghép LEGO để xây câu hỏi cho database**.

Mỗi mảnh LEGO là một đối tượng Java:
- `CriteriaBuilder` là **hộp LEGO** — nơi bạn lấy tất cả các mảnh ghép ra.
- `CriteriaQuery<T>` là **bản thiết kế** — định nghĩa hình dạng cuối cùng của câu hỏi (SELECT gì, kiểu trả về là gì).
- `Root<T>` là **điểm neo chính** — tương đương với FROM clause, xác định entity trung tâm.
- `Predicate` là **mảnh ghép điều kiện** — mỗi `cb.equal()`, `cb.like()`, `cb.between()` tạo ra một mảnh; bạn kết hợp chúng với `cb.and()` / `cb.or()` để tạo WHERE clause.

Khi bạn ghép xong, bạn đưa bản thiết kế cho `EntityManager` và nhận kết quả về. Không có chuỗi nào được nối tay — tất cả đều là đối tượng có kiểu cụ thể.

---

## 4. Where it fits

```
HTTP Request
    |
    v
Controller / REST Layer
    |
    v
Service Layer  <-- business logic, assembles predicates dynamically
    |
    v
Repository / DAO Layer  <-- EntityManager + CriteriaBuilder
    |
    v
JPA Provider (Hibernate)  <-- translates CriteriaQuery to SQL
    |
    v
Database
```

Trong Spring Data JPA, tầng `Repository` thường expose interface `JpaSpecificationExecutor<T>`. `Specification<T>` là một wrapper của `Predicate` — mọi thứ cuối cùng đều đi qua Criteria API bên dưới.

---

## 5. When to use

- Query có nhiều điều kiện filter tùy chọn (optional filters), số lượng điều kiện thay đổi theo input của user — ví dụ trang tìm kiếm với 8-10 filter khác nhau có thể bật/tắt độc lập.
- Cần type safety hoàn toàn: tên field được tham chiếu qua static metamodel (`User_.name`) nên compiler bắt lỗi sai tên ngay lúc biên dịch.
- Cần tái sử dụng các predicate: viết một predicate "isActive" một lần, ghép vào nhiều query khác nhau mà không trùng lặp code.
- Cần phân trang và sắp xếp động kết hợp với filter động — Criteria API tích hợp tự nhiên với `TypedQuery.setFirstResult()` / `setMaxResults()`.
- Dùng cùng Spring Data JPA `Specification` pattern: mỗi Specification đóng gói một `Predicate`, compose chúng bằng `Specification.where().and().or()`.

---

## 6. When NOT to use

- Query tĩnh, đơn giản: nếu điều kiện không thay đổi theo runtime input, dùng `@Query` JPQL hoặc derived query (`findByNameAndAge`) ngắn gọn và dễ đọc hơn nhiều.
- Reporting / analytic queries: query phức tạp với nhiều GROUP BY, HAVING, subquery, window function — Criteria API trở nên cực kỳ verbose và khó maintain hơn cả native SQL.
- Team nhỏ, deadline gấp, project ngắn hạn: Criteria API có learning curve dốc. Với team chưa quen, dùng JPQL string và kiểm tra kỹ có thể deliver nhanh hơn.
- Query chỉ đọc với JOIN phức tạp và nhiều aggregate: nhiều JOIN lồng nhau dễ tạo Cartesian product, và lỗi Cartesian product trong Criteria API khó debug hơn trong SQL thuần.

Hệ quả nếu dùng sai: code trở nên verbose gấp 5-10 lần so với JPQL tương đương, team mới vào khó hiểu intent, và performance không tốt hơn gì vì Hibernate vẫn generate cùng một SQL ở tầng thấp.

---

## 7. Trade-offs

| Pros | Cons |
|------|------|
| Type-safe — lỗi field name bị bắt lúc compile | Verbose — code dài hơn JPQL 3-5 lần |
| Refactor-safe — đổi tên field, IDE cập nhật tự động | Learning curve cao, khó đọc với người mới |
| Dynamic predicate composition linh hoạt | Metamodel cần bước generate code bổ sung khi build |
| Tái sử dụng Predicate như building block độc lập | Debug khó hơn: SQL được generate không hiển thị ngay khi viết code |
| Tích hợp tốt với Spring Data Specification | JOIN với collection dễ gây Cartesian product nếu quên DISTINCT |
| Hỗ trợ phân trang, ordering tích hợp sẵn | Không phù hợp với aggregation / analytic queries phức tạp |

---

## 8. Alternatives

| Alternative | Khi nào dùng | Hạn chế so với Criteria API |
|---|---|---|
| JPQL `@Query` string | Query tĩnh, đơn giản, ít điều kiện | Không type-safe, ghép điều kiện động thủ công |
| Spring Data Derived Query | Query rất đơn giản, 1-3 điều kiện cố định | Tên method dài khủng khiếp khi nhiều điều kiện |
| QueryDSL | Query phức tạp, cần fluent API dễ đọc hơn | Dependency thêm, setup annotation processor phức tạp hơn |
| jOOQ | Query phức tạp, cần control SQL hoàn toàn | Không dùng JPA entity model, cần schema generation riêng |
| Native SQL `@Query(nativeQuery=true)` | Tối ưu performance, dùng SQL-specific feature | Mất tính portable, không type-safe |
| Blaze-Persistence | Entity View, phân trang nâng cao | Ít phổ biến, learning curve cao |

---

## 9. How

### Setup: Static Metamodel (khuyến nghị)

Thêm vào `pom.xml` để generate metamodel tự động khi build:

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jpamodelgen</artifactId>
    <scope>provided</scope>
</dependency>
```

Sau khi build, Hibernate sinh ra class `User_` trong `target/generated-sources`:

```java
// Generated -- khong sua tay
@StaticMetamodel(User.class)
public class User_ {
    public static volatile SingularAttribute<User, Long>    id;
    public static volatile SingularAttribute<User, String>  name;
    public static volatile SingularAttribute<User, String>  email;
    public static volatile SingularAttribute<User, Integer> age;
    public static volatile SingularAttribute<User, Boolean> active;
    public static volatile SetAttribute<User, Role>         roles;
}
```

### Entity

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String email;
    private int    age;
    private boolean active;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "user_roles",
        joinColumns        = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();

    // getters, setters
}
```

### Repository với EntityManager

```java
@Repository
public class UserCriteriaRepository {

    @PersistenceContext
    private EntityManager em;

    public List<User> findUsers(UserSearchFilter filter) {
        CriteriaBuilder     cb   = em.getCriteriaBuilder();
        CriteriaQuery<User> cq   = cb.createQuery(User.class);
        Root<User>          root = cq.from(User.class);

        List<Predicate> predicates = buildPredicates(cb, root, filter);

        cq.select(root)
          .where(cb.and(predicates.toArray(new Predicate[0])))
          .orderBy(cb.asc(root.get(User_.name)));

        TypedQuery<User> query = em.createQuery(cq);

        // Phân trang
        if (filter.getPage() != null && filter.getSize() != null) {
            query.setFirstResult(filter.getPage() * filter.getSize());
            query.setMaxResults(filter.getSize());
        }

        return query.getResultList();
    }

    private List<Predicate> buildPredicates(CriteriaBuilder  cb,
                                             Root<User>       root,
                                             UserSearchFilter filter) {
        List<Predicate> predicates = new ArrayList<>();

        // cb.equal -- exact match
        if (filter.getName() != null) {
            predicates.add(cb.equal(root.get(User_.name), filter.getName()));
        }

        // cb.like -- partial match, case-insensitive
        if (filter.getEmailLike() != null) {
            String pattern = "%" + filter.getEmailLike().toLowerCase() + "%";
            predicates.add(
                cb.like(cb.lower(root.get(User_.email)), pattern)
            );
        }

        // cb.between -- range
        if (filter.getMinAge() != null && filter.getMaxAge() != null) {
            predicates.add(
                cb.between(root.get(User_.age), filter.getMinAge(), filter.getMaxAge())
            );
        }

        // cb.isTrue -- boolean flag
        if (Boolean.TRUE.equals(filter.getActiveOnly())) {
            predicates.add(cb.isTrue(root.get(User_.active)));
        }

        return predicates;
    }

    // Count query riêng biệt để phân trang chính xác
    public long countUsers(UserSearchFilter filter) {
        CriteriaBuilder    cb   = em.getCriteriaBuilder();
        CriteriaQuery<Long> cq  = cb.createQuery(Long.class);
        Root<User>         root = cq.from(User.class);

        List<Predicate> predicates = buildPredicates(cb, root, filter);

        cq.select(cb.count(root))
          .where(cb.and(predicates.toArray(new Predicate[0])));

        return em.createQuery(cq).getSingleResult();
    }
}
```

### JOIN example

```java
// INNER JOIN -- chi lay User co it nhat 1 Role ADMIN
Join<User, Role> roleJoin = root.join(User_.roles, JoinType.INNER);
predicates.add(cb.equal(roleJoin.get(Role_.name), "ADMIN"));

// LEFT JOIN -- lay tat ca User, ke ca khong co Role
Join<User, Role> roleLeft = root.join(User_.roles, JoinType.LEFT);
predicates.add(cb.or(
    cb.isNull(roleLeft.get(Role_.id)),
    cb.equal(roleLeft.get(Role_.name), "VIEWER")
));

// Tránh duplicate khi JOIN với collection
cq.select(root).distinct(true);
```

### cb.or() và cb.and() kết hợp

```java
// WHERE (name = :name OR email LIKE :email) AND active = true
Predicate nameOrEmail = cb.or(
    cb.equal(root.get(User_.name), name),
    cb.like(root.get(User_.email), "%" + email + "%")
);
Predicate isActive = cb.isTrue(root.get(User_.active));
cq.where(cb.and(nameOrEmail, isActive));
```

### Spring Data Specification (wrapper trên Criteria API)

```java
public class UserSpecifications {

    public static Specification<User> hasName(String name) {
        return (root, query, cb) ->
            name == null
                ? null
                : cb.equal(root.get(User_.name), name);
    }

    public static Specification<User> hasEmailLike(String email) {
        return (root, query, cb) ->
            email == null
                ? null
                : cb.like(cb.lower(root.get(User_.email)),
                          "%" + email.toLowerCase() + "%");
    }

    public static Specification<User> isActive() {
        return (root, query, cb) -> cb.isTrue(root.get(User_.active));
    }

    // Tránh fetch join trong count query
    public static Specification<User> withRoles() {
        return (root, query, cb) -> {
            if (!Long.class.equals(query.getResultType())) {
                root.fetch(User_.roles, JoinType.LEFT);
                query.distinct(true);
            }
            return cb.conjunction(); // no additional predicate
        };
    }
}

// Sử dụng:
Specification<User> spec = Specification
    .where(UserSpecifications.hasName(filter.getName()))
    .and(UserSpecifications.hasEmailLike(filter.getEmail()))
    .and(UserSpecifications.isActive());

Page<User> result = userRepository.findAll(spec, pageable);
```

---

## 10. Production Concerns

### Scaling

- Criteria API không tự động tối ưu query. Mọi query được generate vẫn cần index trên database tương ứng. Với bảng lớn (hơn 1 triệu rows), luôn kiểm tra `EXPLAIN ANALYZE` trên SQL thực tế Hibernate generate.
- Bật `spring.jpa.show-sql=true` và `logging.level.org.hibernate.SQL=DEBUG` trong môi trường dev để xem SQL thực tế. Trong production dùng slow query log của database thay vì log ở application layer.
- Với connection pool HikariCP, cấu hình `maximumPoolSize` phù hợp tải — mặc định 10 thường không đủ cho production.

### Failure

- N+1 Problem với LAZY loading: khi Criteria API fetch `User` và sau đó code truy cập `user.getRoles()` trong vòng lặp, Hibernate thực hiện N query phụ. Fix: dùng `root.fetch(User_.roles, JoinType.LEFT)` trong data query (không phải count query) để EAGER fetch trong cùng một SQL, hoặc dùng `@EntityGraph` ở repository method.
- Cartesian product: JOIN với nhiều collection (`roles` và `permissions`) trong cùng một query mà không dùng `DISTINCT` sẽ multiply số row. Luôn thêm `cq.distinct(true)` hoặc tách thành nhiều query.
- Stack overflow / OutOfMemory: với predicate list rất lớn (hàng trăm điều kiện IN), Hibernate có thể generate SQL vượt quá giới hạn parse của database. Chia nhỏ thành nhiều batch query.

### Monitoring

- Bật Hibernate Statistics (`hibernate.generate_statistics=true`) để đếm số query thực thi, số entity được load, cache hit/miss rate.
- Tích hợp với Micrometer / Spring Actuator để expose `hibernate.sessions`, `hibernate.queries` metrics ra Prometheus.
- Với PostgreSQL trên cloud (RDS, Aurora), bật Performance Insights để identify slow queries và lock contention.

---

## 11. Common Mistakes

- Mistake: Quên `distinct(true)` khi JOIN với collection (`@OneToMany`, `@ManyToMany`), dẫn đến kết quả duplicate.
  Fix: Thêm `cq.select(root).distinct(true)` bất cứ khi nào JOIN với collection relationship.

- Mistake: Dùng `root.fetch()` trong count query, khiến Hibernate ném exception hoặc generate SQL sai khi dùng `JpaSpecificationExecutor.count()`.
  Fix: Tách riêng query lấy data (dùng `fetch`) và query đếm (không dùng `fetch`). Trong Specification, kiểm tra `query.getResultType()` trước khi gọi fetch:
  ```java
  (root, query, cb) -> {
      if (!Long.class.equals(query.getResultType())) {
          root.fetch(User_.roles, JoinType.LEFT);
      }
      return cb.conjunction();
  }
  ```

- Mistake: Truyền thẳng giá trị user input vào `cb.like()` mà không escape ký tự đặc biệt `%` và `_`, khiến user có thể accidentally hoặc intentionally match nhiều hơn dự kiến.
  Fix: Escape input trước khi ghép vào pattern:
  ```java
  String safe = input.replace("\\", "\\\\")
                     .replace("%",  "\\%")
                     .replace("_",  "\\_");
  cb.like(root.get(User_.email), "%" + safe + "%", '\\');
  ```

- Mistake: Dùng hardcoded string `root.get("name")` thay vì static metamodel `root.get(User_.name)`, mất toàn bộ lợi ích type-safe.
  Fix: Cấu hình Hibernate JPA Metamodel Generator trong build tool và luôn dùng `User_.name`, `User_.age`, v.v.

---

## 12. Sample Project

**Dự án: API tìm kiếm sản phẩm cho e-commerce**

Constraint cứng:
- Phải hỗ trợ ít nhất 8 filter đồng thời: category, price range, brand, rating minimum, in-stock only, keyword (LIKE trên tên sản phẩm), created date range, seller ID.
- Phải trả về kết quả phân trang với tổng số record chính xác (total elements).
- Không được dùng QueryDSL hoặc native SQL.
- Mỗi filter là optional — query phải hoạt động đúng khi bất kỳ tổ hợp filter nào được bỏ trống.
- Filter price range chỉ áp dụng khi cả min và max đều được cung cấp.

Cách implement:
1. Tạo entity `Product` với các field tương ứng và generate static metamodel `Product_`.
2. Tạo `ProductSearchFilter` POJO chứa tất cả filter params, mỗi field nullable.
3. Implement `ProductSpecifications` với mỗi static method trả về một `Specification<Product>`.
4. Repository extends `JpaRepository<Product, Long>` và `JpaSpecificationExecutor<Product>`.
5. Service compose các specification và gọi `productRepository.findAll(spec, pageable)`.
6. Viết integration test với `@DataJpaTest` cho từng tổ hợp filter để bảo đảm đúng kết quả.

---

## 13. Interview

### Core Q&A

**Q: Các thành phần chính của Criteria API và vai trò của từng thành phần?**
A: `CriteriaBuilder` được lấy từ `EntityManager.getCriteriaBuilder()`, là factory tạo tất cả expression và predicate. `CriteriaQuery<T>` định nghĩa cấu trúc toàn bộ query (SELECT, FROM, WHERE, ORDER BY). `Root<T>` tương đương mệnh đề FROM, là điểm neo để navigate tới các field và join. `Predicate` là biểu thức boolean tương đương một điều kiện trong WHERE clause.

**Q: Root, Join, và Path khác nhau như thế nào?**
A: `Path<X>` là interface cơ sở đại diện cho việc navigate đến một attribute — `root.get(User_.name)` trả về `Path<String>`. `Root<X>` extend `Path`, đại diện cho entity chính trong FROM. `Join<X,Y>` cũng extend `Path`, là kết quả của `root.join()`. Nói cách khác, `Root` và `Join` là những loại `Path` đặc biệt — tất cả đều dùng được trong `cb.equal()`, `cb.like()`, v.v.

**Q: Static Metamodel là gì và tại sao nên dùng?**
A: Là các class Java được generate tự động (ví dụ `User_`) chứa các `SingularAttribute` và `CollectionAttribute` tương ứng với từng field của entity. Dùng `User_.name` thay vì string `"name"` để compiler kiểm tra tên field, IDE có autocomplete, refactor tự động khi đổi tên field.

**Q: Làm thế nào để tránh N+1 problem khi dùng Criteria API?**
A: Dùng `root.fetch(User_.roles, JoinType.LEFT)` để yêu cầu Hibernate JOIN FETCH trong cùng một SQL. Lưu ý quan trọng: không gọi fetch trong count query. Hoặc dùng `@EntityGraph` trên repository method để khai báo fetch plan mà không cần viết Criteria.

**Q: Tại sao cần tách count query và data query khi phân trang?**
A: Vì fetch join và COUNT không tương thích — Hibernate không cho phép ORDER BY trong COUNT query và fetch join có thể làm COUNT sai do duplicate. Tách riêng đảm bảo mỗi query đúng mục đích, data query dùng fetch join để tránh N+1, count query chỉ đếm entity chính.

**Q: cb.and(predicates) nhận array khác gì cb.and(p1, p2) nhận varargs?**
A: Về SQL cuối cùng là như nhau. Về code, `cb.and(Predicate[])` tiện hơn khi xây list predicate động (dùng với `new ArrayList<>()` rồi `toArray()`). `cb.and(p1, p2)` tiện hơn khi biết trước số predicate cố định.

**Q: Spring Data Specification liên quan đến Criteria API như thế nào?**
A: `Specification<T>` là functional interface với method `toPredicate(Root<T>, CriteriaQuery<?>, CriteriaBuilder)` — chính là Criteria API. `SimpleJpaRepository` gọi `toPredicate()` để lấy `Predicate` và truyền vào `CriteriaQuery`. Specification chỉ là pattern wrapper giúp compose (`and`, `or`, `not`) gọn gàng hơn, không thêm gì mới về mặt JPA.

**Q: Khi nào nên dùng QueryDSL thay vi Criteria API?**
A: QueryDSL khi: team muốn fluent API dễ đọc hơn (`.from(QUser.user).where(user.name.eq("foo"))`), query phức tạp nhiều join, hoặc cần sử dụng các biểu thức như subquery complex. Criteria API khi: muốn zero extra dependency, project đã dùng Spring Data Specification, hoặc chỉ cần dynamic filter đơn giản.

### Scenario

**Scenario 1**: Team có query tìm kiếm user với 10 optional filter. Junior dev đề xuất dùng `@Query` JPQL với điều kiện `(:name IS NULL OR u.name = :name)`. Bạn có nhận xét gì?

Trả lời: Cách đó hoạt động nhưng có hai vấn đề: (1) Optimizer của một số database (MySQL đặc biệt) khó optimize tốt các điều kiện IS NULL check phức tạp khi có nhiều tham số, (2) Mỗi khi thêm filter mới phải sửa chuỗi JPQL để đấu runtime error. Với 10 filter, Criteria API + Specification sẽ maintainable hơn: mỗi filter là một method riêng, dễ test độc lập, dễ thêm/bỏ filter mà không ảnh hưởng đến các điều kiện khác.

**Scenario 2**: Bạn dùng Criteria API JOIN vào bảng `orders` (OneToMany). Kết quả trả về 150 records nhưng trong database chỉ có 50 user thỏa điều kiện. Tại sao?

Trả lời: Đây là Cartesian product do JOIN với collection mà không dùng DISTINCT. Mỗi user có trung bình 3 orders nên 50 users x 3 = 150 rows. Fix: Thêm `cq.distinct(true)` hoặc thay `root.join()` bằng `root.fetch()` nếu mục đích là load data cùng entity.

**Scenario 3**: Bạn dùng `root.fetch(User_.roles)` trong Specification. Khi gọi `userRepository.count(spec)`, Spring Data JPA ném exception. Tại sao và fix thế nào?

Trả lời: `count()` tạo `CriteriaQuery<Long>` và Hibernate không cho phép fetch join trong COUNT query (không ambiguity với aggregate). Fix: kiểm tra `query.getResultType()` trong Specification trước khi fetch:
```java
if (!Long.class.equals(query.getResultType())) {
    root.fetch(User_.roles, JoinType.LEFT);
    query.distinct(true);
}
```

**Scenario 4**: Query Criteria API chạy đúng trên H2 (test) nhưng chậm trên PostgreSQL production với 5 triệu record. Bước đầu tiên debug là gi?

Trả lời: Bật `spring.jpa.show-sql=true` để lấy SQL thực tế. Copy SQL đó vào PostgreSQL và chạy `EXPLAIN (ANALYZE, BUFFERS)`. Tìm Sequential Scan thay vì Index Scan. Kiểm tra các column trong WHERE clause đã có index chưa. Nếu dùng LIKE với `%` ở đầu (`%keyword%`) thì index B-tree không được dùng — cần xem xét GIN index cho full-text search. Kiểm tra xem DISTINCT có làm chậm hash aggregation không.

**Scenario 5**: Làm thế nào viết unit test cho một Specification mà không cần Spring context?

Trả lời: Có hai cách thực tế: (1) Dùng `@DataJpaTest` với H2 in-memory — không phải unit test thuần nhưng nhanh, kiểm tra được SQL output cuối cùng. (2) Mock `CriteriaBuilder`, `Root`, `CriteriaQuery` bằng Mockito và verify `toPredicate()` gọi đúng method với đúng argument. Cách (1) thực tế hơn vì đảm bảo cả behavior Hibernate generate đúng SQL mong muốn.

---

## 14. References

- JPA 3.1 Specification — Criteria API chapter: https://jakarta.ee/specifications/persistence/3.1/
- Hibernate ORM Documentation — Criteria: https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/Hibernate_User_Guide.html#criteria
- Spring Data JPA Reference — Specifications: https://docs.spring.io/spring-data/jpa/reference/jpa/specifications.html
- Hibernate JPA Metamodel Generator: https://hibernate.org/orm/tooling/
- Baeldung — Introduction to JPA Criteria API: https://www.baeldung.com/hibernate-criteria-queries
- Baeldung — Spring Data JPA Specifications: https://www.baeldung.com/rest-api-search-language-spring-data-specifications
- Vlad Mihalcea — The best way to use JPA Criteria API: https://vladmihalcea.com/jpa-criteria-api/
- Thorben Janssen — Hibernate Tips: Criteria Queries: https://thorben-janssen.com/hibernate-tips-criteria-queries/

---

## 15. Real-world Code

- Spring Data JPA source — `SimpleJpaRepository` (xem cách Spring tích hợp Specification vào CriteriaQuery): https://github.com/spring-projects/spring-data-jpa/blob/main/spring-data-jpa/src/main/java/org/springframework/data/jpa/repository/support/SimpleJpaRepository.java
- JHipster generator — generated Criteria/Filter/Specification classes dùng làm reference cho production pattern: https://github.com/jhipster/generator-jhipster/tree/main/generators/server/templates/src/main/java/package/service/criteria
- Keycloak — dùng Criteria API intensively để query user, realm, client trong `model/jpa`: https://github.com/keycloak/keycloak/tree/main/model/jpa/src/main/java/org/keycloak/models/jpa
- Spring PetClinic — minimal reference app Spring Boot + JPA: https://github.com/spring-projects/spring-petclinic

---

## 16. Community

- Stack Overflow — "Spring Data Specifications with fetch join causes count query issue": https://stackoverflow.com/questions/27857120/how-to-avoid-fetch-join-in-count-query-for-spring-data-jpa-specification
- Stack Overflow — "JPA Criteria API: how to use IN expression": https://stackoverflow.com/questions/4768063/criteria-api-in-expression
- Stack Overflow — "Avoid N+1 in JPA Criteria API": https://stackoverflow.com/questions/6994137/jpa-criteria-api-avoid-n1-select
- Stack Overflow — "JPA Criteria API distinct with fetch join": https://stackoverflow.com/questions/22893903/jpa-criteria-api-using-distinct-with-fetch-join
- Reddit r/java — discussion on QueryDSL vs Criteria API in modern Spring projects: https://www.reddit.com/r/java/comments/querydsl_vs_criteria_api
- Vlad Mihalcea blog — "JPA Criteria API Bulk Update and Delete": https://vladmihalcea.com/jpa-criteria-api-bulk-update-delete/
- Thorben Janssen blog — "5 things you need to know when using JPA Criteria API": https://thorben-janssen.com/5-things-you-need-to-know-when-using-jpa-criteria-api/
