---
created: 2026-04-17
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/javascript"
related: "[[WebClient in Spring Boot]]"
---
## 1. What
`Axios` là một thư viện HTTP client dựa trên **Promise**, có thể chạy được trên cả trình duyệt (Browser) và môi trường máy chủ (Node.js). Nó cung cấp một API đơn giản để gửi các request (GET, POST, ...) và xử lý dữ liệu trả về từ máy chủ.

## 2. Why
Trước khi Axios ra đời và Fetch API trở nên phổ biến, việc gọi API thường dùng `XMLHttpRequest` rất phức tạp. Axios giải quyết các vấn đề:
- **Automatic JSON transformation**: Tự động chuyển đổi dữ liệu sang JSON (Fetch yêu cầu gọi `.json()`).
- **Interceptors**: Cho phép can thiệp vào request hoặc response trước khi chúng được xử lý (rất hữu ích để gắn Token hoặc xử lý lỗi tập trung).
- **Isomorphic**: Chạy tốt trên cả client và server mà không cần thay đổi code.
- **Request/Response transformation**: Tùy biến dữ liệu trước khi gửi đi hoặc sau khi nhận về.

## 3. Mental Model
> "Hãy coi Axios như một **'Đội ngũ giao hàng chuyên nghiệp'**. 
> - Với Fetch (mặc định), bạn tự mình đi gửi thư. 
> - Với Axios, bạn đưa thư cho đội ngũ này. Họ sẽ tự động bọc thư vào phong bì đẹp (JSON), kiểm tra xem bạn đã dán tem chưa (Auth Token), và nếu trên đường về họ thấy thư bị rách (Lỗi), họ sẽ tự xử lý hoặc báo lại cho bạn một cách rõ ràng theo quy chuẩn."

## 4. Where it fits
`UI Layer (React/Vue) → Axios (Service Layer) → Network → Backend Server`

## 5. When to use
- Trong hầu hết các ứng dụng web cần gọi REST API.
- Khi cần quản lý tập trung các cấu hình như `baseURL`, `timeout`, `headers`.
- Khi cần xử lý logic đăng nhập/refresh token tự động thông qua Interceptors.
- Khi làm việc trong các dự án lớn cần sự ổn định và hỗ trợ trình duyệt cũ.

## 6. When NOT to use
- Trong các ứng dụng siêu nhỏ, không muốn cài thêm thư viện (Fetch API có sẵn là đủ).
- Khi bạn làm việc với GraphQL (nên dùng Apollo Client hoặc Urql).
- Nếu bundle size là ưu tiên hàng đầu và bạn chỉ cần gọi 1-2 API đơn giản.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| API cực kỳ dễ dùng và tường minh. | Thêm ~4.5kb vào bundle size. |
| Hỗ trợ Interceptors mạnh mẽ. | Là một thư viện bên thứ ba, cần quản lý cập nhật. |
| Tự động xử lý lỗi (4xx, 5xx sẽ throw error). | - |

## 8. Alternatives
- **Fetch API**: Có sẵn trong trình duyệt, không cần cài đặt nhưng ít tính năng hơn.
- **Ky**: Một wrapper nhẹ của Fetch API dành cho trình duyệt.
- **Got**: HTTP client mạnh mẽ dành riêng cho Node.js.

## 9. How (Minimal Example)
```javascript
import axios from 'axios';

// 1. Tạo instance để dùng chung
const api = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 5000,
});

// 2. Sử dụng Interceptor (Gắn Token)
api.interceptors.request.use((config) => {
  config.headers.Authorization = `Bearer ${localStorage.getItem('token')}`;
  return config;
});

// 3. Gọi API
async function getUser() {
  try {
    const response = await api.get('/user/1');
    console.log(response.data); // Axios tự động chuyển sang object
  } catch (error) {
    console.error(error.response?.status); // Xử lý lỗi dễ dàng
  }
}
```

## 10. Production concerns
- **Timeout**: Luôn luôn đặt `timeout` để tránh việc request bị treo vô hạn làm ảnh hưởng đến trải nghiệm người dùng.
- **Error Handling**: Sử dụng một Interceptor cho Response để bắt các lỗi 401 (Hết hạn token) và chuyển hướng người dùng về trang Login một cách tập trung.
- **Security**: Không lưu các thông tin nhạy cảm vào `axios.defaults` vì nó có thể bị tấn công XSS truy cập.

## 11. Common mistakes
- ❌ **Mistake**: Không kiểm tra `error.response` dẫn đến lỗi `undefined` khi server không phản hồi (Network Error).
  ✅ **Fix**: Luôn kiểm tra sự tồn tại của `error.response` trước khi truy cập status code.
- ❌ **Mistake**: Tạo instance mới của axios bên trong mỗi component.
  ✅ **Fix**: Tạo một file `api.js` duy nhất chứa instance dùng chung cho toàn bộ app.

## 12. Sample project
**Tên project**: Auth-Ready API Client.
**Constraint**: Phải có cơ chế tự động thử lại (Retry) request 2 lần nếu gặp lỗi 5xx.
**Yêu cầu**: 
- Gắn Header `X-Request-Id` cho mọi request để dễ dàng tracking.
- Nếu server trả về lỗi 401, tự động gọi API `/refresh-token` một lần duy nhất trước khi báo lỗi cho UI.

## 13. Interview
### Core Q&A
1. Q: Axios khác gì so với Fetch API?
   A: Axios hỗ trợ trình duyệt cũ tốt hơn, tự động chuyển đổi JSON, hỗ trợ Interceptors, có khả năng hủy (cancel) request và tự động throw error khi gặp mã lỗi 4xx/5xx.
2. Q: Interceptor là gì và ứng dụng thực tế của nó?
   A: Là các hàm được chạy trước khi gửi request hoặc sau khi nhận response. Ứng dụng: Gắn token vào header, log dữ liệu, format lại response, hoặc xử lý refresh token tự động.
3. Q: Làm thế nào để hủy một request trong Axios?
   A: Sử dụng `AbortController` (trong các bản mới) hoặc `CancelToken` (bản cũ). Điều này hữu ích khi người dùng chuyển trang trước khi dữ liệu kịp tải xong.
4. Q: Axios xử lý lỗi như thế nào?
   A: Axios mặc định coi các mã trạng thái ngoài dải 2xx là lỗi và sẽ đưa vào khối `catch`. Bạn có thể tùy biến dải mã này qua thuộc tính `validateStatus`.

### Scenario
1. Tình huống: Bạn cần gọi 10 API cùng lúc và chỉ hiển thị kết quả khi tất cả đã xong.
   Giải quyết: Sử dụng `axios.all` kết hợp với `axios.spread` hoặc dùng `Promise.all` với các request của Axios.
2. Tình huống: Website của bạn bị chậm và bạn phát hiện nhiều request giống hệt nhau bị gửi đi liên tục.
   Giải quyết: Sử dụng một biến cờ (flag) hoặc triển khai một Cache adapter cho Axios để lưu kết quả các request trùng lặp trong một khoảng thời gian ngắn.
3. Tình huống: Bạn muốn thay đổi URL server từ `dev.api.com` sang `prod.api.com` mà không muốn sửa từng file gọi API.
   Giải quyết: Sử dụng Environment Variables (`process.env.VITE_API_URL`) để gán vào thuộc tính `baseURL` khi khởi tạo instance Axios.
