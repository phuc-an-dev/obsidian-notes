---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/javascript"
  - "#topic/performance"
related:
  - "[[Core Web Vitals]]"
  - "[[HTTP Caching]]"
---

## 1. What

SEO (Search Engine Optimization) là tập hợp các kỹ thuật và phương pháp nhằm tối ưu hóa một trang web để các công cụ tìm kiếm (Google, Bing, ...) có thể thu thập, hiểu và xếp hạng nội dung cao hơn trong trang kết quả tìm kiếm (SERP). SEO không chỉ là viết từ khóa đúng chỗ mà còn bao gồm kỹ thuật, nội dung, và uy tín của trang web.

## 2. Why

Trước khi có SEO, các công cụ tìm kiếm phân loại trang web theo thứ tự xuất hiện hoặc mật độ từ khóa thuần túy, dẫn đến kết quả chất lượng thấp và dễ bị thao túng. Google giải quyết vấn đề này bằng thuật toán PageRank (1998) và sau đó liên tục cập nhật để phạt các kỹ thuật thao túng. SEO ra đời như một hướng dẫn giúp chủ trang web làm đúng để xuất hiện tự nhiên trên công cụ tìm kiếm mà không cần trả tiền quảng cáo.

## 3. Mental Model

Hãy tưởng tượng thư viện đại học với hàng triệu cuốn sách. Người thủ thư (Google crawler) đến lấy sách của bạn, đọc lướt, rồi xếp vào danh mục. Khi sinh viên hỏi về một chủ đề, thủ thư tìm cuốn liên quan nhất, rõ ràng nhất, được nhiều người trích dẫn nhất.

SEO là cách bạn trình bày cuốn sách của mình:
- Bìa rõ ràng (title tag, meta description)
- Mục lục dễ hiểu (heading hierarchy)
- Nội dung chất lượng (content relevance)
- Nhiều tác giả khác trích dẫn (backlinks)
- Cuốn sách mở nhanh (page speed)

Thủ thư không đọc hết từng từ mà dựa vào cấu trúc và tín hiệu bên ngoài để đánh giá.

## 4. Where it fits

```
User Query
    |
    v
Search Engine (Google, Bing)
    |
    +-- Crawling:    Googlebot thu thập HTML, JS, CSS
    |
    +-- Indexing:    Phân tích nội dung, xây dựng chỉ mục
    |
    +-- Ranking:     Sắp xếp theo 200+ yếu tố (E-E-A-T, backlinks, UX)
    |
    v
SERP (Search Engine Results Page)
    |
    v
User clicks -> trang web của bạn
```

SEO tác động vào giai đoạn Crawling, Indexing, và Ranking.

## 5. When to use

- Trang web cần lưu lượng truy cập tự nhiên dài hạn (organic traffic) mà không muốn phụ thuộc quảng cáo trả phí.
- Doanh nghiệp muốn xây dựng thương hiệu và uy tín lâu dài.
- Blog, trang tin tức, e-commerce cần tăng khả năng hiển thị cho từ khóa cụ thể.
- Ứng dụng SaaS muốn thu hút người dùng qua nội dung (content-led growth).
- Các trang cạnh tranh cao cần technical SEO để có lợi thế kỹ thuật.

## 6. When NOT to use

- Khi cần kết quả ngay lập tức: SEO mất ít nhất 3-6 tháng để thấy tác động. Dùng Google Ads thay thế.
- Sản phẩm không có nhu cầu tìm kiếm: nếu không ai gõ từ khóa liên quan, SEO vô nghĩa.
- Trang nội bộ (intranet) không cần được index bởi Google.
- Các trang landing page ngắn hạn cho chiến dịch: PPC (Pay-Per-Click) phù hợp hơn.

Hậu quả nếu làm sai:
- Google Penalty (hình phạt thuật toán) có thể xóa trang web khỏi kết quả tìm kiếm.
- Keyword stuffing làm xấu trải nghiệm người dùng và bị phạt.

## 7. Trade-offs

| Ưu điểm | Nhược điểm |
|---------|-----------|
| Traffic tự nhiên miễn phí sau khi đầu tư | Kết quả chậm (3-12 tháng) |
| Hiệu quả tích lũy theo thời gian | Phụ thuộc vào thuật toán Google (có thể thay đổi) |
| Xây dựng uy tín thương hiệu bền vững | Cần đầu tư liên tục về nội dung và kỹ thuật |
| Không mất tiền mỗi click | Cạnh tranh từ khóa cao rất khó xếp hạng |
| Kết hợp tốt với content marketing | Không kiểm soát hoàn toàn thứ hạng |

## 8. Alternatives

| Phương pháp | Khi nào dùng | So sánh với SEO |
|------------|-------------|----------------|
| Google Ads (PPC) | Cần traffic ngay, ngân sách đủ | Nhanh hơn nhưng tốn tiền mỗi click |
| Social Media Marketing | Audience trên mạng xã hội | Không phụ thuộc search intent |
| Email Marketing | Đã có danh sách khách hàng | Không thu hút khách mới |
| Influencer Marketing | Brand awareness nhanh | Không bền vững, phụ thuộc người khác |
| App Store Optimization (ASO) | Mobile app | SEO tương đương cho app store |

## 9. How

### Technical SEO cơ bản trong HTML

```html
<!-- Title tag: tối đa 60 ký tự, chứa từ khóa chính -->
<title>Hướng dẫn SEO cho Developer 2024 | TechBlog</title>

<!-- Meta description: 120-160 ký tự, mô tả nội dung + CTA -->
<meta name="description" content="Tìm hiểu kỹ thuật SEO thiết thực cho lập trình viên: structured data, Core Web Vitals, sitemap XML và robots.txt.">

<!-- Canonical URL: tránh duplicate content -->
<link rel="canonical" href="https://example.com/seo-guide">

<!-- Open Graph (cho social sharing) -->
<meta property="og:title" content="Hướng dẫn SEO cho Developer">
<meta property="og:description" content="Kỹ thuật SEO thiết thực cho lập trình viên.">
<meta property="og:image" content="https://example.com/og-image.jpg">
<meta property="og:url" content="https://example.com/seo-guide">

<!-- Robots meta tag -->
<meta name="robots" content="index, follow">

<!-- Structured Data (JSON-LD) -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Hướng dẫn SEO cho Developer 2024",
  "author": {
    "@type": "Person",
    "name": "An Phuc"
  },
  "datePublished": "2026-04-24",
  "image": "https://example.com/og-image.jpg"
}
</script>
```

### Sitemap XML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com/</loc>
    <lastmod>2026-04-24</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
  <url>
    <loc>https://example.com/seo-guide</loc>
    <lastmod>2026-04-24</lastmod>
    <changefreq>monthly</changefreq>
    <priority>0.8</priority>
  </url>
</urlset>
```

### robots.txt

```
User-agent: *
Allow: /
Disallow: /admin/
Disallow: /api/

Sitemap: https://example.com/sitemap.xml
```

### Next.js metadata API (App Router)

```typescript
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next';

export async function generateMetadata({ params }): Promise<Metadata> {
  const post = await getPost(params.slug);
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [{ url: post.coverImage }],
    },
    alternates: {
      canonical: `https://example.com/blog/${params.slug}`,
    },
  };
}
```

## 10. Production concerns

**Core Web Vitals (quan trọng nhất):**
- LCP (Largest Contentful Paint) < 2.5s: tối ưu ảnh hero, preload font
- INP (Interaction to Next Paint) < 200ms: giảm JavaScript blocking
- CLS (Cumulative Layout Shift) < 0.1: đặt kích thước cố định cho ảnh và ads

**Crawl budget:**
- Trang lớn (> 10,000 URL) cần quản lý crawl budget.
- Tránh URL có tham số vô hạn (`?page=1`, `?color=red&size=M`) bằng canonical hoặc noindex.
- Dùng `robots.txt` chặn các đường dẫn không cần index.

**Monitoring:**
- Google Search Console: theo dõi impressions, clicks, average position, crawl errors.
- Google Analytics 4: theo dõi organic traffic và hành vi người dùng.
- Ahrefs / Semrush: phân tích từ khóa, backlink, và đối thủ.
- PageSpeed Insights / Lighthouse: đo Core Web Vitals.

**Failure modes:**
- Vô tình thêm `noindex` lên toàn bộ trang trên production (thường do môi trường staging được copy lên).
- Thay đổi URL mà không có 301 redirect: mất toàn bộ link equity.
- Duplicate content không có canonical: Google chia sẻ ranking power giữa các trang trùng.
- JavaScript rendering không hoàn chỉnh: Googlebot có thể không thấy nội dung render bởi JS.

## 11. Common mistakes

**Lỗi 1: Đặt `noindex` lên toàn bộ trang khi deploy**
- Lỗi: Copy cấu hình từ staging sang production, bao gồm `<meta name="robots" content="noindex">`.
- Sửa: Dùng biến môi trường để điều kiện hóa robots meta tag. Kiểm tra Google Search Console ngay sau deploy.

**Lỗi 2: Không có 301 redirect khi đổi URL**
- Lỗi: Đổi slug từ `/blog/post-1` sang `/blog/bai-viet-1` mà không redirect, mất hết backlink và ranking.
- Sửa: Luôn thêm 301 redirect từ URL cũ sang URL mới. Cập nhật internal link đồng thời.

**Lỗi 3: Keyword stuffing trong title và nội dung**
- Lỗi: Nhồi từ khóa quá nhiều: "SEO SEO tốt nhất 2024 SEO hướng dẫn SEO".
- Sửa: Viết tự nhiên cho người đọc, sử dụng từ khóa và từ đồng nghĩa một cách hữu cơ.

**Lỗi 4: Bỏ qua internal linking**
- Lỗi: Các trang không liên kết với nhau, Googlebot khó khám phá nội dung mới.
- Sửa: Thêm contextual internal link từ trang có authority cao sang trang muốn tăng thứ hạng.

## 12. Sample project

Xây dựng một blog cá nhân với Next.js App Router, đảm bảo:
- Hard constraint: Điểm Lighthouse SEO >= 95 và Performance >= 90 trên trang bài viết.

Các bước:
1. Tạo `generateMetadata` động cho mỗi bài viết với title, description, og:image riêng.
2. Thêm structured data JSON-LD kiểu `Article` cho từng bài.
3. Tạo `sitemap.xml` tự động từ danh sách bài viết.
4. Cấu hình `robots.txt` đúng cho production.
5. Tối ưu ảnh bằng `next/image` với `priority` cho ảnh hero.
6. Đo điểm Lighthouse và sửa cho đến khi đạt ngưỡng.

## 13. Interview

### Core Q&A

**Q: Sự khác nhau giữa on-page SEO và off-page SEO là gì?**
A: On-page SEO là các yếu tố bạn kiểm soát trực tiếp trên trang web: title tag, meta description, heading, nội dung, tốc độ trang, structured data. Off-page SEO là các tín hiệu bên ngoài: backlink từ trang khác, brand mentions, social signals. On-page là nền tảng; off-page quyết định thứ hạng trong các từ khóa cạnh tranh cao.

**Q: Canonical URL dùng để làm gì?**
A: Khai báo cho Google biết đâu là URL "chính thức" khi nhiều URL có nội dung giống nhau (ví dụ: trang có và không có trailing slash, trang với tham số filter). Ngăn chặn duplicate content làm loãng ranking power.

**Q: Core Web Vitals ảnh hưởng thế nào đến SEO?**
A: Từ tháng 5/2021, Google dùng Core Web Vitals (LCP, INP, CLS) như một ranking signal trong thuật toán Page Experience. Trang có UX tốt được ưu tiên hơn trang chậm, dù nội dung tương đương. Tác động không phải là yếu tố quyết định nhưng là lợi thế cạnh tranh.

**Q: JavaScript rendering ảnh hưởng thế nào đến crawling?**
A: Googlebot crawl theo hai làn sóng: lần đầu lấy HTML thô, lần hai render JavaScript (có thể mất vài ngày đến vài tuần). Nội dung quan trọng nên có trong HTML ban đầu (SSR hoặc SSG) để được index nhanh. SPA thuần client-side rendering có thể bị index chậm.

**Q: Sự khác nhau giữa 301 và 302 redirect trong SEO?**
A: 301 (Permanent Redirect) chuyển toàn bộ link equity (PageRank) sang URL mới - dùng khi đổi URL vĩnh viễn. 302 (Temporary Redirect) không chuyển link equity - Google giữ URL cũ trong index. Dùng 301 khi đổi URL thật sự.

### Scenarios

**Scenario 1:** Trang web bị Google Penalty, traffic giảm 80% sau một đêm.
- Kiểm tra Google Search Console xem có manual action không.
- Nếu là algorithmic penalty (Panda, Penguin): phân tích nội dung kém chất lượng hoặc backlink toxic, xử lý rồi chờ Google recrawl.
- Nếu là manual action: đọc lý do, sửa vi phạm, gửi reconsideration request.

**Scenario 2:** Bạn đổi toàn bộ URL structure của trang e-commerce từ `/product?id=123` sang `/san-pham/ten-san-pham`.
- Lên kế hoạch: crawl toàn bộ URL cũ trước khi deploy.
- Deploy: thêm 301 redirect từng URL cũ sang URL mới.
- Sau deploy: submit sitemap mới, theo dõi Google Search Console 4-6 tuần để đảm bảo re-index hoàn tất.
- Cập nhật tất cả internal link và backlink quan trọng.

**Scenario 3:** Sản phẩm mới cần ranking nhanh trong 3 tháng.
- SEO không đủ nhanh một mình: kết hợp Google Ads cho traffic ngay lập tức.
- Tập trung vào long-tail keyword ít cạnh tranh để rank nhanh hơn.
- Tạo nội dung chất lượng cao, xây dựng internal link mạnh từ các trang có authority.
- Đẩy backlink từ các nguồn uy tín trong ngành.

## 14. References

- Google Search Central: https://developers.google.com/search/docs
- Google Search Console: https://search.google.com/search-console
- Schema.org: https://schema.org
- Web.dev (Core Web Vitals): https://web.dev/explore/learn-core-web-vitals
- Sitemaps protocol: https://www.sitemaps.org/protocol.html
- Next.js Metadata API: https://nextjs.org/docs/app/building-your-application/optimizing/metadata

## 15. Real-world Code

- **Vercel Next.js Commerce**: https://github.com/vercel/commerce — triển khai SEO hoàn chỉnh với generateMetadata, sitemap, structured data cho e-commerce.
- **Taxonomy (shadcn blog template)**: https://github.com/shadcn-ui/taxonomy — ví dụ về metadata, OG image generation bằng @vercel/og.
- **Cal.com**: https://github.com/calend/cal.com — hệ thống SEO lớn với dynamic sitemap, structured data, i18n SEO.

## 16. Community

- Reddit: r/SEO (thảo luận thuật toán, case study), r/TechSEO (SEO kỹ thuật)
- Stack Overflow: tag `seo` và `google-search-console`
- Blog: Moz Blog (https://moz.com/blog), Ahrefs Blog (https://ahrefs.com/blog), Search Engine Journal
- Conference talks: Google Search Central YouTube channel — các video từ John Mueller và Gary Illyes về crawling, indexing, ranking.
- Twitter/X: theo dõi @JohnMu (John Mueller, Google), @SEngineLand, @aleyda (Aleyda Solis, Technical SEO expert)
