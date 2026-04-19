---
created: 2026-04-17
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/javascript"
  - "#topic/i18n"
related: "[[react-i18next]]"
---

# i18next

## 1. What

i18next là một framework quốc tế hóa (internationalization) mạnh mẽ viết bằng JavaScript, hoạt động hoàn toàn độc lập với mọi UI framework. Nó cung cấp hệ thống dịch thuật đầy đủ tính năng bao gồm nội suy, số nhiều, ngữ cảnh, namespace, và kiến trúc plugin mở rộng. i18next là lớp lõi mà các binding như react-i18next, next-i18next và vue-i18next đều xây dựng dựa trên.

## 2. Why

Trước khi có i18next, developer phải tự xây dựng toàn bộ logic dịch thuật từ đầu: duy trì các object JSON lồng nhau thủ công, viết tay hàm tra cứu chuỗi theo key, xử lý số nhiều bằng `if/else` (sai với nhiều ngôn ngữ), và ghép nối chuỗi động dẫn đến lỗi khi dịch sang ngôn ngữ có cấu trúc câu khác tiếng Anh. Không có chuẩn chung nên mỗi dự án lại có cách tiếp cận riêng, không tái sử dụng được và cực kỳ khó bảo trì khi ứng dụng scale lên hàng chục ngôn ngữ.

## 3. Mental Model

Hãy tưởng tượng i18next như một **thư viện sách tra cứu đa ngôn ngữ**. Mỗi ngăn tủ là một **language** (`en`, `vi`, `ja`). Trong mỗi ngăn tủ có nhiều kệ — đó là các **namespace** (`common`, `auth`, `dashboard`). Mỗi kệ chứa một cuốn sách tra cứu — đó là file JSON translation. Khi bạn gọi `t('auth:login.title')`, thủ thư (i18next engine) đi thẳng đến ngăn tủ ngôn ngữ hiện tại, tìm kệ `auth`, lật đến mục `login`, và đọc mục con `title`. Nếu không tìm thấy, thủ thư thử ngăn tủ dự phòng (fallback language) trước khi chịu thua và trả về key gốc dưới dạng string. Không bao giờ throw error — luôn trả về thứ gì đó.

## 4. Where it fits

```
App Code  ->  i18n.t('namespace:key')
              |
              v
          i18next Core Instance
              |
              +---> ResourceStore (in-memory cache)
              |         |
              |         v
              |     Language Fallback Chain
              |     (e.g. en-US -> en -> fallbackLng)
              |
              +---> Interpolation Engine  {{ variable }}
              |
              +---> Pluralization Rules (per locale, CLDR)
              |
              v
          Final translated string -> Render
```

Plugins tích hợp vào trước khi `init()` chạy:

```
Browser/Server
  -> LanguageDetector Plugin  (URL / cookie / localStorage / navigator)
  -> HttpBackend Plugin       (lazy load JSON files từ CDN)
  -> initReactI18next Plugin  (nếu dùng với React)
  -> i18next Core
  -> App
```

## 5. When to use

- Ứng dụng cần hỗ trợ từ 2 ngôn ngữ trở lên, kể cả trong tương lai gần.
- Cần xử lý số nhiều phức tạp (tiếng Nga, Ả Rập có đến 6 dạng số nhiều).
- Cần lazy load translations để giảm initial bundle size.
- Dự án dùng nhiều platform khác nhau (React web + React Native + Node.js backend) và muốn dùng chung một hệ thống i18n.
- Cần tích hợp với Translation Management System (Locize, Phrase, Crowdin) cho workflow dịch thuật chuyên nghiệp.
- Team có nhiều người, cần tách translation theo feature để tránh conflict khi làm việc song song.

## 6. When NOT to use

- Ứng dụng nội bộ chỉ có một ngôn ngữ và không có roadmap mở rộng — thêm i18next lúc này là over-engineering.
- Microsite tĩnh đơn giản với vài chục chuỗi cố định — một object JavaScript đơn giản là đủ.
- Nếu dùng Next.js App Router (server components), cần setup riêng phía server và phía client vì i18next instance không shared được qua RSC boundary.

Hậu quả nếu dùng sai: thêm ~32KB vào bundle cho tính năng không cần thiết, tăng độ phức tạp của codebase, và tạo ra điểm lỗi tiềm ẩn khi configure sai plugin.

## 7. Trade-offs

| Pros | Cons |
|---|---|
| Ecosystem phong phú, hỗ trợ mọi JS framework | API khá verbose, learning curve ban đầu cao |
| Pluralization tuân theo chuẩn Unicode CLDR | Config ban đầu phức tạp với nhiều options |
| Plugin system linh hoạt, dễ mở rộng | Bundle size ~32KB (minified+gzipped) |
| Hỗ trợ lazy loading qua HttpBackend | Debug khi translation bị missing không trực quan |
| Fallback language chain mạnh mẽ | Pluralization rules thay đổi giữa các major version |
| Tích hợp tốt với các TMS (Locize, Phrase) | TypeScript type-safe keys cần thêm config phức tạp |
| Community lớn, tài liệu đầy đủ | SSR yêu cầu setup bổ sung, dễ bị hydration mismatch |

## 8. Alternatives

| Library | Bundle size | Pluralization | Lazy load | Đặc điểm nổi bật |
|---|---|---|---|---|
| **i18next** | ~32KB | CLDR-compliant | HttpBackend plugin | Ecosystem lớn nhất, linh hoạt nhất |
| **FormatJS / react-intl** | ~25KB | CLDR-compliant | Manual code splitting | Chuẩn ICU message format, dùng bởi Airbnb |
| **Lingui** | ~5KB runtime | Tốt | Babel/SWC compile-time | Bundle nhỏ, extract keys qua AST |
| **Paraglide JS** | <1KB runtime | Tốt | Tree-shaking per message | Mới nhất, bundle cực nhỏ |
| **Custom solution** | ~0KB | Tự implement | Tự implement | Phù hợp khi yêu cầu cực kỳ đơn giản |

Chọn i18next khi cần ecosystem và plugin đa dạng. Chọn FormatJS khi cần chuẩn ICU message format (nhiều translator tool hỗ trợ). Chọn Lingui khi ưu tiên bundle size tối thiểu.

## 9. How

**Cài đặt:**

```bash
npm install i18next i18next-http-backend i18next-browser-languagedetector
```

**Khởi tạo (`src/i18n.js`):**

```javascript
import i18next from 'i18next';
import HttpBackend from 'i18next-http-backend';
import LanguageDetector from 'i18next-browser-languagedetector';

const i18n = i18next
  .use(HttpBackend)
  .use(LanguageDetector);

await i18n.init({
  fallbackLng: 'en',
  supportedLngs: ['en', 'vi', 'ja'],
  defaultNS: 'common',
  ns: ['common'],          // Chỉ preload namespace này
  backend: {
    loadPath: '/locales/{{lng}}/{{ns}}.json',
  },
  detection: {
    order: ['querystring', 'cookie', 'localStorage', 'navigator'],
    caches: ['localStorage', 'cookie'],
    lookupQuerystring: 'lng',
    lookupCookie: 'i18next',
    lookupLocalStorage: 'i18nextLng',
  },
  interpolation: {
    escapeValue: true,     // Escape HTML để tránh XSS
  },
  saveMissing: process.env.NODE_ENV === 'development',
  missingKeyHandler: (lngs, ns, key) => {
    console.warn(`[i18n] Missing key: ${ns}:${key} for langs: ${lngs}`);
  },
});

export default i18n;
```

**File translation (`public/locales/en/common.json`):**

```json
{
  "greeting": "Hello, {{name}}!",
  "itemCount_one": "{{count}} item",
  "itemCount_other": "{{count}} items",
  "nav": {
    "home": "Home",
    "about": "About Us"
  },
  "status": {
    "active": "Active",
    "inactive": "Inactive"
  }
}
```

**File translation tiếng Việt (`public/locales/vi/common.json`):**

```json
{
  "greeting": "Xin chào, {{name}}!",
  "itemCount": "{{count}} mục",
  "nav": {
    "home": "Trang chủ",
    "about": "Giới thiệu"
  },
  "status": {
    "active": "Đang hoạt động",
    "inactive": "Không hoạt động"
  }
}
```

**Sử dụng các tính năng cốt lõi:**

```javascript
// Chờ init xong (nếu không dùng React Suspense)
await i18n.init(config);

// Simple key lookup
i18n.t('nav.home');
// -> "Home"

// Interpolation: thay {{name}} bằng giá trị thực
i18n.t('greeting', { name: 'An Phuc' });
// -> "Hello, An Phuc!"

// Pluralization: i18next tự chọn key _one hay _other
i18n.t('itemCount', { count: 1 });   // -> "1 item"
i18n.t('itemCount', { count: 5 });   // -> "5 items"

// Namespace cụ thể (dùng dấu : để chỉ định)
i18n.t('auth:login.title');

// Đổi ngôn ngữ runtime (returns Promise)
await i18n.changeLanguage('vi');
i18n.t('nav.home');  // -> "Trang chủ"

// Đọc ngôn ngữ hiện tại
console.log(i18n.language);          // 'vi'
console.log(i18n.languages);         // ['vi', 'en'] (fallback chain)

// Kiểm tra key có tồn tại không
i18n.exists('nav.home');             // true
i18n.exists('nav.nonexistent');      // false
```

**Pluralization cho tiếng Nga (6 dạng theo CLDR):**

```json
// ru/common.json
{
  "apples_one":   "{{count}} яблоко",
  "apples_few":   "{{count}} яблока",
  "apples_many":  "{{count}} яблок",
  "apples_other": "{{count}} яблока"
}
```

```javascript
i18n.t('apples', { count: 1 });   // "1 яблоко"
i18n.t('apples', { count: 3 });   // "3 яблока"
i18n.t('apples', { count: 11 });  // "11 яблок"
i18n.t('apples', { count: 21 });  // "21 яблоко"
```

**Context (ngữ cảnh như giới tính):**

```json
{
  "friend": "A friend",
  "friend_male": "A male friend",
  "friend_female": "A female friend"
}
```

```javascript
i18n.t('friend');                        // "A friend"
i18n.t('friend', { context: 'female' }); // "A female friend"
```

**Thêm resources động (không cần reload trang):**

```javascript
// Load thêm một namespace mới vào runtime
await i18n.loadNamespaces('dashboard');

// Thêm resource bundle thủ công
i18n.addResourceBundle('en', 'featureX', {
  title: 'Feature X',
  description: 'This does something cool',
}, true, true);
// true, true = deep merge + overwrite nếu đã có
```

## 10. Production Concerns

**Scaling:**

Split translations thành nhiều namespace nhỏ (mỗi file <20KB) và lazy load theo route. Cache translation files ở CDN với `Cache-Control: max-age=31536000, immutable` và dùng content hash trong filename để cache bust khi cập nhật.

```
/locales/en/common.abc123.json    <- cache 1 năm
/locales/en/auth.def456.json      <- chỉ load khi vào trang auth
/locales/en/dashboard.ghi789.json <- chỉ load khi vào dashboard
```

Với ứng dụng quy mô lớn (50+ ngôn ngữ), dùng TMS như Locize thay vì commit JSON vào repo để tránh Git repo phình to và hỗ trợ workflow dịch thuật cho non-technical translator.

**Failure:**

Nếu HttpBackend thất bại (mạng chậm, 404), i18next fallback về `fallbackLng` nếu đã được load. Nếu cả fallback chưa load, `t('key')` trả về key string thô — behavior an toàn nhưng cần monitor. Implement retry logic trong backend plugin hoặc service worker để cache translation files offline.

**Monitoring:**

```javascript
i18next.init({
  saveMissing: true,
  missingKeyHandler: (lngs, ns, key, fallbackValue) => {
    // Gửi lên error tracking service
    Sentry.captureMessage(`Missing i18n key: ${ns}:${key}`, {
      level: 'warning',
      extra: { lngs, fallbackValue },
    });
  },
});
```

Theo dõi: tỉ lệ fallback key được serve thay vì translation thật, thời gian load của HttpBackend, số lần `changeLanguage()` được gọi để hiểu language distribution của user.

## 11. Common Mistakes

- Mistake: Gọi `t()` trước khi i18next hoàn tất `init()`, dẫn đến trả về key thô thay vì translation.
  Fix: Luôn `await i18next.init()` trước khi render app, hoặc dùng `i18next.on('initialized', callback)`. Với React, dùng `useSuspense: true` và wrap component trong `<Suspense>`.

- Mistake: Dùng key là câu tiếng Anh đầy đủ như `t('Hello World')`, khiến JSON phình to và refactor khó.
  Fix: Dùng key theo cấu trúc có namespace: `t('home:hero.headline')`. Key ngắn, dễ quản lý, không bị phá vỡ khi đổi nội dung bản dịch.

- Mistake: Ghép chuỗi để xử lý số nhiều: `` `${count} ${t('item')}${count > 1 ? 's' : ''}` ``.
  Fix: Dùng `t('itemCount', { count })` với các key `_one` và `_other` trong JSON. Logic ngôn ngữ nằm trong file dịch, không nằm trong code.

- Mistake: Load tất cả namespaces ngay từ đầu trong `ns: ['common', 'auth', 'dashboard', 'settings', ...]`.
  Fix: Chỉ preload namespace thật sự cần thiết cho initial render. Load các namespace còn lại bằng `i18next.loadNamespaces()` khi user navigate đến route tương ứng.

- Mistake: Không set `escapeValue: true` (hoặc tắt đi), rồi render translation chứa nội dung user-generated vào innerHTML.
  Fix: Giữ `escapeValue: true` (đây là default). Khi dùng với React, set `escapeValue: false` vì React đã tự escape — nhưng đảm bảo không dùng `dangerouslySetInnerHTML` với translation.

## 12. Sample Project

**Project: Multi-language documentation site**

Constraint cứng: Toàn bộ translations phải lazy load theo route. Không được bundle bất kỳ translation nào vào initial JS chunk ngoài namespace `layout` (tối đa 15 keys cho header/footer).

**Cấu trúc file:**

```
public/
  locales/
    en/
      layout.json      # 15 keys: header, footer, nav
      home.json        # Load tại route /
      docs.json        # Load tại route /docs/*
      api-ref.json     # Load tại route /api-reference/*
    vi/
      (tương tự)
src/
  i18n.js              # Config với preload chỉ ['layout']
  components/
    Layout.jsx         # useTranslation('layout')
  pages/
    Home.jsx           # useTranslation('home') + Suspense
    Docs.jsx           # useTranslation('docs') + Suspense
    ApiReference.jsx   # useTranslation('api-ref') + Suspense
```

**i18n config chỉ preload `layout`:**

```javascript
await i18next.use(HttpBackend).use(LanguageDetector).init({
  preload: ['en'],
  ns: ['layout'],
  defaultNS: 'layout',
  backend: { loadPath: '/locales/{{lng}}/{{ns}}.json' },
  fallbackLng: 'en',
});
```

**Route component dùng Suspense để lazy load:**

```javascript
// pages/Docs.jsx
import { Suspense } from 'react';
import { useTranslation } from 'react-i18next';

function DocsContent() {
  // Gọi useTranslation với namespace 'docs' trigger lazy load
  const { t } = useTranslation('docs');
  return (
    <main>
      <h1>{t('title')}</h1>
      <p>{t('intro')}</p>
    </main>
  );
}

export default function Docs() {
  return (
    <Suspense fallback={<div>Loading translations...</div>}>
      <DocsContent />
    </Suspense>
  );
}
```

**Kết quả đạt được:** Initial bundle chỉ load ~3KB translation JSON (layout). Mỗi route chỉ load thêm translation của route đó khi user điều hướng đến — giảm ~70% translation data trên mỗi page view.

## 13. Interview

**Core Q&A:**

Q: i18next namespace là gì và tại sao cần dùng?
A: Namespace là cách tổ chức translation resources thành các nhóm riêng biệt theo domain hoặc feature, mỗi nhóm là một file JSON. Lợi ích: (1) lazy load từng namespace khi cần thay vì load tất cả, (2) tránh naming conflicts khi nhiều team làm việc song song, (3) tăng maintainability vì mỗi team chỉ chịu trách nhiệm file của mình.

Q: Interpolation trong i18next hoạt động như thế nào?
A: i18next dùng `{{variableName}}` làm placeholder trong string. Khi gọi `t('key', { variableName: value })`, interpolation engine replace placeholder bằng value truyền vào. Mặc định bật `escapeValue: true` để escape HTML entities, tránh XSS. Khi dùng với React, nên set `escapeValue: false` vì React đã tự escape.

Q: Giải thích cách i18next xử lý pluralization?
A: i18next tuân theo Unicode CLDR plural rules. Mỗi ngôn ngữ có số dạng số nhiều khác nhau (tiếng Anh: 2 dạng `one/other`; tiếng Nga: 4 dạng `one/few/many/other`; tiếng Ả Rập: 6 dạng). Developer định nghĩa key với suffix tương ứng (`_one`, `_few`, `_many`, `_other`) trong JSON. Khi gọi `t('key', { count: n })`, i18next tự tính toán dạng số nhiều đúng dựa trên giá trị `n` và locale hiện tại.

Q: `fallbackLng` và `fallbackNS` khác nhau như thế nào?
A: `fallbackLng` là ngôn ngữ dự phòng khi key không tìm thấy trong ngôn ngữ hiện tại (ví dụ: `en-US` -> `en` -> `fallbackLng: 'en'`). `fallbackNS` là namespace dự phòng khi key không tìm thấy trong namespace đang dùng — thường set là `'common'` để share translation chung giữa các feature.

Q: `i18next.exists('key')` có tác dụng gì?
A: Kiểm tra xem một key có tồn tại trong ResourceStore hiện tại hay không, trả về boolean. Hữu ích khi xây dựng conditional rendering: ví dụ chỉ hiển thị optional tooltip nếu có translation cho nó, thay vì hiển thị key thô.

Q: Plugin LanguageDetector detect ngôn ngữ từ những nguồn nào?
A: Theo thứ tự trong `detection.order`: `querystring` (`?lng=vi`), `cookie`, `localStorage`, `sessionStorage`, `navigator` (Accept-Language header của browser), `htmlTag` (`<html lang="vi">`), `path` (`/vi/about`), `subdomain` (`vi.example.com`). Thứ tự này có thể tùy chỉnh.

Q: Tại sao không nên commit translation files lớn trực tiếp vào Git?
A: Với ứng dụng lớn (50+ ngôn ngữ, hàng nghìn keys), mỗi lần cập nhật translation sẽ tạo ra rất nhiều diff trong Git, làm chậm CI và làm phình repository. Ngoài ra, non-technical translator không thể làm việc trực tiếp với Git. Giải pháp: dùng TMS (Locize, Phrase, Crowdin) lưu translations riêng và đồng bộ qua API.

**Scenario:**

Q: App hiển thị key `dashboard:stats.totalUsers` thô thay vì text. Debug bước nào?
A: Kiểm tra theo thứ tự: (1) Mở Network tab xem request `/locales/en/dashboard.json` có trả về 200 không. (2) Kiểm tra file JSON có key `stats.totalUsers` đúng cú pháp không (trailing comma, Unicode escape). (3) Kiểm tra namespace `dashboard` đã được load chưa qua `i18n.hasResourceBundle('en', 'dashboard')`. (4) Kiểm tra i18next đã init xong chưa: `i18n.isInitialized`. (5) Kiểm tra `defaultNS` hoặc prefix namespace trong lời gọi `t()`.

Q: Cần hiển thị số notification với đúng grammar cho 6 ngôn ngữ. Approach?
A: Định nghĩa key `notification` trong từng file ngôn ngữ với đủ các suffix theo CLDR của ngôn ngữ đó (`_one`, `_other` cho tiếng Anh; `_one`, `_few`, `_many`, `_other` cho tiếng Nga; v.v.). Gọi `t('notification', { count: n })` — i18next tự chọn đúng form. Tuyệt đối không viết `count > 1 ? t('notifications') : t('notification')` trong code.

Q: Translation file phình lên 800KB, làm tăng TTI. Giải pháp?
A: (1) Audit bằng `i18next-parser` để xóa key không dùng. (2) Split thành nhiều namespace nhỏ theo feature/route. (3) Dùng HttpBackend với lazy loading — chỉ load namespace khi route tương ứng được visit. (4) Bật gzip/Brotli compression ở CDN (JSON compress rất tốt, giảm 70-80%). (5) Với Next.js, dùng `getStaticProps` để chỉ pass translation cần thiết cho từng page xuống client.

## 14. References

- i18next official documentation: https://www.i18next.com/
- Configuration options: https://www.i18next.com/overview/configuration-options
- Pluralization guide: https://www.i18next.com/translation-function/plurals
- Interpolation guide: https://www.i18next.com/translation-function/interpolation
- Context feature: https://www.i18next.com/translation-function/context
- i18next-http-backend GitHub: https://github.com/i18next/i18next-http-backend
- i18next-browser-languagedetector GitHub: https://github.com/i18next/i18next-browser-languageDetector
- Unicode CLDR plural rules: https://cldr.unicode.org/index/cldr-spec/plural-rules
- i18next-parser (extract keys from source): https://github.com/i18next/i18next-parser

## 15. Real-world Code

- Outline (open-source Wiki) — i18next setup với HttpBackend: https://github.com/outline/outline/blob/main/app/utils/i18n.ts
- Excalidraw — i18n config và lazy loading: https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/i18n.ts
- Grafana frontend — large-scale i18next usage: https://github.com/grafana/grafana/blob/main/public/app/core/utils/i18n.ts
- Cal.com — Next.js + i18next setup với many locales: https://github.com/calcom/cal.com/tree/main/apps/web/public/static/locales
- Docusaurus — i18n plugin wrapping i18next: https://github.com/facebook/docusaurus/tree/main/packages/docusaurus-theme-translations

## 16. Community

- i18next GitHub Discussions (official support): https://github.com/i18next/i18next/discussions
- Stack Overflow — i18next tag (10,000+ câu hỏi): https://stackoverflow.com/questions/tagged/i18next
- Locize Blog (viết bởi i18next creators, nhiều deep-dive): https://locize.com/blog/
- DEV.to — "How does i18next work under the hood?": https://dev.to/adrai/how-does-i18next-work-under-the-hood-3po1
- LogRocket Blog — React Internationalization with i18next: https://blog.logrocket.com/react-internationalization-i18next/
- r/reactjs — discussions về i18n library choices: https://www.reddit.com/r/reactjs/search/?q=i18next
