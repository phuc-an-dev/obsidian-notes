---
created: 2026-04-17
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/react"
  - "#topic/i18n"
related:
  - "[[react-i18next]]"
  - "[[i18next]]"
---

# useTranslation

## 1. What

`useTranslation` là React hook chính của react-i18next, cung cấp hàm `t()` để lấy translated strings và object `i18n` để truy cập i18next instance bên trong React functional components. Hook này tự động subscribe component vào các thay đổi ngôn ngữ, đảm bảo re-render đúng thời điểm mà không gây re-render thừa. Đây là API được dùng trong >90% trường hợp khi làm việc với react-i18next.

## 2. Why

Trước khi có hook này, developer phải wrap component với `withTranslation` HOC (Higher-Order Component) — cú pháp verbose, tạo thêm component nesting không cần thiết trong DevTools, và khó kết hợp với các HOC khác. Với class components thì còn phức tạp hơn khi phải manually subscribe và unsubscribe vào i18next events. `useTranslation` đóng gói toàn bộ logic này vào một dòng code, phù hợp tự nhiên với functional component paradigm của React hiện đại.

## 3. Mental Model

Hãy nghĩ `useTranslation` như một **radio receiver cá nhân**. Mỗi component gọi `useTranslation('namespace')` giống như bật một chiếc radio và dò đúng kênh (namespace). Khi đài phát (i18next store) phát sóng thông báo "ngôn ngữ vừa đổi sang tiếng Việt", tất cả radio đang bật và đang dò kênh liên quan đều nhận được tín hiệu và tự cập nhật. Radio tắt (component unmount) thì tự động ngừng nhận sóng — không cần gọi hàm cleanup thủ công. Hàm `t()` là nút bấm trên radio: bạn nhập key, radio tìm bản dịch đúng trên kênh hiện tại và đọc ra cho bạn nghe.

## 4. Where it fits

```
i18next instance (ResourceStore + language state)
  |
  v
I18nextProvider (React Context)
  |
  v
Component tree
  |
  +-- useTranslation('auth')
  |     |
  |     +-- { t, i18n, ready }
  |           |
  |           +-- t('login.title')         -> lookup in ResourceStore
  |           +-- t('greeting', { name })  -> interpolation
  |           +-- t('count', { count: n }) -> pluralization
  |           +-- i18n.changeLanguage()    -> updates all subscribers
  |           +-- i18n.language            -> current language string
  |           +-- ready                   -> boolean: namespace loaded?
```

Luồng re-render khi đổi ngôn ngữ:

```
i18n.changeLanguage('vi')
  -> i18next fires 'languageChanged' event
  -> initReactI18next catches event
  -> Notifies all active useTranslation subscribers
  -> React schedules re-render for affected components
  -> t() calls now return Vietnamese translations
```

## 5. When to use

- Trong mọi functional component cần hiển thị text đã dịch.
- Khi cần truy cập `i18n.language` để conditional render theo ngôn ngữ hiện tại.
- Khi cần `i18n.changeLanguage()` để xây dựng language switcher.
- Khi cần kiểm tra `ready` flag để render loading state thay vì hiển thị key thô.
- Khi một component cần translations từ nhiều namespace khác nhau.

## 6. When NOT to use

- Trong class components — dùng `withTranslation` HOC thay thế.
- Trong React Server Components (Next.js App Router) — hooks không hoạt động trong RSC, phải dùng server-side translation utilities riêng.
- Bên ngoài React component tree (ví dụ: trong utility functions, API calls) — dùng i18next instance trực tiếp (`import i18n from './i18n'; i18n.t('key')`).

Hậu quả nếu dùng sai: Gọi `useTranslation` ngoài component function body vi phạm Rules of Hooks, gây crash toàn bộ app.

## 7. Trade-offs

| Pros | Cons |
|---|---|
| API cực kỳ đơn giản, một dòng setup | Chỉ dùng được trong functional components |
| Tự động cleanup khi component unmount | Phải hiểu namespace concept để dùng hiệu quả |
| Granular re-render: chỉ re-render khi namespace liên quan thay đổi | `ready` flag dễ bị bỏ qua, gây hiển thị key thô |
| Type-safe với TypeScript khi setup đúng | TypeScript setup yêu cầu module augmentation |
| Hỗ trợ multiple namespaces trong một lần gọi | Dùng nhiều namespace cùng lúc làm code phức tạp hơn |
| `i18n` object cho phép full control | Truy cập sai `i18n` methods có thể gây side effects |

## 8. Alternatives

| Approach | Dùng khi | Hạn chế |
|---|---|---|
| **useTranslation** | Functional components, phổ biến nhất | Chỉ client-side |
| **withTranslation HOC** | Class components cũ | Verbose, nesting phức tạp |
| **Trans component** | Cần JSX trong translation string | Không dùng được cho prop values |
| **i18n.t() trực tiếp** | Outside component tree | Không reactive, không re-render |
| **useT từ next-intl** | Next.js App Router RSC | Chỉ dùng với next-intl |

## 9. How

**Cú pháp cơ bản:**

```jsx
import { useTranslation } from 'react-i18next';

function MyComponent() {
  const { t, i18n } = useTranslation('myNamespace');
  // hoặc không truyền namespace -> dùng defaultNS
  const { t } = useTranslation();

  return <h1>{t('page.title')}</h1>;
}
```

**Hàm `t()` — tất cả các cách dùng:**

```jsx
function TranslationExamples() {
  const { t } = useTranslation('common');

  return (
    <div>
      {/* Simple key */}
      <p>{t('nav.home')}</p>
      {/* -> "Home" */}

      {/* Interpolation với {{ }} */}
      <p>{t('greeting', { name: 'An Phuc' })}</p>
      {/* -> "Hello, An Phuc!" */}

      {/* Pluralization: truyền count, i18next tự chọn đúng form */}
      <p>{t('itemCount', { count: 1 })}</p>
      {/* -> "1 item" */}
      <p>{t('itemCount', { count: 5 })}</p>
      {/* -> "5 items" */}

      {/* Namespace prefix trong key string */}
      <p>{t('auth:login.title')}</p>
      {/* -> Lấy từ namespace 'auth', key 'login.title' */}

      {/* Default value nếu key không tồn tại */}
      <p>{t('nonexistent.key', 'Fallback text')}</p>
      {/* -> "Fallback text" */}

      {/* Nested key với dot notation */}
      <p>{t('errors.validation.required')}</p>
      {/* -> "This field is required" */}

      {/* Dùng t() cho attribute của HTML element */}
      <input placeholder={t('form.emailPlaceholder')} />

      {/* Context */}
      <p>{t('role', { context: 'admin' })}</p>
      {/* -> "Administrator" (dùng key 'role_admin' trong JSON) */}
    </div>
  );
}
```

**Object `i18n` — truy cập instance:**

```jsx
function LanguageControls() {
  const { i18n } = useTranslation();

  return (
    <div>
      {/* Đọc ngôn ngữ hiện tại */}
      <p>Current: {i18n.language}</p>
      {/* -> "vi" */}

      {/* Đọc toàn bộ fallback chain */}
      <p>Chain: {i18n.languages.join(' -> ')}</p>
      {/* -> "vi -> en" */}

      {/* Đổi ngôn ngữ */}
      <button onClick={() => i18n.changeLanguage('en')}>English</button>
      <button onClick={() => i18n.changeLanguage('vi')}>Tiếng Việt</button>

      {/* Kiểm tra key có tồn tại không */}
      {i18n.exists('common:nav.home') && (
        <p>Key exists!</p>
      )}
    </div>
  );
}
```

**`ready` flag và Suspense:**

Có hai cách xử lý trạng thái chờ translations load:

**Cách 1: Dùng `ready` flag (khi `useSuspense: false`):**

```jsx
// i18n config: react: { useSuspense: false }

function UserProfile() {
  const { t, ready } = useTranslation('profile');

  if (!ready) {
    // Translations đang load — render skeleton thay vì key thô
    return <ProfileSkeleton />;
  }

  return (
    <div>
      <h1>{t('title')}</h1>
      <p>{t('bio')}</p>
    </div>
  );
}
```

**Cách 2: Dùng React Suspense (khi `useSuspense: true` — recommended):**

```jsx
// i18n config: react: { useSuspense: true }

function UserProfileContent() {
  const { t } = useTranslation('profile');
  // Component này sẽ tự suspend nếu 'profile' namespace chưa load
  // Không cần check ready

  return (
    <div>
      <h1>{t('title')}</h1>
      <p>{t('bio')}</p>
    </div>
  );
}

// Wrapper với Suspense boundary
function UserProfile() {
  return (
    <Suspense fallback={<ProfileSkeleton />}>
      <UserProfileContent />
    </Suspense>
  );
}
```

**Multiple namespaces trong một component:**

```jsx
function ComplexPage() {
  // Option 1: Array — namespace đầu tiên là default
  const { t } = useTranslation(['page', 'common', 'errors']);

  return (
    <div>
      {t('page:main.title')}         // Từ 'page' namespace
      {t('common:nav.home')}         // Từ 'common' namespace
      {t('errors:validation.email')} // Từ 'errors' namespace
      {t('main.title')}              // Mặc định tìm trong 'page' (index 0)
    </div>
  );
}
```

**Namespace-specific vs global translation:**

```jsx
// Namespace cụ thể — tốt cho feature-scoped components
const { t } = useTranslation('checkout');
t('summary.totalLabel');  // Chỉ tìm trong 'checkout' namespace

// Global (defaultNS) — tốt cho shared components
const { t } = useTranslation();
t('common:buttons.submit');  // Phải explicit namespace prefix
```

**Pattern: Custom hook bọc useTranslation:**

Khi nhiều components trong cùng feature đều dùng cùng namespace:

```jsx
// hooks/useAuthTranslation.js
import { useTranslation } from 'react-i18next';

export function useAuthTranslation() {
  return useTranslation('auth');
}

// Trong LoginForm, RegisterForm, ForgotPassword...
function LoginForm() {
  const { t } = useAuthTranslation();
  // Không cần nhớ namespace name ở mọi nơi
  return <h1>{t('login.title')}</h1>;
}
```

**TypeScript type-safe keys:**

```typescript
// types/i18n.d.ts
import 'react-i18next';
import commonEn from '../public/locales/en/common.json';
import authEn from '../public/locales/en/auth.json';

declare module 'react-i18next' {
  interface CustomTypeOptions {
    defaultNS: 'common';
    resources: {
      common: typeof commonEn;
      auth: typeof authEn;
    };
  }
}

// Bây giờ trong component:
const { t } = useTranslation('auth');
t('login.title');          // OK - TypeScript biết key này tồn tại
t('login.nonexistent');    // TypeScript ERROR - key không tồn tại
```

## 10. Production Concerns

**Performance:**

`useTranslation` không gây unnecessary re-render vì nó chỉ subscribe vào changes của namespace được truyền vào. Nếu component A dùng `useTranslation('auth')` và component B dùng `useTranslation('dashboard')`, khi namespace `dashboard` thay đổi (ví dụ: load xong), chỉ component B re-render, component A không bị ảnh hưởng.

Cơ chế bên trong: react-i18next dùng `useState` + event listener. Khi i18next phát event `languageChanged` hoặc `loaded`, hook so sánh namespace trước và sau để quyết định có update state (trigger re-render) hay không.

**Scaling:**

Với app lớn có hàng trăm components, việc nhiều components cùng re-render khi đổi ngôn ngữ là bình thường và React xử lý tốt qua batching. Không cần optimize ở đây trừ khi profiling cho thấy vấn đề cụ thể.

**Failure:**

Nếu `useTranslation` được gọi trước khi `I18nextProvider` mount (ví dụ: component render ngoài provider tree), sẽ throw error. Đảm bảo `I18nextProvider` wrap toàn bộ component tree hoặc ít nhất là tất cả components dùng translation.

**Monitoring:**

Log khi `ready` là false lâu hơn expected threshold (ví dụ: >3 giây) để phát hiện network issues với translation file loading.

## 11. Common Mistakes

- Mistake: Gọi `t()` bên ngoài component function body, ví dụ trong module scope: `const title = t('page.title');` ở top level của file.
  Fix: `useTranslation` là hook — chỉ được gọi trong function body của React component hoặc custom hook. Mọi sử dụng `t()` ở module scope đều vi phạm Rules of Hooks.

- Mistake: Bỏ qua `ready` flag khi `useSuspense: false`, render trực tiếp `{t('key')}` khi translations chưa load xong.
  Fix: Check `if (!ready) return <Skeleton />;` hoặc chuyển sang `useSuspense: true` với `<Suspense>` wrapper.

- Mistake: Dùng `i18n.changeLanguage()` trong render phase (trực tiếp trong JSX hoặc trong component body ngoài event handler).
  Fix: Chỉ gọi `i18n.changeLanguage()` trong event handlers hoặc `useEffect`. Gọi trong render phase gây vòng lặp re-render vô tận.

- Mistake: Destructure `t` và lưu vào biến ngoài scope rồi dùng sau khi component đã re-render với ngôn ngữ mới.
  Fix: `t` từ `useTranslation` luôn trả về function mới sau mỗi re-render với ngôn ngữ cập nhật. Không lưu `t` vào biến ngoài component lifecycle.

- Mistake: Truyền wrong namespace — dùng `useTranslation('Auth')` trong khi file là `auth.json` (case-sensitive trên Linux server).
  Fix: Namespace names phải khớp chính xác (case-sensitive) với tên file và với config trong `ns` array. Dùng constant để tránh typo: `const AUTH_NS = 'auth'`.

## 12. Sample Project

**Project: Dashboard ứng dụng quản lý nhân sự**

Constraint cứng: Mỗi module (employees, payroll, reports) phải dùng namespace riêng. Không được pass translation key hoặc translated string qua props — mỗi component tự fetch translation của mình qua `useTranslation`.

**Cấu trúc translation files:**

```
public/locales/en/
  common.json      # buttons, labels, errors chung
  employees.json   # tất cả text của module employees
  payroll.json     # tất cả text của module payroll
  reports.json     # tất cả text của module reports
```

**employees.json:**

```json
{
  "list": {
    "title": "Employee Directory",
    "searchPlaceholder": "Search by name or ID...",
    "totalCount_one": "{{count}} employee",
    "totalCount_other": "{{count}} employees",
    "emptyState": "No employees found matching your search."
  },
  "detail": {
    "title": "Employee Profile",
    "department": "Department",
    "joinDate": "Join Date",
    "status": {
      "active": "Active",
      "inactive": "Inactive",
      "onLeave": "On Leave"
    }
  }
}
```

**EmployeeList component:**

```jsx
import { useTranslation } from 'react-i18next';
import { Suspense } from 'react';

function EmployeeListContent({ employees }) {
  const { t } = useTranslation('employees');

  return (
    <div>
      <h1>{t('list.title')}</h1>
      <p>{t('list.totalCount', { count: employees.length })}</p>
      <input placeholder={t('list.searchPlaceholder')} />
      {employees.length === 0 && (
        <p>{t('list.emptyState')}</p>
      )}
    </div>
  );
}

export default function EmployeeList({ employees }) {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <EmployeeListContent employees={employees} />
    </Suspense>
  );
}
```

**EmployeeStatus component — không nhận translated string qua props:**

```jsx
// BAD (vi phạm constraint): <EmployeeStatus label={t('detail.status.active')} />
// GOOD (mỗi component tự fetch):

function EmployeeStatus({ status }) {
  const { t } = useTranslation('employees');
  // Dùng dynamic key với status value
  const label = t(`detail.status.${status}`, { defaultValue: status });
  return <span className={`status status--${status}`}>{label}</span>;
}
```

**Language switcher kết hợp với URL routing:**

```jsx
function LanguageSwitcher() {
  const { i18n } = useTranslation();
  const navigate = useNavigate();
  const location = useLocation();

  const switchLanguage = async (lang) => {
    await i18n.changeLanguage(lang);
    // Update URL để bookmark/share đúng ngôn ngữ
    const newPath = `/${lang}${location.pathname.replace(/^\/[a-z]{2}/, '')}`;
    navigate(newPath);
  };

  return (
    <nav>
      <button onClick={() => switchLanguage('en')}>EN</button>
      <button onClick={() => switchLanguage('vi')}>VI</button>
    </nav>
  );
}
```

## 13. Interview

**Core Q&A:**

Q: `useTranslation` trả về những gì?
A: Trả về object `{ t, i18n, ready }`. `t` là hàm translation. `i18n` là i18next instance đầy đủ. `ready` là boolean cho biết namespace đã load xong chưa (quan trọng khi `useSuspense: false`).

Q: Tại sao `useTranslation` không gây unnecessary re-render?
A: Hook chỉ subscribe vào thay đổi của namespace được truyền vào. Bên trong, nó dùng `useRef` để track namespace và `useState` để trigger re-render khi cần. Khi i18next phát event, hook so sánh namespace liên quan — nếu namespace của component không thay đổi thì không update state, không re-render.

Q: Sự khác biệt giữa `t('ns:key')` và `useTranslation('ns')` rồi `t('key')`?
A: Về kết quả là như nhau, nhưng `useTranslation('ns')` đảm bảo namespace `ns` được preload khi component mount. Dùng `t('ns:key')` mà không declare namespace trong `useTranslation` sẽ không trigger lazy loading của namespace đó nếu chưa load.

Q: `i18n.changeLanguage()` trả về gì?
A: Trả về `Promise<TFunction>`. Promise resolve sau khi ngôn ngữ mới được load xong (nếu dùng HttpBackend) và tất cả subscribed components đã được notify. Nên `await` khi cần đảm bảo UI đã cập nhật trước khi làm gì tiếp theo.

Q: Làm thế nào để dùng `t()` bên ngoài component (trong util functions)?
A: Import i18next instance trực tiếp: `import i18n from './i18n'; i18n.t('key')`. Tuy nhiên kết quả này không reactive — nếu ngôn ngữ thay đổi, function phải được gọi lại để lấy giá trị mới. Với React, luôn ưu tiên dùng `useTranslation` trong component.

Q: Có thể dùng `useTranslation` trong custom hook không?
A: Có, đây là pattern phổ biến để đóng gói logic translation theo feature. Custom hook gọi `useTranslation` với namespace cố định và export `t` cùng với các helper functions. Phải tuân theo Rules of Hooks: custom hook phải được gọi trong component.

**Scenario:**

Q: Component hiển thị key thô `profile:user.name` thay vì translation. Debug?
A: (1) Kiểm tra `ready` — nếu `false`, namespace `profile` chưa load xong. (2) Kiểm tra Network tab xem `/locales/en/profile.json` có được request không và có trả về 200 không. (3) Kiểm tra JSON syntax của file (parse error khiến toàn bộ namespace fail). (4) Kiểm tra key `user.name` có tồn tại trong JSON đúng không (case-sensitive). (5) Chạy `i18n.hasResourceBundle('en', 'profile')` trong console để verify namespace đã load.

Q: User báo ngôn ngữ bị reset về English sau khi refresh trang, dù đã chọn Tiếng Việt. Fix?
A: LanguageDetector đang không persist đúng. Kiểm tra: (1) `caches: ['localStorage', 'cookie']` có trong detection config không. (2) Browser có block localStorage không (private browsing mode). (3) `detection.order` có include `localStorage` trước `navigator` không. (4) Nếu dùng SSR, server cần đọc cookie/header để detect ngôn ngữ đã chọn.

Q: Bạn cần format currency và date theo locale cùng lúc với translation. Kết hợp như thế nào?
A: Dùng `i18n.language` từ `useTranslation` để pass vào Intl API:
```jsx
const { t, i18n } = useTranslation('billing');
const formatCurrency = (amount) =>
  new Intl.NumberFormat(i18n.language, { style: 'currency', currency: 'USD' })
    .format(amount);
const formatDate = (date) =>
  new Intl.DateTimeFormat(i18n.language, { dateStyle: 'long' })
    .format(new Date(date));

return <p>{t('invoice.total', { amount: formatCurrency(1234.56) })}</p>;
```
i18next xử lý translation strings, Intl API xử lý locale-specific formatting.

## 14. References

- useTranslation hook API docs: https://react.i18next.com/latest/usetranslation-hook
- react-i18next overview: https://react.i18next.com/
- TypeScript với react-i18next: https://react.i18next.com/latest/typescript
- i18next t() function docs: https://www.i18next.com/translation-function/essentials
- Pluralization: https://www.i18next.com/translation-function/plurals
- i18next interpolation: https://www.i18next.com/translation-function/interpolation
- React Suspense docs: https://react.dev/reference/react/Suspense
- react-i18next GitHub: https://github.com/i18next/react-i18next

## 15. Real-world Code

- Excalidraw — useTranslation trong nhiều components: https://github.com/excalidraw/excalidraw/search?q=useTranslation
- Outline — useTranslation với TypeScript: https://github.com/outline/outline/search?q=useTranslation
- Mattermost — large-scale useTranslation patterns: https://github.com/mattermost/mattermost/search?q=useTranslation&type=code
- Ghost Admin — useTranslation hooks: https://github.com/TryGhost/Ghost/search?q=useTranslation
- Umami Analytics — useTranslation với Next.js: https://github.com/umami-software/umami/search?q=useTranslation

## 16. Community

- react-i18next useTranslation hook documentation: https://react.i18next.com/latest/usetranslation-hook
- Stack Overflow — useTranslation tag: https://stackoverflow.com/search?q=useTranslation+react-i18next
- GitHub Issues — react-i18next discussions: https://github.com/i18next/react-i18next/issues
- DEV.to — "useTranslation deep dive": https://dev.to/search?q=useTranslation
- Locize Blog — React hooks i18n patterns: https://locize.com/blog/react-i18next/
- r/reactjs — i18n hooks discussion: https://www.reddit.com/r/reactjs/search/?q=useTranslation
