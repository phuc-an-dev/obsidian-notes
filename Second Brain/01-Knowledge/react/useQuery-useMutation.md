---
created: 2026-04-17
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/react"
  - "#topic/state-management"
related:
  - "[[tanstack-react-query]]"
  - "[[query-key-array]]"
---

# useQuery and useMutation

## 1. What

`useQuery` và `useMutation` là hai hook cốt lõi của TanStack Query v5 — `useQuery` dùng để đọc dữ liệu từ server (GET), còn `useMutation` dùng để ghi dữ liệu (POST, PUT, PATCH, DELETE). Hai hook này tạo thành cặp đôi hoàn chỉnh cho mọi tác vụ tương tác với server trong một React component.

---

## 2. Why

Trước khi có hai hook này, mỗi developer tự viết pattern riêng cho data fetching và mutation:

- Fetch data: `useEffect` + `useState` cho `loading/data/error` — viết lại ở mọi component.
- Submit form: event handler gọi `fetch`, tự quản lý loading state, tự handle lỗi, tự quyết định khi nào cần refresh data ở màn hình khác.
- Không có cơ chế tự động invalidate cache sau mutation — data trên màn hình list có thể stale ngay sau khi user tạo mới một item.
- Race conditions: fetch chạy song song, response về theo thứ tự ngẫu nhiên.

`useQuery` và `useMutation` đóng gói toàn bộ complexity này vào interface nhất quán, type-safe, và dễ test.

---

## 3. Mental Model

**Phép ẩn dụ: `useQuery` là trợ lý đặt hàng tự động, `useMutation` là nhân viên giao dịch có ủy quyền.**

`useQuery` giống như một subscription: bạn nói "tôi muốn nhận dữ liệu về todos, luôn luôn giữ nó fresh". Trợ lý sẽ tự động fetch khi cần, tự động cập nhật khi data thay đổi, và thông báo cho bạn biết trạng thái hiện tại (đang lấy hàng / đã có hàng / hàng hỏng).

`useMutation` giống như một giao dịch: bạn nói "khi tôi gọi, hãy thực hiện hành động này với server và báo cho tôi biết kết quả". Không tự động trigger — chỉ chạy khi bạn gọi `mutate()`. Sau khi giao dịch xong, bạn quyết định cần cập nhật gì.

Điểm khác biệt then chốt: **`useQuery` là reactive** (tự động chạy, tự động cập nhật), **`useMutation` là imperative** (chỉ chạy khi được gọi).

---

## 4. Where it fits

```
useQuery flow:
Component mounts
  -> useQuery({ queryKey, queryFn }) called
  -> TanStack Query checks cache by queryKey
       |-- Cache exists + fresh -> return { data, isPending: false, isFetching: false }
       |-- Cache exists + stale -> return { data, isPending: false, isFetching: true }
                                   + run queryFn in background
       |-- No cache -> return { data: undefined, isPending: true, isFetching: true }
                       + run queryFn
  -> queryFn resolves -> cache updated -> component re-renders with new data

useMutation flow:
User triggers action (button click, form submit)
  -> mutate(variables) called
  -> onMutate(variables) runs (optional, sync)
  -> mutationFn(variables) runs (async, calls server)
  -> Success path: onSuccess(data, variables, context)
  -> Error path:   onError(error, variables, context)
  -> Always:       onSettled(data, error, variables, context)
  -> Component re-renders with { isSuccess, isError, data, error }
```

---

## 5. When to use

**`useQuery`:**
- Mọi trường hợp cần fetch data từ server và hiển thị trong component.
- Khi cần caching giữa các lần navigate.
- Khi cần dependent queries (query B chỉ chạy sau khi query A có kết quả).
- Khi cần conditional fetching (`enabled` option).
- Khi cần transform data trước khi đưa vào component (`select` option).

**`useMutation`:**
- Tạo, cập nhật, xóa dữ liệu trên server.
- Gửi form.
- Bất kỳ action nào có side effect trên server và cần track trạng thái (loading, success, error).
- Khi cần optimistic updates — cập nhật UI ngay lập tức trước khi server confirm.

---

## 6. When NOT to use

- **`useQuery` cho mutations**: không dùng `useQuery` để gọi POST/PUT/DELETE. `useQuery` được thiết kế để idempotent — nó có thể chạy lại bất cứ lúc nào do caching.
- **`useMutation` để fetch data mà không có side effect**: dùng `useQuery`. Mutations không có caching.
- **Lồng `useMutation` kết quả vào `useQuery` queryFn**: đây là anti-pattern. Mỗi hook có mục đích riêng biệt.
- **Dùng `refetch` từ `useQuery` để trigger fetch sau user action**: dùng `invalidateQueries` thay vì gọi `refetch` thủ công — nó đúng semantics hơn và hoạt động với mọi observer của query đó.

---

## 7. Trade-offs

| Pros | Cons |
|---|---|
| Interface nhất quán cho mọi data fetching | Cần hiểu rõ các trạng thái: isPending vs isFetching vs isLoading |
| Automatic deduplication requests | Callback hell trong useMutation nếu không tổ chức tốt |
| `select` option giúp tối ưu re-renders | `select` chỉ transform data trả về, không filter cache |
| `enabled` flag cho dependent/conditional queries | Dependent query chain dài trở nên phức tạp |
| onSuccess/onError/onSettled callbacks rõ ràng | v5 xóa callbacks trên useQuery — phải dùng useEffect nếu cần side effects |
| Tự động track mutation state (isPending, isSuccess) | Mutation không có cache — không share state giữa components |

---

## 8. Alternatives

| Approach | Caching | Mutation tracking | Boilerplate | Trade-off |
|---|---|---|---|---|
| useQuery + useMutation | Có | Có | Thấp | Phụ thuộc TanStack Query |
| SWR useSWR + useSWRMutation | Có | Có | Thấp | API đơn giản hơn, ít tính năng hơn |
| RTK Query endpoints | Có | Có | Trung bình | Cần Redux setup |
| Raw useEffect + useState | Không | Thủ công | Cao | Không phụ thuộc |
| Apollo useQuery + useMutation | Có, normalized | Có | Trung bình | Chỉ cho GraphQL |

---

## 9. How

### useQuery — Đầy đủ options

```tsx
import { useQuery } from '@tanstack/react-query'

interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'user'
}

async function fetchUser(userId: number): Promise<User> {
  const res = await fetch(`/api/users/${userId}`)
  if (!res.ok) throw new Error(`HTTP ${res.status}`)
  return res.json()
}

function useUser(userId: number | undefined) {
  return useQuery({
    queryKey: ['users', userId],         // Cache key — unique per userId
    queryFn: () => fetchUser(userId!),   // Non-null assertion an toàn vì enabled check
    enabled: userId !== undefined,        // Chỉ fetch khi có userId
    staleTime: 1000 * 60 * 5,           // Fresh trong 5 phút
    gcTime: 1000 * 60 * 30,             // Giữ cache 30 phút sau khi unmount
    retry: 1,                            // Thử lại 1 lần khi fail
    select: (data) => ({                 // Transform trước khi trả về component
      ...data,
      displayName: data.name.toUpperCase(),
      isAdmin: data.role === 'admin',
    }),
    placeholderData: (previousData) => previousData, // Giữ data cũ khi key thay đổi
  })
}

// Sử dụng trong component
function UserProfile({ userId }: { userId: number | undefined }) {
  const { data: user, isPending, isFetching, isError, error } = useUser(userId)

  // isPending: chưa có data gì trong cache, đang fetch lần đầu
  if (isPending) return <Skeleton />

  // isError: query thất bại, không có data
  if (isError) return <ErrorMessage message={error.message} />

  return (
    <div>
      {/* isFetching: đang background refetch — show subtle indicator */}
      {isFetching && <RefreshIndicator />}
      <h1>{user.displayName}</h1>
      {user.isAdmin && <AdminBadge />}
    </div>
  )
}
```

### useQuery — Dependent queries

```tsx
// Query B chỉ chạy khi Query A có kết quả
function useUserPosts(username: string) {
  // Query A: lấy userId từ username
  const userQuery = useQuery({
    queryKey: ['users', 'byUsername', username],
    queryFn: () => fetchUserByUsername(username),
  })

  // Query B: lấy posts của user đó
  const postsQuery = useQuery({
    queryKey: ['posts', 'byUser', userQuery.data?.id],
    queryFn: () => fetchPostsByUserId(userQuery.data!.id),
    enabled: !!userQuery.data?.id, // Chỉ chạy khi có userId
  })

  return {
    user: userQuery.data,
    posts: postsQuery.data,
    isPending: userQuery.isPending || postsQuery.isPending,
    isError: userQuery.isError || postsQuery.isError,
  }
}
```

### useQuery — Parallel queries

```tsx
// Chạy nhiều queries song song — tất cả fetch cùng lúc
function useDashboardData(userId: number) {
  const profileQuery = useQuery({
    queryKey: ['profile', userId],
    queryFn: () => fetchProfile(userId),
  })

  const statsQuery = useQuery({
    queryKey: ['stats', userId],
    queryFn: () => fetchStats(userId),
  })

  const notificationsQuery = useQuery({
    queryKey: ['notifications', userId],
    queryFn: () => fetchNotifications(userId),
  })

  return {
    profile: profileQuery.data,
    stats: statsQuery.data,
    notifications: notificationsQuery.data,
    isLoading: profileQuery.isPending || statsQuery.isPending || notificationsQuery.isPending,
  }
}

// Hoặc dùng useQueries cho số lượng query động
import { useQueries } from '@tanstack/react-query'

function useMultipleUsers(userIds: number[]) {
  return useQueries({
    queries: userIds.map(id => ({
      queryKey: ['users', id],
      queryFn: () => fetchUser(id),
    })),
    combine: (results) => ({
      users: results.map(r => r.data).filter(Boolean),
      isPending: results.some(r => r.isPending),
    }),
  })
}
```

### useMutation — Đầy đủ pattern

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query'

interface UpdateUserInput {
  id: number
  name: string
  email: string
}

function useUpdateUser() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (input: UpdateUserInput) => {
      const res = await fetch(`/api/users/${input.id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(input),
      })
      if (!res.ok) throw new Error('Update failed')
      return res.json() as Promise<User>
    },

    onMutate: async (input) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['users', input.id] })

      // Snapshot
      const previousUser = queryClient.getQueryData<User>(['users', input.id])

      // Optimistic update
      queryClient.setQueryData<User>(['users', input.id], (old) =>
        old ? { ...old, name: input.name, email: input.email } : old
      )

      return { previousUser }
    },

    onError: (_err, input, context) => {
      // Rollback
      if (context?.previousUser) {
        queryClient.setQueryData(['users', input.id], context.previousUser)
      }
    },

    onSuccess: (updatedUser, input) => {
      // Cập nhật cache với data chính xác từ server
      queryClient.setQueryData(['users', input.id], updatedUser)
    },

    onSettled: (_data, _error, input) => {
      // Đảm bảo sync cuối cùng
      queryClient.invalidateQueries({ queryKey: ['users', input.id] })
      queryClient.invalidateQueries({ queryKey: ['users', 'list'] })
    },
  })
}

// Sử dụng trong component
function EditUserForm({ user }: { user: User }) {
  const updateUser = useUpdateUser()

  const handleSubmit = (formData: { name: string; email: string }) => {
    updateUser.mutate(
      { id: user.id, ...formData },
      {
        // Có thể override callbacks ở level component (chạy sau level hook)
        onSuccess: () => {
          toast.success('User updated!')
        },
      }
    )
  }

  return (
    <form onSubmit={...}>
      {/* ... */}
      <button
        type="submit"
        disabled={updateUser.isPending}
      >
        {updateUser.isPending ? 'Saving...' : 'Save'}
      </button>
      {updateUser.isError && (
        <p className="error">{updateUser.error.message}</p>
      )}
    </form>
  )
}
```

### isLoading vs isPending vs isFetching — Summary

```tsx
function StatusDemo() {
  const query = useQuery({ queryKey: ['todos'], queryFn: fetchTodos })

  // isPending: true khi:
  //   - Chưa bao giờ fetch (không có data trong cache)
  //   - Status là 'pending'
  console.log(query.isPending)

  // isFetching: true khi:
  //   - Đang có request bay (bất kỳ loại request nào)
  //   - Bao gồm cả background refetch
  console.log(query.isFetching)

  // isLoading (v5): tương đương isPending && isFetching
  //   - True chỉ khi là lần fetch đầu tiên và chưa có data
  console.log(query.isLoading)

  // isRefetching: true khi isFetching && !isPending
  //   - Đang refetch nhưng đã có data cũ trong cache
  console.log(query.isRefetching)

  // Pattern đúng dùng:
  // - Show skeleton/spinner: isPending
  // - Show "refreshing" indicator nhẹ: isFetching && !isPending
  // - Disable button submit: isFetching
}
```

---

## 10. Production Concerns

### Scaling

- **Custom hooks cho mỗi query**: không gọi `useQuery` trực tiếp trong component. Bọc vào custom hooks (`useTodos`, `useUser`) để dễ test, dễ mock, dễ thay đổi API endpoint.
- **`select` để giảm re-renders**: `select` được memo hóa — nếu kết quả `select` không thay đổi (theo referential equality), component sẽ không re-render dù cache có cập nhật.
  ```tsx
  // Chỉ re-render khi count thay đổi, không re-render khi data khác của todos thay đổi
  const { data: count } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
    select: (todos) => todos.filter(t => !t.completed).length,
  })
  ```

### Failure Handling

- **v5 xóa `onSuccess`/`onError` khỏi `useQuery`**: trong v5, các callbacks này đã bị xóa khỏi `useQuery` options vì chúng gây confusion. Nếu cần side effect khi query thành công/thất bại, dùng `useEffect` với `data` hoặc `error` làm dependency, hoặc đặt logic trong custom hook.
  ```tsx
  const { data, error } = useQuery({ queryKey: ['user'], queryFn: fetchUser })
  
  useEffect(() => {
    if (error) toast.error(error.message)
  }, [error])
  ```
- **`throwOnError`**: set `throwOnError: true` để throw error lên React Error Boundary thay vì handle inline ở mỗi component.

### Monitoring

- **Track mutation errors**: dùng `MutationCache` `onError` để log tất cả mutation failures tập trung, tương tự `QueryCache`.
- **mutation.reset()**: gọi sau khi hiển thị error notification để clear error state, tránh user thấy error message cũ khi mở lại form.

---

## 11. Common Mistakes

- Mistake: Dùng `useQuery` với `queryFn` thực hiện POST request (mutation), khiến TanStack Query có thể tự động chạy lại request do background refetch hay window focus.
  Fix: Tất cả operations có side effect (tạo, sửa, xóa) phải dùng `useMutation`. `useQuery` chỉ dành cho idempotent GET operations.

- Mistake: Truyền inline function vào `queryFn` mà không bọc vào custom hook, dẫn đến mỗi render tạo một function reference mới (dù TanStack Query handle điều này, vẫn là bad practice về organization).
  Fix: Tách `queryFn` thành named async function ở ngoài component, hoặc bọc vào custom hook.

- Mistake: Gọi `refetch()` sau mutation thay vì `invalidateQueries()`.
  Fix: `refetch()` chỉ refetch query hiện tại, `invalidateQueries()` mark stale và refetch tất cả queries khớp với key — bao gồm cả list queries, filtered queries, v.v. Dùng `invalidateQueries` sau mutations.

- Mistake: Không dùng `enabled: false` khi `queryKey` chưa đủ dữ liệu, dẫn đến `queryFn` chạy với `undefined` argument và crash.
  Fix: Khi `queryKey` phụ thuộc vào giá trị có thể undefined, luôn kết hợp với `enabled`:
  ```tsx
  useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId!),
    enabled: userId != null,
  })
  ```

---

## 12. Sample Project

**Project: "Team Task Manager" (Asana-like)**

**Constraint cứng**: Tất cả thao tác tạo/sửa/xóa task phải hiển thị kết quả ngay lập tức trong UI (under 50ms sau khi user action), không chờ đợi server response. Nếu server trả về lỗi, UI phải tự động rollback về trạng thái trước đó.

**Yêu cầu kỹ thuật**:
- `useQuery` với `staleTime: 30_000` cho task list — tránh refetch mỗi khi focus.
- `useMutation` với optimistic updates cho mỗi action: create task, update status, reorder, delete.
- `onMutate` snapshot + rollback pattern cho mỗi mutation.
- `onSettled` luôn gọi `invalidateQueries` để đảm bảo sync cuối cùng.
- Dùng `select` để đếm số task theo status: pending/in-progress/done — tránh re-render toàn bộ list khi chỉ count thay đổi.
- Dependent query: chỉ load task details khi user click vào task (`enabled: selectedTaskId !== null`).
- Parallel queries: load assignee info cùng lúc với task list.

---

## 13. Interview

### Core Q&A

**Q: Khi nào nên dùng `useQuery`, khi nào nên dùng `useMutation`?**
A: Nguyên tắc đơn giản: nếu operation là idempotent và đọc dữ liệu, dùng `useQuery`. Nếu operation có side effect trên server (tạo, sửa, xóa), dùng `useMutation`. `useQuery` có caching và tự động chạy lại, `useMutation` không có caching và chỉ chạy khi được gọi explicit.

**Q: `select` option trong `useQuery` làm gì? Nó có làm thay đổi cache không?**
A: `select` là pure transform function chạy SAU khi data được lấy từ cache, trước khi trả về component. Nó không ảnh hưởng đến cache — data trong cache vẫn là raw data từ server. `select` được memo hóa: nếu raw data thay đổi nhưng output của `select` không thay đổi (theo `Object.is`), component sẽ không re-render. Đây là cách tối ưu re-renders khi chỉ cần một phần nhỏ của data lớn.

**Q: Sự khác biệt giữa `onSuccess` trong `useMutation` option và `onSuccess` truyền vào khi gọi `mutate()`?**
A: Cả hai đều chạy khi mutation thành công, nhưng theo thứ tự: option-level `onSuccess` chạy trước, sau đó component-level `onSuccess` trong `mutate()` chạy sau. Option-level phù hợp cho side effects chung (invalidate cache). Component-level phù hợp cho UI actions cụ thể (show toast, navigate, reset form) — vì nó gần với component, không nên đặt logic UI trong hook.

**Q: Tại sao `isPending` mà không phải `isLoading` trong v5?**
A: Trong v4, `isLoading` nghĩa là "chưa có data và đang fetch". v5 rename thành `isPending` để đồng nhất với React's concurrent mode và Suspense terminology (promise pending state). `isLoading` vẫn tồn tại trong v5 nhưng có nghĩa là `isPending && isFetching`, tức là chỉ true khi đã thực sự bắt đầu request — phân biệt với trường hợp query `isPending` nhưng bị disabled (`enabled: false`).

**Q: `placeholderData` và `initialData` khác nhau thế nào?**
A: `initialData` được coi là real data — nó được lưu vào cache và có thể trigger stale check. `placeholderData` chỉ là data tạm thời cho đến khi real data về, không được lưu vào cache. Dùng `initialData` khi bạn có data thật từ SSR hoặc từ query khác. Dùng `placeholderData` khi muốn show data cũ trong khi filter/pagination đang thay đổi — `placeholderData: (prev) => prev` là pattern phổ biến để tránh UI trống trắng khi chuyển trang.

### Scenario

**Scenario: User đang trên trang list, click vào một item để xem detail. Làm sao để detail page hiển thị data ngay lập tức thay vì show spinner?**
A: Dùng `placeholderData` hoặc `initialData` kết hợp với data đã có trong list cache:
```tsx
const { data: todo } = useQuery({
  queryKey: ['todos', todoId],
  queryFn: () => fetchTodo(todoId),
  placeholderData: () => {
    // Lấy data từ list cache nếu có
    const todos = queryClient.getQueryData<Todo[]>(['todos'])
    return todos?.find(t => t.id === todoId)
  },
})
```
Detail component hiển thị ngay với data từ list (có thể thiếu fields), sau đó update khi detail API trả về.

**Scenario: Có 5 components khác nhau cùng cần user data. Làm sao đảm bảo chỉ có 1 request được gửi?**
A: Dùng cùng `queryKey` trong tất cả 5 components — TanStack Query tự động deduplicate. Chỉ 1 request được gửi, tất cả 5 components đều nhận data cùng lúc. Nên bọc vào một custom hook `useCurrentUser()` sử dụng cùng queryKey `['users', 'me']` để đảm bảo consistency.

**Scenario: Sau khi mutation thất bại nhiều lần, user thấy error message cũ. Làm sao reset?**
A: Gọi `mutation.reset()` để clear `isError`, `error`, `data` về trạng thái initial. Thông thường gọi trong `onClose` của error dialog hoặc khi user bắt đầu nhập liệu lại. Có thể cấu hình `mutation.reset()` tự động sau một khoảng thời gian bằng cách kết hợp với `useEffect`.

---

## 14. References

- useQuery API Reference: https://tanstack.com/query/v5/docs/framework/react/reference/useQuery
- useMutation API Reference: https://tanstack.com/query/v5/docs/framework/react/reference/useMutation
- Mutations Guide: https://tanstack.com/query/v5/docs/framework/react/guides/mutations
- Query Options: https://tanstack.com/query/v5/docs/framework/react/guides/query-options
- Dependent Queries: https://tanstack.com/query/v5/docs/framework/react/guides/dependent-queries
- Parallel Queries: https://tanstack.com/query/v5/docs/framework/react/guides/parallel-queries
- Placeholder Data: https://tanstack.com/query/v5/docs/framework/react/guides/placeholder-query-data

---

## 15. Real-world Code

- Bulletproof React — custom hooks pattern với useQuery/useMutation: https://github.com/alan2207/bulletproof-react/tree/master/src/features
- Hoppscotch (API client) — React Query usage in real production app: https://github.com/hoppscotch/hoppscotch
- Notesnook — offline-first notes app dùng React Query: https://github.com/streetwriters/notesnook

---

## 16. Community

- TkDodo: "You Might Not Need React Query" — khi nào KHÔNG nên dùng: https://tkdodo.eu/blog/you-might-not-need-react-query
- TkDodo: "Mastering Mutations in React Query": https://tkdodo.eu/blog/mastering-mutations-in-react-query
- TkDodo: "React Query and TypeScript": https://tkdodo.eu/blog/react-query-and-type-script
- Reddit thread "useQuery vs useEffect for data fetching": https://www.reddit.com/r/reactjs/
- Stack Overflow: https://stackoverflow.com/questions/tagged/react-query
