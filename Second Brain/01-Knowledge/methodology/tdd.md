---
created: 2026-04-21
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/java"
  - "#topic/performance"
related:
  - "[[ddd.md]]"
  - "[[Reproduce.md]]"
---

# Test-Driven Development (TDD)

## 1. What
Test-Driven Development (TDD) là một quy trình phát triển phần mềm trong đó các unit test được viết **trước** khi viết code chức năng. Quy trình TDD tuân thủ chặt chẽ vòng lặp "Red-Green-Refactor": viết một test thất bại (Red), viết code vừa đủ để vượt qua test (Green), và tối ưu hóa code (Refactor).

## 2. Why
Viết code xong mới viết test thường dẫn đến tâm lý "viết test cho có" hoặc test không bao quát hết các trường hợp biên. TDD buộc lập trình viên phải suy nghĩ về thiết kế, interface và yêu cầu nghiệp vụ trước khi implement, giúp code dễ test hơn, giảm bớt lỗi logic và tạo sự tự tin khi thực hiện refactor.

## 3. Mental Model
Hãy tưởng tượng TDD giống như một người thợ may đo quần áo:
- **Red:** Vẽ phác thảo trên vải và thử khoác lên người (chưa may hoàn thiện, không vừa).
- **Green:** May đường chỉ đầu tiên để bộ đồ có thể mặc vừa (dù chưa đẹp, nhưng đã mặc được).
- **Refactor:** Cắt gọt, là ủi, chỉnh sửa cho bộ đồ vừa vặn, đẹp và bền hơn mà vẫn đảm bảo nó mặc vừa (test vẫn pass).

## 4. Where it fits
```
[Red: Write failing test]
        |
        v
[Green: Write minimal code to pass]
        |
        v
[Refactor: Improve code quality]
        |
        v
[Repeat]
```

## 5. When to use
- Khi bắt đầu một tính năng mới có logic nghiệp vụ phức tạp.
- Khi làm việc trong các hệ thống đòi hỏi độ ổn định cao (Core banking, API platform).
- Khi bạn cần refactor mã nguồn cũ (legacy code) mà không muốn làm hỏng chức năng hiện tại.

## 6. When NOT to use
- Các dự án thử nghiệm (PoC) cần tốc độ cực nhanh.
- Khi bạn chưa rõ requirements hoặc logic nghiệp vụ còn thay đổi liên tục hàng giờ.
- Các UI code hoặc code tích hợp (integration) quá phức tạp để unit test (khi đó nên dùng BDD hoặc Integration Test).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Code có tính modular cao, dễ test | Tốn thời gian hơn trong giai đoạn đầu |
| Giảm tỷ lệ bug trong production | Learning curve cho người mới |
| Documentation sống động qua test | Dễ bị quá tập trung vào "test pass" mà quên thiết kế tổng thể |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Test-Last | Viết code trước, test sau. Dễ code hơn nhưng khó phát hiện lỗi thiết kế |
| BDD (Behavior Driven) | Viết test bằng ngôn ngữ tự nhiên, tập trung vào hành vi (Gherkin syntax) |

## 9. How
```java
// 1. Red: Viết test cho phương thức tính thuế
@Test
void shouldCalculateTaxCorrectly() {
    TaxCalculator calculator = new TaxCalculator();
    assertEquals(10.0, calculator.calculate(100.0, 0.1));
}

// 2. Green: Viết code tối giản
public class TaxCalculator {
    public double calculate(double amount, double rate) {
        return amount * rate;
    }
}

// 3. Refactor: Nếu cần, chuyển đổi logic phức tạp hơn (ví dụ dùng Strategy pattern)
```

## 10. Production concerns
### Test Suite Speed
Nếu bộ test chạy mất 30 phút, lập trình viên sẽ lười chạy test. TDD đòi hỏi bộ unit test phải cực nhanh (< 5 giây).

### Maintenance
Test là code, và test cũng cần được bảo trì. Nếu viết test quá chi tiết (mocking quá mức), khi thay đổi cấu trúc code, test sẽ hỏng hàng loạt.

## 11. Common mistakes
- Mistake: Viết quá nhiều test cùng một lúc.
  Fix: Chỉ viết test cho một hành vi nhỏ nhất có thể (Atomic test).
- Mistake: Bỏ qua bước "Refactor".
  Fix: Refactor là bước quan trọng nhất để giữ code sạch. Nếu không làm, codebase sẽ tích tụ "nợ kỹ thuật" dù test pass.

## 12. Sample project
**Tên: TDD Calculator**
Viết một máy tính cộng/trừ/nhân/chia.
- Yêu cầu: Mỗi method phải có test case cho trường hợp bình thường, trường hợp biên (chia cho 0), và trường hợp giá trị âm. Không được viết code nếu chưa có test thất bại.

## 13. Interview
### Core Q&A
1. Q: Làm sao để biết một test case là tốt?
   A: Phải tuân thủ tiêu chuẩn F.I.R.S.T (Fast, Independent, Repeatable, Self-validating, Timely).
2. Q: TDD có thay thế được integration test không?
   A: Không. TDD tập trung vào unit/logic. Integration test vẫn cần thiết để đảm bảo các thành phần làm việc tốt với nhau.

### Scenario
**Tình huống:** Codebase hiện tại là legacy, không có test. Bạn muốn áp dụng TDD nhưng không biết bắt đầu từ đâu. Bạn làm gì?
**Giải đáp:** Sử dụng "Golden Master" (Characterization testing): Viết test dựa trên hành vi hiện tại của code, dù code đó xấu, để tạo một lớp bảo vệ. Sau đó, khi cần sửa đổi hoặc thêm tính năng, áp dụng TDD trên các phần code mới hoặc refactor dần dần.

## 14. References
- Kent Beck - Test Driven Development: By Example
- Clean Code (Robert C. Martin)
- Martin Fowler - TDD: https://martinfowler.com/bliki/TestDrivenDevelopment.html

## 15. Real-world Code
- Spring Framework unit tests (mẫu mực về TDD).
- JUnit 5 Examples: https://github.com/junit-team/junit5-samples

## 16. Community
- TDD Community: https://tdd.nyc/
- Stack Overflow - TDD tag
