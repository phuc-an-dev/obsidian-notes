---
created: 2026-04-21
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/system-design"
  - "#topic/performance"
related:
  - "[[RESTful API]]"
---

# Domain-Driven Design (DDD)

## 1. What
Domain-Driven Design (DDD) là một cách tiếp cận phát triển phần mềm phức tạp bằng cách tập trung vào "Domain" (miền nghiệp vụ) và logic nghiệp vụ cốt lõi. DDD nhấn mạnh việc xây dựng các mô hình phần mềm phản ánh chính xác các khái niệm và quy tắc trong thế giới thực thông qua sự hợp tác chặt chẽ giữa chuyên gia nghiệp vụ và lập trình viên.

## 2. Why
Khi dự án trở nên phức tạp, việc chỉ tập trung vào CRUD đơn thuần sẽ dẫn đến "Big Ball of Mud" (mã nguồn rối rắm, khó hiểu, khó bảo trì). DDD ra đời để giải quyết vấn đề này bằng cách chia nhỏ hệ thống thành các context rõ ràng, giúp code phản ánh đúng tư duy nghiệp vụ, từ đó dễ mở rộng và thay đổi hơn.

## 3. Mental Model
Hãy tưởng tượng việc xây dựng một hệ thống quản lý thư viện khổng lồ:
- CRUD approach: Bạn chỉ quan tâm đến bảng `Book` và các thuộc tính.
- DDD approach: Bạn xây dựng mô hình dựa trên thực tế: Thư viện gồm `Library`, trong đó có `Book` (cuốn sách cụ thể) và `Title` (đầu sách chung). Bạn định nghĩa quy tắc như "Sách chỉ có thể mượn nếu chưa được mượn" (`CheckOutPolicy`). Code trở thành ngôn ngữ mà cả thủ thư và lập trình viên đều hiểu.

## 4. Where it fits
```
[User/Expertise]
       |
[Ubiquitous Language] (Ngôn ngữ chung)
       |
[Bounded Contexts] (Phân ranh giới)
       |
[Entities / Value Objects / Aggregates]
       |
[Infrastructure Layer]
```

## 5. When to use
- Các hệ thống có nghiệp vụ cực kỳ phức tạp (Ví dụ: Core Banking, hệ thống tính cước viễn thông, thương mại điện tử lớn).
- Khi bạn cần xây dựng hệ thống mà trong đó các quy tắc nghiệp vụ thường xuyên thay đổi hoặc yêu cầu sự hiểu biết sâu sắc về miền nghiệp vụ.
- Các hệ thống Microservices (DDD là nền tảng để chia các Bounded Context thành các Service).

## 6. When NOT to use
- Các dự án CRUD đơn giản (hệ thống blog, landing page, ứng dụng quản lý nhỏ).
- Khi team không có chuyên gia nghiệp vụ để làm việc cùng hoặc không sẵn sàng thay đổi tư duy sang mô hình hóa nghiệp vụ.
- Khi chi phí đầu tư thời gian cho thiết kế không mang lại giá trị tương xứng so với quy mô dự án.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Code phản ánh sát thực tế nghiệp vụ | Learning curve rất dốc |
| Tách biệt code phức tạp khỏi infra | Tốn thời gian thiết kế ban đầu |
| Hỗ trợ tốt cho microservices | Có thể gây over-engineering cho dự án nhỏ |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| CRUD-centric | Đơn giản, nhanh nhưng khó bảo trì khi phức tạp |
| Anemic Domain Model | Tách biệt data và logic, dễ làm nhưng không khai thác sức mạnh object-oriented |

## 9. How
- **Ubiquitous Language**: Thống nhất thuật ngữ giữa team Dev và Business (ví dụ: dùng "Order" chứ không dùng "Invoice" nếu business chỉ gọi là "Order").
- **Bounded Context**: Chia ranh giới, ví dụ trong context "Shipping" thì khái niệm "Order" chỉ là "tập hợp các món cần giao", trong context "Sales" thì "Order" là "hợp đồng giao dịch".
- **Aggregates**: Nhóm các thực thể lại thành một khối (Aggregate Root) để kiểm soát các thay đổi dữ liệu đảm bảo tính toàn vẹn.

## 10. Production concerns
### Complexity
DDD yêu cầu team có kỷ luật cao. Nếu làm sai (ví dụ chia Bounded Context sai), hệ thống sẽ trở nên cực kỳ khó debug vì code bị phân tán.

### Performance
Việc ánh xạ giữa Domain Model và Database (ORM) có thể làm giảm performance nếu không cẩn thận. Cần kết hợp với CQRS (Command Query Responsibility Segregation) để tối ưu hóa việc truy vấn.

## 11. Common mistakes
- Mistake: Dùng một Model duy nhất cho toàn bộ hệ thống (dẫn đến sự cồng kềnh, không Bounded Context).
  Fix: Chia Bounded Context dựa trên ranh giới nghiệp vụ.
- Mistake: Quá chú trọng vào các pattern phức tạp của DDD mà quên mất mục tiêu là giải quyết vấn đề nghiệp vụ.
  Fix: Quay lại triết lý "Domain first".

## 12. Sample project
**Tên: DDD-based E-commerce**
Xây dựng một service "Order Management":
- Aggregate: `Order` là Aggregate Root.
- Rule: Chỉ có thể "Cancel" đơn hàng nếu nó ở trạng thái "Pending".
- Code: Thể hiện logic này trực tiếp trong Entity `Order` (không dùng Service để kiểm tra rồi update setter).

## 13. Interview
### Core Q&A
1. Q: "Ubiquitous Language" là gì?
   A: Là ngôn ngữ chung được sử dụng bởi tất cả các thành viên (Dev, Tester, PM, Business Experts) trong một Bounded Context để mô tả nghiệp vụ mà không bị nhầm lẫn.
2. Q: Sự khác biệt giữa Entity và Value Object?
   A: Entity có định danh (ID) duy nhất và vòng đời thay đổi. Value Object không có ID, được xác định bởi thuộc tính của nó và thường là immutable.

### Scenario
**Tình huống:** Bạn đang xây dựng hệ thống Microservices cho một công ty bán lẻ. Khái niệm "Product" được dùng ở cả service "Inventory" và "Catalog". Bạn có nên chia sẻ chung một class `Product` giữa hai service không?
**Giải đáp:** Không. Trong DDD, mỗi context nên định nghĩa lại khái niệm đó theo nhu cầu riêng. Service "Catalog" cần thuộc tính mô tả (mô tả, ảnh), còn "Inventory" cần thuộc tính vận hành (vị trí kho, số lượng). Chia sẻ class sẽ tạo ra sự phụ thuộc nguy hiểm giữa các services.

## 14. References
- Eric Evans - Domain-Driven Design (Blue Book)
- Vaughn Vernon - Implementing DDD (Red Book)
- Martin Fowler - DDD patterns: https://martinfowler.com/tags/domain%20driven%20design.html

## 15. Real-world Code
- Spring Petclinic (Architecture refactoring examples)
- JHipster (có các blueprint áp dụng DDD/Microservices)

## 16. Community
- DDD Community: https://dddcommunity.org/
- Reddit r/ddd
