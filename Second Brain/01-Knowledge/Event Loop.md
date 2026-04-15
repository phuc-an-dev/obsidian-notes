---
created: 2026-04-15
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/javascript"
related: "[[WebClient in Spring Boot]]"
---

## 1. What
**Event Loop** là cơ chế cốt lõi cho phép JavaScript thực hiện các thao tác non-blocking I/O mặc dù nó là một ngôn ngữ đơn luồng (single-threaded). Nó liên tục kiểm tra Call Stack và Callback Queue để quyết định khi nào nên đưa một hàm vào thực thi.

## 2. Why (problem it solves)
- Giải quyết vấn đề "treo" giao diện (UI blocking): Nếu không có Event Loop, khi gọi một API mất 5 giây, toàn bộ trình duyệt sẽ bị đơ cho đến khi có kết quả.
- Tận dụng tối đa tài nguyên: Cho phép CPU xử lý các tác vụ khác trong khi đợi các thao tác I/O (đọc file, gọi mạng, truy vấn DB) hoàn tất.
- Khả năng mở rộng (Scalability): Giúp Node.js xử lý hàng ngàn kết nối đồng thời chỉ với một luồng chính duy nhất.

## 3. Mental Model
> "Hãy tưởng tượng Event Loop như một **Người phục vụ bàn (Waiter)** trong nhà hàng.
> 1. Bạn gọi món (`Request`).
> 2. Người phục vụ ghi order và đưa vào bếp (`Call Stack -> Web APIs`).
> 3. Thay vì đứng đợi ở cửa bếp, người phục vụ đi lấy order của bàn khác (`Non-blocking`).
> 4. Khi món ăn xong, đầu bếp bấm chuông (`Callback Queue`).
> 5. Người phục vụ thấy chuông reo và bàn mình đang trống, anh ta sẽ mang món ăn ra cho bạn (`Event Loop`)."

## 4. Where it fits (architecture)
`JS Engine (Call Stack) ↔ [Event Loop] ↔ [Callback Queue / Microtask Queue] ↔ Web APIs (Browser) / Libuv (Node.js)`

## 5. When to use
- Luôn luôn hiện diện trong mọi ứng dụng JavaScript (Browser & Node.js).
- Cần hiểu rõ khi xử lý các tác vụ bất đồng bộ: `setTimeout`, `Promises`, `async/await`, `fetch`, `fs.readFile`.

## 6. When NOT to use
- Event Loop **không phù hợp** cho các tác vụ tính toán nặng (CPU Intensive) như xử lý ảnh, giải mã video, hoặc tính toán ma trận lớn. Vì nó sẽ chặn luồng chính (Block the Event Loop), khiến mọi yêu cầu khác bị treo.
- Đối với các tác vụ này, nên dùng `Worker Threads` hoặc tách ra một service riêng (Microservice).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tiết kiệm tài nguyên (không cần tạo nhiều thread). | Dễ bị "nghẽn cổ chai" nếu có một tác vụ đồng bộ chạy quá lâu. |
| Code đơn giản hơn (không cần lo lắng về Race Condition giữa các luồng). | Callback Hell (nếu không dùng Promise/Async-Await). |
| Hiệu suất I/O cực cao. | Khó debug luồng thực thi vì tính chất bất đồng bộ. |

## 8. Alternatives (with comparison)
| Option | So sánh |
|--------|---------|
| **Multi-threading (Java/C++)** | Mỗi request một thread. Tốt cho tính toán nặng nhưng tốn RAM và overhead quản lý thread. |
| **Worker Threads (JS)** | Giải pháp bổ trợ cho Event Loop để xử lý CPU Intensive Task trên một thread riêng biệt. |

## 9. How (minimal example)
```javascript
console.log("1. Bắt đầu"); // Chạy ngay (Call Stack)

setTimeout(() => {
    console.log("2. Timeout 0ms"); // Đưa vào Macrotask Queue
}, 0);

Promise.resolve().then(() => {
    console.log("3. Promise (Microtask)"); // Đưa vào Microtask Queue
});

console.log("4. Kết thúc"); // Chạy ngay (Call Stack)

// Thứ tự in ra: 1 -> 4 -> 3 -> 2
// Giải thích: Call Stack chạy hết -> Microtask (Promise) chạy -> Macrotask (Timeout) chạy.
```

## 10. Production concerns
### Scaling
- Trong Node.js, sử dụng module `Cluster` để tận dụng tối đa các core CPU (chạy nhiều Event Loop trên các process khác nhau).
### Failure
- Nếu Event Loop bị chặn (`Event Loop Lag`), latency của ứng dụng sẽ tăng vọt. Cần monitor chỉ số này chặt chẽ.
### Monitoring
- Sử dụng các công cụ như `clinic.js` hoặc `Prometheus` để đo lường Event Loop Lag và thời gian xử lý của các task.

## 11. Common mistakes / anti-patterns
- ❌ **Mistake**: Viết vòng lặp vô tận hoặc đệ quy quá sâu (Synchronous blocking).
  ✅ **Fix**: Chia nhỏ task lớn thành các task nhỏ bằng `setImmediate` hoặc `process.nextTick`.
- ❌ **Mistake**: Hiểu lầm rằng `setTimeout(fn, 0)` sẽ chạy ngay lập tức.
  ✅ **Fix**: Nó sẽ chạy sau khi Call Stack và Microtasks trống.

## 12. Sample project (with constraint)
**Tên project**: "Non-blocking Log Processor"
**Constraint**: Đọc một file log 1GB và đếm số dòng lỗi (`ERROR`). Không được làm treo server (vẫn phải phản hồi được các request HTTP khác trong lúc đọc).
**Output**: Sử dụng `Streams` và `Event Loop` để đọc từng chunk của file thay vì load toàn bộ vào memory.

## 13. Interview
### Core Q&A
1. **Q**: Sự khác biệt giữa Microtask và Macrotask là gì?
   **A**: Microtask (Promises, process.nextTick) có ưu tiên cao hơn. Event Loop sẽ xử lý **tất cả** Microtasks hiện có trước khi chuyển sang Macrotask (setTimeout, setInterval, I/O) tiếp theo.
2. **Q**: Điều gì xảy ra nếu Call Stack không bao giờ trống?
   **A**: Event Loop sẽ bị treo, các callback trong queue sẽ không bao giờ được thực thi, ứng dụng sẽ bị "đơ".
### Scenario
> "Tình huống: Server Node.js của bạn có một API thực hiện vòng lặp `for` chạy mất 10 giây. Chuyện gì xảy ra với các user khác đang truy cập vào trang web?"
**A**: Tất cả các user khác sẽ bị treo hoàn toàn và không nhận được phản hồi cho đến khi vòng lặp `for` đó kết thúc, vì Event Loop đã bị chặn bởi luồng chính duy nhất.
