---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/performance"
related:
  - "[[nextjs]]"
  - "[[app-router]]"
---

## 1. What
SEO & Metadata trong Next.js (App Router) là một hệ thống Metadata API tích hợp cho phép lập trình viên định nghĩa các thẻ meta (title, description, open graph) một cách khai báo. Next.js tự động chèn các thẻ này vào phần `<head>` của HTML được render trên server.

## 2. Why
Trước đây, việc quản lý SEO trong React khá thủ công (thường dùng `react-helmet`) và khó đảm bảo tính nhất quán giữa Server và Client. SEO tốt là yếu tố sống còn cho các trang web thương mại điện tử, blog và tin tức để thu hút lưu lượng truy cập từ Google, Bing và các mạng xã hội (Facebook, Twitter).

## 3. Mental Model
Hãy tưởng tượng mỗi trang web của bạn là một cuốn sách trong một thư viện khổng lồ. Metadata chính là bìa sách và trang giới thiệu. Nếu bìa sách trắng tinh (thiếu Metadata), thủ thư (Google Crawler) sẽ không biết xếp sách vào đâu, và độc giả (người dùng) cũng không muốn cầm lên xem. Next.js Metadata API giúp bạn in ấn "bìa sách" chuyên nghiệp một cách tự động.

## 4. Where it fits
Page/Layout Configuration -> Metadata API -> Server-side Rendering -> HTML `<head>` -> Search Engine Crawler.

## 5. When to use
- Luôn luôn sử dụng cho mọi trang công khai (Public pages).
- Khi cần tạo các thẻ Open Graph động cho từng bài viết, sản phẩm.
- Khi cần cấu hình các thẻ kỹ thuật như `canonical`, `robots`, `viewport`.

## 6. When NOT to use
- Các trang quản trị nội bộ (Admin Dashboard) không cần SEO.
- Các ứng dụng web cực kỳ bảo mật mà bạn không muốn Search Engine lập chỉ mục (Index).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Hỗ trợ SEO mặc định, hiệu năng cao | Cần hiểu cơ chế merge metadata từ Layout đến Page |
| Hỗ trợ Open Graph Image động (Image Response) | Khó debug nếu có nhiều tầng metadata lồng nhau |
| Type-safety mạnh mẽ với TypeScript | Một số thẻ meta đặc thù vẫn cần dùng file `robots.ts` hoặc `sitemap.ts` |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Next/Head (Pages Router) | Cách cũ, linh hoạt hơn nhưng không có tính kế thừa (inheritance) tốt bằng. |
| react-helmet-async | Dành cho ứng dụng React SPA thuần túy. |

## 9. How
```tsx
// Static Metadata
export const metadata = {
  title: 'My Awesome App',
  description: 'Welcome to the future of web',
};

// Dynamic Metadata
export async function generateMetadata({ params }) {
  const product = await getProduct(params.id);
  return {
    title: product.name,
    description: product.summary,
    openGraph: {
      images: [product.image],
    },
  };
}

export default function Page() {
  return <h1>My Page</h1>;
}
```

## 10. Production concerns
### Scaling
Sử dụng `sitemap.ts` và `robots.ts` để giúp Google crawler hiểu cấu trúc của hàng triệu trang web một cách hiệu quả.

### Failure
Nếu hàm `generateMetadata` bị lỗi hoặc quá chậm, nó sẽ ảnh hưởng đến TTFB của trang. Cần xử lý lỗi và caching dữ liệu trong hàm này.

### Monitoring
Sử dụng Google Search Console để theo dõi hiệu quả SEO thực tế và phát hiện các lỗi Metadata trên môi trường Production.

## 11. Common mistakes
- Mistake: Khai báo Metadata trong Client Component.
  Fix: Metadata chỉ được khai báo trong Server Components (Layout hoặc Page).

- Mistake: Quên thẻ `canonical` dẫn đến lỗi trùng lặp nội dung (Duplicate Content).
  Fix: Luôn cấu hình `metadataBase` và `canonical` URL.

## 12. Sample project
Tạo một blog cá nhân:
- **Root Layout**: Chứa metadata chung (site name, twitter handle).
- **Category Page**: Metadata động theo tên danh mục.
- **Post Page**: Metadata chi tiết theo bài viết và tự động sinh ảnh Open Graph bằng `OG Image Generation`.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào Next.js xử lý việc trùng lặp Metadata giữa Layout và Page?
   A: Next.js sử dụng cơ chế ghi đè (overriding). Metadata ở cấp độ sâu nhất (Page) sẽ ghi đè các giá trị trùng lặp từ các cấp độ cao hơn (Layout). Các giá trị không trùng sẽ được hợp nhất (merged).

### Scenario
1. Q: Bạn làm thế nào để hiển thị ảnh preview khi chia sẻ link lên Facebook?
   A: Tôi sẽ cấu hình thuộc tính `openGraph` trong Metadata API, cung cấp các trường `title`, `description` và đặc biệt là `images` (URL tuyệt đối của ảnh).

## 14. References
- Next.js Metadata: https://nextjs.org/docs/app/building-your-application/optimizing/metadata
- OG Image Generation: https://nextjs.org/docs/app/building-your-application/optimizing/metadata#dynamic-image-generation

## 15. Real-world Code
Nghiên cứu cách các trang như `LeeRob.it` hoặc chính trang `Nextjs.org` triển khai dynamic metadata và sitemaps.

## 16. Community
- YouTube: "SEO in Next.js 13+ App Router" - Delba de Oliveira.
- Twitter: @leeerob (Lee Robinson) thường xuyên chia sẻ tip về SEO.
