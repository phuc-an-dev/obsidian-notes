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
Rendering Strategies trong Next.js là tập hợp các phương pháp khác nhau để chuyển đổi mã React thành HTML và hiển thị cho người dùng, bao gồm: Static Site Generation (SSG), Server-Side Rendering (SSR), Incremental Static Regeneration (ISR), và Client-Side Rendering (CSR).

## 2. Why
Mỗi loại nội dung trên web có yêu cầu khác nhau về độ tươi mới của dữ liệu (data freshness) và tốc độ tải trang (performance). Next.js cung cấp nhiều chiến lược để nhà phát triển có thể tối ưu hóa trải nghiệm người dùng và SEO cho từng trang cụ thể thay vì áp dụng một cách tiếp cận duy nhất cho toàn bộ ứng dụng.

## 3. Mental Model
Hãy tưởng tượng một nhà hàng:
- **SSG**: Giống như đồ ăn đóng hộp sẵn, khách đến là có ngay nhưng không thể thay đổi nguyên liệu.
- **SSR**: Giống như món ăn nấu theo yêu cầu, khách đợi một chút để đầu bếp nấu món nóng hổi ngay tại chỗ.
- **ISR**: Giống như quầy buffet được làm mới mỗi 15 phút, đồ ăn khá tươi và khách không phải đợi lâu.
- **CSR**: Giống như bộ kit tự nấu, nhà hàng đưa nguyên liệu và khách tự nấu tại bàn của mình.

## 4. Where it fits
User Request -> Next.js Server (Check Cache for SSG/ISR) -> (Optional) Fetch Data & Render (SSR) -> Send HTML -> Browser (CSR/Hydration).

## 5. When to use
- **SSG**: Trang landing page, blog, tài liệu kỹ thuật (dữ liệu ít thay đổi).
- **SSR**: Trang cá nhân hóa (Dashboard người dùng), trang cần dữ liệu thời gian thực và SEO.
- **ISR**: Trang danh sách sản phẩm E-commerce, trang tin tức (dữ liệu nhiều nhưng cần cập nhật định kỳ).
- **CSR**: Các phần tương tác nhỏ trên trang, dashboard nội bộ không cần SEO.

## 6. When NOT to use
- Đừng dùng **SSR** cho các trang tĩnh hoàn toàn vì sẽ tốn tài nguyên server vô ích.
- Đừng dùng **SSG** cho các trang có hàng triệu path thay đổi liên tục (nên kết hợp ISR).
- Đừng dùng **CSR** cho các trang cần SEO mạnh hoặc yêu cầu First Contentful Paint cực nhanh.

## 7. Trade-offs
| Chiến lược | Tốc độ (TTFB) | SEO | Độ tươi dữ liệu | Tải Server |
|------------|---------------|-----|----------------|------------|
| SSG | Rất nhanh | Tốt | Thấp | Rất thấp |
| SSR | Chậm hơn | Tốt | Rất cao | Cao |
| ISR | Nhanh | Tốt | Trung bình | Thấp |
| CSR | Chậm ban đầu | Kém | Cao | Rất thấp |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Remix | Ưu tiên SSR và tối ưu hóa chuyển đổi trạng thái. |
| Astro | Island Architecture, mặc định là Static và chỉ thêm JS khi cần. |

## 9. How
Trong App Router, chiến lược được quyết định qua cách cấu hình hàm `fetch`:

```tsx
// SSG (mặc định)
const data = await fetch('https://api.com/data');

// SSR (force-dynamic)
const data = await fetch('https://api.com/data', { cache: 'no-store' });

// ISR (revalidate sau 60s)
const data = await fetch('https://api.com/data', { next: { revalidate: 60 } });
```

## 10. Production concerns
### Scaling
SSG/ISR dễ dàng scale nhờ CDN (Vercel Edge Network). SSR yêu cầu tài nguyên server (Node.js runtime) mạnh mẽ hơn khi traffic tăng cao.

### Failure
Với ISR, nếu lần revalidate thất bại, Next.js sẽ tiếp tục phục vụ bản cache cũ (stale) thay vì trả về lỗi, đảm bảo tính sẵn sàng cao.

### Monitoring
Theo dõi tỉ lệ Cache Hit/Miss trên CDN để điều chỉnh thời gian revalidate của ISR một cách hợp lý.

## 11. Common mistakes
- Mistake: Nhầm lẫn rằng dùng App Router thì mọi thứ đều là SSR.
  Fix: Hiểu rằng mặc định App Router sẽ cố gắng SSG mọi thứ nếu không có các hàm "dynamic" như `cookies()`, `headers()`.

- Mistake: Set thời gian ISR quá ngắn (ví dụ 1s) gây quá tải API nguồn.
  Fix: Cân đối giữa yêu cầu kinh doanh và khả năng chịu tải của hệ thống backend.

## 12. Sample project
Tạo một trang tin tức tổng hợp:
- Trang chủ: SSG (Build lúc deploy).
- Chuyên mục: ISR (Cập nhật mỗi 10 phút).
- Bài báo cụ thể: ISR (Cập nhật nếu có sửa đổi).
- Dashboard cá nhân: SSR (Render theo session người dùng).

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa SSR và ISR là gì?
   A: SSR render lại trang cho mỗi request, đảm bảo dữ liệu luôn mới nhất nhưng chậm hơn. ISR phục vụ trang từ cache và render lại ở chế độ nền theo chu kỳ, giúp trang nhanh như tĩnh nhưng vẫn cập nhật được dữ liệu.

### Scenario
1. Q: Làm thế nào để implement một trang web có 100,000 sản phẩm mà không làm tăng thời gian build?
   A: Sử dụng ISR. Chỉ build sẵn 1,000 sản phẩm phổ biến nhất. 99,000 sản phẩm còn lại sẽ được render theo yêu cầu (on-demand) khi có người truy cập lần đầu và sau đó được cache lại.

## 14. References
- Next.js Rendering: https://nextjs.org/docs/app/building-your-application/rendering
- Vercel Data Cache: https://vercel.com/docs/infrastructure/data-cache

## 15. Real-world Code
Tham khảo cách các trang thương mại điện tử lớn (như Nike, TikTok Shop) sử dụng kết hợp các chiến lược này để tối ưu tốc độ load.

## 16. Community
- YouTube: "Next.js Rendering Strategies Explained" bởi Theo - t3․gg.
- Blog: Vercel - "Static by default, Dynamic when needed".
