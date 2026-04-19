---
tags:
  - type/library
  - status/draft
  - lang/javascript
date: 2026-04-17
---

# Day.js

## 1. What

Day.js là một thư viện JavaScript siêu nhẹ (chỉ 2KB sau khi gzip) dùng để parse, validate, manipulate và format ngày tháng/thời gian. API của nó được thiết kế tương thích gần như hoàn toàn với Moment.js, giúp việc chuyển đổi dễ dàng. Điểm khác biệt quan trọng nhất so với Moment.js là Day.js có tính bất biến (immutable) — mỗi phép toán trên một đối tượng Day.js đều trả về một instance mới thay vì thay đổi instance gốc.

## 2. Why

Trước khi Day.js phổ biến, Moment.js là lựa chọn mặc định để xử lý ngày tháng trong JavaScript. Nhưng Moment.js gây ra nhiều vấn đề đáng kể: nó nặng khoảng 280KB (gzip), ảnh hưởng trực tiếp đến bundle size và performance của ứng dụng web. Quan trọng hơn, Moment.js là mutable — khi bạn gọi `.add(7, 'day')` trên một object, nó thay đổi object gốc, gây ra các bug khó tìm khi nhiều nơi cùng dùng chung một reference. Ngoài ra, Moment.js đã chính thức vào "maintenance mode" từ năm 2020, ngừng phát triển tính năng mới. Day.js giải quyết cả ba vấn đề: siêu nhẹ, immutable, và vẫn đang được duy trì tích cực.

## 3. Mental Model

Hãy nghĩ đến Day.js như một máy in báo chí (printing press) thời cổ. Khi bạn đặt lệnh in một bản tin (thao tác ngày tháng như `.add()`, `.subtract()`), máy in KHÔNG sửa trên bản gốc mà luôn in ra một bản sao mới với nội dung được cập nhật. Bản gốc vẫn nguyên vẹn. Mỗi lệnh tiếp theo lại tạo ra thêm một bản sao mới.

Plugin là như các mô đun mở rộng cho máy in: mặc định máy chỉ in chữ đen trắng. Khi bạn lắp thêm mô đun "in màu" (plugin `relativeTime`), máy có thêm khả năng in màu — nhưng bạn phải lắp mô đun trước khi sử dụng.

Tốc độ: máy in rất nhẹ, tốc độ, chỉ có những chức năng cơ bản. Nếu cần in 3D (xử lý timezone cực phức tạp), bạn cần một máy khác (Luxon).

## 4. Where It Fits

```
Data source (API response / database / user input)
          |
          v
       Day.js
    (parse + manipulate)
          |
          v
    Formatted string / timestamp
          |
          v
  UI Display / API request body
```

Day.js sống trong lớp Utility / Helper của ứng dụng — giữa lớp data thô và lớp hiển thị.

```
Backend API (ISO string: "2026-04-17T10:30:00Z")
  |
  v
dayjs("2026-04-17T10:30:00Z")     <- parse
  |
  v
.tz("Asia/Ho_Chi_Minh")           <- chuyển timezone
  |
  v
.format("DD/MM/YYYY HH:mm")       <- format hiển thị
  |
  v
UI: "17/04/2026 17:30"
```

## 5. When to Use

- Khi cần xử lý, so sánh, hoặc format ngày tháng trong bất kỳ ứng dụng web nào (React, Vue, Node.js, vanilla JS).
- Khi ưu tiên bundle size tối ưu — đặc biệt là ứng dụng mobile web hoặc ứng dụng với nhiều người dùng bằng data 3G.
- Khi đang chuyển đổi từ Moment.js vì API rất tương đồng, refactor nhanh.
- Khi cần hỗ trợ relative time ("5 phút trước", "2 ngày nữa"), timezone conversion, hoặc locale (ngôn ngữ địa phương).
- Khi dự án là Node.js backend cần xử lý ngày tháng (luật lưu trữ log, tính toán deadline, schedule).

## 6. When NOT to Use

- Khi ứng dụng chỉ cần format ngày tháng cơ bản và không cần xử lý timezone — `Intl.DateTimeFormat` có sẵn trong trình duyệt là đủ, không cần thêm thư viện.
- Khi xử lý timezone phức tạp là nghiệp vụ cốt lõi (ứng dụng lịch công ty toàn cầu, tính toán chứng khoán theo giờ thị trường) — Luxon có hỗ trợ IANA timezone đầy đủ hơn và xử lý edge case tốt hơn.
- Khi dự án yêu cầu tree-shaking tuyệt đối và bạn chỉ dùng vài tính năng — `date-fns` với function-per-file approach có thể tree-shake tốt hơn.
- Khi dùng Temporal API (đề ra) — Temporal là tiêu chuẩn ECMAScript tương lai cho xử lý ngày tháng, sẽ thay thế mọi thư viện hiện tại.

## 7. Trade-offs

| Pros | Cons |
|------|------|
| Siêu nhẹ: 2KB (gzip), không ảnh hưởng bundle | Plugin system: phải import và extend trước khi dùng |
| Immutable: an toàn, không có shared-state bug | Một số plugin có độ ẩn phụ với nhau (utc + timezone phải dùng đúng thứ tự) |
| API tương thích Moment.js: migrate nhanh | Không hỗ trợ Temporal API (tiêu chuẩn tương lai) |
| Plugin linh hoạt: chỉ load những gì cần | Plugin timezone phụ thuộc vào `Intl.DateTimeFormat` của trình duyệt (có vấn đề với trình duyệt rất cũ) |
| Hỗ trợ locale nhiều ngôn ngữ | Một số edge case timezone ít phổ biến có thể cho kết quả sai |
| Tree-shaking tốt với module bundler | Không có built-in duration formatting phong phú như Luxon |

## 8. Alternatives

| Thư viện | Kích thước (gzip) | Immutable | Timezone | Tree-shaking | Ghi chú |
|----------|-------------------|-----------|----------|-------------|---------|
| Day.js | ~2KB (core) | Có | Plugin | Tốt | Khuyên dùng cho phần lớn dự án |
| Moment.js | ~280KB | Không | Built-in (nhưng cũ) | Kém | Chỉ cho legacy, không dùng mới |
| date-fns | ~12KB (dùng cả) | Có (functional) | Plugin (date-fns-tz) | Tuyệt vời | Mỗi function là một module riêng |
| Luxon | ~23KB | Có | IANA đầy đủ | Trung bình | Dành cho timezone phức tạp |
| Temporal API | 0 (native) | Có | Native | N/A | Tiêu chuẩn tương lai, chưa đầy đủ hỗ trợ trình duyệt |
| Intl.DateTimeFormat | 0 (native) | N/A | Native | N/A | Chỉ format, không manipulate |

## 9. How

### Setup và Plugin cơ bản

```javascript
// Cài đặt
// npm install dayjs

import dayjs from 'dayjs';

// Import plugins cần thiết
import relativeTime from 'dayjs/plugin/relativeTime';
import utc from 'dayjs/plugin/utc';
import timezone from 'dayjs/plugin/timezone';
import duration from 'dayjs/plugin/duration';
import customParseFormat from 'dayjs/plugin/customParseFormat';
import isBefore from 'dayjs/plugin/isBefore'; // Note: isBefore là built-in, ko phải plugin
import 'dayjs/locale/vi'; // Import locale Việt Nam

// Extend plugins (phải gọi trước khi sử dụng)
dayjs.extend(relativeTime);
dayjs.extend(utc);
dayjs.extend(timezone);
dayjs.extend(duration);
dayjs.extend(customParseFormat);

// Đặt locale mặc định (có thể thay đổi sau)
dayjs.locale('vi');
```

### Khởi tạo (Parsing)

```javascript
import dayjs from 'dayjs';

// Từ thời điểm hiện tại
const now = dayjs();

// Từ ISO 8601 string (định dạng chuẩn từ API)
const fromISO = dayjs('2026-04-17T10:30:00Z');

// Từ timestamp Unix (milliseconds)
const fromTimestamp = dayjs(1713351000000);

// Từ định dạng tùy chỉnh (yêu cầu plugin customParseFormat)
const fromCustom = dayjs('17/04/2026', 'DD/MM/YYYY');
const fromCustom2 = dayjs('2026-04-17 10:30', 'YYYY-MM-DD HH:mm');

// Kiểm tra tính hợp lệ
console.log(dayjs('invalid-date').isValid()); // false
console.log(dayjs('2026-04-17').isValid());   // true

// Từ Date object của JavaScript
const fromDate = dayjs(new Date(2026, 3, 17)); // Tháng tính từ 0
```

### Formatting

```javascript
const d = dayjs('2026-04-17T14:30:00');

// Các định dạng phổ biến
d.format('YYYY-MM-DD');           // "2026-04-17"
d.format('DD/MM/YYYY');           // "17/04/2026"
d.format('DD/MM/YYYY HH:mm:ss'); // "17/04/2026 14:30:00"
d.format('YYYY-MM-DDTHH:mm:ssZ'); // ISO 8601 với timezone offset

// Viết tắt token
d.format('ddd, MMM D YYYY');     // "Fri, Apr 17 2026"

// Lấy giá trị số
d.year();    // 2026
d.month();   // 3 (0-indexed! Tháng 4 = index 3)
d.date();    // 17 (ngày trong tháng)
d.day();     // 5 (0=CN, 1=T2, ..., 5=T6, 6=T7)
d.hour();    // 14
d.minute();  // 30
d.unix();    // Unix timestamp (giây)
d.valueOf(); // Unix timestamp (milliseconds)
```

### Manipulation (Immutable)

```javascript
const today = dayjs('2026-04-17');

// Mỗi operation trả về INSTANCE MỚI, today không thay đổi
const tomorrow = today.add(1, 'day');          // 2026-04-18
const nextWeek = today.add(1, 'week');         // 2026-04-24
const nextMonth = today.add(1, 'month');       // 2026-05-17
const nextYear = today.add(1, 'year');         // 2027-04-17
const twoHoursLater = today.add(2, 'hour');
const thirtyMinsBefore = today.subtract(30, 'minute');

console.log(today.format()); // Vẫn là 2026-04-17 — KHÔNG bị thay đổi

// Set giá trị cụ thể
const startOfMonth = today.startOf('month'); // 2026-04-01 00:00:00
const endOfMonth = today.endOf('month');     // 2026-04-30 23:59:59
const startOfDay = today.startOf('day');     // 2026-04-17 00:00:00

// Chain operations
const result = dayjs()
  .add(1, 'month')
  .startOf('month')
  .add(2, 'week')
  .format('DD/MM/YYYY');
```

### So sánh

```javascript
const d1 = dayjs('2026-04-17');
const d2 = dayjs('2026-04-20');
const d3 = dayjs('2026-04-17');

// Built-in methods
d1.isBefore(d2);       // true
d1.isAfter(d2);        // false
d1.isSame(d3);         // true
d1.isSame(d2, 'month'); // true (cùng tháng 4)
d1.isSame(d2, 'year');  // true (cùng năm 2026)

// diff: tính khoảng cách
d2.diff(d1, 'day');        // 3
d2.diff(d1, 'hour');       // 72
d2.diff(d1, 'month');      // 0 (chưa đủ 1 tháng)
d2.diff(d1, 'month', true); // 0.096... (số thập phân)
```

### Relative Time (plugin relativeTime)

```javascript
import dayjs from 'dayjs';
import relativeTime from 'dayjs/plugin/relativeTime';
import 'dayjs/locale/vi';
dayjs.extend(relativeTime);

// "Tính từ bây giờ, bao lâu trước/sau?"
dayjs().subtract(5, 'minute').fromNow(); // "5 phút trước" (nếu locale = vi)
dayjs().subtract(2, 'hour').fromNow();   // "2 giờ trước"
dayjs().subtract(1, 'day').fromNow();    // "một ngày trước"
dayjs().add(3, 'day').fromNow();         // "3 ngày nữa"
dayjs().add(1, 'month').fromNow();       // "một tháng nữa"

// "Tính từ một mốc thời gian khác"
const post = dayjs('2026-01-01');
const now = dayjs('2026-04-17');
post.from(now); // "3 tháng trước"

// Đặt locale viết trong component React:
// dayjs.locale('vi') - đặt global
// hoặc dùng locale per-instance:
dayjs().subtract(5, 'minute').locale('vi').fromNow();
```

### UTC và Timezone (plugin utc + timezone)

```javascript
import dayjs from 'dayjs';
import utc from 'dayjs/plugin/utc';
import timezone from 'dayjs/plugin/timezone';
dayjs.extend(utc);
dayjs.extend(timezone);

// Parse UTC time từ API
const utcTime = dayjs.utc('2026-04-17T07:30:00Z');
console.log(utcTime.format()); // "2026-04-17T07:30:00Z"

// Chuyển sang timezone cụ thể
const hoChiMinh = utcTime.tz('Asia/Ho_Chi_Minh');
console.log(hoChiMinh.format('DD/MM/YYYY HH:mm')); // "17/04/2026 14:30"

const newYork = utcTime.tz('America/New_York');
console.log(newYork.format('DD/MM/YYYY HH:mm'));    // "17/04/2026 03:30"

// Tạo ngày tháng TRONG timezone cụ thể (không convert, tạo mới)
const localTime = dayjs.tz('2026-04-17 14:30', 'Asia/Ho_Chi_Minh');
console.log(localTime.utc().format()); // Convert sang UTC để gửi lên server

// Lấy timezone của máy hiện tại
const userTimezone = dayjs.tz.guess(); // "Asia/Ho_Chi_Minh"
```

### Duration (plugin duration)

```javascript
import dayjs from 'dayjs';
import duration from 'dayjs/plugin/duration';
dayjs.extend(duration);

// Tạo duration
const dur = dayjs.duration(1, 'hour');
const dur2 = dayjs.duration({ hours: 2, minutes: 30, seconds: 15 });
const dur3 = dayjs.duration(90, 'minute'); // 1.5 giờ

// Format duration
dur2.format('HH:mm:ss'); // "02:30:15"
dur3.hours();            // 1
dur3.minutes();          // 30
dur3.asMinutes();        // 90 (tổng số phút)
dur3.asSeconds();        // 5400

// Tính duration giữa 2 mốc thời gian (kết hợp với diff)
const start = dayjs('2026-04-17 09:00');
const end = dayjs('2026-04-17 17:30');
const workDuration = dayjs.duration(end.diff(start));
console.log(`Làm việc: ${workDuration.hours()} tiếng ${workDuration.minutes()} phút`);
// "Làm việc: 8 tiếng 30 phút"
```

## 10. Production Concerns

**Plugin initialization:** Trong ứng dụng React/Vue/Node.js, `dayjs.extend()` phải được gọi TRƯỚC khi bất kỳ component nào sử dụng plugin. Đặt tất cả plugin extensions vào một file `src/lib/dayjs.ts` và import file này ở entry point của ứng dụng (main.tsx hoặc App.tsx). Nếu quên, sẽ gặp lỗi `TypeError: dayjs(...).fromNow is not a function` rất khó debug.

**Locale management:** Nếu ứng dụng hỗ trợ đa ngôn ngữ (i18n), khi user đổi ngôn ngữ, gọi `dayjs.locale(newLocale)` để cập nhật locale cho toàn bộ app. Tuy nhiên, đây là global state — trong ứng dụng có SSR (Next.js), global locale có thể bị race condition giữa các request. Giải pháp: dùng per-instance locale: `dayjs().locale(userLocale).fromNow()`.

**Bundle size:** Dù Day.js core chỉ 2KB, mỗi plugin thêm ~1-3KB. Không import plugin không dùng. Kiểm tra bằng `vite-plugin-visualizer` hoặc `webpack-bundle-analyzer`.

**Timezone data:** Plugin timezone của Day.js dựa vào `Intl.DateTimeFormat` của JavaScript runtime. Trên Node.js versions cũ (< 14), IANA timezone database có thể không đầy đủ. Kiểm tra timezone support: `Intl.DateTimeFormat().resolvedOptions().timeZone`.

**Server-Client consistency:** Khi render ngày tháng trên server (Next.js SSR), đảm bảo server và client dùng cùng locale và timezone. Sử dụng `dayjs.utc()` trên server và chuyển timezone ở client, tránh hydration mismatch.

## 11. Common Mistakes

- Mistake: Gọi `.fromNow()`, `.from()`, `.to()` mà quên `dayjs.extend(relativeTime)`, dẫn đến lỗi runtime.
  Fix: Tập trung tất cả `dayjs.extend()` vào một file cấu hình duy nhất (`src/config/dayjs.ts`) và import ở entry point. Document rõ các plugin đang dùng.

- Mistake: Nghĩ `dayjs()` tự hiểu mọi định dạng string, rồi gọi `dayjs('17/04/2026')` mà không có plugin.
  Fix: `dayjs()` chỉ hiểu ISO 8601 mặc định. Với định dạng tùy chỉnh, phải extend `customParseFormat` và truyền format string: `dayjs('17/04/2026', 'DD/MM/YYYY')`.

- Mistake: Dùng `.month()` mà quên nó trả về 0-indexed (0 = tháng 1, 11 = tháng 12), dẫn đến tính toán sai.
  Fix: Khi hiển thị tháng cho user, thêm 1: `dayjs().month() + 1`. Khi truyền vào hàm `dayjs()` với object `{month: ...}`, cũng phải trừ 1: `dayjs({month: userMonth - 1})`.

- Mistake: Mutate result của Day.js operation rồi sử dụng cả hai biến (nghĩ là độc lập nhưng thực ra không).
  Fix: Day.js là immutable, nên đây không phải bug của Day.js. Nhưng nếu đang chuyển từ Moment.js sang, hãy kiểm tra lại các chỗ dùng biến chung.

## 12. Sample Project

**Project: Twitter-style Post Feed với Relative Time và i18n**

Constraint khó: (1) Bài đăng < 60 giây: "Vừa xong". (2) Bài đăng < 24h: "5 phút trước", "2 giờ trước". (3) Bài đăng >= 24h: hiển thị "17/04/2026". (4) Khi user đổi ngôn ngữ (VI <-> EN), tất cả thời gian trên feed phải cập nhật ngay lập tức. (5) Server trả về UTC ISO string; client ở nhiều timezone khác nhau.

```typescript
// src/config/dayjs.ts — Entry point setup
import dayjs from 'dayjs';
import relativeTime from 'dayjs/plugin/relativeTime';
import utc from 'dayjs/plugin/utc';
import timezone from 'dayjs/plugin/timezone';
import 'dayjs/locale/vi';
import 'dayjs/locale/en';

dayjs.extend(relativeTime);
dayjs.extend(utc);
dayjs.extend(timezone);

export { dayjs };

// src/utils/formatPostTime.ts
import { dayjs } from '../config/dayjs';

export function formatPostTime(utcIsoString: string, locale: 'vi' | 'en'): string {
  const postTime = dayjs.utc(utcIsoString).tz(dayjs.tz.guess()); // Chuyển sang timezone user
  const now = dayjs();
  const diffSeconds = now.diff(postTime, 'second');
  const diffHours = now.diff(postTime, 'hour');

  if (diffSeconds < 60) {
    return locale === 'vi' ? 'Vừa xong' : 'Just now';
  }

  if (diffHours < 24) {
    // fromNow() sử dụng locale được set per-instance
    return postTime.locale(locale).fromNow();
  }

  // >= 24h: hiển thị ngày cụ thể theo locale
  return locale === 'vi'
    ? postTime.format('DD/MM/YYYY')
    : postTime.format('MMM D, YYYY');
}

// src/components/PostCard.tsx
import { useMemo } from 'react';
import { formatPostTime } from '../utils/formatPostTime';
import { useLocale } from '../hooks/useLocale'; // Custom hook đọc locale từ store

interface Post {
  id: string;
  content: string;
  createdAt: string; // ISO UTC string từ API
  author: { name: string };
}

function PostCard({ post }: { post: Post }) {
  const { locale } = useLocale(); // 'vi' | 'en'

  // useMemo: tính lại chỉ khi post.createdAt hoặc locale thay đổi
  const formattedTime = useMemo(
    () => formatPostTime(post.createdAt, locale),
    [post.createdAt, locale]
  );

  return (
    <article>
      <strong>{post.author.name}</strong>
      <p>{post.content}</p>
      <time dateTime={post.createdAt}>{formattedTime}</time>
    </article>
  );
}
```

## 13. Interview

### Core Q&A

**Q: Tại sao Day.js tốt hơn Moment.js cho ứng dụng hiện đại?**
A: Ba lý do chính: (1) Bundle size: Day.js chỉ 2KB so với 280KB của Moment.js — ảnh hưởng trực tiếp đến thời gian tải trang, đặc biệt trên mobile. (2) Immutability: Moment.js mutable gây ra shared-state bug khó tìm, Day.js luôn trả về instance mới. (3) Moment.js đã chính thức vào maintenance mode từ 2020, không còn phát triển tính năng mới.

**Q: Immutability trong Day.js có nghĩa là gì trong thực tế?**
A: Mỗi method như `.add()`, `.subtract()`, `.startOf()`, `.endOf()` đều KHÔNG thay đổi instance gốc mà trả về một instance Day.js mới. Ví dụ: `const d = dayjs(); const tomorrow = d.add(1, 'day');` — sau lệnh này, `d` vẫn là hôm nay, `tomorrow` là ngày mai. Trong Moment.js, `d.add(1, 'day')` sẽ thay đổi `d` luôn.

**Q: Khi nào dùng `dayjs()`, `dayjs.utc()`, và `dayjs().tz()`?**
A: `dayjs(str)`: parse string, kết quả là local time của máy (phụ thuộc vào timezone hệ thống). Dùng khi chỉ cần xử lý local time. `dayjs.utc(str)`: parse string như UTC, kết quả là UTC time. Dùng khi xử lý time từ server (server thường gửi UTC). `dayjs().tz('Asia/Ho_Chi_Minh')`: convert một Day.js object sang timezone cụ thể. Dùng khi hiển thị time cho user ở timezone khác hoặc chuyển đổi giữa các timezone.

**Q: Day.js khác gì date-fns?**
A: Day.js dùng OOP approach với method chaining (`dayjs().add(1, 'day').format()`). date-fns dùng functional approach: mỗi tính năng là một function riêng (`addDays(date, 1)`, `format(date, 'dd/MM/yyyy')`). date-fns có tree-shaking tốt hơn (chỉ bundle đúng hàm đang dùng), nhưng API verbose hơn. Day.js đơn giản hơn cho người mới bắt đầu.

**Q: Làm thế nào để xử lý date comparison chính xác khi có timezone?**
A: Luôn convert về cùng timezone trước khi so sánh. Nếu server trả về UTC, dùng `dayjs.utc()` cho cả hai giá trị rồi compare. Nếu cần compare "cùng ngày" theo local time, dùng `.startOf('day')` trước. Ví dụ: `dayjs.utc(date1).startOf('day').isSame(dayjs.utc(date2).startOf('day'))`.

**Q: Plugin `relativeTime` hoạt động như thế nào?**
A: Plugin này add các method `.fromNow()`, `.from(date)`, `.toNow()`, `.to(date)`. Nó tính khoảng cách thời gian bằng diff, rồi map sang string tương ứng theo locale đang active. Locale 'vi' map "5 phút trước", locale 'en' map "5 minutes ago".

### Scenario

**S: User ở Việt Nam đang xem bài post của user ở New York. Thời gian hiển thị phải theo timezone của ai?**
A: Thường hiển thị theo timezone của NGƯỜI XEM (viewer), tức user ở Việt Nam. Server trả về UTC, frontend chuyển sang timezone của viewer: `dayjs.utc(post.createdAt).tz(dayjs.tz.guess()).format(...)`. `dayjs.tz.guess()` tự động lấy timezone của trình duyệt hiện tại.

**S: App có bug: tháng 1 hiển thị là tháng 2. Fix như thế nào?**
A: Dùng `.month()` trả về 0-indexed. Ngoài ra, kiểm tra nếu đang dùng `new Date(year, month, day)` — tham số `month` của constructor này cũng 0-indexed. Fix: khi lấy giá trị để hiển thị: `dayjs().month() + 1`. Khi set tháng: `dayjs().month(userInputMonth - 1)`.

**S: Ứng dụng cần hiển thị countdown timer (bao nhiêu ngày/giờ còn lại trước deadline). Làm thế nào?**
A: Dùng `diff` kết hợp với `duration` plugin:
```javascript
const deadline = dayjs('2026-12-31');
const now = dayjs();
const diff = deadline.diff(now);
const dur = dayjs.duration(diff);
console.log(`${dur.days()} ngày ${dur.hours()} giờ ${dur.minutes()} phút`);
```
Nếu muốn real-time, đặt vào `setInterval` trong `useEffect` và `clearInterval` khi unmount.

## 14. References

- Day.js Official Docs: https://day.js.org/docs/en/installation/installation
- Day.js Plugin List: https://day.js.org/docs/en/plugin/plugin
- Day.js GitHub Repository: https://github.com/iamkun/dayjs
- You Dont Need Momentjs (alternatives): https://github.com/you-dont-need/You-Dont-Need-Momentjs
- Moment.js Project Status (maintenance mode): https://momentjs.com/docs/#/-project-status/

## 15. Real-world Code

- Day.js source code và examples: https://github.com/iamkun/dayjs/tree/dev/docs/en/plugin
- Bulletproof React (date handling patterns): https://github.com/alan2207/bulletproof-react
- Cal.com (open source scheduling app, xử lý timezone): https://github.com/calcom/cal.com
- date-fns examples (để so sánh approach): https://github.com/date-fns/date-fns/tree/main/docs

## 16. Community

- Reddit r/javascript — "Day.js vs date-fns 2024": https://www.reddit.com/r/javascript/search/?q=dayjs+vs+date-fns
- Stack Overflow — dayjs tag: https://stackoverflow.com/questions/tagged/day.js
- Dev.to — "Stop using Moment.js": https://dev.to/search?q=dayjs+moment
- You Don't Need Momentjs GitHub (community alternatives): https://github.com/you-dont-need/You-Dont-Need-Momentjs
