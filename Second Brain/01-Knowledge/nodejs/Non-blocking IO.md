---
created: 2026-04-15
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
related:
  - "[[Event Loop.md]]"
---

## 1. What
**Non-blocking I/O (NIO)** là một mô hình xử lý vào/ra (Input/Output) mà trong đó một lời gọi hàm không bắt luồng (thread) hiện tại phải đợi cho đến khi thao tác I/O hoàn tất. Thay vào đó, nó trả về ngay lập tức với trạng thái hiện tại, cho phép luồng đó tiếp tục làm việc khác.

## 2. Why (problem it solves)
- **Cạn kiệt tài nguyên**: Trong mô hình Blocking I/O truyền thống, mỗi kết nối cần 1 thread. Nếu có 10,000 user đang đợi dữ liệu từ DB, server cần 10,000 thread -> tốn cực kỳ nhiều RAM (mỗi thread chiếm ~1MB).
- **Lãng phí CPU**: Thread ở trạng thái Blocking (Idle) không làm gì cả nhưng vẫn chiếm tài nguyên hệ thống và gây ra overhead khi Context Switch.
- **Latency**: Giảm thời gian chờ đợi tổng thể của hệ thống khi phải giao tiếp với các thành phần chậm hơn (Disk, Network).

## 3. Mental Model
> "Hãy tưởng tượng bạn đi mua trà sữa:
> - **Blocking I/O**: Bạn đứng tại quầy, nhìn chằm chằm vào nhân viên làm trà sữa và không làm gì khác cho đến khi nhận được cốc trà. Những người sau bạn phải đợi bạn rời đi mới được order.
> - **Non-blocking I/O**: Bạn order xong, nhân viên đưa cho bạn một cái **Thẻ rung (Token)**. Bạn quay ra chỗ khác lướt điện thoại hoặc làm việc. Khi trà xong, thẻ rung lên và bạn quay lại lấy. Trong lúc bạn đợi, nhân viên vẫn có thể phục vụ thêm hàng chục người khác."

## 4. Where it fits (architecture)
`Application → [NIO Library (Netty/Java NIO)] → [OS Kernel (Epoll/Kqueue)] → Hardware (Network Card/Disk)`

## 5. When to use
- Hệ thống cần xử lý hàng ngàn kết nối đồng thời (High Concurrency) như Chat app, Real-time Dashboard, Gateway.
- Các ứng dụng Network-intensive (giao tiếp mạng nhiều).
- Khi xây dựng Microservices Gateway hoặc Proxy.

## 6. When NOT to use
- Ứng dụng đơn giản, lượng người dùng thấp (Blocking I/O code dễ viết và debug hơn nhiều).
- Tác vụ xử lý dữ liệu nặng trên CPU (CPU-intensive): Non-blocking I/O không giúp ích gì cho tốc độ tính toán của CPU.
- Khi làm việc với các thư viện cũ không hỗ trợ Non-blocking (ví dụ: JDBC truyền thống).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| **Scalability**: Xử lý cực nhiều kết nối với rất ít thread. | **Complexity**: Logic code trở nên phức tạp (Callback, Future, Reactive). |
| **Efficiency**: Tối ưu hóa việc sử dụng RAM và CPU. | **Debugging**: Khó theo dõi stack trace vì code không chạy tuần tự. |
| Phù hợp hoàn hảo cho Microservices. | Nguy cơ bị "block" nếu vô tình gọi một hàm blocking. |

## 8. Alternatives (with comparison)
| Option | So sánh |
|--------|---------|
| **Blocking I/O** | Dễ viết, phù hợp cho logic nghiệp vụ phức tạp, nhưng không scale được. |
| **Asynchronous I/O (AIO)** | Cao cấp hơn NIO, OS sẽ tự làm mọi thứ và báo cho app khi xong hoàn toàn (True Async). |

## 9. How (minimal example - Java NIO style)
```java
// Giả lập Non-blocking check
while (true) {
    // Thử đọc dữ liệu từ Channel
    int bytesRead = socketChannel.read(buffer); 
    
    if (bytesRead > 0) {
        // Có dữ liệu -> Xử lý
        processData(buffer);
        break;
    } else if (bytesRead == 0) {
        // Chưa có dữ liệu -> Làm việc khác thay vì đứng đợi
        doOtherWork(); 
    } else {
        // Connection closed
        break;
    }
}
```

## 10. Production concerns
### Scaling
- Sử dụng cơ chế **Multiplexing** (như `epoll` trên Linux) để một thread có thể theo dõi hàng ngàn socket cùng lúc.
### Failure
- Cần có cơ chế xử lý lỗi khi "Thẻ rung" không bao giờ rung (Timeout handling).
### Monitoring
- Theo dõi số lượng File Descriptors và Thread Pool usage của thư viện NIO (như Netty).

## 11. Common mistakes / anti-patterns
- ❌ **Mistake**: Gọi hàm Blocking (như `Thread.sleep()` hoặc JDBC) bên trong luồng Non-blocking.
  ✅ **Fix**: Chuyển các tác vụ đó sang một Dedicated Thread Pool riêng.
- ❌ **Mistake**: Không xử lý "Backpressure" khi producer gửi dữ liệu quá nhanh khiến consumer bị tràn bộ nhớ.

## 12. Sample project (with constraint)
**Tên project**: "High-speed File Proxy"
**Constraint**: Phải forward dữ liệu từ một API chậm sang một API nhanh mà không được dùng quá 10MB RAM, kể cả khi file nặng 1GB.
**Output**: Sử dụng **Streams/Pipes** với cơ chế Non-blocking để truyền dữ liệu theo từng chunk (mảnh nhỏ).

## 13. Interview
### Core Q&A
1. **Q**: Sự khác biệt lớn nhất giữa NIO và BIO (Blocking I/O) là gì?
   **A**: BIO là luồng (thread) bị chặn cho đến khi có dữ liệu. NIO là luồng gửi yêu cầu và có thể quay lại kiểm tra sau hoặc được thông báo khi có dữ liệu, giúp giải phóng luồng cho các task khác.
2. **Q**: Tại sao Node.js chỉ có 1 thread mà lại xử lý được nhiều request hơn Java (truyền thống)?
   **A**: Vì Node.js sử dụng hoàn toàn Non-blocking I/O kết hợp với Event Loop, trong khi Java truyền thống dùng 1-thread-per-request và bị lãng phí thread vào việc đợi I/O.
### Scenario
> "Tình huống: Hệ thống dùng WebFlux (Non-blocking) của bạn bỗng dưng chậm lại khi bạn thêm tính năng lưu log vào File. Tại sao?"
**A**: Có khả năng thư viện ghi File bạn dùng là Blocking I/O. Khi đó, luồng Event Loop duy nhất bị chặn bởi việc ghi đĩa, khiến toàn bộ request khác phải đợi. Giải pháp là dùng thư viện Async File I/O hoặc đẩy việc ghi log sang một thread pool khác.
