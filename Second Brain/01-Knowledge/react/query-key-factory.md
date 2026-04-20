---
created: 2026-04-17
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/react"
  - "#topic/state-management"
related:
  - "[[query-key-array]]"
  - "[[tanstack-react-query]]"
---

# Query Key Factory Pattern

## 1. What

Query Key Factory là một TypeScript pattern — thường là một plain object hoặc module — chứa tất cả các hàm tạo query keys của một entity, đặt tập trung tại một nơi duy nhất. Thay vì viết `['todos', 'list', filters]` rải rác khắp codebase, bạn viết một lần trong factory: `todoKeys.list(filters)` và gọi đi ở bất cứ đâu. Đây là pattern, không phải thư viện — có thể implement thủ công hoặc dùng helper `@lukemorales/query-key-factory`.

---

## 2. Why

Khi dự án lớn ra, query keys bị viết lặp lại ở nhiều chỗ:

- `useQuery({ queryKey: ['todos', 'list', { status: 'active' }] })` trong `TodoList.tsx`
- `queryClient.invalidateQueries({ queryKey: ['todos', 'list'] })` trong `CreateTodo.tsx`
- `queryClient.prefetchQuery({ queryKey: ['todos', 'list', filters] })` trong `router.ts`

Khi cần đổi tên entity từ `'todos'` sang `'tasks'`, hoặc thêm field mới vào key structure, bạn phải tìm kiếm và sửa tất cả các nơi đó. Một typo nhỏ (`'todo'` thay vì `'todos'`) tạo ra cache entry riêng biệt — bug rất khó phát hiện.

Query Key Factory giải quyết:
- **Single source of truth** cho tất cả keys.
- **Refactoring an toàn**: sửa một nơi, ảnh hưởng toàn bộ.
- **Type safety**: TypeScript biết chính xác shape của key.
- **Discoverability**: developer mới chỉ cần xem factory để biết tất cả keys tồn tại.

---

## 3. Mental Model

**Phép ẩn dụ: Query Key Factory như bảng giá cả chuẩn của một quán cà phê.**

Thay vì mỗi nhân viên tự nhớ giá của từng món, quán có một bảng giá (the menu board) treo trên tường — ai cũng nhìn vào đó. Khi giá Americano đổi từ 45k lên 50k, chỉ sửa bảng giá một lần, tất cả nhân viên tự động biết giá mới mà không cần thông báo riêng.

Query Key Factory là bảng giá đó. Mỗi entity (`todos`, `users`, `posts`) có bản "menu" của mình. Tất cả custom hooks, mutation handlers, và prefetch logic đều "nhìn vào menu" này thay vì tự nhớ giá.

**Kết hợp với custom hooks**: Factory hiệu quả nhất khi kết hợp với custom hooks làm data layer:

```
Component
  -> custom hook (useActiveTodos)
    -> useQuery({ queryKey: todoKeys.list({ status: 'active' }), ... })
    -> (ben duoi): todoKeys.list() -> ['todos', 'list', { status: 'active' }]
```

---

## 4. Where it fits

```
src/
  features/
    todos/
      api/
        todo.keys.ts       <- Query Key Factory (định nghĩa)
        todo.api.ts        <- API functions (fetchTodos, createTodo)
        todo.hooks.ts      <- Custom hooks (useTodos, useCreateTodo)
      components/
        TodoList.tsx       <- Chỉ dùng custom hooks, không biết về query keys
        CreateTodoForm.tsx

Luồng sử dụng:
TodoList.tsx
  -> useTodos() [from todo.hooks.ts]
     -> useQuery({ queryKey: todoKeys.list(), queryFn: fetchTodos })
     -> todoKeys.list() returns ['todos', 'list']

CreateTodoForm.tsx
  -> useCreateTodo() [from todo.hooks.ts]
     -> useMutation({ mutationFn: createTodo })
     -> onSuccess: queryClient.invalidateQueries({ queryKey: todoKeys.lists() })
     -> todoKeys.lists() returns ['todos', 'list'] (parent prefix)
```

---

## 5. When to use

- Dự án có trên 5-10 queries khác nhau — lúc này query keys bắt đầu bị lặp lại.
- Team có nhiều người — cần enforce consistency.
- Có các query liên quan phân tầng (list, detail, nested resources).
- Cần global invalidation sau logout (xóa tất cả queries của user).
- Khi refactoring API — cần đổi key structure mà không muốn tìm-sửa khắp codebase.
- Bất cứ lúc nào bạn thấy mình copy-paste query key từ nơi này sang nơi khác.

---

## 6. When NOT to use

- Dự án rất nhỏ (1-2 query keys đơn giản) — overhead không đáng.
- Prototype nhanh — có thể thêm vào sau khi codebase ổn định.
- Không nên dùng Factory Pattern mà có cấu trúc phức tạp, có quá nhiều levels làm cho việc đọc key khó khăn hơn.

---

## 7. Trade-offs

| Pros | Cons |
|------|------|
| Single source of truth cho tất cả keys | Thêm một file/module cần maintain |
| Refactoring an toàn — sửa một chỗ | Over-engineering nếu app nhỏ |
| Type safety và TypeScript inference | Phải nhớ gọi factory thay vì viết key trực tiếp |
| Discoverability: xem factory biết tất cả keys tồn tại | Cần team agreement về cách tổ chức factory |
| Global invalidation (e.g., sau logout) đơn giản | |
| Dễ viết unit test cho factory | |

---

## 8. Alternatives

| Approach | Type safety | Centralized | Maintenance |
|---|---|---|---|
| Raw array keys (no factory) | Không | Không | Kém khi scale |
| QUERY_KEYS constants object (strings) | Partial | Có | Tốt hơn, nhưng vẫn manual |
| Manual factory (plain object/functions) | Có (manual) | Có | Tốt |
| @lukemorales/query-key-factory | Có (auto inference) | Có | Rất tốt |

---

## 9. How

### Manual Factory — cách đơn giản nhất

```ts
// features/todos/api/todo.keys.ts

interface TodoFilters {
  status?: 'active' | 'done' | 'all'
  assigneeId?: number
  page?: number
}

// Pattern phổ biến nhất: object với các hàm tạo keys
export const todoKeys = {
  // Root: cha của tất cả todo keys
  all: ['todos'] as const,

  // Lists: cha của tất cả list queries
  lists: () => [...todoKeys.all, 'list'] as const,

  // List cụ thể với filters
  list: (filters?: TodoFilters) =>
    [...todoKeys.lists(), filters ?? {}] as const,

  // Details: cha của tất cả detail queries
  details: () => [...todoKeys.all, 'detail'] as const,

  // Detail cụ thể của một todo
  detail: (id: number) => [...todoKeys.details(), id] as const,

  // Nested resource: comments của một todo
  comments: (todoId: number) =>
    [...todoKeys.detail(todoId), 'comments'] as const,
}

// Kết quả:
// todoKeys.all        => ['todos']
// todoKeys.lists()    => ['todos', 'list']
// todoKeys.list()     => ['todos', 'list', {}]
// todoKeys.list({ status: 'active' }) => ['todos', 'list', { status: 'active' }]
// todoKeys.details()  => ['todos', 'detail']
// todoKeys.detail(1)  => ['todos', 'detail', 1]
// todoKeys.comments(1)=> ['todos', 'detail', 1, 'comments']
```

### Kết hợp với custom hooks

```ts
// features/todos/api/todo.api.ts
import type { TodoFilters } from './todo.keys'

export interface Todo {
  id: number
  title: string
  status: 'active' | 'done'
  assigneeId: number
}

export async function fetchTodos(filters?: TodoFilters): Promise<Todo[]> {
  const params = new URLSearchParams()
  if (filters?.status) params.set('status', filters.status)
  if (filters?.assigneeId) params.set('assigneeId', String(filters.assigneeId))
  if (filters?.page) params.set('page', String(filters.page))

  const res = await fetch(`/api/todos?${params}`)
  if (!res.ok) throw new Error('Failed to fetch todos')
  return res.json()
}

export async function fetchTodoById(id: number): Promise<Todo> {
  const res = await fetch(`/api/todos/${id}`)
  if (!res.ok) throw new Error('Failed to fetch todo')
  return res.json()
}

export async function createTodo(title: string): Promise<Todo> {
  const res = await fetch('/api/todos', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ title }),
  })
  if (!res.ok) throw new Error('Failed to create todo')
  return res.json()
}
```

```ts
// features/todos/api/todo.hooks.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { todoKeys, type TodoFilters } from './todo.keys'
import { fetchTodos, fetchTodoById, createTodo } from './todo.api'

// Hook cho list với filters
export function useTodos(filters?: TodoFilters) {
  return useQuery({
    queryKey: todoKeys.list(filters),
    queryFn: () => fetchTodos(filters),
    staleTime: 1000 * 60 * 2,
  })
}

// Hook cho detail
export function useTodo(id: number | undefined) {
  return useQuery({
    queryKey: todoKeys.detail(id!),
    queryFn: () => fetchTodoById(id!),
    enabled: id !== undefined,
  })
}

// Mutation hook — invalidate đúng cách
export function useCreateTodo() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: createTodo,
    onSuccess: (newTodo) => {
      // Invalidate tất cả list queries (vì list sẽ thay đổi)
      queryClient.invalidateQueries({ queryKey: todoKeys.lists() })

      // Optionally: thêm todo mới vào cache ngay lập tức
      queryClient.setQueryData(todoKeys.detail(newTodo.id), newTodo)
    },
  })
}

export function useDeleteTodo() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (id: number) => {
      const res = await fetch(`/api/todos/${id}`, { method: 'DELETE' })
      if (!res.ok) throw new Error('Failed to delete todo')
    },
    onSuccess: (_data, deletedId) => {
      // Xóa cache của todo đã bị xóa
      queryClient.removeQueries({ queryKey: todoKeys.detail(deletedId) })
      // Invalidate tất cả lists
      queryClient.invalidateQueries({ queryKey: todoKeys.lists() })
    },
  })
}
```

### Global invalidation sau logout

```ts
// features/auth/auth.hooks.ts
import { useQueryClient } from '@tanstack/react-query'
import { todoKeys } from '../todos/api/todo.keys'
import { userKeys } from '../users/api/user.keys'
import { postKeys } from '../posts/api/post.keys'

export function useLogout() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async () => {
      await fetch('/api/auth/logout', { method: 'POST' })
    },
    onSuccess: () => {
      // Option 1: Xóa tất cả cache
      queryClient.clear()

      // Option 2: Xóa có chọn lọc — giữ static data (categories, config)
      queryClient.removeQueries({ queryKey: todoKeys.all })
      queryClient.removeQueries({ queryKey: userKeys.all })
      queryClient.removeQueries({ queryKey: postKeys.all })

      // Navigate về login page
      window.location.href = '/login'
    },
  })
}
```

### Dùng @lukemorales/query-key-factory

```ts
// Cài đặt: npm install @lukemorales/query-key-factory
import { createQueryKeys, inferQueryKeys } from '@lukemorales/query-key-factory'

// Định nghĩa — ngắn gọn hơn manual factory
export const todoKeys = createQueryKeys('todos', {
  all: null,
  lists: {
    queryKey: null,
    contextQueries: {
      filtered: (filters: TodoFilters) => ({
        queryKey: [filters],
      }),
    },
  },
  detail: (id: number) => ({
    queryKey: [id],
    contextQueries: {
      comments: {
        queryKey: null,
      },
    },
  }),
})

// TypeScript tự động infer types:
// todoKeys.all.queryKey           => readonly ['todos', 'all']
// todoKeys.lists.queryKey         => readonly ['todos', 'lists']
// todoKeys.lists.filtered(f).queryKey => readonly ['todos', 'lists', 'filtered', filters]
// todoKeys.detail(1).queryKey     => readonly ['todos', 'detail', 1]
// todoKeys.detail(1).comments.queryKey => readonly ['todos', 'detail', 1, 'comments']

// Sử dụng trong hooks:
useQuery({
  ...todoKeys.detail(todoId),  // Spread cả queryKey và queryFn scope
  queryFn: () => fetchTodoById(todoId),
})
```

### Type inference với manual factory

```ts
// Lấy type của query key để dùng trong function signatures
import { todoKeys } from './todo.keys'

type TodoListKey = ReturnType<typeof todoKeys.list>
// => readonly ['todos', 'list', TodoFilters | {}]

type TodoDetailKey = ReturnType<typeof todoKeys.detail>
// => readonly ['todos', 'detail', number]

// Dùng trong custom prefetch function
async function prefetchTodoList(
  queryClient: QueryClient,
  filters?: TodoFilters
) {
  await queryClient.prefetchQuery({
    queryKey: todoKeys.list(filters),
    queryFn: () => fetchTodos(filters),
    staleTime: 1000 * 60 * 5,
  })
}
```

---

## 10. Production Concerns

### Scaling

- **Một factory file per entity**: `user.keys.ts`, `todo.keys.ts`, `post.keys.ts`. Mỗi entity độc lập — tránh một file "allKeys.ts" khổng lồ.
- **Barrel export**: tạo `features/todos/api/index.ts` export `todoKeys`, `useTodos`, `useCreateTodo`, v.v. — component chỉ cần import từ một nơi.
- **Giữ factory đơn giản**: factory chỉ tạo keys, không chứa logic fetch. Logic fetch thuộc về `api.ts`, logic hook thuộc về `hooks.ts`.

### Failure Handling

- **Nếu đổi key structure**: vì tất cả keys đi qua factory, chỉ cần sửa factory — nhưng phải kiểm tra lại tất cả `invalidateQueries` calls bởi vì chúng có thể đang dùng parent key mà giờ có thể thay đổi shape.
- **Test factory**: factory là pure functions, rất dễ unit test:
  ```ts
  describe('todoKeys', () => {
    it('list key includes filters', () => {
      expect(todoKeys.list({ status: 'active' }))
        .toEqual(['todos', 'list', { status: 'active' }])
    })
    it('detail key is child of details', () => {
      expect(todoKeys.detail(1)[0]).toBe('todos')
      expect(todoKeys.detail(1)[1]).toBe('detail')
      expect(todoKeys.detail(1)[2]).toBe(1)
    })
  })
  ```

### Monitoring

- Với factory, tất cả keys có cấu trúc cần thiết. Trong TanStack Query Devtools, bạn có thể dễ dàng nhận ra entity từ phần tử đầu của key — làm cho debugging trong DevTools trực quan hơn.

---

## 11. Common Mistakes

- Mistake: Định nghĩa factory key không dùng spread parent, dẫn đến mất hierarchical relationship:
  ```ts
  // WRONG: lists() và all không có parent-child relationship
  const todoKeys = {
    all: ['todos'] as const,
    lists: () => ['todos', 'list'] as const, // Hard-coded, không spread all
    detail: (id: number) => ['todos', 'detail', id] as const,
  }
  ```
  Fix: Luôn spread parent key để đảm bảo hierarchy chính xác:
  ```ts
  const todoKeys = {
    all: ['todos'] as const,
    lists: () => [...todoKeys.all, 'list'] as const,
    detail: (id: number) => [...todoKeys.all, 'detail', id] as const,
  }
  ```

- Mistake: Dùng factory trong component trực tiếp thay vì bọc vào custom hook, dẫn đến component biết quá nhiều về cache internals.
  Fix: Component chỉ dùng custom hooks (`useTodos`, `useCreateTodo`). Chỉ custom hooks mới import và dùng factory. Component không bao giờ tự gọi `queryClient.invalidateQueries` hay import `todoKeys`.

- Mistake: Tạo factory với tất cả keys là strings phẳng thay vì hierarchical arrays:
  ```ts
  // WRONG: mất hierarchical invalidation
  export const todoKeys = {
    all: 'todos',
    list: 'todos-list',
    detail: (id: number) => `todos-detail-${id}`,
  }
  ```
  Fix: Keys phải là arrays để TanStack Query có thể làm prefix matching khi invalidate.

- Mistake: Quên update factory khi thêm query mới, dẫn đến developer khác viết key trực tiếp trong component.
  Fix: Code review checklist: mỗi `useQuery` mới phải đi qua factory. Enforce bằng ESLint rule custom nếu cần: cấm import `useQuery` trực tiếp vào components, chỉ allow trong `*.hooks.ts` files.

---

## 12. Sample Project

**Project: "Multi-tenant SaaS Dashboard"**

**Constraint cứng**: Khi user chuyển đổi giữa các tenants (organizations), KHÔNG ĐƯỢC có bất kỳ data nào của tenant A hiện ra khi đang ở tenant B. Đồng thời, static data không phụ thuộc tenant (như danh sách quốc gia, cấu hình hệ thống) KHÔNG ĐƯỢC bị xóa khi switch tenant — để tránh refetch không cần thiết.

**Yêu cầu kỹ thuật**:
- Mỗi entity factory bao gồm `tenantId` trong key: `todoKeys.list(tenantId, filters)` => `['todos', tenantId, 'list', filters]`.
- Sau khi switch tenant, gọi `queryClient.removeQueries({ queryKey: ['todos', previousTenantId] })` — chỉ xóa data của tenant cũ, giữ data chung.
- `staticKeys` factory cho static data: `staticKeys.countries` => `['static', 'countries']`. Keys này KHÔNG bao gồm tenantId và không bao giờ bị xóa khi switch tenant.
- TypeScript enforce `tenantId` là required param trong tất cả tenant-specific factory methods.
- Viết unit tests cho factory đảm bảo:
  - `removeQueries({ queryKey: ['todos', tenantA] })` không affect `['todos', tenantB]`.
  - `removeQueries({ queryKey: ['static'] })` chỉ khi admin clear cache thủ công.
- Custom hook `useSwitchTenant()` tập trung toàn bộ cache cleanup logic.

---

## 13. Interview

### Core Q&A

**Q: Query Key Factory giải quyết vấn đề gì mà raw array keys không giải quyết được?**
A: Raw array keys vẫn hoạt động đúng về mặt kĩ thuật, nhưng gây ra hai vấn đề khi scale: (1) "magic strings" lặp lại khắp codebase — `['todos', 'list']` viết ở 10 nơi, một typo gây ra bug khó debug; (2) Refactoring nguy hiểm — đổi key structure là "search and replace" thủ công toàn codebase. Factory giải quyết cả hai: single source of truth và type-safe refactoring.

**Q: Tại sao factory method `lists()` (plural) lại khác với `list(filters)` (singular)?**
A: Đây là convention để phân biệt hai level trong hierarchy. `lists()` trả về prefix key cho TẤT CẢ list queries — dùng trong `invalidateQueries` khi muốn invalidate tất cả lists bao gồm cả filtered lists. `list(filters)` trả về key cho một list cụ thể với filters đó — dùng trong `useQuery`. Khi tạo một todo mới, bạn gọi `invalidateQueries({ queryKey: todoKeys.lists() })` để affect tất cả filtered views, không chỉ một.

**Q: Làm sao đảm bảo developer mới trong team không bỏ qua factory và viết key trực tiếp?**
A: Ba cách: (1) Documentation/onboarding: giới thiệu pattern ngay từ đầu. (2) Code review: check mỗi `useQuery` call nếu có query key là raw array thay vì factory call. (3) ESLint custom rule: cấm import `@tanstack/react-query` vào component files, chỉ allow trong `*.hooks.ts` — buộc developer phải dùng custom hooks, và custom hooks dùng factory.

**Q: Có cần dùng `@lukemorales/query-key-factory` hay viết manual factory là đủ?**
A: Manual factory là đủ cho hầu hết trường hợp và không có dependency ngoài. `@lukemorales/query-key-factory` thêm automatic TypeScript inference và có cấu trúc factory nhất quán — hữu ích khi team lớn cần enforce convention. Trade-off: thêm dependency, phức tạp hơn. Khuyến nghị: bắt đầu với manual factory, switch sang library khi team thấy cần enforce hơn.

**Q: Query Key Factory có thể phối hợp với React Query Devtools thế nào?**
A: Vì factory tạo ra keys có cấu trúc nhất quán `[entityName, ...]`, trong Devtools bạn dễ dàng nhận ra entity từ phần tử đầu của key. Mỗi cache entry hiển thị key đầy đủ — `['todos', 'list', { status: 'active' }]` — giúp debug invalidation và cache state dễ hơn nhiều so với keys ngẫu nhiên.

### Scenario

**Scenario: Cần thêm field `organizationId` vào tất cả todo query keys vì backend đổi API. Làm thế nào với factory pattern?**
A: Với factory, chỉ cần sửa một file `todo.keys.ts`:
```ts
export const todoKeys = {
  all: (orgId: string) => ['todos', orgId] as const,
  lists: (orgId: string) => [...todoKeys.all(orgId), 'list'] as const,
  list: (orgId: string, filters?: TodoFilters) =>
    [...todoKeys.lists(orgId), filters ?? {}] as const,
  detail: (orgId: string, id: number) =>
    [...todoKeys.all(orgId), 'detail', id] as const,
}
```
TypeScript sẽ compile error ở tất cả những chỗ chưa truyền `orgId` — không cần grep codebase. Sửa theo compile errors là đủ.

**Scenario: Sau khi logout, tất cả queries phải bị xóa TRỪ cache của static data (country list, feature flags). Implement thế nào?**
A: Mỗi entity factory bao gồm entity prefix riêng: `todoKeys.all = ['todos']`, `userKeys.all = ['users']`, `staticKeys.all = ['static']`. Khi logout:
```ts
// Xóa chỉ user-specific data, giữ static data
queryClient.removeQueries({ queryKey: todoKeys.all })
queryClient.removeQueries({ queryKey: userKeys.all })
// ['static'] keys không bị xóa — vẫn còn trong cache
```
Đây là ưu điểm chính của factory: global invalidation/removal theo entity trở nên một dòng code.

---

## 14. References

- TkDodo: "The Query Key Factory" — bài viết chính thức mô tả pattern này: https://tkdodo.eu/blog/effective-react-query-keys#use-query-key-factories
- @lukemorales/query-key-factory on GitHub: https://github.com/lukemorales/query-key-factory
- @lukemorales/query-key-factory docs: https://github.com/lukemorales/query-key-factory#readme
- TanStack Query Query Keys Guide: https://tanstack.com/query/v5/docs/framework/react/guides/query-keys

---

## 15. Real-world Code

- Bulletproof React — feature-based structure với query keys tập trung: https://github.com/alan2207/bulletproof-react
- query-key-factory examples trong official repo: https://github.com/lukemorales/query-key-factory/tree/main/examples
- Cal.com — large Next.js app với React Query và organized query keys: https://github.com/calcom/cal.com

---

## 16. Community

- TkDodo: "Effective React Query Keys" — Section "Use Query Key Factories": https://tkdodo.eu/blog/effective-react-query-keys
- Luke Morales (tác giả query-key-factory) trên Twitter/X: https://twitter.com/lukemorales_
- Reddit r/reactjs — tham khảo các thread về "react query key management": https://www.reddit.com/r/reactjs/
- Stack Overflow: https://stackoverflow.com/questions/tagged/react-query
- GitHub Discussions TanStack Query — nhiều real-world factory pattern được chia sẻ: https://github.com/TanStack/query/discussions
