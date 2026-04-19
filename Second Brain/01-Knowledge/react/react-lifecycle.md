---
tags:
  - type/concept
  - status/draft
  - lang/react
date: 2026-04-17
related:
  - "[[react-suspense]]"
---

# React Component Lifecycle

## 1. What

React component lifecycle mô tả toàn bộ hành trình mà một component trải qua kể từ khi nó được tạo ra (mounting), qua các lần cập nhật (updating), cho đến khi bị gỡ bỏ khỏi DOM (unmounting). Với class components, React cung cấp các phương thức vòng đời cụ thể như `componentDidMount`, `componentDidUpdate`, `componentWillUnmount`. Với function components (React 16.8+), hook `useEffect` và `useLayoutEffect` là cách thống nhất để can thiệp vào các giai đoạn này.

## 2. Why

Nếu không có cơ chế lifecycle, chúng ta không có cách biết chính xác khi nào nên thực hiện các side effects (gọi API, đăng ký sự kiện, khởi tạo thư viện bên ngoài). Nếu gọi API trực tiếp trong thân hàm component (render), nó sẽ được gọi lại mỗi lần render — gây ra vòng lặp vô hạn. Không có khái niệm "cleanup", timer và event listener sẽ tồn tại mãi mãi sau khi component bị gỡ bỏ, gây memory leak. Lifecycle cung cấp các điểm mốc chính xác để thực hiện logic đúng lúc, đúng một lần, và được dọn dẹp đúng cách.

## 3. Mental Model

Hãy nhìn theo góc độ của một vở kịch trên sân khấu:

**Mounting (Lên sân khấu):** Diễn viên (component) lần đầu tiên bước ra trước khán giả. Trước khi màn màn lên, diễn viên chuẩn bị trang phục và prop (constructor/state init). Khi màn lên, khán giả thấy diễn viên lần đầu (render). Sau khi màn lên hoàn toàn, diễn viên bắt đầu tương tác — bắt tay chào đảo, kết nối micro (componentDidMount / useEffect với `[]`).

**Updating (Diễn xuất):** Kịch bản thay đổi (props mới) hoặc cảm xúc thay đổi (state mới), diễn viên cập nhật thoại và hành động. Sau mỗi lần thay đổi, diễn viên có thể phải thích nghi thêm (componentDidUpdate / useEffect với `[deps]`).

**Unmounting (Hạ màn):** Diễn viên cúi chào, rời khỏi sân khấu, thu dọn đạo cụ của mình (componentWillUnmount / cleanup function trong useEffect). Nếu bỏ qua việc thu dọn, sân khấu sẽ đầy đạo cụ rời rạc — là memory leak.

## 4. Where It Fits

```
Component được tạo (JSX render)
  |
  v
MOUNTING
  render() / return JSX
  |
  v
DOM được cập nhật
  |
  v
componentDidMount / useEffect(..., [])   <-- side effects lần đầu
  |
  v
UPDATING (mỗi khi props hoặc state thay đổi)
  render() / return JSX (mới)
  |
  v
DOM được cập nhật
  |
  v
componentDidUpdate / useEffect(..., [deps])  <-- phản ứng với thay đổi
  |
  v
UNMOUNTING
  |
  v
componentWillUnmount / cleanup function  <-- dọn dẹp
  |
  v
Component bị xóa khỏi DOM
```

Render phase vs Commit phase:

```
Render Phase (pure, có thể bị hủy/lặp lại bởi React)
  Constructor / State init
  render() / function body
  |
  v
Commit Phase (thao tác với DOM thực, không thể hủy)
  React cập nhật DOM
  useLayoutEffect chạy (đồng bộ, trước khi browser paint)
  Browser paint
  useEffect chạy (bất đồng bộ, sau khi browser paint)
```

## 5. When to Use

- **Mounting side effects** (`useEffect` với `[]`): Gọi API lần đầu, khởi tạo WebSocket, đăng ký global event listener, khởi tạo analytics, focus input.
- **Reactive effects** (`useEffect` với `[deps]`): Fetch lại khi ID thay đổi, đồng bộ hóa external store với state, cập nhật document title khi route thay đổi.
- **Cleanup** (return function trong `useEffect`): Hủy AbortController, clearInterval/clearTimeout, removeEventListener, đóng WebSocket, unsubscribe Observable.
- **DOM measurement** (`useLayoutEffect`): Đo kích thước phần tử trước khi browser paint, tạo tooltip/popover cần biết vị trí chính xác, tránh flicker.

## 6. When NOT to Use

- Không đặt API call trực tiếp trong thân hàm component (ngoài `useEffect`) — sẽ gọi lại mỗi render, gây lặp.
- Không dùng `useEffect` để xử lý các sự kiện người dùng (click, submit) — dùng event handler trực tiếp.
- Không chuyển đổi props sang state trong `useEffect` rồi update — đây là anti-pattern "derived state", gây double render và bug khó tìm. Tính toán trực tiếp từ props trong render.
- Không bỏ qua dependency array (`useEffect` không có arg thứ 2) trừ khi thực sự muốn chạy sau mỗi render — đó là behavior vô tình, rất dễ gây bug.

## 7. Trade-offs

| Khía cạnh | Class Component | Function Component + Hooks |
|-----------|----------------|---------------------------|
| Mounting | componentDidMount | useEffect(..., []) |
| Updating (specific dep) | componentDidUpdate + if check | useEffect(..., [dep]) |
| Unmounting cleanup | componentWillUnmount | return () => cleanup trong useEffect |
| DOM measurement | componentDidMount (sync) | useLayoutEffect |
| Code re-use giữa components | HOC / Render Props (phức tạp) | Custom hooks (đơn giản, có thể compose) |
| Điều kiện re-render | shouldComponentUpdate / PureComponent | React.memo, useMemo, useCallback |
| Có thể suspend (Concurrent) | Không tốt (lifecycle order thay đổi) | useEffect tương thích hoàn toàn |

## 8. Alternatives

| Cách tiếp cận | Khi nào phù hợp | Hạn chế |
|--------------|----------------|---------|
| useEffect (built-in) | Phần lớn side effects | Cần hiểu dependency array |
| TanStack Query | Server data fetching | Thêm thư viện |
| Zustand / Jotai subscriptions | Global state sync | Thêm thư viện |
| SWR | Data fetching nhẹ | Ít tính năng hơn TQ |
| xstate | State machine phức tạp | Learning curve cao |

## 9. How

### Class Component — Đầy đủ các phương thức lifecycle

```tsx
import { Component } from 'react';

interface Props { userId: string; }
interface State { user: User | null; loading: boolean; error: string | null; }

class UserProfile extends Component<Props, State> {
  private abortController: AbortController | null = null;

  // 1. MOUNTING: Khởi tạo state
  constructor(props: Props) {
    super(props);
    this.state = { user: null, loading: true, error: null };
  }

  // 2. MOUNTING: Chạy sau khi component xuất hiện trên DOM
  async componentDidMount() {
    this.fetchUser(this.props.userId);
  }

  // 3. UPDATING: Chạy sau mỗi update, nhận snapshot trước đó
  async componentDidUpdate(prevProps: Props) {
    // Chỉ re-fetch nếu userId thực sự thay đổi
    if (prevProps.userId !== this.props.userId) {
      this.abortController?.abort(); // Hủy fetch cũ
      this.fetchUser(this.props.userId);
    }
  }

  // 4. UNMOUNTING: Dọn dẹp trước khi component bị xóa
  componentWillUnmount() {
    this.abortController?.abort();
  }

  private async fetchUser(userId: string) {
    this.abortController = new AbortController();
    this.setState({ loading: true, error: null });
    try {
      const res = await fetch(`/api/users/${userId}`, {
        signal: this.abortController.signal,
      });
      if (!res.ok) throw new Error('Fetch failed');
      const user = await res.json();
      this.setState({ user, loading: false });
    } catch (err) {
      if ((err as Error).name !== 'AbortError') {
        this.setState({ error: (err as Error).message, loading: false });
      }
    }
  }

  render() {
    const { user, loading, error } = this.state;
    if (loading) return <div>Đang tải...</div>;
    if (error) return <div>Lỗi: {error}</div>;
    return <div>{user?.name}</div>;
  }
}
```

### Function Component + useEffect — Cách hiện đại

```tsx
import { useState, useEffect } from 'react';

function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    // AbortController để hủy fetch khi:
    // (a) userId thay đổi (dependency update)
    // (b) Component unmount
    const controller = new AbortController();
    let cancelled = false;

    async function fetchUser() {
      setLoading(true);
      setError(null);
      try {
        const res = await fetch(`/api/users/${userId}`, { signal: controller.signal });
        if (!res.ok) throw new Error('Fetch failed');
        const data = await res.json();
        // cancelled check: tránh set state sau khi đã cleanup
        if (!cancelled) {
          setUser(data);
          setLoading(false);
        }
      } catch (err) {
        if ((err as Error).name !== 'AbortError' && !cancelled) {
          setError((err as Error).message);
          setLoading(false);
        }
      }
    }

    fetchUser();

    // Cleanup: chạy khi userId thay đổi HOẶC khi component unmount
    return () => {
      cancelled = true;
      controller.abort();
    };
  }, [userId]); // Chỉ re-run khi userId thay đổi

  if (loading) return <div>Đang tải...</div>;
  if (error) return <div>Lỗi: {error}</div>;
  return <div>{user?.name}</div>;
}
```

### useEffect — Ba dạng dependency array

```tsx
function Examples() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');

  // (A) Chạy sau MỖI render — ít khi cần, dùng khi thật sự muốn observe mỗi update
  useEffect(() => {
    console.log('Chạy sau mỗi render:', count, name);
  }); // Không có array

  // (B) Chạy MỘT LẦN sau khi mount — tương đương componentDidMount
  useEffect(() => {
    console.log('Component đã mount');
    const subscription = someExternalStore.subscribe(() => { /* ... */ });
    return () => subscription.unsubscribe(); // Cleanup khi unmount
  }, []); // Array rỗng

  // (C) Chạy khi `count` thay đổi — tương đương componentDidUpdate
  useEffect(() => {
    document.title = `Count: ${count}`;
    // Cleanup chạy trước mỗi lần effect này chạy lại
    return () => { document.title = 'App'; };
  }, [count]); // Phụ thuộc vào count

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>Tăng</button>
    </div>
  );
}
```

### useLayoutEffect — Khi cần đo DOM trước khi paint

```tsx
import { useState, useLayoutEffect, useRef } from 'react';

function Tooltip({ text, targetRef }: { text: string; targetRef: React.RefObject<HTMLElement> }) {
  const tooltipRef = useRef<HTMLDivElement>(null);
  const [position, setPosition] = useState({ top: 0, left: 0 });

  // useLayoutEffect: chạy TRƯỚC khi browser paint
  // Đảm bảo tooltip được đặt vị trí chính xác trước khi user thấy
  // Nếu dùng useEffect, có thể thấy tooltip "nhảy" vị trí
  useLayoutEffect(() => {
    if (!targetRef.current || !tooltipRef.current) return;
    const targetRect = targetRef.current.getBoundingClientRect();
    const tooltipRect = tooltipRef.current.getBoundingClientRect();
    setPosition({
      top: targetRect.top - tooltipRect.height - 8,
      left: targetRect.left + (targetRect.width - tooltipRect.width) / 2,
    });
  }, [targetRef]);

  return (
    <div
      ref={tooltipRef}
      style={{ position: 'fixed', top: position.top, left: position.left }}
    >
      {text}
    </div>
  );
}
```

### React 18 Strict Mode — Tại sao useEffect chạy 2 lần

```tsx
// Trong React 18 Strict Mode (development only):
// React mount component -> chạy effect
// React UNMOUNT component -> chạy cleanup
// React mount lại component -> chạy effect lần 2
// Mục đích: Phát hiện side effects KHÔNG thể chạy lại an toàn
// (tức là cleanup của bạn có hoạt động đúng không?)

// SAI — bug bị lộ ra với Strict Mode:
useEffect(() => {
  const ws = new WebSocket('ws://localhost');
  ws.onmessage = handler;
  // Không có cleanup! Sau khi unmount, ws vẫn tồn tại
  // Strict Mode chạy 2 lần -> 2 WebSocket connection
}, []);

// ĐÚNG — cleanup rõ ràng:
useEffect(() => {
  const ws = new WebSocket('ws://localhost');
  ws.onmessage = handler;
  return () => ws.close(); // Cleanup đảm bảo chỉ có 1 connection tại 1 thời điểm
}, []);
```

## 10. Production Concerns

**Memory Leaks:** Lỗi phổ biến nhất là quên cleanup trong `useEffect`. Dấu hiệu: sau khi chuyển trang, console vẫn log, RAM tăng dần. Kiểm tra bằng React DevTools Profiler — component có bị highlight sau khi unmount không? Luôn đặt `AbortController.abort()`, `clearInterval`, `removeEventListener` trong return function.

**Expensive Re-renders:** `useEffect` với dependency array sai (quá nhiều dep hoặc thiếu dep) sẽ chạy lại liên tục. Dùng `eslint-plugin-react-hooks` (`exhaustive-deps` rule) để phát hiện vấn đề này tự động. Trong production, dùng React DevTools Profiler để đếm số lần render và xác định component nào render quá nhiều.

**useLayoutEffect và SSR:** `useLayoutEffect` bị warn trên server (Next.js) vì SSR không có browser paint phase. Nếu cần, dùng `useEffect` thay thế (chấp nhận flicker nhỏ), hoặc dùng pattern: `const isClient = typeof window !== 'undefined'` trước khi gọi `useLayoutEffect`.

**Concurrent Mode:** Trong Concurrent React 18, render phase có thể bị ngắt và chạy lại nhiều lần. Effect setup/cleanup có thể chạy nhiều hơn mong đợi trong Strict Mode (development). Đảm bảo cleanup idempotent: gọi cleanup nhiều lần phải an toàn.

## 11. Common Mistakes

- Mistake: Gọi API trực tiếp trong thân component, ngoài useEffect, nên gọi lại mỗi lần render.
  Fix: Bao giờ cũng đặt fetch bên trong `useEffect` với dependency array hợp lý. Sử dụng TanStack Query nếu có thể để tránh phải tự quản lý hoàn toàn.

- Mistake: Quên cleanup — đặt `setInterval` hoặc `addEventListener` mà không return cleanup function.
  Fix: Mỗi `useEffect` tạo ra "kênh kết nối" tới thế giới bên ngoài đều phải có return cleanup đóng kênh đó. Quy tắc: "Mở thì phải đóng."

- Mistake: Xử lý `useEffect` như `componentDidMount` — nghĩ rằng nó chỉ chạy một lần nên không cần cleanup.
  Fix: Với Strict Mode (React 18), `useEffect` với `[]` vẫn chạy cleanup-mount-cleanup-mount (trong dev). Cleanup vẫn cần thiết. Ngoài ra, trong tương lai Concurrent Mode, React có thể unmount/remount bất kỳ lúc nào.

- Mistake: Đặt object hoặc array literal trong dependency array: `useEffect(() => {...}, [{ id: userId }])`.
  Fix: Object literal tạo ra reference mới mỗi render, khiến effect chạy lại mãi mãi. Chỉ đặt primitive values hoặc stable references vào deps. Nếu phải dùng object, dùng `useMemo` hoặc destructure lấy trường cụ thể: `[userId]`.

## 12. Sample Project

**Project: Real-time User Presence System**

Constraint khó: (1) Khi user mở trang, subscribe realtime presence của những người khác. (2) Khi user chuyển sang user khác, unsubscribe channel cũ và subscribe channel mới — không được có double subscription. (3) Khi tab bị ẩn (visibilitychange), tạm thời disconnet để tiết kiệm resource. (4) Khi unmount, đảm bảo đã cleanup hoàn toàn.

```tsx
// hooks/usePresence.ts
import { useState, useEffect, useRef } from 'react';

interface PresenceUser { id: string; name: string; online: boolean; }

function usePresence(channelId: string) {
  const [users, setUsers] = useState<PresenceUser[]>([]);
  const [connected, setConnected] = useState(false);
  // useRef để giữ stable reference đến cleanup function
  const cleanupRef = useRef<(() => void) | null>(null);

  // Effect 1: Quản lý WebSocket subscription theo channelId
  useEffect(() => {
    const ws = new WebSocket(`wss://api.example.com/presence/${channelId}`);

    ws.onopen = () => setConnected(true);
    ws.onclose = () => setConnected(false);
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setUsers(data.users);
    };

    cleanupRef.current = () => ws.close();

    // Cleanup chạy khi channelId thay đổi HOẶC unmount
    // Đảm bảo chuyển channel không tạo double connection
    return () => {
      ws.close();
      setConnected(false);
    };
  }, [channelId]); // Re-subscribe khi channel thay đổi

  // Effect 2: Tạm dừng khi tab bị ẩn
  useEffect(() => {
    function handleVisibility() {
      if (document.hidden) {
        cleanupRef.current?.(); // Đóng connection
      } else {
        // Reconnect sẽ được xử lý bởi effect 1 khi channelId vẫn như cũ
        // Trong thực tế, cần trigger reconnect (có thể qua state reconnectKey)
      }
    }

    document.addEventListener('visibilitychange', handleVisibility);
    return () => document.removeEventListener('visibilitychange', handleVisibility);
  }, []); // Chỉ đăng ký một lần

  return { users, connected };
}

// Cách dùng:
function RoomPage({ roomId }: { roomId: string }) {
  const { users, connected } = usePresence(roomId);

  return (
    <div>
      <span>{connected ? 'Kết nối' : 'Ngắt kết nối'}</span>
      <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>
    </div>
  );
}
```

## 13. Interview

### Core Q&A

**Q: Sự khác biệt giữa `useEffect` và `useLayoutEffect` là gì?**
A: Cả hai đều chạy sau khi React cập nhật DOM. Nhưng `useLayoutEffect` chạy đồng bộ (synchronous) trước khi browser thực hiện paint — thích hợp để đo DOM và thay đổi vị trí trước khi user thấy. `useEffect` chạy bất đồng bộ sau khi browser paint — thích hợp cho side effects không liên quan đến visual như fetch data, analytics. Dùng sai `useLayoutEffect` có thể block browser paint.

**Q: Tại sao `useEffect` chạy 2 lần trong React 18 Strict Mode?**
A: Đây là hành vi có ý (deliberate) trong development. React mount component, chạy effects, rồi UNMOUNT, rồi mount lại. Mục đích là kiểm tra xem cleanup function của bạn có làm việc đúng không: nếu ứng dụng chạy bình thường sau khi bị unmount-remount, nghĩa là cleanup của bạn tốt. Đây bộc lộ các bug double-subscription, double-fetch mà có thể bị âm hiện trong production Concurrent Mode.

**Q: Sự khác biệt giữa `[]`, `[deps]`, và không có arg nào trong useEffect?**
A: Không có arg: chạy sau MỖI render. `[]`: chạy một lần sau mounting và cleanup khi unmounting (tương đương componentDidMount + componentWillUnmount). `[dep1, dep2]`: chạy sau mounting và mỗi khi `dep1` hoặc `dep2` thay đổi giá trị (tương đương componentDidUpdate với kiểm tra dep).

**Q: Tại sao không nên đặt object literal vào dependency array?**
A: Object literal trong JSX hoặc hàm tạo ra reference mới mỗi render: `{} !== {}`. `useEffect` so sánh deps bằng `Object.is()` (reference equality), nên object mỗi lần render sẽ bị coi là "thay đổi", khiến effect chạy lại sau mỗi render — tạo ra vòng lặp. Giải pháp: destructure lấy các trường primitive, hoặc dùng `useMemo`.

**Q: Render phase và commit phase là gì?**
A: Render phase: React chạy hàm component / `render()` để tính toán JSX output. Giai đoạn này có thể bị lặp lại nhiều lần (Concurrent Mode), phải pure. Commit phase: React áp dụng thay đổi vào DOM thực — không thể hủy, chạy đồng bộ. `useLayoutEffect` chạy cuối commit phase, `useEffect` chạy sau đó bất đồng bộ.

**Q: Custom hook là gì và nó liên quan đến lifecycle như thế nào?**
A: Custom hook là hàm JavaScript bắt đầu bằng `use` và có thể gọi các built-in hooks. Nó cho phép tách logic lifecycle (useEffect, useState) ra khỏi component và tái sử dụng. Ví dụ: `useWindowSize`, `useDebounce`, `usePresence`. Mỗi lần custom hook được gọi trong một component, nó tạo ra instance độc lập — lifecycle gặp theo component chứa nó.

**Q: Phân biệt componentDidUpdate (class) và useEffect (hook) khi xử lý thay đổi props?**
A: `componentDidUpdate(prevProps, prevState)` cung cấp snapshot của lần render trước để so sánh. Với hooks, bạn phải tự lưu giá trị trước bằng `useRef`: `const prevIdRef = useRef(userId)`. Tuy nhiên, trong phần lớn trường hợp, khai báo dep trong `useEffect([userId])` là đủ — React tự động chỉ chạy effect khi `userId` thay đổi, không cần so sánh thủ công.

### Scenario

**S: User ở trang User A, click sang User B, trang vẫn hiện User A. Bug ở đâu?**
A: `useEffect` fetch data nhưng `userId` không ở trong dependency array. Do đó, khi `userId` props thay đổi (update phase), effect không chạy lại. Fix: thêm `userId` vào array: `useEffect(() => { fetchUser(userId); }, [userId])`.

**S: Sau khi chuyển trang, console log vẫn nhảy liên tục, RAM tăng. Memory leak ở đâu?**
A: Component đã unmount nhưng `setInterval` (hoặc WebSocket, event listener) vẫn chạy. Fix: trả về cleanup function trong `useEffect`: `return () => clearInterval(intervalId)` (hoặc `ws.close()`, `removeEventListener`).

**S: Bạn cần fetch data chỉ một lần khi app khởi động, nhưng `useEffect` với `[]` bị chạy 2 lần trong dev.**
A: Đây là Strict Mode trong development. Trong production, sẽ chỉ chạy 1 lần. Design code để chịu được việc chạy 2 lần bằng cleanup: hủy request lần 1 (AbortController) trước khi request lần 2 bắt đầu. Hoặc dùng TanStack Query có deduplication built-in.

**S: Bạn muốn update document.title theo tên của trang hiện tại, nhưng title bị trễ 1 render.**
A: Dùng `useLayoutEffect` thay vì `useEffect`. `useLayoutEffect` chạy trước khi browser paint, đảm bảo title được set trước khi user thấy trang. Lưu ý: cần check SSR compatibility (`typeof window !== 'undefined'`) nếu dùng Next.js.

**S: Component có prop `theme` (object). `useEffect([theme])` chạy lại mỗi render dù theme không thay đổi nội dung.**
A: Vì `theme` là object được tạo mới mỗi render (reference thay đổi). Fix: destructure props lấy trường cụ thể: `useEffect(() => {...}, [theme.color, theme.fontSize])`. Hoặc dùng `useMemo` để stabilize object: `const stableTheme = useMemo(() => theme, [theme.color, theme.fontSize])`.

## 14. References

- React Docs — useEffect: https://react.dev/reference/react/useEffect
- React Docs — useLayoutEffect: https://react.dev/reference/react/useLayoutEffect
- React Docs — Synchronizing with Effects: https://react.dev/learn/synchronizing-with-effects
- React Docs — You Might Not Need an Effect: https://react.dev/learn/you-might-not-need-an-effect
- React Docs — Lifecycle of Reactive Effects: https://react.dev/learn/lifecycle-of-reactive-effects
- React 18 Strict Mode changes: https://react.dev/blog/2022/03/08/react-18-upgrade-guide#updates-to-strict-mode

## 15. Real-world Code

- Bulletproof React — hooks examples: https://github.com/alan2207/bulletproof-react/tree/master/apps/react-vite/src/hooks
- TanStack Query — source (useEffect patterns): https://github.com/TanStack/query/blob/main/packages/react-query/src/useBaseQuery.ts
- use-hooks collection: https://github.com/uidotdev/usehooks
- react-use library: https://github.com/streamich/react-use

## 16. Community

- Reddit r/reactjs — "useEffect vs componentDidMount": https://www.reddit.com/r/reactjs/search/?q=useEffect+componentDidMount
- Stack Overflow — react-hooks tag: https://stackoverflow.com/questions/tagged/react-hooks
- Kent C. Dodds — "useEffect vs useLayoutEffect": https://kentcdodds.com/blog/useeffect-vs-uselayouteffect
- Dan Abramov — "A Complete Guide to useEffect": https://overreacted.io/a-complete-guide-to-useeffect/
- Robin Wieruch — "React useEffect Hook": https://www.robinwieruch.de/react-useeffect-hook/
