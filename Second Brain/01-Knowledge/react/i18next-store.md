---
created: 2026-04-17
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/javascript"
  - "#topic/i18n"
related:
  - "[[i18next]]"
  - "[[react-i18next]]"
---

# i18next Store Architecture

## 1. What

i18next Store là hệ thống lưu trữ in-memory nơi tất cả translation resources được cache và quản lý trong suốt lifecycle của ứng dụng. Nó bao gồm `ResourceStore` (lớp chính) và `ResourceLanguage`/`ResourceNamespace` objects bên trong. Mọi thao tác dịch thuật — từ `t()` lookup đến lazy loading HTTP backend — đều đi qua store này. Hiểu store giúp debug translation issues và tối ưu performance.

## 2. Why

Nếu không có store tập trung, mỗi lần gọi `t('key')` sẽ phải đọc lại file translation từ disk hoặc network — cực kỳ chậm. Store giải quyết bằng cách load một lần, cache trong memory, và serve tất cả subsequent lookups từ RAM. Ngoài ra, store cung cấp single source of truth để quản lý state phức tạp như fallback chain, lazy-loaded namespaces, và merge của resources từ nhiều nguồn khác nhau (static bundle, HTTP backend, runtime additions).

## 3. Mental Model

Hãy nghĩ ResourceStore như một **Redis cache với cấu trúc 3 cấp lồng nhau**. Cấp 1 là language code (`'en'`, `'vi'`). Cấp 2 là namespace (`'common'`, `'auth'`). Cấp 3 là key-value pairs của translations.

```
ResourceStore (in-memory)
{
  'en': {                           <- ResourceLanguage
    'common': {                     <- ResourceNamespace
      'nav.home': 'Home',
      'nav.about': 'About Us',
      'greeting': 'Hello, {{name}}!'
    },
    'auth': {
      'login.title': 'Sign In',
      'login.submitButton': 'Log In'
    }
  },
  'vi': {
    'common': {
      'nav.home': 'Trang chủ',
      ...
    }
  }
}
```

Khi bạn gọi `t('common:nav.home')`, i18next thực hiện lookup như một database query: `store['en']['common']['nav.home']`. Nếu không tìm thấy, thử `store['en']['common']['nav.home']` trong fallback namespace, rồi thử `store['en-fallback']['common']['nav.home']`.

## 4. Where it fits

```
i18next initialization
  |
  +-> Plugins register backends/detectors
  |
  +-> init() runs
        |
        +-> LanguageDetector determines active language
        |
        +-> ResourceStore created (empty)
        |
        +-> Backend plugin (HttpBackend) fetches JSON files
        |         |
        |         v
        +-> addResourceBundle() populates store
        |
        v
    Store is ready: { 'en': { 'common': {...} } }
        |
        v
    t('key') -> Store.lookup(language, namespace, key)
                  -> Language Fallback Chain if not found
                  -> Interpolation Engine
                  -> Return translated string
```

Lazy loading flow:

```
useTranslation('dashboard')  <- Component requests namespace
  |
  v
Store.hasResourceBundle('en', 'dashboard')?
  |
  NO -> HttpBackend fetches /locales/en/dashboard.json
  |         |
  |         v
  |    addResourceBundle('en', 'dashboard', data)
  |         |
  v         v
  YES -> Store serves data directly (cache hit)
```

## 5. When to use

Hiểu về Store Architecture cần thiết khi:

- Debug tại sao một key cụ thể hiển thị thay vì translation — cần biết store có chứa namespace đó không.
- Implement dynamic translation loading (load translations từ API response, không phải static files).
- Build custom plugin cho i18next cần đọc/ghi vào store.
- Tối ưu memory khi app cần hỗ trợ nhiều ngôn ngữ lớn — biết khi nào remove resources không dùng.
- Testing: populate store trực tiếp thay vì mock HTTP calls.

## 6. When NOT to use

- Không nên manipulate store trực tiếp cho các use case thông thường — dùng API public như `t()`, `useTranslation`, `addResourceBundle()`.
- Không nên giữ reference đến internal store object và đọc/ghi trực tiếp vào `store.data` — đây là internal implementation detail có thể thay đổi giữa các phiên bản.

Hậu quả: Code phụ thuộc vào internal store structure sẽ bị break khi update i18next major version.

## 7. Trade-offs

| Pros | Cons |
|---|---|
| Lookup cực nhanh O(1) — tra cứu trong object JS | Toàn bộ translation data sống trong RAM |
| Single source of truth cho mọi translation state | Memory tăng tuyến tính theo số ngôn ngữ x namespaces |
| Hỗ trợ lazy addition resources tại runtime | Internal structure là private API, dễ break khi update |
| Language fallback chain xử lý tự động | Không có TTL — resources không tự expire |
| Có thể serialize/deserialize cho SSR | Debugging store state cần biết internal API |

## 8. Alternatives

| Approach | Mô tả | Khi dùng |
|---|---|---|
| **i18next ResourceStore (default)** | In-memory, JavaScript object | Mọi app thông thường |
| **Custom backend plugin** | Fetch từ custom source (database, CMS API) | Khi translations quản lý ở backend |
| **Pre-bundled resources** | Import JSON trực tiếp vào JS bundle | App nhỏ, cần instant availability |
| **i18next-localstorage-backend** | Cache trong localStorage | Giảm network requests khi user quay lại |

## 9. How

**Đọc state của store:**

```javascript
import i18n from './i18n';

// Sau khi init() hoàn tất:

// Kiểm tra một resource bundle có tồn tại không
const hasEnCommon = i18n.hasResourceBundle('en', 'common');
// -> true nếu đã load, false nếu chưa

// Lấy toàn bộ data của một namespace
const authTranslations = i18n.getResourceBundle('en', 'auth');
// -> { 'login.title': 'Sign In', 'login.submitButton': 'Log In', ... }

// Lấy một giá trị cụ thể từ store
const value = i18n.getResource('en', 'common', 'nav.home');
// -> 'Home'

// Kiểm tra xem một key có tồn tại không (không cần biết value)
const exists = i18n.exists('common:nav.home');
// -> true
```

**Thêm resources vào store:**

```javascript
// Thêm toàn bộ namespace (ví dụ: nhận từ API response)
i18n.addResourceBundle(
  'en',           // language
  'featureX',     // namespace
  {               // data
    title: 'Feature X',
    description: 'This is a dynamic feature',
  },
  true,           // deep: merge với data đã có (default: false = replace)
  true            // overwrite: ghi đè key đã tồn tại (default: false)
);

// Thêm nhiều keys lẻ (không ghi đè toàn bộ namespace)
i18n.addResources('en', 'common', {
  'buttons.save': 'Save Changes',
  'buttons.cancel': 'Cancel',
});

// Thêm một giá trị đơn lẻ
i18n.addResource('en', 'common', 'nav.settings', 'Settings');
```

**Language Fallback Chain:**

Đây là tính năng quan trọng nhất của store — cơ chế dự phòng khi key không tìm thấy:

```javascript
// Config:
i18n.init({
  lng: 'en-US',
  fallbackLng: {
    'en-US': ['en', 'dev'],  // en-US -> en -> dev
    'vi-VN': ['vi', 'en'],   // vi-VN -> vi -> en
    default: ['en'],          // Mọi ngôn ngữ khác -> en
  },
});
```

```
t('someKey') với lng = 'en-US':

Bước 1: Tìm trong store['en-US']['namespace']['someKey']
  -> Không có 'en-US' bundle? Sang bước 2

Bước 2: Tìm trong store['en']['namespace']['someKey']
  -> Tìm thấy! Trả về giá trị.

Nếu bước 2 cũng không có:
Bước 3: Tìm trong store['dev']['namespace']['someKey']
  -> 'dev' language là special: dùng cho developer defaults

Nếu tất cả đều không có:
Bước 4: Trả về key string ('someKey') như fallback cuối cùng
```

**Ví dụ thực tế — partial translation:**

```json
// vi/common.json (bản dịch chưa hoàn chỉnh)
{
  "nav.home": "Trang chủ",
  "nav.about": "Giới thiệu"
  // nav.contact CHƯA được dịch
}

// en/common.json (đầy đủ)
{
  "nav.home": "Home",
  "nav.about": "About Us",
  "nav.contact": "Contact Us"
}
```

```javascript
i18n.init({ lng: 'vi', fallbackLng: 'en' });

i18n.t('nav.home');     // -> "Trang chủ" (tìm thấy trong vi)
i18n.t('nav.contact');  // -> "Contact Us" (vi không có, fallback sang en)
```

**Lazy loading namespaces và store update:**

```javascript
// Component yêu cầu namespace 'reporting' chưa load
await i18n.loadNamespaces(['reporting']);
// -> HttpBackend fetch /locales/en/reporting.json
// -> addResourceBundle('en', 'reporting', data) tự động
// -> Store bây giờ có: store['en']['reporting'] = {...}

console.log(i18n.hasResourceBundle('en', 'reporting')); // true
```

**Xóa resources khỏi store (memory management):**

```javascript
// Xóa một namespace không còn cần (ví dụ: user logout khỏi tính năng premium)
i18n.removeResourceBundle('en', 'premiumFeatures');

// Kiểm tra:
i18n.hasResourceBundle('en', 'premiumFeatures'); // false
```

**Serialize store cho SSR:**

```javascript
// Server-side: serialize state sau khi render
const serverI18nState = {
  initialLanguage: i18n.language,
  initialI18nStore: i18n.services.resourceStore.data,
};

// HTML output
const html = `
  <script>
    window.__I18N_STATE__ = ${JSON.stringify(serverI18nState)};
  </script>
`;

// Client-side: rehydrate từ server state
i18n.init({
  ...config,
  lng: window.__I18N_STATE__.initialLanguage,
  resources: window.__I18N_STATE__.initialI18nStore,
  // i18next skip HTTP fetch vì resources đã có sẵn
});
```

**Debug store trong development:**

```javascript
// Log toàn bộ store state
console.log(JSON.stringify(i18n.services.resourceStore.data, null, 2));

// Nghe sự kiện khi store thay đổi
i18n.on('added', (lng, ns) => {
  console.log(`[i18n store] Added: ${lng}/${ns}`);
});

i18n.on('removed', (lng, ns) => {
  console.log(`[i18n store] Removed: ${lng}/${ns}`);
});

// Hook vào missing key event để detect khi store thiếu key
i18n.on('missingKey', (lngs, namespace, key, res) => {
  console.warn(`[i18n store] Missing: ${namespace}:${key} in ${lngs}`);
});
```

## 10. Production Concerns

**Memory:**

Store giữ tất cả translations trong RAM. Tính toán ước lượng: 1000 keys x 50 bytes/key x 5 ngôn ngữ x 10 namespaces = ~2.5MB. Đây thường không phải vấn đề với app web desktop. Với mobile web hoặc low-end devices, cân nhắc `removeResourceBundle()` để unload namespaces của các route không còn active.

**Scaling:**

Với >50 ngôn ngữ, không preload tất cả — dùng lazy loading và chỉ giữ trong store ngôn ngữ hiện tại + 1-2 fallback. Implement LRU-like eviction bằng cách track namespace usage và remove những namespace không được dùng trong >30 phút.

**Failure:**

Nếu HttpBackend fetch thất bại và namespace chưa có trong store, `t()` sẽ trả về key string. Implement retry mechanism và error boundary để hiển thị graceful fallback UI thay vì key strings.

**Monitoring:**

Track `store.added` events để measure namespace loading performance. So sánh số lượng unique namespaces được load với số namespace tối đa dự kiến để phát hiện namespace leak.

## 11. Common Mistakes

- Mistake: Đọc trực tiếp từ `i18n.services.resourceStore.data` trong production code để fetch translations.
  Fix: Dùng API public: `i18n.getResourceBundle(lng, ns)` hoặc `i18n.getResource(lng, ns, key)`. Internal `store.data` là implementation detail.

- Mistake: Gọi `addResourceBundle` với `overwrite: false` (default) rồi thắc mắc tại sao updates không có tác dụng sau khi hot-reload translation files trong development.
  Fix: Trong development, gọi `addResourceBundle(lng, ns, data, true, true)` với cả `deep: true` và `overwrite: true` để override existing data.

- Mistake: Không kiểm tra `hasResourceBundle()` trước khi gọi `getResourceBundle()`, dẫn đến `undefined` khi namespace chưa load.
  Fix: Luôn kiểm tra `hasResourceBundle()` trước, hoặc dùng `await i18n.loadNamespaces([ns])` để đảm bảo namespace đã load trước khi access.

- Mistake: Serialize `i18n.services.resourceStore.data` vào SSR HTML mà không sanitize, dẫn đến XSS nếu translations chứa script tags.
  Fix: Dùng `JSON.stringify` với proper escaping: `JSON.stringify(data).replace(/</g, '\\u003c')` để tránh `</script>` injection.

- Mistake: Quên rằng store là shared mutable state trong SSR — nhiều concurrent requests dùng chung một i18n instance, gây race condition khi gọi `changeLanguage()`.
  Fix: Trong SSR, tạo một i18n instance riêng cho mỗi request thay vì dùng shared singleton:
  ```javascript
  const i18nForRequest = i18n.cloneInstance({ lng: detectedLang });
  ```

## 12. Sample Project

**Project: Debugging tool cho i18n — Translation Inspector**

Constraint cứng: Phải hiển thị real-time state của ResourceStore trong dev mode, bao gồm danh sách namespaces đã load, keys bị missing, và fallback chain visualization. Không được dùng external state management library — chỉ dùng i18next store APIs.

**TranslationInspector component:**

```jsx
import { useState, useEffect } from 'react';
import { useTranslation } from 'react-i18next';

function TranslationInspector() {
  const { i18n } = useTranslation();
  const [storeSnapshot, setStoreSnapshot] = useState({});
  const [missingKeys, setMissingKeys] = useState([]);

  useEffect(() => {
    // Chụp snapshot của store hiện tại
    const takeSnapshot = () => {
      const data = i18n.services.resourceStore.data;
      const snapshot = {};
      Object.keys(data).forEach((lng) => {
        snapshot[lng] = Object.keys(data[lng]).map((ns) => ({
          namespace: ns,
          keyCount: Object.keys(data[lng][ns]).length,
          keys: Object.keys(data[lng][ns]).slice(0, 5), // Preview 5 keys
        }));
      });
      setStoreSnapshot(snapshot);
    };

    takeSnapshot();

    // Cập nhật snapshot khi namespace được thêm
    i18n.on('added', takeSnapshot);
    i18n.on('removed', takeSnapshot);

    // Track missing keys
    const missingHandler = (lngs, ns, key) => {
      setMissingKeys((prev) => [
        ...prev.slice(-19), // Giữ 20 entries gần nhất
        { lngs, ns, key, timestamp: new Date().toISOString() },
      ]);
    };
    i18n.on('missingKey', missingHandler);

    return () => {
      i18n.off('added', takeSnapshot);
      i18n.off('removed', takeSnapshot);
      i18n.off('missingKey', missingHandler);
    };
  }, [i18n]);

  if (process.env.NODE_ENV !== 'development') return null;

  return (
    <details style={{ position: 'fixed', bottom: 0, right: 0, background: '#fff', border: '1px solid #ccc', padding: '8px', maxHeight: '300px', overflow: 'auto', zIndex: 9999 }}>
      <summary>i18n Store Inspector</summary>

      <h4>Loaded Resources</h4>
      {Object.entries(storeSnapshot).map(([lng, namespaces]) => (
        <div key={lng}>
          <strong>{lng}</strong>
          {namespaces.map(({ namespace, keyCount }) => (
            <div key={namespace} style={{ marginLeft: '12px' }}>
              {namespace}: {keyCount} keys
              {' '}
              <button onClick={() => i18n.removeResourceBundle(lng, namespace)}>
                Unload
              </button>
            </div>
          ))}
        </div>
      ))}

      <h4>Missing Keys ({missingKeys.length})</h4>
      {missingKeys.map(({ ns, key, lngs, timestamp }, idx) => (
        <div key={idx} style={{ color: 'red', fontSize: '12px' }}>
          [{timestamp.slice(11, 19)}] {ns}:{key} (tried: {lngs.join(', ')})
        </div>
      ))}

      <h4>Fallback Chain</h4>
      <pre style={{ fontSize: '11px' }}>
        {JSON.stringify(i18n.languages, null, 2)}
      </pre>
    </details>
  );
}
```

Kết quả: Developer thấy được real-time state của store, phát hiện missing keys ngay khi chúng xảy ra, và có thể manually unload namespaces để test lazy loading behavior.

## 13. Interview

**Core Q&A:**

Q: i18next ResourceStore lưu data theo cấu trúc nào?
A: Ba cấp lồng nhau: `{ [language]: { [namespace]: { [key]: value } } }`. Ví dụ: `{ 'en': { 'common': { 'nav.home': 'Home' } } }`. Lookup là O(1) vì dùng JavaScript object property access.

Q: Language fallback chain hoạt động như thế nào khi key bị missing?
A: i18next duyệt qua `i18n.languages` array theo thứ tự — mảng này được build từ `lng` + `fallbackLng` config. Với mỗi language, tìm key trong namespace hiện tại rồi `fallbackNS`. Nếu không tìm thấy sau khi duyệt toàn bộ chain, trả về key string hoặc `defaultValue` nếu được cung cấp.

Q: `addResourceBundle` với deep=true và overwrite=true khác gì deep=false?
A: Với `deep: false`: thay thế toàn bộ namespace data bằng object mới. Với `deep: true`: merge sâu (recursive merge) với data hiện có. `overwrite: true` cho phép ghi đè key đã tồn tại trong merge; `overwrite: false` giữ nguyên key cũ.

Q: Tại sao SSR với i18next dễ gây race condition?
A: i18next instance mặc định là singleton. Trong SSR, nhiều requests concurrent cùng share một instance. Nếu request A gọi `changeLanguage('vi')` trong khi request B đang render với `'en'`, store state bị corrupt. Fix: dùng `i18n.cloneInstance()` tạo instance riêng cho mỗi request, hoặc truyền language context qua React Context thay vì mutate global instance.

Q: Làm thế nào để pre-populate store trong unit tests mà không cần mock HTTP?
A: Dùng `resources` option trong `init()` để cung cấp translations trực tiếp, bypass hoàn toàn backend plugin:
```javascript
i18n.init({
  resources: {
    en: { common: { 'nav.home': 'Home' } },
  },
  lng: 'en',
});
```
Hoặc sau init: `i18n.addResourceBundle('en', 'test', { key: 'value' })`.

Q: Khi nào nên dùng `removeResourceBundle()`?
A: Khi app cần giải phóng memory — ví dụ: single-page app với nhiều route lớn, sau khi user navigate khỏi một feature nặng. Cũng dùng trong tests để reset state giữa các test cases. Trong production web thường không cần vì browser refresh sẽ clear memory.

**Scenario:**

Q: App hiển thị "en" translations cho một số keys dù user đã chọn "vi". Debug store?
A: Chạy `i18n.hasResourceBundle('vi', 'namespaceName')` — nếu false, namespace `vi` chưa được load. Check Network tab xem `/locales/vi/namespace.json` có được request không. Nếu file 404, fallback chain tự động dùng `en`. Nếu file tồn tại nhưng thiếu key, fallback sang `en` cho đúng key đó. Dùng `i18n.getResourceBundle('vi', 'namespaceName')` để inspect data thực tế trong store.

Q: Bạn cần load translations từ database thay vì JSON files. Approach với store?
A: Implement custom backend plugin:
```javascript
const DatabaseBackend = {
  type: 'backend',
  init(services, backendOptions) { /* save db connection */ },
  read(language, namespace, callback) {
    // Fetch từ database
    db.getTranslations(language, namespace)
      .then((data) => callback(null, data))
      .catch((err) => callback(err, null));
  },
};
i18n.use(DatabaseBackend).init(config);
```
Plugin sẽ tự động gọi `addResourceBundle()` để populate store khi backend gọi `callback(null, data)`.

Q: Sau khi deploy bản dịch mới, user cần refresh trang mới thấy. Làm sao update store mà không reload?
A: Implement polling hoặc WebSocket để nhận notification khi translation files thay đổi. Khi nhận được update signal:
```javascript
// Remove old bundle và reload
i18n.removeResourceBundle('en', 'changedNamespace');
await i18n.reloadResources(['en'], ['changedNamespace']);
// React components tự re-render vì i18next phát event 'added'
```

## 14. References

- i18next store API docs: https://www.i18next.com/overview/api#addresourcebundle
- i18next resource store source code: https://github.com/i18next/i18next/blob/master/src/ResourceStore.js
- Language fallback docs: https://www.i18next.com/principles/fallback
- Backend plugin interface: https://www.i18next.com/misc/creating-own-plugins#backend
- SSR with i18next: https://www.i18next.com/misc/server-side-rendering
- i18next events API: https://www.i18next.com/overview/api#onmissingkey
- addResourceBundle docs: https://www.i18next.com/overview/api#addresourcebundle
- cloneInstance API: https://www.i18next.com/overview/api#cloneinstance

## 15. Real-world Code

- i18next ResourceStore source code: https://github.com/i18next/i18next/blob/master/src/ResourceStore.js
- i18next-http-backend — cách plugin gọi addResourceBundle: https://github.com/i18next/i18next-http-backend/blob/master/lib/index.js
- next-i18next SSR store serialization: https://github.com/i18next/next-i18next/blob/master/src/serverSideTranslations.ts
- i18next-localstorage-backend — custom backend plugin example: https://github.com/i18next/i18next-localstorage-backend
- Locize backend plugin — production TMS integration: https://github.com/locize/i18next-locize-backend

## 16. Community

- i18next GitHub — ResourceStore implementation: https://github.com/i18next/i18next/blob/master/src/ResourceStore.js
- Stack Overflow — i18next store questions: https://stackoverflow.com/search?q=i18next+addResourceBundle
- i18next GitHub Issues — store-related bugs: https://github.com/i18next/i18next/issues?q=ResourceStore
- DEV.to — "i18next internals explained": https://dev.to/adrai/how-does-i18next-work-under-the-hood-3po1
- Locize Blog — dynamic translations with i18next: https://locize.com/blog/i18next-dynamic-loading/
- r/javascript — SSR i18n discussion: https://www.reddit.com/r/javascript/search/?q=i18next+SSR+store
