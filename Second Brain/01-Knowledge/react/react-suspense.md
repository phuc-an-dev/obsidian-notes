---
tags:
  - type/concept
  - status/draft
  - lang/react
  - topic/async
date: 2026-04-17
related:
  - "[[react-lifecycle]]"
  - "[[tanstack-react-query]]"
---

# React Suspense

## 1. What

React Suspense là một cơ chế cho phép component khai báo rằng nó đang "chờ" một thứ gì đó — có thể là code chunk chưa được tải về, hoặc dữ liệu chưa sẵn sàng — và hiển thị một UI dự phòng (fallback) trong lúc chờ đợi. Nó hoạt động thông qua một giao thức đặc biệt: khi một component ném ra (throw) một Promise, React sẽ bắt lấy Promise đó, render fallback, và thử render lại component khi Promise resolve. Suspense không phải là một hook hay một higher-order component mà là một phần cốt lõi của cơ chế render của React.

## 2. Why

Trước khi có Suspense, việc xử lý các trạng thái bất đồng bộ trong UI rất rối rắm và lặp lại. Mỗi component cần tự quản lý `isLoading`, `isError`, `data` — dẫn đến vô số điều kiện lồng nhau và khó kiểm soát. Với code splitting, cách duy nhất là dùng `import()` động kết hợp với các cờ trạng thái thủ công. Không có cách nhất quán để toàn bộ cây component nói với React "tôi chưa sẵn sàng, hãy hiển thị cái gì đó khác đi". Kết quả là mỗi lập trình viên tự phát minh lại wheel với các pattern loading state khác nhau, làm cho code không đồng nhất và khó bảo trì.

## 3. Mental Model

Hãy tưởng tượng một nhà hàng với một tấm biển "Xin chờ — bàn đang được chuẩn bị" ở lối vào. Khách hàng (user) không cần biết bên trong bếp đang làm gì. Họ chỉ thấy biển chờ (fallback), và khi bàn đã sẵn sàng, họ được dẫn vào ngồi (component render với dữ liệu). Người quản lý nhà hàng (Suspense boundary) là người quyết định khi nào thì đặt biển "Xin chờ" và khi nào thì cho khách vào.

Component bên trong là "đầu bếp" — nó có thể bất kỳ lúc nào ném ra một tờ phiếu (Promise) nói "tôi cần thêm thời gian", và người quản lý sẽ bắt lấy tờ phiếu đó, dựng biển chờ lên, rồi hỏi lại đầu bếp sau khi tờ phiếu được xử lý xong.

Điểm mấu chốt: component không quan tâm đến trạng thái loading. Nó chỉ viết code như thể dữ liệu đã có sẵn. Suspense boundary xử lý phần còn lại.

## 4. Where It Fits

```
User interaction
      |
      v
React Tree (render phase)
      |
      v
Component throws Promise  -->  Suspense Boundary catches it
      |                               |
      |                               v
      |                         Render <fallback />
      |                               |
      v                               v
Promise resolves             React retries render
      |                               |
      v                               v
Component renders normally   Fallback removed, component shown
```

Với Streaming SSR (Next.js App Router):

```
Server
  |
  v
Render HTML shell (instant)  -->  Send to browser  -->  TTFB thấp
  |
  v
Async chunks resolve one by one
  |
  v
Stream HTML fragments to browser
  |
  v
React hydrates each chunk as it arrives
```

## 5. When to Use

- Khi dùng `React.lazy()` để code-split các route hoặc component nặng (biểu đồ, editor, map).
- Khi dùng thư viện hỗ trợ Suspense như TanStack Query (`useSuspenseQuery`), SWR, hoặc Relay để fetch dữ liệu.
- Khi muốn xử lý loading state một cách khai báo và nhất quán trên toàn bộ cây component mà không lặp lại `isLoading` checks.
- Khi cần Streaming SSR với Next.js 13+ App Router để cải thiện Time To First Byte và cho phép trang hiển thị shell trước khi data sẵn sàng.
- Khi dùng `startTransition` hoặc `useDeferredValue` để tránh hiển thị fallback khi chuyển trang (giữ UI cũ trong khi tải UI mới).

## 6. When NOT to Use

- Không dùng Suspense để xử lý lỗi — đó là việc của `ErrorBoundary`. Nếu dùng sai, lỗi mạng có thể làm treo UI mãi mãi mà không có thông báo.
- Không bọc toàn bộ ứng dụng trong một Suspense boundary duy nhất — điều này sẽ ẩn toàn bộ UI khi bất kỳ component nào đang tải, gây trải nghiệm tệ.
- Không tự ý implement "throw a Promise" bên trong component trừ khi bạn đang xây dựng thư viện — đây là API nội bộ, dễ bị thay đổi.
- Không dùng khi logic loading đơn giản và cần kiểm soát chi tiết (ví dụ: skeleton loader theo hình dạng cụ thể của từng field) — lúc đó quản lý state thủ công rõ ràng hơn.

## 7. Trade-offs

| Pros | Cons |
|------|------|
| Code loading state sạch, khai báo, không lặp lại isLoading | Yêu cầu data source phải "Suspense-enabled" — không phải thư viện nào cũng hỗ trợ |
| Tách biệt hoàn toàn logic "chờ" khỏi business logic | Khó debug: khi component suspend, stack trace ít thông tin hơn |
| Hoạt động tốt với Concurrent Mode (startTransition, useDeferredValue) | Cần hiểu rõ vị trí đặt Suspense boundary để tránh fallback quá nhiều |
| Streaming SSR giúp TTFB nhanh hơn đáng kể | ErrorBoundary phải được dùng song song, tăng boilerplate |
| Tự động retry khi Promise resolve, không cần viết lại | Nếu Promise không bao giờ resolve, UI bị treo (cần timeout ở data layer) |

## 8. Alternatives

| Approach | Bundle overhead | Suspense support | SSR streaming | Khó | Ghi chú |
|----------|-----------------|-----------------|---------------|-----|---------|
| Manual useState + useEffect | 0 | Không | Không | Thấp | Phù hợp app nhỏ |
| React Suspense + TanStack Query | ~13kb (TQ) | Native (useSuspenseQuery) | Có (Next.js) | Trung bình | Khuyến dùng cho React 18+ |
| SWR | ~4kb | Có (hạn chế) | Hạn chế | Thấp | Nhẹ hơn TanStack Query |
| Relay | Lớn | Native (Facebook) | Có | Cao | Dùng với GraphQL |
| Apollo Client | ~30kb | Có | Có | Trung bình | GraphQL-focused |

## 9. How

### Code Splitting với React.lazy()

```tsx
import React, { Suspense, lazy } from 'react';

// Component này chỉ được download khi cần thiết (route-level split)
const HeavyChart = lazy(() => import('./HeavyChart'));
const SettingsPage = lazy(() => import('../pages/SettingsPage'));

function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      {/* fallback hiển thị trong khi JS chunk đang được download */}
      <Suspense fallback={<div className="skeleton skeleton-chart" />}>
        <HeavyChart />
      </Suspense>
    </div>
  );
}
```

### Data Fetching với TanStack Query (Suspense mode)

```tsx
import { Suspense } from 'react';
import { useSuspenseQuery } from '@tanstack/react-query';
import { ErrorBoundary } from 'react-error-boundary';

// Component này viết như thể data luôn có sẵn — không có isLoading check
function UserProfile({ userId }: { userId: string }) {
  // useSuspenseQuery throw Promise nếu data chưa có
  // throw Error nếu request thất bại -> ErrorBoundary bắt
  const { data: user } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(r => {
      if (!r.ok) throw new Error('Failed to fetch user');
      return r.json();
    }),
  });

  // Tại đây, user luôn tồn tại — TypeScript biết điều này
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}

// Parent phải bọc cả ErrorBoundary (xử lý lỗi) và Suspense (xử lý loading)
function UserPage({ userId }: { userId: string }) {
  return (
    <ErrorBoundary fallback={<ErrorCard message="Không thể tải thông tin người dùng." />}>
      <Suspense fallback={<UserProfileSkeleton />}>
        <UserProfile userId={userId} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### startTransition: Giữ UI cũ trong khi tải UI mới

```tsx
import { useState, useTransition, Suspense, lazy } from 'react';

const tabs = {
  home: lazy(() => import('./HomeTab')),
  settings: lazy(() => import('./SettingsTab')),
  analytics: lazy(() => import('./AnalyticsTab')),
} as const;

type TabKey = keyof typeof tabs;

function App() {
  const [tab, setTab] = useState<TabKey>('home');
  const [isPending, startTransition] = useTransition();
  const ActiveTab = tabs[tab];

  function switchTab(newTab: TabKey) {
    // startTransition đánh dấu navigation là "non-urgent"
    // React giữ UI cũ trong khi tab mới đang load
    // thay vì show fallback ngay lập tức (tránh flash)
    startTransition(() => setTab(newTab));
  }

  return (
    <div>
      <nav>
        {(Object.keys(tabs) as TabKey[]).map(t => (
          <button
            key={t}
            onClick={() => switchTab(t)}
            disabled={isPending}
            aria-current={tab === t ? 'page' : undefined}
          >
            {t}
          </button>
        ))}
      </nav>
      {/* opacity giảm khi pending, không ẩn toàn bộ UI */}
      <div style={{ opacity: isPending ? 0.6 : 1, transition: 'opacity 0.2s' }}>
        <Suspense fallback={<TabSkeleton />}>
          <ActiveTab />
        </Suspense>
      </div>
    </div>
  );
}
```

### useDeferredValue: Search-as-you-type không bị block UI

```tsx
import { useState, useDeferredValue, Suspense } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  // deferredQuery "đi chậm" hơn query
  // Component dùng deferredQuery sẽ suspend với giá trị cũ
  // trong khi giá trị mới đang render — tránh fallback nhảy
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Tìm kiếm..."
      />
      <div style={{ opacity: isStale ? 0.6 : 1 }}>
        <Suspense fallback={<SearchResultsSkeleton />}>
          <SearchResults query={deferredQuery} />
        </Suspense>
      </div>
    </div>
  );
}
```

### Nested Suspense Boundaries (Granular control)

```tsx
function ProductPage({ id }: { id: string }) {
  return (
    <div>
      {/* Header load nhanh, hiển thị trước */}
      <Suspense fallback={<ProductHeaderSkeleton />}>
        <ProductHeader id={id} />
      </Suspense>

      <div className="product-body">
        {/* Reviews và Recommendations load độc lập, không block nhau */}
        <ErrorBoundary fallback={<ReviewsError />}>
          <Suspense fallback={<ReviewsSkeleton />}>
            <Reviews id={id} />
          </Suspense>
        </ErrorBoundary>

        <ErrorBoundary fallback={null}> {/* Recommendations lỗi thì bỏ qua */}
          <Suspense fallback={<RecommendationsSkeleton />}>
            <Recommendations id={id} />
          </Suspense>
        </ErrorBoundary>
      </div>
    </div>
  );
}
```

## 10. Production Concerns

**Scaling — Streaming SSR:** Với Next.js App Router, mỗi Suspense boundary tạo ra một "hole" trong HTML được stream. Server gửi HTML shell trước (TTFB thấp), sau đó stream từng fragment khi Promise resolve. Điều này cải thiện cảm giác load của user nhưng tăng độ phức tạp của caching: response không còn là một HTML tĩnh mà là một stream, nên không cache được ở CDN theo cách thông thường. Cần xem xét stale-while-revalidate hoặc partial caching.

**Failure — ErrorBoundary là bắt buộc:** Nếu một Suspense-enabled component throw Error thay vì Promise, ErrorBoundary sẽ bắt nó. Nếu không có ErrorBoundary, lỗi sẽ bắn lên root và crash toàn bộ app. Luôn pair `ErrorBoundary > Suspense > Component`. Với `react-error-boundary`, sử dụng `useErrorBoundary` để reset boundary khi user retry.

**Monitoring:** Dùng `React.Profiler` để đo `actualDuration` của các Suspense boundary. Trên Sentry, enable React component tracking (`@sentry/react`) để theo dõi "Suspense fallback shown" events. Theo dõi metric "Interaction to Next Paint (INP)" và "Time to Interactive (TTI)" thay vì chỉ "TTFB" khi sử dụng Streaming SSR.

**Timeout — Không có built-in:** React hiện tại không có built-in timeout cho Suspense. Nếu Promise không bao giờ resolve, component suspend mãi mãi. Giải quyết bằng cách đặt `signal: AbortSignal.timeout(5000)` trong fetch, hoặc `timeout` trong axios instance.

## 11. Common Mistakes

- Mistake: Đặt một Suspense boundary duy nhất ở root, bọc toàn bộ app, mong chờ fallback xử lý tất cả.
  Fix: Đặt nhiều boundary ở các vị trí chiến lược — mỗi section có thể load và fallback độc lập, giữ phần còn lại của UI tiếp tục hoạt động. Granularity là chiếc chìa khóa.

- Mistake: Dùng Suspense mà không có ErrorBoundary bên cạnh, nên khi fetch thất bại UI bị treo ở fallback mãi mãi.
  Fix: Luôn pair Suspense với ErrorBoundary. Pattern chuẩn: `<ErrorBoundary><Suspense fallback={...}><DataComponent /></Suspense></ErrorBoundary>`.

- Mistake: Gọi `useSuspenseQuery` bên trong `useEffect` hoặc sau một điều kiện (conditional Suspense).
  Fix: Suspense chỉ hoạt động trong render phase. Data fetching phải được kích hoạt trong body của component. Không đặt hooks sau `if` statements.

- Mistake: Dùng `startTransition` cho mọi state update vì nghĩ nó tốt hơn mặc định.
  Fix: Chỉ dùng `startTransition` cho navigation hoặc update có thể defer mà không cần phản hồi ngay. Update tức thời như typing, toggle, checkbox phải dùng state update thông thường để tránh lag cảm giác.

## 12. Sample Project

**Project: E-commerce Product Listing với Streaming SSR**

Constraint khó: (1) HTML shell phải đến browser trong vòng 100ms. (2) Product grid và Recommendations phải load độc lập. (3) Lỗi fetch Recommendations không được làm hỏng Product grid. (4) Khi user filter, giữ grid cũ hiển thị trong khi grid mới đang fetch (không flash loading).

```
app/
  products/
    page.tsx              <- Server Component shell, không await bất kỳ gì
    ProductGrid.tsx       <- Client Component, dùng useSuspenseQuery
    Recommendations.tsx   <- Client Component, ErrorBoundary riêng (silent fail)
    FilterBar.tsx         <- Client Component, dùng startTransition
    skeletons/
      ProductGridSkeleton.tsx
      RecommendationsSkeleton.tsx
```

```tsx
// app/products/page.tsx (Next.js App Router — Server Component)
// Không await gì ở đây. Để Suspense xử lý tất cả.
export default function ProductsPage() {
  return (
    <main>
      <h1>Sản phẩm</h1>
      <FilterBar />

      <ErrorBoundary fallback={<ProductsErrorCard />}>
        <Suspense fallback={<ProductGridSkeleton count={12} />}>
          <ProductGrid />
        </Suspense>
      </ErrorBoundary>

      {/* Recommendations: lỗi thì bỏ qua hoàn toàn (fallback=null) */}
      <ErrorBoundary fallback={null}>
        <Suspense fallback={<RecommendationsSkeleton />}>
          <Recommendations />
        </Suspense>
      </ErrorBoundary>
    </main>
  );
}

// app/products/FilterBar.tsx
'use client';
import { useRouter, useSearchParams } from 'next/navigation';
import { useTransition } from 'react';

export function FilterBar() {
  const router = useRouter();
  const searchParams = useSearchParams();
  const [isPending, startTransition] = useTransition();

  function applyFilter(category: string) {
    const params = new URLSearchParams(searchParams);
    params.set('category', category);
    // startTransition giữ grid cũ hiển thị trong khi grid mới load
    startTransition(() => router.push(`/products?${params}`));
  }

  return (
    <div style={{ opacity: isPending ? 0.7 : 1 }}>
      {['all', 'electronics', 'clothing'].map(cat => (
        <button key={cat} onClick={() => applyFilter(cat)}>{cat}</button>
      ))}
    </div>
  );
}
```

## 13. Interview

### Core Q&A

**Q: React Suspense hoạt động như thế nào ở mức cơ bản (under the hood)?**
A: Khi một component cần "chờ" gì đó, nó throw một Promise. React bắt Promise đó, tìm Suspense boundary gần nhất trên cây component, render fallback của boundary đó, đăng ký `.then()` trên Promise, và khi Promise resolve, schedule re-render của component đó. Component được render lại như bình thường với data sẵn sàng.

**Q: "Suspense-enabled data source" có nghĩa là gì và tại sao không thể dùng với mọi async function?**
A: Là một data source tương thích với giao thức của Suspense: (1) Khi data chưa có, throw một Promise. (2) Khi data đã có (cache hit), trả về kết quả đồng bộ. Thư viện như TanStack Query duy trì internal cache và implement giao thức này. Một `async function` thông thường trả về Promise ngay lập tức — React không có cách biết khi nào nên retry hay lấy kết quả, nên phải có thư viện làm trung gian.

**Q: Khác nhau giữa `React.lazy()` và data fetching với Suspense?**
A: `React.lazy()` là use case gốc (React 16.6+), dùng cho code splitting — download JavaScript bundle theo yêu cầu. Data fetching với Suspense là use case mở rộng (ổn định từ React 18), yêu cầu thư viện ngoài implement giao thức throw-a-Promise và quản lý cache. Cả hai cùng dùng cơ chế Suspense nhưng phục vụ mục đích khác nhau.

**Q: `startTransition` liên quan đến Suspense như thế nào?**
A: `startTransition` đánh dấu một state update là "non-urgent" (transition). Khi state update đó trigger một Suspense boundary suspend, React sẽ giữ UI cũ hiển thị (thay vì show fallback ngay) và âm thầm render UI mới, chỉ swap khi sẵn sàng. Điều này tránh "flash of loading state" khi chuyển trang hoặc filter. Không có `startTransition`, React sẽ show fallback ngay lập tức.

**Q: Tại sao phải đặt ErrorBoundary bên ngoài Suspense?**
A: Suspense chỉ xử lý trường hợp throw Promise — tức là "đang chờ". Nó không bắt lỗi (throw Error). Nếu fetch thất bại và throw Error mà không có ErrorBoundary, lỗi sẽ bắn lên root và có thể crash toàn bộ app. ErrorBoundary xử lý "thất bại", Suspense xử lý "đang chờ" — hai vai trò tách biệt.

**Q: Suspense trên server (Streaming SSR) hoạt động khác gì trên client?**
A: Trên server, React render HTML shell ngay lập tức (phần ngoài Suspense boundary) và gửi về browser — TTFB thấp. Khi các Promise resolve, React stream thêm HTML fragments xuống browser kèm theo script nhỏ để insert vào đúng vị trí. Browser nhận và hiển thị từng phần khi đến. Trên client, React render fallback và swap sang real component khi Promise resolve — không có stream.

**Q: `useDeferredValue` và Suspense tương tác như thế nào?**
A: `useDeferredValue(value)` trả về một version "cũ" của giá trị trong khi version mới đang render. Nếu component dùng deferred value suspend, React hiển thị component với giá trị cũ (không show fallback), trong khi âm thầm render phiên bản với giá trị mới. Hữu ích cho search-as-you-type: user gõ typing không bị block, kết quả cũ vẫn hiển thị cho đến khi kết quả mới sẵn sàng.

### Scenario

**S: Dashboard có 5 widget độc lập. Mỗi widget fetch data riêng. Nếu một widget lỗi, các widget khác vẫn phải hiển thị. Giải pháp?**
A: Bọc mỗi widget trong `ErrorBoundary + Suspense` riêng. ErrorBoundary của từng widget catch lỗi cục bộ (chỉ ẩn widget đó, hiển thị error card), không ảnh hưởng đến phần còn lại của dashboard. Pattern: `<ErrorBoundary fallback={<WidgetError />}><Suspense fallback={<WidgetSkeleton />}><Widget /></Suspense></ErrorBoundary>`.

**S: User click chuyển trang, trang mới mất 1 giây để load và toàn bộ trang hiện Flash bằng fallback. Làm sao tránh?**
A: Bọc state update chuyển trang trong `startTransition`. React sẽ giữ trang cũ hiển thị (có thể với opacity mờ hoặc spinner nhỏ trong button) trong khi trang mới đang load. Với Next.js App Router, dùng `useRouter().push()` bên trong `startTransition`.

**S: Thư viện bạn dùng không hỗ trợ Suspense. Có nên tự implement "throw a Promise" không?**
A: Không nên trong production code. Giao thức này là "unofficial API" và có thể thay đổi. Thay vào đó: (1) Chuyển sang thư viện hỗ trợ Suspense (TanStack Query, SWR). (2) Hoặc quản lý `isLoading` / `isError` thủ công với `useState` — đơn giản hơn, rõ ràng hơn với thư viện không hỗ trợ.

**S: Component suspend liên tục (Promise resolve, render lại, throw lại ngay). Điều gì có thể gây ra?**
A: Thường là do data fetching được khởi tạo inside component không có caching — mỗi render tạo một Promise mới, khiến cache miss và suspend tiếp. Kiểm tra: (1) Query key có ổn định không? (2) Thư viện có cache kết quả giữa các render không? (3) Có ai invalidate cache ngay sau khi fetch xong không?

**S: TTFB của trang Next.js bị báo cáo là cao. Streaming SSR giải quyết được hoàn toàn không?**
A: Chỉ giải quyết một phần. Streaming SSR gửi HTML shell ngay lập tức (TTFB thấp về mặt cảm giác). Nhưng nếu shell cũng mất 500ms để render (vì query DB nặng ở Server Component trước Suspense boundary), TTFB vẫn cao. Cần phân tích bottleneck cụ thể: tối ưu query DB, dùng cache (Redis), hoặc chuyển query vào bên trong Suspense boundary để shell được gửi trước.

## 14. References

- React Docs — Suspense: https://react.dev/reference/react/Suspense
- React Docs — React.lazy: https://react.dev/reference/react/lazy
- React Docs — startTransition: https://react.dev/reference/react/startTransition
- React Docs — useDeferredValue: https://react.dev/reference/react/useDeferredValue
- React 18 Release + Concurrent Features: https://react.dev/blog/2022/03/29/react-v18
- Next.js — Loading UI and Streaming: https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming
- TanStack Query — Suspense Guide: https://tanstack.com/query/latest/docs/framework/react/guides/suspense
- react-error-boundary: https://github.com/bvaughn/react-error-boundary

## 15. Real-world Code

- Next.js examples — with-suspense: https://github.com/vercel/next.js/tree/canary/examples/with-suspense
- TanStack Query — Suspense examples: https://github.com/TanStack/query/tree/main/examples/react/suspense
- Vercel Commerce (Next.js App Router + Streaming SSR): https://github.com/vercel/commerce
- Bulletproof React (architecture patterns): https://github.com/alan2207/bulletproof-react

## 16. Community

- Reddit r/reactjs — Suspense data fetching discussions: https://www.reddit.com/r/reactjs/search/?q=suspense+data+fetching&sort=top
- Stack Overflow — react-suspense tag: https://stackoverflow.com/questions/tagged/react-suspense
- Kent C. Dodds — "React Suspense for data fetching": https://kentcdodds.com/blog/suspense-for-data-fetching-in-react-18
- Josh W Comeau — "The new wave of React state management": https://www.joshwcomeau.com/react/the-perils-of-rehydration/
- Reactiflux Discord — #suspense channel: https://www.reactiflux.com/
