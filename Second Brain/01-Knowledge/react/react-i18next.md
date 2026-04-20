---
created: 2026-04-17
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/react"
  - "#topic/i18n"
related:
  - "[[i18next]]"
  - "[[useTranslation]]"
  - "[[i18next-store]]"
---

# react-i18next

## 1. What

react-i18next là thư viện binding chính thức kết nối i18next với React, cung cấp hooks, components và Higher-Order Components để tích hợp dịch thuật một cách tự nhiên vào React component tree. Nó không thay thế i18next mà bổ sung thêm lớp tích hợp React: quản lý re-render khi ngôn ngữ thay đổi, hỗ trợ Suspense cho lazy-loading, và cung cấp cách nhúng JSX vào translated strings qua component `<Trans>`.

## 2. Why

i18next là thư viện JavaScript thuần túy không biết gì về React component lifecycle hay re-render model. Nếu dùng i18next trực tiếp trong React mà không có react-i18next, developer phải tự đăng ký lắng nghe sự kiện `languageChanged`, tự force re-render toàn bộ component tree, và tự xử lý trường hợp translations chưa load xong. react-i18next đóng gói toàn bộ phần tích hợp này, đặc biệt là hook `useTranslation` giúp component tự động re-render khi đúng ngôn ngữ và namespace cần thiết thay đổi.

## 3. Mental Model

Hãy nghĩ react-i18next như một **hệ thống ống dẫn nước thông minh** trong tòa nhà. i18next là bể chứa nước trung tâm (translation resources). react-i18next là mạng lưới đường ống và van. `I18nextProvider` là van tổng ở tầng trệt — mở van này thì toàn bộ tòa nhà có nước. `useTranslation` là vòi nước trong từng phòng — component nào cần translation thì gắn vòi vào, nước (translation) chảy ra ngay. Khi bể chứa trung tâm cập nhật (đổi ngôn ngữ), tất cả vòi đang mở tự động nhận nước mới — không cần làm gì thêm. `<Trans>` là vòi nước đặc biệt có thể pha trộn: không chỉ rót nước mà còn nhúng cả đá viên (React elements) vào trong.

## 4. Where it fits

```
React App
  |
  +-- <I18nextProvider i18n={i18nInstance}>   <- Inject i18n vào Context
        |
        +-- <Suspense fallback={<Spinner/>}>  <- Chờ translations load
              |
              +-- <Layout>                    <- useTranslation('common')
              |
              +-- <AuthPage>                  <- useTranslation('auth')
              |     |
              |     +-- <LoginForm>           <- useTranslation('auth')
              |           |
              |           +-- <Trans>         <- JSX trong translated string
              |
              +-- <Dashboard>                 <- useTranslation('dashboard')
```

Pipeline kết nối:

```
i18next instance
  -> .use(initReactI18next)    <- Plugin bắt buộc
  -> .init(config)
  -> I18nextProvider           <- Đưa instance vào React Context
  -> useTranslation hook       <- Components subscribe vào translations
  -> t() / i18n object         <- API cuối cùng component dùng
```

## 5. When to use

- Bất kỳ React app nào đã dùng i18next và cần tích hợp với component system.
- Khi cần React components tự động re-render khi ngôn ngữ thay đổi mà không cần viết listener thủ công.
- Khi translated string cần chứa React elements (links, bold text, icons) — phải dùng `<Trans>`.
- Khi dùng React Suspense để lazy-load translations theo route.
- Khi muốn language switcher component đơn giản sử dụng `i18n.changeLanguage()`.

## 6. When NOT to use

- Nếu không dùng React — dùng i18next trực tiếp hoặc các binding khác (vue-i18next, angular-i18next).
- Nếu ứng dụng đã dùng FormatJS/react-intl — không nên mix hai hệ thống, gây confusion.
- Trong React Server Components (Next.js App Router), `useTranslation` không hoạt động vì là hook. Phải dùng server-side utilities riêng hoặc thư viện khác như `next-intl`.

Hậu quả nếu dùng sai: Dùng i18next mà thiếu `initReactI18next` plugin sẽ khiến components không re-render khi đổi ngôn ngữ. Quên wrap Suspense sẽ gây crash khi translations chưa load xong.

## 7. Trade-offs

| Pros | Cons |
|---|---|
| useTranslation hook API rất tự nhiên với React | Yêu cầu hiểu i18next trước khi dùng hiệu quả |
| Tự động re-render khi ngôn ngữ/namespace thay đổi | `<Trans>` component có API phức tạp, dễ sai |
| Suspense integration cho lazy-loading translations | withTranslation HOC (legacy) tạo boilerplate |
| `<Trans>` cho phép nhúng JSX vào strings | SSR phức tạp, dễ bị hydration mismatch |
| TypeScript support tốt với generics | Bundle size cộng thêm ~5KB trên i18next |
| Tích hợp tốt với React DevTools | Debugging namespace loading issues khó |

## 8. Alternatives

| Approach | Ưu điểm | Nhược điểm |
|---|---|---|
| **react-i18next** | Đầy đủ tính năng, official | Cần học i18next trước |
| **react-intl (FormatJS)** | ICU format chuẩn, type-safe | Verbose hơn, ít flexible |
| **next-intl** | Tối ưu cho Next.js App Router | Chỉ dùng được với Next.js |
| **Lingui React** | Bundle nhỏ, compile-time | Cần Babel/SWC plugin |
| **i18next trực tiếp** | Không overhead | Không tự re-render |

## 9. How

**Cài đặt:**

```bash
npm install react-i18next i18next
```

**Setup i18next instance với `initReactI18next` plugin:**

```javascript
// src/i18n.js
import i18next from 'i18next';
import { initReactI18next } from 'react-i18next';
import HttpBackend from 'i18next-http-backend';

const i18n = i18next
  .use(HttpBackend)
  .use(initReactI18next);  // Plugin kết nối i18next với React

await i18n.init({
  fallbackLng: 'en',
  supportedLngs: ['en', 'vi'],
  defaultNS: 'common',
  backend: {
    loadPath: '/locales/{{lng}}/{{ns}}.json',
  },
  react: {
    useSuspense: true,     // Dùng React Suspense để chờ translations
  },
});

export default i18n;
```

**Wrap app với `I18nextProvider`:**

```jsx
// src/main.jsx
import React, { Suspense } from 'react';
import ReactDOM from 'react-dom/client';
import { I18nextProvider } from 'react-i18next';
import i18n from './i18n';
import App from './App';

ReactDOM.createRoot(document.getElementById('root')).render(
  <I18nextProvider i18n={i18n}>
    <Suspense fallback={<div>Loading...</div>}>
      <App />
    </Suspense>
  </I18nextProvider>
);
```

**Lưu ý:** Nếu gọi `i18n.init()` trước khi render và dùng `useSuspense: false`, có thể dùng `I18nextProvider` mà không cần `Suspense`. Nhưng với HttpBackend (lazy loading), luôn cần `Suspense`.

**`useTranslation` hook — cách dùng phổ biến nhất:**

```jsx
// components/LoginForm.jsx
import { useTranslation } from 'react-i18next';

export default function LoginForm() {
  const { t, i18n } = useTranslation('auth');
  // 'auth' là namespace — sẽ load /locales/en/auth.json

  const handleSwitchLanguage = () => {
    const next = i18n.language === 'en' ? 'vi' : 'en';
    i18n.changeLanguage(next);
    // Component TỰ ĐỘNG re-render sau khi đổi ngôn ngữ
  };

  return (
    <form>
      <h1>{t('login.title')}</h1>
      <label>{t('login.emailLabel')}</label>
      <input placeholder={t('login.emailPlaceholder')} />
      <label>{t('login.passwordLabel')}</label>
      <input type="password" placeholder={t('login.passwordPlaceholder')} />
      <button type="submit">{t('login.submitButton')}</button>
      <button type="button" onClick={handleSwitchLanguage}>
        {i18n.language === 'en' ? 'Tiếng Việt' : 'English'}
      </button>
    </form>
  );
}
```

**`<Trans>` component — cho JSX trong translated strings:**

Đây là tính năng không thể làm được với `t()` đơn giản. Dùng khi translation cần chứa HTML tags hoặc React components.

```json
// en/auth.json
{
  "agreeToTerms": "By signing up, you agree to our <1>Terms of Service</1> and <3>Privacy Policy</3>."
}
```

```jsx
import { Trans, useTranslation } from 'react-i18next';
import { Link } from 'react-router-dom';

function SignupForm() {
  const { t } = useTranslation('auth');

  return (
    <p>
      <Trans
        i18nKey="auth:agreeToTerms"
        components={[
          <span />,                   // index 0 — wrapper ngoài
          <Link to="/terms" />,       // index 1 — <1>...</1>
          <span />,                   // index 2 — " and "
          <Link to="/privacy" />,     // index 3 — <3>...</3>
        ]}
      />
    </p>
  );
}
```

Kết quả render:
```
By signing up, you agree to our Terms of Service and Privacy Policy.
```
Với "Terms of Service" và "Privacy Policy" là React Router `<Link>` thật sự.

**`<Trans>` với interpolation và count:**

```json
{
  "welcomeMessage": "Hello <strong>{{name}}</strong>, you have <1>{{count}} new messages</1>."
}
```

```jsx
<Trans
  i18nKey="welcomeMessage"
  values={{ name: 'An Phuc', count: 5 }}
  components={[<span />, <a href="/messages" />]}
/>
```

**`withTranslation` HOC (legacy, class components):**

```jsx
import { withTranslation } from 'react-i18next';

class LegacyHeader extends React.Component {
  render() {
    const { t } = this.props; // t được inject qua props
    return <h1>{t('header.title')}</h1>;
  }
}

export default withTranslation('common')(LegacyHeader);
```

Dùng `withTranslation` chỉ khi phải làm việc với class components cũ. Với functional components, luôn dùng `useTranslation`.

**`useTranslation` với multiple namespaces:**

```jsx
// Load nhiều namespace cùng lúc
const { t } = useTranslation(['auth', 'common']);

t('auth:login.title');    // Lấy từ namespace auth
t('common:nav.home');     // Lấy từ namespace common
t('login.title');         // Lấy từ namespace đầu tiên trong array ('auth')
```

**Language switcher component:**

```jsx
import { useTranslation } from 'react-i18next';

const LANGUAGES = [
  { code: 'en', label: 'English' },
  { code: 'vi', label: 'Tiếng Việt' },
  { code: 'ja', label: '日本語' },
];

export default function LanguageSwitcher() {
  const { i18n } = useTranslation();

  return (
    <select
      value={i18n.language}
      onChange={(e) => i18n.changeLanguage(e.target.value)}
    >
      {LANGUAGES.map(({ code, label }) => (
        <option key={code} value={code}>{label}</option>
      ))}
    </select>
  );
}
```

## 10. Production Concerns

**Scaling:**

Khi app lớn lên, tổ chức namespace theo feature team. Mỗi team sở hữu namespace của mình, tránh conflict. Dùng dynamic namespace loading với `useTranslation` kết hợp Suspense để chỉ load translations cần thiết.

```jsx
// Lazy load namespace 'reporting' chỉ khi component này mount
function ReportingDashboard() {
  const { t, ready } = useTranslation('reporting');
  if (!ready) return null;
  return <div>{t('title')}</div>;
}
```

**Failure:**

Hydration mismatch trong SSR là vấn đề phổ biến: server render với ngôn ngữ A (detect từ Accept-Language header), client hydrate với ngôn ngữ B (detect từ localStorage). Giải pháp: pass detected language từ server xuống client qua cookie hoặc HTML attribute, đảm bảo cả hai dùng cùng ngôn ngữ trong lần render đầu.

**Monitoring:**

Theo dõi bundle size của translation namespaces theo route. Nếu một route load quá nhiều namespace, cân nhắc tách nhỏ hơn. Track `ready` state trong custom analytics để đo thời gian chờ translation.

## 11. Common Mistakes

- Mistake: Quên thêm `.use(initReactI18next)` khi khởi tạo i18next, dẫn đến components không re-render khi gọi `i18n.changeLanguage()`.
  Fix: Luôn chain `.use(initReactI18next)` trước `.init()`. Plugin này đăng ký React render subscription vào i18next event system.

- Mistake: Dùng `t()` bên trong callback hoặc event handler sau khi component unmount, gây memory leak vì subscription vẫn còn active.
  Fix: react-i18next tự cleanup subscription khi component unmount. Tuy nhiên nếu gọi `i18n.changeLanguage()` bên ngoài component tree, hãy đảm bảo component đã mount trước.

- Mistake: Dùng `<Trans>` với `dangerouslySetInnerHTML` thay vì `components` prop, gây XSS.
  Fix: Luôn dùng `components` prop của `<Trans>` để pass React elements. Không bao giờ render translation HTML qua `dangerouslySetInnerHTML` trừ khi content 100% trusted và sanitized.

- Mistake: Không wrap app với `<Suspense>` khi dùng `useSuspense: true`, dẫn đến React crash với error "Suspense boundary not found".
  Fix: Luôn bọc `<I18nextProvider>` hoặc ít nhất là components dùng `useTranslation` trong `<Suspense fallback={...}>`.

- Mistake: Pass toàn bộ i18n instance vào `<Trans i18nKey>` kèm theo `t` prop không cần thiết, làm code verbose.
  Fix: `<Trans i18nKey="key" />` là đủ nếu i18n đã được setup đúng. `<Trans>` tự lấy i18n instance từ Context.

## 12. Sample Project

**Project: E-commerce storefront với 3 ngôn ngữ**

Constraint cứng: Trang product detail phải render đúng ngôn ngữ ngay khi page load (SSR/SSG), không được có flash of untranslated content. Language switcher phải thay đổi ngôn ngữ mà không reload trang.

**Cấu trúc:**

```
src/
  i18n.js                         # init với initReactI18next
  main.jsx                        # I18nextProvider + Suspense
  components/
    LanguageSwitcher.jsx          # useTranslation + i18n.changeLanguage
    ProductCard.jsx               # useTranslation('products')
    CheckoutTerms.jsx             # <Trans> với Link components
  pages/
    ProductDetail.jsx             # useTranslation(['products', 'common'])
public/
  locales/
    en/common.json
    en/products.json
    vi/common.json
    vi/products.json
    ja/common.json
    ja/products.json
```

**CheckoutTerms dùng `<Trans>`:**

```jsx
// components/CheckoutTerms.jsx
import { Trans } from 'react-i18next';
import { Link } from 'react-router-dom';

export default function CheckoutTerms() {
  return (
    <p className="terms">
      <Trans
        i18nKey="checkout:termsAgreement"
        components={[
          <Link to="/terms" className="link" />,
          <Link to="/refund-policy" className="link" />,
        ]}
      />
    </p>
  );
}
```

**SSR pre-loading để tránh flash:**

```javascript
// server.js (Express)
import i18n from './src/i18n.js';

app.get('*', async (req, res) => {
  const lng = detectLanguage(req); // Từ Accept-Language hoặc cookie
  await i18n.changeLanguage(lng);
  await i18n.loadNamespaces(['common', 'products']);

  // Server render với translations đã có sẵn
  const html = renderToString(<App />);
  // Serialize i18n store state và pass xuống client
  const initialI18nStore = JSON.stringify(i18n.services.resourceStore.data);

  res.send(`
    <script>window.__initialI18nStore = ${initialI18nStore};</script>
    ${html}
  `);
});
```

```javascript
// client i18n.js — rehydrate từ server state
i18next.init({
  ...config,
  resources: window.__initialI18nStore || undefined,
  // Nếu có server state, không cần fetch lại
});
```

## 13. Interview

**Core Q&A:**

Q: `initReactI18next` plugin làm gì và tại sao bắt buộc phải dùng?
A: Plugin này đăng ký React vào hệ thống event của i18next. Cụ thể, nó override `bindI18n` để React components có thể subscribe vào events `languageChanged` và `loaded`. Không có plugin này, `useTranslation` và `withTranslation` không thể trigger re-render khi ngôn ngữ thay đổi.

Q: Tại sao dùng `<Trans>` thay vì `dangerouslySetInnerHTML`?
A: `<Trans>` cho phép nhúng React elements thật vào translated string, bảo toàn React event handling, styling, và navigation. `dangerouslySetInnerHTML` chỉ nhúng HTML string tĩnh, mất đi React reactivity và tạo XSS risk. `<Trans>` an toàn vì các component placeholder không render raw HTML.

Q: Giải thích `ready` flag trong `useTranslation`.
A: `ready` là `true` khi namespace được yêu cầu đã load xong và sẵn sàng dùng. Khi `useSuspense: false`, component có thể render trước khi translation load xong — `ready` giúp conditional render để tránh hiển thị key thô. Với `useSuspense: true`, React tự suspend component nên không cần check `ready` thủ công.

Q: Khác biệt giữa `I18nextProvider` và dùng instance i18n global?
A: `I18nextProvider` inject i18n instance vào React Context, cho phép testing (inject mock instance), SSR (mỗi request có instance riêng), và tránh global state. Dùng instance global (`import i18n from './i18n'`) hoạt động trong nhiều trường hợp nhưng phá vỡ test isolation và gây shared state issues trong SSR.

Q: Khi nào dùng `withTranslation` thay vì `useTranslation`?
A: Chỉ khi phải làm việc với class components không thể convert sang functional components. `withTranslation` là HOC legacy, inject `t` và `i18n` qua props. Mọi functional component nên dùng `useTranslation` hook.

Q: Làm thế nào để type-safe translation keys với TypeScript?
A: Declare module augmentation cho `react-i18next`:
```typescript
declare module 'react-i18next' {
  interface CustomTypeOptions {
    defaultNS: 'common';
    resources: {
      common: typeof import('./locales/en/common.json');
      auth: typeof import('./locales/en/auth.json');
    };
  }
}
```
Sau đó `t('nonexistent.key')` sẽ báo TypeScript error tại compile time.

**Scenario:**

Q: User đổi ngôn ngữ qua dropdown nhưng một số component không re-render. Debug?
A: Kiểm tra: (1) `initReactI18next` đã được add vào i18next instance chưa. (2) Component đang dùng `useTranslation` hay import `t` function từ đâu đó khác (static import không reactive). (3) Component có đang memoize sai với `React.memo` hoặc `useMemo` cắt đứt re-render không. (4) `I18nextProvider` có wrap đúng tất cả components cần translation không.

Q: App bị hydration mismatch khi deploy lên SSR. Nguyên nhân và fix?
A: Nguyên nhân phổ biến: server detect ngôn ngữ từ Accept-Language header là `vi`, nhưng client detect từ localStorage là `en`, dẫn đến render khác nhau. Fix: (1) Server ghi ngôn ngữ đã detect vào cookie. (2) Client đọc cookie trước các nguồn khác trong `detection.order`. (3) Pass serialized i18n store từ server xuống client để client không cần detect lại, tái sử dụng state từ server.

Q: `<Trans>` render sai — text xuất hiện nhưng không có styling của Link component. Debug?
A: Kiểm tra `components` array: index phải khớp với số trong translation key (`<1>`, `<3>` dùng index 1 và 3). Nếu `components` array có ít phần tử hơn index lớn nhất, phần tử bị thiếu sẽ render thành plain text. Dùng `components` object thay vì array để rõ ràng hơn: `components={{ link: <Link to="/terms" /> }}` và trong JSON dùng `<link>...</link>`.

## 14. References

- react-i18next official documentation: https://react.i18next.com/
- Getting started guide: https://react.i18next.com/getting-started
- useTranslation hook API: https://react.i18next.com/latest/usetranslation-hook
- Trans component guide: https://react.i18next.com/latest/trans-component
- withTranslation HOC: https://react.i18next.com/latest/withtranslation-hoc
- Suspense integration: https://react.i18next.com/guides/lazy-loading-translations
- SSR guide: https://react.i18next.com/misc/server-side-rendering
- react-i18next GitHub repository: https://github.com/i18next/react-i18next

## 15. Real-world Code

- Excalidraw — useTranslation hooks throughout components: https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/components/App.tsx
- Outline Wiki — I18nextProvider setup và Trans usage: https://github.com/outline/outline/blob/main/app/components/i18n.tsx
- Cal.com — react-i18next in Next.js Pages Router: https://github.com/calcom/cal.com/blob/main/apps/web/pages/_app.tsx
- Docusaurus — Trans component usage in docs: https://github.com/facebook/docusaurus/search?q=Trans+i18next
- Mattermost webapp — large-scale react-i18next: https://github.com/mattermost/mattermost/tree/master/webapp/channels/src/i18n

## 16. Community

- react-i18next GitHub Discussions: https://github.com/i18next/react-i18next/discussions
- Stack Overflow — react-i18next tag: https://stackoverflow.com/questions/tagged/react-i18next
- Locize Blog — React i18n deep dives: https://locize.com/blog/react-i18next/
- DEV.to — "React Internationalization – How to" by Jan Mühlemann: https://dev.to/adrai/react-internationalization-how-to-6e1
- LogRocket — "Complete guide to react-i18next": https://blog.logrocket.com/complete-guide-react-i18next/
- r/reactjs thread về best practices i18n: https://www.reddit.com/r/reactjs/search/?q=react-i18next+best+practices
