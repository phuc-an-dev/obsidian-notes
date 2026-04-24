---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/performance"
related:
  - "[[nextjs]]"
---

## 1. What
Image Optimization trong Next.js là một tập hợp các kỹ thuật tự động nhằm tối ưu hóa hiệu suất tải hình ảnh. Thành phần cốt lõi là component `next/image`, giúp tự động thay đổi kích thước, nén ảnh, và hỗ trợ các định dạng hiện đại như WebP, AVIF.

## 2. Why
Hình ảnh thường chiếm phần lớn dung lượng của một trang web. Việc xử lý ảnh thủ công (cắt nhiều kích thước, chuyển định dạng, viết mã lazy load) rất tốn thời gian và dễ sai sót. Next.js tự động hóa toàn bộ quy trình này để cải thiện Core Web Vitals, đặc biệt là Largest Contentful Paint (LCP).

## 3. Mental Model
Hãy tưởng tượng Next.js Image như một thợ may chuyên nghiệp. Thay vì đưa cho mọi khách hàng cùng một cỡ áo khổng lồ (ảnh gốc), người thợ này sẽ đo đạc kích thước màn hình của khách (viewport) và may đo một chiếc áo vừa vặn nhất, nhẹ nhất trước khi giao đến tay họ.

## 4. Where it fits
React Component (`next/image`) -> Image Optimization API (Server-side) -> CDN/Cache -> Browser.

## 5. When to use
- Luôn luôn sử dụng cho các hình ảnh nội dung: ảnh sản phẩm, ảnh bài viết, banner.
- Khi cần đảm bảo ảnh không gây ra hiện tượng nhảy giao diện (Layout Shift).
- Khi muốn hỗ trợ tự động Lazy Loading mà không cần viết JS thủ công.

## 6. When NOT to use
- Không dùng cho các icon SVG cực nhỏ (nên nhúng trực tiếp hoặc dùng icon font).
- Khi hình ảnh đã được tối ưu hoàn toàn và quản lý bởi các dịch vụ bên thứ ba có API biến đổi mạnh mẽ (như Cloudinary, Imgix) và bạn muốn kiểm soát hoàn toàn URL.
- Cho các hình ảnh trang trí thuần túy bằng CSS (nên dùng `background-image`).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tăng điểm Lighthouse/SEO đáng kể | Tốn tài nguyên CPU trên server để xử lý ảnh lần đầu |
| Tránh Cumulative Layout Shift (CLS) | Yêu cầu cấu hình `remotePatterns` cho ảnh từ domain khác |
| Tự động hỗ trợ WebP/AVIF | Có thể gây chậm build nếu xử lý quá nhiều ảnh tĩnh cùng lúc |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Standard `<img>` | Đơn giản nhưng không có tối ưu hóa, dễ gây chậm trang. |
| Cloudinary / Imgix | Dịch vụ chuyên dụng, mạnh mẽ hơn nhưng tốn phí và cần SDK riêng. |

## 9. How
```tsx
import Image from 'next/image'

export default function Profile() {
  return (
    <div className="container">
      {/* Ảnh local: Tự động xác định width/height */}
      <Image
        src="/me.png"
        alt="Author"
        width={500}
        height={500}
      />
      
      {/* Ảnh remote: Phải cung cấp width/height hoặc dùng fill */}
      <Image
        src="https://example.com/hero.jpg"
        alt="Hero image"
        fill
        className="object-cover"
        priority // Tải ngay lập tức (cho LCP)
      />
    </div>
  )
}
```

## 10. Production concerns
### Scaling
Việc xử lý ảnh tiêu tốn CPU. Khi scale lớn, nên sử dụng các trình tối ưu hóa ảnh bên ngoài hoặc tận dụng cache của CDN một cách triệt để.

### Failure
Nếu domain ảnh không được khai báo trong `next.config.js`, ứng dụng sẽ crash hoặc không hiển thị ảnh.

### Monitoring
Theo dõi các cảnh báo về "Large Images" trong console của trình duyệt và sử dụng công cụ phân tích bundle để xem `next/image` có được cấu hình đúng không.

## 11. Common mistakes
- Mistake: Không cung cấp thuộc tính `width` và `height` (hoặc `fill`), dẫn đến lỗi layout shift.
  Fix: Luôn xác định tỉ lệ khung hình hoặc dùng `fill` kết hợp với `aspect-ratio` trong CSS.

- Mistake: Dùng `priority` cho quá nhiều ảnh trên một trang.
  Fix: Chỉ dùng `priority` cho ảnh là thành phần LCP (thường là banner đầu trang).

## 12. Sample project
Tạo một trang Portfolio hiển thị danh sách các tác phẩm nghệ thuật có độ phân giải cao. Sử dụng `next/image` với thuộc tính `placeholder="blur"` để tạo hiệu ứng load mượt mà.

## 13. Interview
### Core Q&A
1. Q: Cumulative Layout Shift (CLS) là gì và Next.js Image giải quyết nó như thế nào?
   A: CLS là hiện tượng các thành phần giao diện bị nhảy khi ảnh được load xong. Next.js Image buộc lập trình viên xác định kích thước ảnh trước, từ đó trình duyệt dành sẵn không gian cho ảnh, ngăn chặn việc nhảy giao diện.

### Scenario
1. Q: Bạn làm thế nào để hiển thị ảnh từ một URL bất kỳ được người dùng nhập vào?
   A: Tôi phải cấu hình `remotePatterns` trong `next.config.js` để cho phép các domain đó. Nếu domain là hoàn toàn ngẫu nhiên, tôi có thể phải sử dụng một giải pháp Proxy hoặc chấp nhận rủi ro bảo mật (không khuyến khích).

## 14. References
- Next.js Image Docs: https://nextjs.org/docs/app/building-your-application/optimizing/images
- Web Vitals: https://web.dev/vitals/

## 15. Real-world Code
Học cách các trang tin tức lớn như BBC hay New York Times (sử dụng Next.js) tối ưu hóa hàng ngàn tấm ảnh mỗi ngày.

## 16. Community
- YouTube: "Next.js Image Component - Everything You Need to Know".
- Blog: Vercel - "How we optimized images for millions of users".
