---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/async"
related:
  - "[[Transient]]"
---

## 1. What
`AtomicInteger` là một lớp trong gói `java.util.concurrent.atomic` cung cấp một giá trị kiểu `int` có thể được cập nhật một cách nguyên tử (atomically). Nó được thiết kế để sử dụng trong các môi trường đa luồng (multi-threaded) mà không cần sử dụng từ khóa `synchronized`.

## 2. Why
Trong môi trường đa luồng, các thao tác đơn giản như `count++` thực chất gồm 3 bước: đọc giá trị, tăng giá trị, và ghi lại giá trị. Nếu hai luồng cùng thực hiện đồng thời, có thể dẫn đến hiện tượng "Lost Update" (một luồng ghi đè lên kết quả của luồng kia). `AtomicInteger` giải quyết vấn đề này bằng cách sử dụng các chỉ thị CPU đặc biệt để đảm bảo thao tác cập nhật là duy nhất và không bị ngắt quãng.

## 3. Mental Model
Hãy tưởng tượng một **"Bảng số thứ tự (Counter)"** ở ngân hàng. Thay vì để mọi người tự viết số tiếp theo lên bảng (dễ gây nhầm lẫn), cái bảng này có một cơ chế **"Chốt an toàn"**. Khi bạn muốn tăng số, bạn phải nhìn số hiện tại (ví dụ: 5), tính số mới (6), và nói với bảng: "Nếu số hiện tại vẫn là 5, hãy đổi nó thành 6". Nếu trong lúc bạn tính, có người khác đã đổi nó thành 6, bảng sẽ từ chối yêu cầu của bạn và bạn phải làm lại. Đây gọi là cơ chế CAS (Compare-And-Swap).

## 4. Where it fits
Nó nằm ở tầng Low-level Concurrency, cung cấp giải pháp cho các biến chia sẻ (shared variables):
`Multiple Threads -> AtomicInteger (CAS Operations) -> Main Memory`

## 5. When to use
- Dùng làm bộ đếm (counter) trong các ứng dụng đa luồng (ví dụ: đếm số lượng request, số lượng user online).
- Dùng để tạo ra các ID duy nhất (Sequence generator) mà không muốn bị bottleneck bởi `synchronized`.
- Dùng trong các thuật toán Non-blocking.

## 6. When NOT to use
- Khi bạn cần thực hiện nhiều thao tác phức tạp trên nhiều biến khác nhau một cách nguyên tử (trường hợp này nên dùng `Lock` hoặc `synchronized`).
- Khi chỉ có một luồng duy nhất truy cập vào biến (dùng `int` thường sẽ nhanh hơn).
- Khi giá trị bị cập nhật quá thường xuyên bởi cực kỳ nhiều luồng (High contention), dẫn đến việc các luồng phải thử lại (retry) quá nhiều lần gây lãng phí CPU. Trong trường hợp này, `LongAdder` sẽ hiệu quả hơn.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Hiệu năng cực cao vì không gây "block" luồng (Non-blocking). | Có thể gây lãng phí CPU (Spin-wait) nếu có quá nhiều luồng tranh chấp. |
| Tránh được các vấn đề Deadlock thường gặp khi dùng lock. | Chỉ bảo vệ được một biến đơn lẻ. |
| Code ngắn gọn, dễ đọc hơn so với dùng try-finally lock. | Gặp vấn đề ABA (mặc dù hiếm với Integer). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `synchronized` | An toàn nhưng chậm hơn do luồng phải đợi nhau (blocking). |
| `LongAdder` | Tốt hơn AtomicInteger/AtomicLong trong trường hợp tranh chấp cực cao. |
| `Volatile int` | Chỉ đảm bảo tính hiển thị (visibility), không đảm bảo tính nguyên tử (atomicity) cho phép toán `++`. |

## 9. How
```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounterExample {
    private static AtomicInteger counter = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) counter.incrementAndGet();
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) counter.incrementAndGet();
        });

        t1.start(); t2.start();
        t1.join(); t2.join();

        System.out.println("Final count: " + counter.get()); // Luôn là 2000
    }
}
```

## 10. Production concerns
### Scaling
Trong các hệ thống phân tán, `AtomicInteger` chỉ có tác dụng trong phạm vi một JVM. Nếu cần đếm trên toàn hệ thống, bạn phải dùng Redis (INCR) hoặc Database.

### Failure
Cơ chế CAS có thể thất bại liên tục nếu có một luồng khác luôn cập nhật giá trị nhanh hơn. Đây gọi là "Live-lock" nhẹ.

### Monitoring
Theo dõi tỉ lệ CAS failure có thể giúp nhận diện các điểm nóng (hotspots) về tranh chấp tài nguyên trong ứng dụng.

## 11. Common mistakes
- **Mistake**: Sử dụng `AtomicInteger` nhưng vẫn thực hiện các lệnh logic bên ngoài không nguyên tử.
  ```java
  // SAI: check-then-act bị hổng
  if (atomicInt.get() == 5) {
      atomicInt.set(10); 
  }
  ```
  **Fix**: Sử dụng các phương thức nguyên tử có sẵn: `atomicInt.compareAndSet(5, 10)`.

- **Mistake**: Nghĩ rằng `AtomicInteger` thay thế được hoàn toàn `synchronized` cho mọi logic.
  **Fix**: Chỉ dùng cho các biến đơn lẻ.

## 12. Sample project
Xây dựng một lớp `RateLimiter` đơn giản cho phép tối đa 100 requests mỗi giây. Sử dụng `AtomicInteger` để đếm số request và reset nó mỗi giây.
**Ràng buộc**: Không được dùng `synchronized`.

## 13. Interview
### Core Q&A
1. **Q**: Cơ chế CAS (Compare-And-Swap) là gì?
   **A**: Là một kỹ thuật so sánh giá trị hiện tại của biến với một giá trị kỳ vọng. Nếu khớp, nó sẽ cập nhật biến thành giá trị mới. Thao tác này được thực hiện ở cấp độ phần cứng (atomic instruction).
2. **Q**: Tại sao `incrementAndGet()` lại an toàn hơn `i++`?
   **A**: Vì `i++` là thao tác 3 bước (read-modify-write), còn `incrementAndGet()` thực hiện vòng lặp CAS đảm bảo giá trị không bị thay đổi bởi luồng khác trong quá trình tính toán.
3. **Q**: Vấn đề ABA là gì và `AtomicInteger` có bị ảnh hưởng không?
   **A**: ABA là khi giá trị thay đổi từ A -> B -> A, khiến CAS tưởng rằng giá trị chưa hề thay đổi. Với Integer thường không quan trọng, nhưng với các cấu trúc dữ liệu dựa trên Node/Pointer thì rất nguy hiểm.

### Scenario
**Tình huống**: Bạn cần đếm tổng số lượng lỗi xảy ra trong toàn bộ hệ thống Microservice khi xử lý một batch lớn. Bạn sẽ dùng gì?
**Trả lời**: Nếu chỉ đếm trong một instance, tôi dùng `LongAdder` (hiệu quả hơn `AtomicInteger` cho việc ghi dồn dập). Nếu cần tổng toàn hệ thống, tôi dùng một shared counter trên Redis.

## 14. References
- Official Docs: [https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/atomic/AtomicInteger.html](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/atomic/AtomicInteger.html)
- GitHub Repo: OpenJDK `AtomicInteger.java` source code.

## 15. Real-world Code
- Được dùng trong `ThreadPoolExecutor` để theo dõi số lượng thread đang chạy.
- Dùng trong các thư viện Metrics như Micrometer, Dropwizard.

## 16. Community
- Reddit: r/java
- Stack Overflow: Tag [atomic-integer]
- Blog: Jenkov's Java Concurrency.
