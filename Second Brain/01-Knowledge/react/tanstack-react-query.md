---
created: 2026-04-17
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/react"
  - "#topic/state-management"
related:
  - "[[useQuery-useMutation]]"
  - "[[query-key-array]]"
  - "[[query-key-factory]]"
---

# TanStack Query (React Query)

## 1. What

TanStack Query (tên cũ: React Query) là thư viện quản lý **server state** trong ứng dụng React, cung cấp cơ chế fetching, caching, synchronizing, và updating dữ liệu từ server một cách tự động. Thư viện tách biệt hoàn toàn **server state** (dữ liệu từ API, có thể stale, cần sync) khỏi **client state** (UI state như modal mở/đóng, form values). Phiên bản hiện tại là v5, với nhiều breaking changes so với v4.

---

## 2. Why

Trước khi có TanStack Query, developer phải tự quản lý toàn bộ vòng đời của một API call trong component:

- Tạo `useEffect` để fetch data khi component mount.
- Tạo các state riêng biệt: `isLoading`, `data`, `error`.
- Không có cơ chế cache — mỗi lần navigate lại là fetch lại từ đầu.
- Race conditions: khi user navigate nhanh, response cũ có thể ghi đè response mới.
- Không có background refetch — data có thể stale mà user không biết.
- Invalidation thủ công: sau khi tạo một todo mới, phải tự quyết định khi nào refetch danh sách.
- Code lặp lại ở mọi component cần fetch data.

Tất cả những vấn đề trên dẫn đến codebase khó bảo trì, nhiều bug, và UX kém.

---

## 3. Mental Model

**Phép ẩn dụ: TanStack Query như một người trợ lý thông minh giữ bản photocopy tài liệu cho bạn.**

Hãy tưởng tượng bạn cần một văn bản từ kho lưu trữ (server). Thay vì mỗi lần cần là phải đích thân vào kho lấy, bạn có một trợ lý:

1. **Lần đầu bạn yêu cầu**: trợ lý vào kho lấy bản gốc, photo copy lại, và đưa cho bạn. Bản photocopy này được lưu trong tủ hồ sơ (cache).
2. **Lần sau bạn yêu cầu cùng tài liệu đó**: trợ lý lấy ngay bản photocopy từ tủ hồ sơ, đưa cho bạn ngay lập tức (không phải chờ). Đồng thời, họ âm thầm vào kho kiểm tra bản gốc có thay đổi không (background refetch).
3. **Nếu bản gốc đã thay đổi**: trợ lý photo copy lại và cập nhật tủ hồ sơ — bạn tự động nhận được bản mới mà không cần yêu cầu lại.
4. **Nếu bạn không hỏi đến tài liệu đó lâu**: trợ lý tự động hủy bản photocopy cũ đi để tiết kiệm chỗ (garbage collection sau `gcTime`).
5. **`staleTime`**: khoảng thời gian mà trợ lý tin rằng bản photocopy vẫn còn mới, không cần kiểm tra lại kho.

Điểm mấu chốt: **bạn luôn nhận được data ngay lập tức** (từ cache), trong khi việc sync với server xảy ra ngầm phía sau.

---

## 4. Where it fits

```
User Action (click, navigate)
  -> React Component renders
  -> useQuery() called with queryKey + queryFn
  -> TanStack Query checks QueryCache
       |-- Cache HIT + fresh (within staleTime) -> return cached data immediately
       |-- Cache HIT + stale -> return cached data immediately + trigger background fetch
       |-- Cache MISS -> show isPending=true, run queryFn
  -> queryFn calls your fetch/axios/graphql client
  -> Server responds
  -> QueryCache updated
  -> All subscribers re-render with new data
```

```
Mutation Flow:
User submits form
  -> useMutation().mutate(variables)
  -> mutationFn runs (POST/PUT/DELETE to server)
  -> onSuccess: queryClient.invalidateQueries({ queryKey: ['todos'] })
  -> TanStack Query marks all 'todos' queries as stale
  -> Mounted queries with that key refetch automatically
  -> UI updates
```

---

## 5. When to use

- Ứng dụng cần fetch data từ REST API, GraphQL, hoặc bất kỳ async source nào.
- Cần caching để tránh fetch lại khi navigate giữa các trang.
- Cần background sync để UI luôn hiển thị data mới nhất.
- Cần optimistic updates (hiển thị kết quả trước khi server confirm).
- Cần pagination, infinite scroll với ít boilerplate.
- Khi team cần chuẩn hóa cách handle async data trên toàn ứng dụng.
- Khi muốn tách biệt rõ ràng server state ra khỏi global client state (Redux/Zustand).

---

## 6. When NOT to use

- **Form state hoàn toàn local**: dùng `useState` hoặc React Hook Form, không cần TanStack Query.
- **Real-time data yêu cầu WebSocket thuần túy**: TanStack Query hỗ trợ manual cache updates nhưng không phải là giải pháp WebSocket-first. Kết hợp bằng cách dùng socket events để gọi `queryClient.setQueryData()`.
- **Ứng dụng cực kỳ đơn giản** với 1-2 API call không cần cache: overhead setup không đáng.
- **Thay thế cho client state management**: đừng dùng TanStack Query để lưu theme, language, hay UI state — đó là việc của Zustand/Context.
- **Nhầm lẫn với server-side data fetching**: TanStack Query là client-side library. Với Next.js App Router, cần kết hợp đúng cách (prefetch trên server, hydrate trên client qua `HydrationBoundary`).

---

## 7. Trade-offs

| Pros | Cons |
|---|---|
| Loại bỏ hoàn toàn loading/error boilerplate | Bundle size tăng (~13KB minified+gzipped) |
| Automatic caching và background refetch | Learning curve: hiểu staleTime, gcTime, invalidation |
| Deduplication: cùng queryKey chỉ fetch 1 lần dù nhiều component dùng | v5 breaking changes: cần migrate nếu từ v4 |
| Devtools trực quan để debug cache | Cần tư duy lại về server state vs client state |
| Window focus refetch giúp data luôn fresh | Có thể over-fetch nếu không set staleTime phù hợp |
| Retry logic tự động khi request thất bại | Optimistic update pattern phức tạp khi cần rollback |
| Type-safe với TypeScript | Config QueryClient cần suy nghĩ kỹ cho production |
| Stale-while-revalidate: UI không bị trắng khi refetch | Không normalize cache theo ID như Apollo |

---

## 8. Alternatives

| Library | Caching | Mutations | Bundle Size | Use Case |
|---|---|---|---|---|
| TanStack Query v5 | Mạnh, gcTime | Tốt, optimistic | ~13KB | General purpose, REST/GraphQL |
| SWR (Vercel) | Tốt, stale-while-revalidate | Cơ bản hơn | ~4KB | Đơn giản hơn, Next.js projects |
| RTK Query (Redux Toolkit) | Tốt | Tốt | ~10KB (cần Redux) | Đã dùng Redux, muốn tích hợp |
| Apollo Client | Tốt, normalized by ID | Tốt | ~33KB | GraphQL-first |
| Relay | Rất mạnh, normalized | Tốt | ~57KB | GraphQL large-scale |
| Raw useEffect + fetch | Không có | Thủ công | 0KB | Prototype, rất đơn giản |

---

## 9. How

### Setup cơ bản

```tsx
// main.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,  // 5 phút: data fresh trong 5 phút
      gcTime: 1000 * 60 * 10,    // 10 phút: xoa cache sau 10 phut khong dung
      retry: 2,                   // Thu lai 2 lan khi request that bai
      refetchOnWindowFocus: true, // Refetch khi user focus lai tab
    },
  },
})

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <MyApp />
      {/* Chi hien thi trong development */}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

### Fetch data voi useQuery

```tsx
// hooks/useTodos.ts
import { useQuery } from '@tanstack/react-query'

interface Todo {
  id: number
  title: string
  completed: boolean
}

async function fetchTodos(): Promise<Todo[]> {
  const response = await fetch('/api/todos')
  if (!response.ok) throw new Error('Failed to fetch todos')
  return response.json()
}

export function useTodos() {
  return useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
    staleTime: 1000 * 60 * 2, // Override global: fresh trong 2 phut
  })
}

// components/TodoList.tsx
function TodoList() {
  const { data: todos, isPending, isError, error, isFetching } = useTodos()

  if (isPending) return <div>Loading...</div>
  if (isError) return <div>Error: {error.message}</div>

  return (
    <>
      {/* isFetching hien thi khi background refetch xay ra */}
      {isFetching && <span>Refreshing...</span>}
      <ul>
        {todos.map(todo => (
          <li key={todo.id}>{todo.title}</li>
        ))}
      </ul>
    </>
  )
}
```

### Mutation voi invalidation

```tsx
// hooks/useCreateTodo.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'

export function useCreateTodo() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (title: string) => {
      const response = await fetch('/api/todos', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title }),
      })
      if (!response.ok) throw new Error('Failed to create todo')
      return response.json()
    },
    onSuccess: () => {
      // Invalidate va refetch tat ca queries co key bat dau bang 'todos'
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    },
    onError: (error) => {
      console.error('Create todo failed:', error)
    },
  })
}
```

### Optimistic Update

```tsx
export function useDeleteTodo() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (todoId: number) => {
      const response = await fetch(`/api/todos/${todoId}`, { method: 'DELETE' })
      if (!response.ok) throw new Error('Failed to delete todo')
    },
    onMutate: async (todoId) => {
      // Cancel outgoing refetches de tranh overwrite optimistic update
      await queryClient.cancelQueries({ queryKey: ['todos'] })

      // Luu snapshot de rollback neu can
      const previousTodos = queryClient.getQueryData<Todo[]>(['todos'])

      // Optimistically remove todo khoi cache
      queryClient.setQueryData<Todo[]>(['todos'], (old = []) =>
        old.filter(todo => todo.id !== todoId)
      )

      return { previousTodos }
    },
    onError: (_err, _todoId, context) => {
      // Rollback ve data cu khi co loi
      if (context?.previousTodos) {
        queryClient.setQueryData(['todos'], context.previousTodos)
      }
    },
    onSettled: () => {
      // Luon refetch sau mutation de dam bao sync voi server
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    },
  })
}
```

---

## 10. Production Concerns

### Scaling

- **Tránh query keys quá generic**: `['data']` sẽ khó invalidate chính xác. Dùng hierarchical keys như `['users', userId, 'posts']`. Xem [[query-key-array]] và [[query-key-factory]].
- **`staleTime` phù hợp với từng loại data**: data tĩnh (config, categories) có thể set `staleTime: Infinity`. Data thay đổi thường xuyên (notifications, dashboard) nên set `staleTime: 0` hoặc rất thấp.
- **TanStack Query không normalize cache theo ID**: nếu cùng một user xuất hiện trong nhiều queries, mỗi query có một bản copy riêng. Khi update, cần invalidate tất cả queries liên quan.

### Failure Handling

- **Retry thông minh**: mặc định retry 3 lần với exponential backoff. Với 4xx errors, không nên retry:
  ```ts
  retry: (failureCount, error) => {
    if (error.status === 401 || error.status === 403) return false
    return failureCount < 2
  }
  ```
- **Error boundaries**: set `throwOnError: true` để throw error lên React Error Boundary thay vì handle inline trong mọi component.
- **Offline support**: set `networkMode: 'offlineFirst'` để queries không tự động fail khi mất mạng, thay vào đó chờ cho đến khi có kết nối.

### Monitoring

- **Log errors tập trung**: hook vào `QueryCache` events:
  ```ts
  const queryClient = new QueryClient({
    queryCache: new QueryCache({
      onError: (error, query) => {
        Sentry.captureException(error, {
          extra: { queryKey: query.queryKey }
        })
      },
    }),
  })
  ```
- **Devtools**: `@tanstack/react-query-devtools` hiển thị cache state, active queries, query status, và timing. Không bao giờ ship devtools vào production bundle — nó tự động tree-shake khi `process.env.NODE_ENV === 'production'`.

---

## 11. Common Mistakes

- Mistake: Dùng `isLoading` thay vì `isPending` trong v5 để check trạng thái loading ban đầu.
  Fix: Trong v5, dùng `isPending` để kiểm tra query chưa có data và đang fetch lần đầu. `isFetching` để kiểm tra bất kỳ request nào đang chạy (kể cả background refetch). `isLoading` trong v5 = `isPending && isFetching`.

- Mistake: Tạo `new QueryClient()` bên trong function component, khiến QueryClient bị tạo lại mỗi lần render và toàn bộ cache bị xóa.
  Fix: Tạo `QueryClient` ở module scope (bên ngoài component) hoặc dùng `useRef`/`useState` nếu cần lazy init trong component.

- Mistake: Không set `staleTime` phù hợp, giữ mặc định `staleTime: 0`. Mỗi lần component mount hoặc window focus đều trigger refetch, dẫn đến hàng chục request không cần thiết.
  Fix: Set `staleTime` global trong `defaultOptions.queries` dựa trên tần suất thay đổi của data. Hầu hết ứng dụng benefit từ `staleTime` tối thiểu 1-5 phút.

- Mistake: Gọi `queryClient.invalidateQueries()` bên trong `useEffect` của component để sync data sau action.
  Fix: Luôn đặt invalidation logic trong `onSuccess` hoặc `onSettled` của `useMutation`. `useEffect` không phải nơi để orchestrate server state.

---

## 12. Sample Project

**Project: "News Aggregator with Offline-First Reading"**

**Constraint cứng**: Ứng dụng phải hoạt động khi mất mạng sau lần đầu load. Articles đã được xem phải vẫn đọc được. Khi có mạng trở lại, background sync tự động mà không cần user action.

**Yêu cầu kỹ thuật**:
- Dùng `networkMode: 'offlineFirst'` cho queries về articles.
- Set `gcTime: Infinity` và `staleTime: Infinity` cho article detail queries — data không bao giờ bị xóa khỏi cache trong session.
- Dùng `refetchOnReconnect: true` để tự động sync khi có mạng.
- Implement `queryClient.prefetchQuery()` khi user hover vào một article để preload content trước khi click.
- Dùng `useInfiniteQuery` cho danh sách articles với infinite scroll.
- Mutations (bookmark, mark as read) dùng optimistic updates với rollback.
- Log tất cả query errors lên monitoring service qua `QueryCache` `onError` handler.

---

## 13. Interview

### Core Q&A

**Q: Server state khác client state như thế nào, và tại sao điều này quan trọng với TanStack Query?**
A: Client state là data tồn tại hoàn toàn trong browser — UI state như modal open/close, form input, theme. Server state là data thực sự nằm trên server, chỉ được snapshot vào client — có thể stale bất cứ lúc nào, cần async operations để lấy/update, có thể bị thay đổi bởi user khác. TanStack Query được thiết kế để quản lý server state với caching và sync tự động. Zustand/Redux tốt hơn cho client state. Trộn lẫn hai loại này vào một store là anti-pattern.

**Q: `staleTime` và `gcTime` khác nhau như thế nào?**
A: `staleTime` quyết định bao lâu data được coi là "fresh" kể từ khi fetch xong — trong khoảng thời gian này, TanStack Query không trigger background refetch kể cả khi component remount hay window focus. `gcTime` (cacheTime trong v4) quyết định bao lâu data được giữ trong cache SAU KHI không còn observer nào — tức là sau khi tất cả components dùng query đó đã unmount. Mặc định: `staleTime = 0` (data stale ngay sau khi fetch), `gcTime = 5 phút`.

**Q: `isPending` vs `isFetching` khác nhau thế nào trong v5?**
A: `isPending` = `true` khi query chưa có data trong cache và đang trong quá trình fetch lần đầu. Đây là trạng thái để hiển thị skeleton/loading screen. `isFetching` = `true` khi query đang thực sự gửi request, kể cả background refetch khi đã có data cũ rồi. Dùng `isFetching` để hiển thị subtle loading spinner trong khi vẫn show data cũ.

**Q: Tại sao phải `await queryClient.cancelQueries()` trong `onMutate` khi làm optimistic update?**
A: Nếu có background refetch đang chạy khi ta optimistically update cache, refetch đó hoàn thành sau khi mutation của ta hoàn thành sẽ ghi đè lên optimistic state bằng data cũ từ server (vì server chưa nhận được mutation request ở thời điểm đó). `cancelQueries` hủy request đang bay để đảm bảo không có gì ghi đè optimistic state cho đến khi mutation kết thúc và `onSettled` trigger `invalidateQueries` cuối cùng.

**Q: `queryClient.invalidateQueries()` vs `queryClient.refetchQueries()` khác nhau thế nào?**
A: `invalidateQueries` đánh dấu queries là stale, triggering refetch CHỈ với các queries đang có observer (component đang mounted). Queries không có observer được đánh dấu stale nhưng không refetch ngay — chúng sẽ fetch khi component mount tiếp theo. `refetchQueries` force refetch ngay lập tức, kể cả queries không có observer đang mount. Trong hầu hết trường hợp, `invalidateQueries` là lựa chọn đúng đắn hơn.

**Q: TanStack Query handle deduplication như thế nào?**
A: Khi nhiều component cùng `useQuery` với cùng `queryKey`, TanStack Query chỉ gửi 1 request duy nhất đến server dù có bao nhiêu component. Mỗi component subscribe vào cùng một `QueryObserver`. Khi response về, tất cả subscribers nhận được data cùng lúc. Điều này loại bỏ hoàn toàn vấn đề "N components = N requests" thường gặp với raw `useEffect`.

### Scenario

**Scenario: User logout và bạn cần xóa toàn bộ cached data. Làm thế nào?**
A: Gọi `queryClient.clear()` để xóa toàn bộ QueryCache và MutationCache ngay lập tức. Gọi điều này trước khi navigate về login page để tránh flash of stale data. Nếu chỉ muốn xóa data nhạy cảm (không xóa static data như config), dùng `queryClient.removeQueries()` với filter theo queryKey pattern.

**Scenario: Bạn cần fetch user detail nhưng chỉ khi đã có userId từ một query khác. Implement thế nào?**
A: Dùng `enabled` option để tạo dependent query:
```tsx
const { data: session } = useQuery({ queryKey: ['session'], queryFn: fetchSession })
const { data: profile } = useQuery({
  queryKey: ['profile', session?.userId],
  queryFn: () => fetchProfile(session!.userId),
  enabled: !!session?.userId,
})
```
Query thứ hai sẽ ở trạng thái `isPending` cho đến khi `enabled` trở thành `true`.

**Scenario: API trả về 429 Too Many Requests. Bạn configure retry như thế nào?**
A: Customize `retry` và `retryDelay` để handle rate limiting:
```tsx
retry: (failureCount, error) => {
  if (error.status === 429) return failureCount < 3
  if (error.status >= 400 && error.status < 500) return false
  return failureCount < 2
},
retryDelay: (attemptIndex, error) => {
  // Respect Retry-After header neu co
  const retryAfter = error.headers?.get('Retry-After')
  if (retryAfter) return parseInt(retryAfter) * 1000
  return Math.min(1000 * 2 ** attemptIndex, 30000)
},
```

**Scenario: Sau khi user tạo comment, bạn muốn hiển thị comment ngay lập tức không cần chờ refetch. Làm thế nào?**
A: Hai cách. Cách 1 - Optimistic update trong `onMutate`: update cache ngay với comment tạm thời, rollback trong `onError`. Cách 2 - Update cache trong `onSuccess` với data thực từ server response:
```tsx
onSuccess: (newComment) => {
  queryClient.setQueryData<Comment[]>(['comments', postId], (old = []) => [
    ...old,
    newComment
  ])
}
```
Cách 2 an toàn hơn vì dùng data thật. Luôn kết hợp với `invalidateQueries` trong `onSettled` để đảm bảo sync cuối cùng.

---

## 14. References

- TanStack Query v5 Official Docs: https://tanstack.com/query/v5/docs/framework/react/overview
- Migration Guide v4 to v5: https://tanstack.com/query/v5/docs/framework/react/guides/migrating-to-v5
- Caching Guide: https://tanstack.com/query/v5/docs/framework/react/guides/caching
- Optimistic Updates Guide: https://tanstack.com/query/v5/docs/framework/react/guides/optimistic-updates
- Important Defaults: https://tanstack.com/query/v5/docs/framework/react/guides/important-defaults
- Query Invalidation: https://tanstack.com/query/v5/docs/framework/react/guides/query-invalidation
- GitHub Repository: https://github.com/TanStack/query
- Devtools: https://tanstack.com/query/v5/docs/framework/react/devtools

---

## 15. Real-world Code

- Bulletproof React (production-grade architecture với TanStack Query): https://github.com/alan2207/bulletproof-react
- t3-app (TanStack Query + tRPC + Next.js): https://github.com/t3-oss/create-t3-app
- Calcom (real-world Next.js app dung React Query): https://github.com/calcom/cal.com

---

## 16. Community

- TkDodo's Blog — series "Practical React Query" bởi Dominik Dorfmeister (TanStack Query maintainer), đây là nguồn tài liệu sâu nhất ngoài official docs: https://tkdodo.eu/blog/practical-react-query
- TanStack Query GitHub Discussions: https://github.com/TanStack/query/discussions
- Reddit r/reactjs — tìm kiếm "react query" hoặc "tanstack query": https://www.reddit.com/r/reactjs/
- Stack Overflow [react-query] tag: https://stackoverflow.com/questions/tagged/react-query
- Discord TanStack (community hỗ trợ real-time): https://discord.com/invite/WrRKjPJ
