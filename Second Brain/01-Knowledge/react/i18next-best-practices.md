---
created: 2026-04-17
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/react"
  - "#topic/i18n"
related: "[[i18next]], [[react-i18next]], [[useTranslation]]"
---

# i18next Best Practices

## 1. What

i18next Best Practices là tập hợp các pattern và anti-pattern đã được kiểm chứng trong production, bao gồm cách tổ chức namespaces, đặt tên keys, lazy loading, xử lý số nhiều, format ngày giờ và số, tích hợp CI/CD, và xử lý SSR. Đây không phải là feature của i18next mà là kinh nghiệm thực tế giúp ứng dụng i18n dễ bảo trì, scale tốt, và ít lỗi.

## 2. Why

i18next cung cấp công cụ mạnh mẽ nhưng không enforce cách dùng. Thiếu convention rõ ràng dẫn đến: namespace lộn xộn khiến translator không biết file nào dịch trước, key names không nhất quán làm khó search trong codebase, string concatenation phá vỡ ngữ nghĩa ngôn ngữ, và thiếu automation để phát hiện key bị missing dẫn đến bug production chỉ phát hiện khi user gặp. Best practices giải quyết tất cả những vấn đề này một cách hệ thống.

## 3. Mental Model

Hãy nghĩ hệ thống i18n của bạn như **một nhà xuất bản sách đa ngôn ngữ với quy trình sản xuất chuyên nghiệp**. Namespace là các chương sách — mỗi editor (team feature) chịu trách nhiệm một chương. Key là mục lục — phải rõ ràng để translator biết context. Pluralization là grammar rules — không thể cheat, phải follow đúng quy tắc của từng ngôn ngữ. CI/CD với i18next-parser là proof-reader tự động — scan toàn bộ bản thảo trước khi in để đảm bảo không có mục lục nào thiếu nội dung. SSR là in bản preview trước khi giao cho khách — bản preview phải giống hệt bản thật, không được có trang trắng (flash of untranslated content).

## 4. Where it fits

```
Development Workflow
  |
  +-- Write code with t('namespace:key')
  |
  +-- i18next-parser (CI step)
  |     |
  |     +-- Scans source files
  |     +-- Extracts all t() calls
  |     +-- Compares with existing JSON files
  |     +-- Reports: missing keys / unused keys
  |
  +-- Translator updates JSON files (via TMS or PR)
  |
  +-- Review & merge
  |
  +-- Build
  |     |
  |     +-- Bundle namespaces used on initial load
  |     +-- Other namespaces -> separate chunks (lazy load)
  |
  +-- Deploy to CDN
        |
        +-- Browser lazy loads namespaces per route
        +-- SSR pre-loads for first render
```

## 5. When to use

Áp dụng các practices này khi:

- Bắt đầu dự án mới cần i18n — thiết lập đúng từ đầu dễ hơn refactor sau.
- Team có nhiều hơn 2 người làm việc trên cùng translation files.
- App có >5 ngôn ngữ hoặc >500 translation keys.
- Cần CI/CD enforcement để không ai commit code có missing translation keys.
- App dùng SSR và yêu cầu không có Flash of Untranslated Content (FOUC).

## 6. When NOT to use

Không cần áp dụng mọi practice cho mọi dự án:

- App cá nhân nhỏ: namespace strategy đơn giản (1-2 namespaces) là đủ.
- Prototype hoặc PoC: đừng tốn thời gian setup CI pipeline cho i18n.
- App chỉ cần dịch 1 lần và không cập nhật thường xuyên: TMS integration over-engineering.

## 7. Trade-offs

| Pros | Cons |
|---|---|
| Namespace theo feature: tách biệt ownership rõ ràng | Cần quy ước đặt tên namespace nhất quán trong team |
| Lazy loading: giảm initial bundle size đáng kể | Thêm complexity vào routing setup |
| i18next-parser: phát hiện missing keys tự động | Cần cấu hình parser đúng với syntax đặc thù của project |
| Intl API cho date/number: đúng locale, không cần plugin | Cần giữ i18n.language đồng bộ với Intl locale |
| SSR preloading: không có FOUC | Setup phức tạp hơn, cần serialize store |

## 8. Alternatives

| Practice | Alternative | Khi nào dùng alternative |
|---|---|---|
| Namespace theo feature | Namespace theo type (common/errors) | App nhỏ, 1-2 feature teams |
| Dot notation keys | Flat keys | File dịch cực nhỏ, ít key |
| i18next-parser CLI | TypeScript type generation | Khi cần type-safety thay vì linting |
| Lazy loading per route | Bundle tất cả | App nhỏ, translation files tổng <50KB |
| Intl API | i18next-intl plugin | Khi đã dùng FormatJS pattern |

## 9. How

### Namespace Strategy

**Pattern 1 — Theo Feature (khuyến nghị cho team lớn):**

```
public/locales/en/
  common.json       # Shared: buttons, labels, status chung
  errors.json       # Error messages chung
  auth.json         # Login, register, forgot password
  dashboard.json    # Dashboard overview
  products.json     # Product listing, detail, search
  checkout.json     # Cart, payment, order confirmation
  account.json      # Profile, settings, notifications
  admin.json        # Admin panel (chỉ load cho admin users)
```

Quy tắc: Mỗi team/feature sở hữu một namespace. Không ai được sửa namespace của team khác trừ qua PR review.

**Pattern 2 — Theo Type (cho app nhỏ):**

```
public/locales/en/
  common.json    # Text xuất hiện nhiều nơi
  forms.json     # Labels, placeholders, validation messages
  errors.json    # Error messages
  pages.json     # Page-specific content (headlines, descriptions)
```

**Tránh:** Một namespace `translation.json` cho toàn bộ app — merge conflict thường xuyên, file phình to, khó lazy load.

### Key Naming Convention

**Nguyên tắc:**
- Dùng `camelCase` cho mỗi segment: `loginForm.submitButton` thay vì `login_form.submit_button`
- Tối đa 3 cấp lồng nhau: `section.subsection.element`
- Mô tả context, không mô tả style: `form.submitButton` thay vì `form.blueButton`
- Key phản ánh vị trí trong UI: `productCard.price.discountLabel`

**Flat vs Nested:**

```json
// Flat — dễ grep, nhưng dễ naming conflict khi nhiều feature
{
  "loginTitle": "Sign In",
  "loginEmail": "Email Address",
  "loginPassword": "Password",
  "registerTitle": "Create Account",
  "registerEmail": "Email Address"
}

// Nested — rõ ràng context, dễ tổ chức (khuyến nghị)
{
  "login": {
    "title": "Sign In",
    "email": "Email Address",
    "password": "Password"
  },
  "register": {
    "title": "Create Account",
    "email": "Email Address"
  }
}
```

**Dot notation pitfall — tránh dùng dấu chấm trong key name:**

```json
// NGUY HIỂM: key "com.example.app" bị parse thành nested object
{
  "com.example.app": "My App"  // i18next parse thành com -> example -> app
}

// ĐÚNG: escape hoặc dùng array notation
{
  "appName": "My App"  // đơn giản và rõ ràng
}
```

```javascript
// Nếu bắt buộc phải dùng key có dấu chấm:
i18n.init({ keySeparator: false }); // Tắt dot notation hoàn toàn
// Hoặc:
i18n.init({ keySeparator: '|' }); // Dùng | thay vì .
```

### Lazy Loading per Route

```javascript
// i18n.js — chỉ preload common và layout
i18n.init({
  preload: ['en'],
  ns: ['common'],       // Chỉ load common ban đầu
  defaultNS: 'common',
  backend: {
    loadPath: '/locales/{{lng}}/{{ns}}.json',
  },
});
```

```jsx
// Mỗi route component tự load namespace của mình
// react-router v6 với lazy components

// pages/Checkout.jsx
import { Suspense } from 'react';
import { useTranslation } from 'react-i18next';

function CheckoutContent() {
  // useTranslation trigger lazy load 'checkout' namespace
  const { t } = useTranslation('checkout');
  return (
    <div>
      <h1>{t('title')}</h1>
      <CheckoutForm />
    </div>
  );
}

export default function Checkout() {
  return (
    <Suspense fallback={<CheckoutSkeleton />}>
      <CheckoutContent />
    </Suspense>
  );
}
```

```javascript
// router.jsx — React Router code splitting kết hợp namespace lazy load
const routes = [
  { path: '/', element: <Home /> },
  {
    path: '/checkout',
    element: (
      // React.lazy: split JS bundle
      // Suspense + useTranslation: split translation bundle
      <React.Suspense fallback={<PageSkeleton />}>
        {React.createElement(React.lazy(() => import('./pages/Checkout')))}
      </React.Suspense>
    ),
  },
];
```

### Pluralization — Không Bao Giờ Concatenate

```json
// BAD pattern (trong code JS):
// `${count} ${count === 1 ? t('item') : t('items')}`
// Sai với tiếng Đức, Nga, Ả Rập, Ba Lan...

// GOOD pattern — định nghĩa trong JSON, không trong logic JS:
// en/common.json
{
  "fileCount_zero": "No files",
  "fileCount_one": "{{count}} file",
  "fileCount_other": "{{count}} files",
  "fileCount_many": "{{count}} files"
}
```

```jsx
// Component chỉ truyền count, không có if/else:
function FileCounter({ count }) {
  const { t } = useTranslation('common');
  return <span>{t('fileCount', { count })}</span>;
}
// -> "No files" / "1 file" / "5 files"
```

```json
// vi/common.json — tiếng Việt không có plural forms phức tạp như tiếng Anh
// Nhưng vẫn phải dùng count để translator biết context
{
  "fileCount": "{{count}} tập tin",
  "fileCount_zero": "Không có tập tin"
}
```

```json
// ru/common.json — tiếng Nga có 4 plural forms
{
  "fileCount_zero": "Нет файлов",
  "fileCount_one": "{{count}} файл",
  "fileCount_few": "{{count}} файла",
  "fileCount_many": "{{count}} файлов",
  "fileCount_other": "{{count}} файла"
}
```

### Date và Number Formatting với Intl API

i18next không handle date/number formatting — đây là trách nhiệm của `Intl` API built-in của JavaScript. Kết hợp `i18n.language` với `Intl`:

```jsx
import { useTranslation } from 'react-i18next';
import { useMemo } from 'react';

function useFormatters() {
  const { i18n } = useTranslation();
  const locale = i18n.language;

  return useMemo(() => ({
    formatDate: (date, options = { dateStyle: 'long' }) =>
      new Intl.DateTimeFormat(locale, options).format(new Date(date)),

    formatCurrency: (amount, currency = 'USD') =>
      new Intl.NumberFormat(locale, {
        style: 'currency',
        currency,
      }).format(amount),

    formatNumber: (num, options = {}) =>
      new Intl.NumberFormat(locale, options).format(num),

    formatRelativeTime: (date) => {
      const diff = (new Date(date) - Date.now()) / 1000;
      const rtf = new Intl.RelativeTimeFormat(locale, { numeric: 'auto' });
      if (Math.abs(diff) < 60) return rtf.format(Math.round(diff), 'second');
      if (Math.abs(diff) < 3600) return rtf.format(Math.round(diff / 60), 'minute');
      if (Math.abs(diff) < 86400) return rtf.format(Math.round(diff / 3600), 'hour');
      return rtf.format(Math.round(diff / 86400), 'day');
    },
  }), [locale]);
}

// Sử dụng trong component:
function OrderSummary({ order }) {
  const { t } = useTranslation('orders');
  const { formatDate, formatCurrency } = useFormatters();

  return (
    <div>
      <p>{t('summary.orderDate', { date: formatDate(order.createdAt) })}</p>
      <p>{t('summary.total', { amount: formatCurrency(order.total) })}</p>
    </div>
  );
}
```

Output ví dụ:

```
English (en-US):
  Order Date: April 17, 2026
  Total: $1,234.56

Vietnamese (vi-VN):
  Order Date: 17 tháng 4, 2026
  Total: 1.234,56 US$

Japanese (ja-JP):
  Order Date: 2026年4月17日
  Total: $1,234.56
```

### CI/CD với i18next-parser

**Cài đặt:**

```bash
npm install --save-dev i18next-parser
```

**Cấu hình (`i18next-parser.config.js`):**

```javascript
module.exports = {
  locales: ['en', 'vi', 'ja'],
  defaultNamespace: 'common',
  input: ['src/**/*.{js,jsx,ts,tsx}'],
  output: 'public/locales/$LOCALE/$NAMESPACE.json',

  // Giữ keys đã dịch, chỉ thêm keys mới với empty string
  keepRemoved: false,
  createOldCatalogs: true,  // Lưu keys removed vào .json.OLD file

  // Phát hiện t() calls với namespace prefix
  namespaceSeparator: ':',
  keySeparator: '.',

  // Custom patterns nếu dùng custom hook
  lexers: {
    js: ['JavascriptLexer'],
    jsx: ['JsxLexer'],
    ts: ['JavascriptLexer'],
    tsx: ['JsxLexer'],
  },
};
```

**package.json scripts:**

```json
{
  "scripts": {
    "i18n:extract": "i18next-parser",
    "i18n:check": "i18next-parser --dry-run && node scripts/check-missing-translations.js"
  }
}
```

**Script kiểm tra missing translations (`scripts/check-missing-translations.js`):**

```javascript
const fs = require('fs');
const path = require('path');

const LOCALES_DIR = './public/locales';
const BASE_LOCALE = 'en';
const OTHER_LOCALES = ['vi', 'ja'];

function getKeys(obj, prefix = '') {
  return Object.entries(obj).flatMap(([key, value]) =>
    typeof value === 'object' && value !== null
      ? getKeys(value, `${prefix}${key}.`)
      : [`${prefix}${key}`]
  );
}

let hasMissing = false;

const namespaces = fs.readdirSync(path.join(LOCALES_DIR, BASE_LOCALE))
  .filter((f) => f.endsWith('.json'))
  .map((f) => f.replace('.json', ''));

for (const ns of namespaces) {
  const baseData = JSON.parse(
    fs.readFileSync(path.join(LOCALES_DIR, BASE_LOCALE, `${ns}.json`), 'utf8')
  );
  const baseKeys = new Set(getKeys(baseData));

  for (const locale of OTHER_LOCALES) {
    const localeFile = path.join(LOCALES_DIR, locale, `${ns}.json`);
    if (!fs.existsSync(localeFile)) {
      console.error(`MISSING FILE: ${locale}/${ns}.json`);
      hasMissing = true;
      continue;
    }

    const localeData = JSON.parse(fs.readFileSync(localeFile, 'utf8'));
    const localeKeys = new Set(getKeys(localeData));

    for (const key of baseKeys) {
      if (!localeKeys.has(key)) {
        console.error(`MISSING KEY [${locale}/${ns}]: ${key}`);
        hasMissing = true;
      }
    }
  }
}

if (hasMissing) {
  console.error('\nFound missing translations. Build failed.');
  process.exit(1);
}

console.log('All translations complete.');
```

**GitHub Actions CI workflow:**

```yaml
# .github/workflows/i18n-check.yml
name: Check i18n translations

on:
  pull_request:
    paths:
      - 'src/**'
      - 'public/locales/**'

jobs:
  check-translations:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - name: Extract and check translation keys
        run: npm run i18n:check
```

### SSR Considerations

**Vấn đề:** Server render với ngôn ngữ X, client hydrate với ngôn ngữ Y -> hydration mismatch.

**Giải pháp với Next.js Pages Router:**

```javascript
// pages/index.jsx
import { serverSideTranslations } from 'next-i18next/serverSideTranslations';

export async function getStaticProps({ locale }) {
  return {
    props: {
      // Chỉ pass namespaces cần thiết cho trang này
      ...(await serverSideTranslations(locale, ['common', 'home'])),
    },
  };
}
```

**Giải pháp với Express SSR thuần túy:**

```javascript
// server.js
import i18n from './src/i18n.js';

app.use(async (req, res) => {
  // Detect language từ cookie (user đã chọn) hoặc Accept-Language
  const lng = req.cookies.i18next || req.acceptsLanguages(['vi', 'en', 'ja']) || 'en';

  // Clone instance để tránh race condition giữa concurrent requests
  const i18nForRequest = i18n.cloneInstance({ initImmediate: false });
  await i18nForRequest.changeLanguage(lng);
  await i18nForRequest.loadNamespaces(['common', detectPageNamespace(req.path)]);

  // Serialize store để client không phải fetch lại
  const initialI18nStore = i18nForRequest.services.resourceStore.data;

  const html = renderToString(
    <I18nextProvider i18n={i18nForRequest}>
      <App />
    </I18nextProvider>
  );

  // Set cookie để persist language choice
  res.cookie('i18next', lng, { maxAge: 365 * 24 * 60 * 60 * 1000 });

  res.send(`
    <!DOCTYPE html>
    <html lang="${lng}">
    <head>...</head>
    <body>
      <div id="root">${html}</div>
      <script>
        window.__I18N_INITIAL_STORE__ = ${JSON.stringify(initialI18nStore).replace(/</g, '\\u003c')};
        window.__I18N_LANGUAGE__ = "${lng}";
      </script>
    </body>
    </html>
  `);
});
```

```javascript
// client i18n.js — rehydrate từ server
i18n.init({
  lng: window.__I18N_LANGUAGE__ || undefined,   // Dùng ngôn ngữ server đã detect
  resources: window.__I18N_INITIAL_STORE__ || undefined, // Skip HTTP fetch nếu có
  ...restConfig,
});
```

### TypeScript Integration

```typescript
// src/types/i18n.d.ts
import 'react-i18next';
import type commonEn from '../../public/locales/en/common.json';
import type authEn from '../../public/locales/en/auth.json';
import type checkoutEn from '../../public/locales/en/checkout.json';

declare module 'react-i18next' {
  interface CustomTypeOptions {
    defaultNS: 'common';
    resources: {
      common: typeof commonEn;
      auth: typeof authEn;
      checkout: typeof checkoutEn;
    };
  }
}

// Bây giờ:
const { t } = useTranslation('auth');
t('login.title');         // OK - TypeScript biết key này
t('login.nonexistent');   // TypeScript ERROR tại compile time
```

## 10. Production Concerns

**Scaling:**

Khi app vượt 10,000 translation keys:
- Dùng TMS (Locize hoặc Phrase) thay vì quản lý JSON files trong Git — translator workflow tốt hơn, versioning riêng, không pollute Git history.
- Split CI pipeline i18n check thành job riêng để không block main build pipeline.
- Implement namespace-level cache busting: đưa content hash vào filename (`common.abc123.json`) và config HttpBackend dùng đúng URL.

**Failure:**

Implement graceful degradation: nếu translation file không load được, component phải render với fallback text chấp nhận được (tiếng Anh hoặc key formatted), không được crash. Dùng `React.ErrorBoundary` quanh Suspense boundaries để catch translation loading errors.

**Monitoring:**

Dashboard metrics cần track:
- Tỉ lệ key fallback (key được trả về thay vì translation) theo ngôn ngữ — tăng đột biến nghĩa là translation file bị missing.
- Thời gian load trung bình của mỗi namespace — nếu >500ms cần xem xét CDN hoặc split file nhỏ hơn.
- Language distribution của users — để ưu tiên dịch ngôn ngữ nào trước.

## 11. Common Mistakes

- Mistake: Đặt tất cả keys vào một namespace `translation.json` duy nhất.
  Fix: Chia thành ít nhất `common.json` (shared UI elements) và các namespace theo feature/page. Nguyên tắc: nếu chỉ một route/feature dùng key đó, nó thuộc namespace của feature đó, không phải `common`.

- Mistake: Dùng concatenation để build translated string với dynamic parts: `t('hello') + ' ' + userName + '!'`.
  Fix: Dùng interpolation: `t('greeting', { name: userName })` với translation `"greeting": "Hello, {{name}}!"`. Translator cần thấy toàn bộ câu để dịch đúng ngữ nghĩa.

- Mistake: Không chạy i18next-parser trong CI, để missing translations lọt lên production.
  Fix: Thêm `i18n:check` vào CI pipeline như một required check, block merge nếu có missing key trong bất kỳ ngôn ngữ nào.

- Mistake: Format date/number trực tiếp trong translation key: `t('createdAt', { date: '17/04/2026' })` với date đã format bằng hardcoded format.
  Fix: Luôn pass raw Date object hoặc ISO string, dùng `Intl.DateTimeFormat` với `i18n.language` để format trong component. Translation chỉ nên chứa cấu trúc câu, không chứa format logic.

- Mistake: Không test RTL (right-to-left) khi thêm ngôn ngữ Ả Rập hoặc Hebrew.
  Fix: Thêm `dir={i18n.dir()}` vào `<html>` element. i18next có `i18n.dir(lng)` trả về `'rtl'` hoặc `'ltr'` dựa trên ngôn ngữ.

- Mistake: Tắt `escapeValue` globally khi dùng React vì "React đã tự escape" — nhưng sau đó truyền i18n instance sang non-React code mà không biết.
  Fix: Khi set `escapeValue: false` trong React context, chỉ áp dụng cho react-i18next. Nếu dùng i18next instance trực tiếp trong vanilla JS code, `escapeValue` vẫn cần được bật.

## 12. Sample Project

**Project: SaaS B2B platform với 5 ngôn ngữ và strict i18n compliance**

Constraint cứng: (1) Mọi PR fail CI nếu thiếu translation trong bất kỳ ngôn ngữ nào. (2) Initial page load không được load >2 namespaces. (3) Không được có FOUC trên SSR pages. (4) Tất cả dates và currencies phải dùng Intl API với locale đúng.

**File structure:**

```
public/locales/
  en/ vi/ ja/ de/ fr/
    common.json          # <20 keys: nav, buttons, status labels
    errors.json          # Error messages
    auth.json            # Login/register flow
    dashboard.json       # Dashboard home
    billing.json         # Billing & payments
    settings.json        # User settings
    admin.json           # Admin panel (chỉ load cho role=admin)

src/
  i18n/
    config.js            # i18next init, chỉ preload 'common'
    formatters.js        # useFormatters custom hook (Intl wrappers)
    namespaces.ts        # Const enum of all namespace names
  types/
    i18n.d.ts            # TypeScript module augmentation

scripts/
  check-translations.js  # CI script: verify all keys present
  extract-translations.js # Wrapper around i18next-parser

.github/workflows/
  i18n-check.yml         # Required CI check on every PR
```

**namespaces.ts — tránh hardcoded strings:**

```typescript
export const NS = {
  COMMON: 'common',
  ERRORS: 'errors',
  AUTH: 'auth',
  DASHBOARD: 'dashboard',
  BILLING: 'billing',
  SETTINGS: 'settings',
  ADMIN: 'admin',
} as const;

export type Namespace = typeof NS[keyof typeof NS];
```

**Lazy loading per route với preload guard:**

```jsx
// pages/Billing.jsx
import { NS } from '../i18n/namespaces';

function BillingContent() {
  const { t } = useTranslation(NS.BILLING);
  const { formatCurrency, formatDate } = useFormatters();

  return (
    <div>
      <h1>{t('title')}</h1>
      <p>{t('nextBillingDate', { date: formatDate(nextBilling) })}</p>
      <p>{t('currentPlan.price', { amount: formatCurrency(price, 'USD') })}</p>
    </div>
  );
}

export default function Billing() {
  return (
    <Suspense fallback={<BillingSkeleton />}>
      <BillingContent />
    </Suspense>
  );
}
```

**CI check result khi developer quên thêm key:**

```
$ npm run i18n:check

Extracting keys from source...
Found 847 keys across 7 namespaces.

Checking translations...
MISSING KEY [vi/billing]: invoiceHistory.downloadButton
MISSING KEY [ja/billing]: invoiceHistory.downloadButton
MISSING KEY [de/settings]: notifications.emailFrequency.options.weekly
MISSING KEY [fr/settings]: notifications.emailFrequency.options.weekly

Found 4 missing translations. Build failed.
npm ERR! script "i18n:check" exited with code 1
```

Developer thấy ngay phải thêm keys nào trước khi merge PR.

## 13. Interview

**Core Q&A:**

Q: Tại sao nên chia namespace theo feature thay vì theo loại (common/errors/pages)?
A: Namespace theo feature cho phép lazy loading tách biệt — chỉ load translation của feature khi user truy cập. Namespace theo loại thường bị abuse thành một file to, khó split. Ngoài ra, feature team có ownership rõ ràng, tránh conflict khi nhiều người cùng sửa file translation.

Q: Tại sao phải dùng `count` trong `t()` ngay cả với ngôn ngữ như tiếng Việt không có complex plurals?
A: Hai lý do: (1) Translator cần thấy `{{count}}` để biết đây là số đếm, dịch đúng context. (2) Nếu sau này thêm ngôn ngữ có plural phức tạp (tiếng Nga, Ả Rập), code đã sẵn sàng mà không cần refactor. Nguyên tắc: logic pluralization luôn trong translation file, không trong code.

Q: i18next-parser hoạt động như thế nào?
A: Parser scan source code bằng AST (Abstract Syntax Tree) hoặc regex để tìm tất cả lời gọi `t()`, `useTranslation()`, `<Trans>`. Nó extract key strings và so sánh với existing JSON files. Keys có trong code nhưng không có trong JSON được report là "missing". Keys có trong JSON nhưng không có trong code được report là "unused". Output có thể tự động thêm missing keys vào JSON với empty string để translator điền vào.

Q: Làm thế nào tránh hydration mismatch trong SSR với i18next?
A: Đảm bảo server và client dùng cùng ngôn ngữ trong initial render. Cách tốt nhất: server detect ngôn ngữ và ghi vào cookie, serialize i18n store và đưa vào HTML. Client đọc từ `window.__I18N_STATE__` thay vì detect lại. Với Next.js, dùng `serverSideTranslations` từ `next-i18next`.

Q: `i18n.dir()` dùng làm gì?
A: Trả về `'rtl'` hoặc `'ltr'` dựa trên ngôn ngữ hiện tại. Dùng để set `dir` attribute trên `<html>` hoặc `<body>` khi hỗ trợ RTL languages (Arabic, Hebrew, Persian, Urdu). Cần kết hợp với CSS logical properties (`margin-inline-start` thay vì `margin-left`) cho layout đúng.

Q: Khi nào nên dùng TMS (Translation Management System) thay vì quản lý JSON trong Git?
A: Khi (1) có translator không phải developer — họ không thể dùng Git; (2) số lượng keys >5000 và nhiều ngôn ngữ; (3) cần workflow review/approval cho bản dịch; (4) cần in-context translation (translator thấy UI khi dịch). Locize, Phrase, Crowdin đều tích hợp tốt với i18next.

Q: Làm thế nào để test i18n trong unit tests?
A: Khởi tạo i18next với `resources` trực tiếp (không dùng HttpBackend) và `initImmediate: false`:
```javascript
i18n.use(initReactI18next).init({
  resources: { en: { common: { 'nav.home': 'Home' } } },
  lng: 'en',
  fallbackLng: 'en',
  initImmediate: false,
});
```
Không cần mock fetch, test chạy synchronously.

**Scenario:**

Q: Translator phàn nàn không hiểu key `btn1`, `btn2`, `btn3` cần dịch thành gì. Vấn đề và fix?
A: Key names không có context. Fix: đổi sang tên mô tả vị trí và chức năng: `loginForm.submitButton`, `loginForm.cancelButton`, `loginForm.forgotPasswordLink`. Nếu cần thêm context cho translator, dùng `description` field trong workflow TMS hoặc thêm comment trong JSON (không được parse bởi i18next nhưng giúp trong TMS).

Q: App của bạn được yêu cầu thêm ngôn ngữ Arabic (RTL). Cần làm những gì?
A: (1) Thêm `ar` vào `supportedLngs`. (2) Tạo file `public/locales/ar/*.json` với bản dịch. (3) Thêm `document.documentElement.dir = i18n.dir()` vào language change handler. (4) Review toàn bộ CSS thay `margin-left/right` bằng `margin-inline-start/end`. (5) Test layout với RTL — đặc biệt flex direction, absolute positioning, icon placement. (6) Kiểm tra font — không phải mọi font hỗ trợ Arabic script.

Q: Load time trang đầu tiên tăng 2 giây sau khi thêm i18n. Tìm nguyên nhân và fix?
A: Audit: (1) Mở Network tab, xem có bao nhiêu translation files được fetch khi load trang. (2) Đo tổng size của tất cả JSON files được load. Nếu load quá nhiều namespaces: (a) Giảm `ns` array trong `init()` xuống chỉ còn namespace cần cho initial render. (b) Còn lại dùng lazy loading qua Suspense. (c) Enable brotli compression ở CDN — JSON compress rất tốt. (d) Cân nhắc preload chỉ ngôn ngữ default; các ngôn ngữ khác load on-demand khi user switch.

## 14. References

- i18next official best practices: https://www.i18next.com/principles/best-practices
- i18next-parser documentation: https://github.com/i18next/i18next-parser
- react-i18next TypeScript guide: https://react.i18next.com/latest/typescript
- i18next lazy loading guide: https://react.i18next.com/guides/lazy-loading-translations
- next-i18next (Next.js integration): https://github.com/i18next/next-i18next
- i18next SSR documentation: https://www.i18next.com/misc/server-side-rendering
- MDN — Intl API reference: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl
- Unicode CLDR plural rules: https://cldr.unicode.org/index/cldr-spec/plural-rules
- i18n cimode for testing: https://www.i18next.com/overview/configuration-options#initimmediate

## 15. Real-world Code

- Cal.com — namespace strategy và SSR setup: https://github.com/calcom/cal.com/tree/main/apps/web/public/static/locales
- Outline — TypeScript i18n types và namespaces: https://github.com/outline/outline/blob/main/app/utils/i18n.ts
- Mattermost — large-scale i18n với CI checks: https://github.com/mattermost/mattermost/tree/master/webapp/i18n
- Grafana — namespace strategy trong enterprise app: https://github.com/grafana/grafana/tree/main/public/locales
- Excalidraw — Intl API kết hợp với i18next: https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/utils.ts

## 16. Community

- i18next GitHub — best practices discussions: https://github.com/i18next/i18next/discussions/categories/q-a
- Locize Blog — Production i18n patterns: https://locize.com/blog/
- DEV.to — "i18n Best Practices with React and i18next": https://dev.to/adrai/i18n-best-practices-with-react-and-i18next-4f0b
- Stack Overflow — i18next namespace strategy: https://stackoverflow.com/search?q=i18next+namespace+best+practices
- LogRocket — "Optimizing React app i18n performance": https://blog.logrocket.com/optimizing-react-app-i18n-performance/
- r/reactjs — SSR + i18n discussion thread: https://www.reddit.com/r/reactjs/search/?q=i18next+SSR+best+practices
