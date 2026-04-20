---
created: 2026-04-17
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/state-management"
related:
  - "[[tanstack-react-query]]"
  - "[[query-key-factory]]"
  - "[[useQuery-useMutation]]"
---

# Query Key Arrays

## 1. What

Query Key Arrays là cách TanStack Query định danh và quản lý các entries trong cache. Mỗi `useQuery` nhận một `queryKey` là một array — array này đóng vai trò như khóa chính của cache entry đó. Khi key thay đổi, TanStack Query tự động fetch lại data cho key mới. Việc tổ chức keys theo đúng cấu trúc phân cấp (hierarchical) cho phép invalidate tập hợp queries một cách chính xác và hiệu quả.

---

## 2. Why

Nếu dùng plain string làm cache key, bạn chỉ có hai lựa chọn khi invalidate: invalidate chính xác một key, hoặc invalidate tất cả. Không có cách nào invalidate "tất cả queries liên quan đến todos" mà không ảnh hưởng đến "queries liên quan đến users".

Với array keys và hierarchical structure, bạn có thể:
- Invalidate `['todos']` để affect tất cả: `['todos', 'list']`, `['todos', 1]`, `['todos', 1, 'comments']`.
- Invalidate chỉ `['todos', 1]` mà không affect `['todos', 'list']`.
- Invalidate tất cả data của một user (`['users', userId]`) sau khi logout mà không xóa data khác.

Đây là power của hierarchical invalidation — không thể làm được với string keys.

---

## 3. Mental Model

**Phép ẩn dụ: Query keys là hệ thống sổ lưu trữ theo mục (folder hierarchy).**

Hãy tưởng tượng QueryCache như một tủ hồ sơ:
- Tầng ngoài cùng: `['todos']` — toàn bộ ngăn "todos"
- Tầng tiếp theo: `['todos', 'list']` — ngăn "danh sách todos"
- Sâu hơn: `['todos', 'list', { status: 'active' }]` — "danh sách todos đang active"
- Riêng biệt: `['todos', 1]` — "todo có id = 1"
- Sâu hơn nữa: `['todos', 1, 'comments']` — "comments của todo 1"

Khi bạn gọi `invalidateQueries({ queryKey: ['todos'] })`, đây giống như quát mát ngăn "todos" — tất cả giấy tờ (queries) trong ngăn đó đều bị đánh dấu là stale. Khi bạn gọi với `['todos', 1]`, chỉ ngăn con "todos/1" bị ảnh hưởng.

**Serialization**: TanStack Query serialize array key bằng cách deep compare — `{ status: 'active' }` và `{ status: 'active' }` là CÙNG KEY dù là hai object khác nhau trong JavaScript. Thứ tự các keys trong object KHÔNG ảnh hưởng — `{ a: 1, b: 2 }` và `{ b: 2, a: 1 }` là cùng key.

---

## 4. Where it fits

```
useQuery({ queryKey: ['todos', 'list', { status: 'active', page: 1 }], ... })
  -> TanStack Query serializes key deterministically
  -> Key stored as stable string in QueryCache map
  -> invalidateQueries({ queryKey: ['todos'] })
       -> Matches: ['todos', 'list', { status: 'active', page: 1 }]  YES (prefix match)
       -> Matches: ['todos', 1]                                        YES (prefix match)
       -> Matches: ['todos', 1, 'comments']                           YES (prefix match)
       -> Matches: ['users', 1]                                        NO
  -> invalidateQueries({ queryKey: ['todos', 1] })
       -> Matches: ['todos', 1]                                        YES
       -> Matches: ['todos', 1, 'comments']                           YES
       -> Matches: ['todos', 'list', ...]                              NO
```

---

## 5. When to use

- Mọi lúc dùng TanStack Query — query keys là bắt buộc.
- **Dùng array với nhiều phần tử** khi data có thể được filter, paginate, hoặc có quan hệ cha-con.
- **Dùng object trong array** khi có nhiều params tùy chọn (filters, pagination, search term).
- **Đặt entity name làm phần tử đầu tiên** (`'todos'`, `'users'`, `'posts'`) để hierarchical invalidation hoạt động.
- **Đặt dynamic values ở sau** trong array, sao cho prefix chung đối với các queries cùng entity.

---

## 6. When NOT to use

- **Dùng string key đơn** cho complex queries: `useQuery({ queryKey: 'todos-active-page-1', ... })` — mất khả năng hierarchical invalidation, khó maintain.
- **Đặt dynamic params ở trước entity name**: `['user_123_todos']` là anti-pattern. Phải là `['todos', { userId: 123 }]`.
- **Dựa vào JavaScript object identity**: `const filters = { status: 'active' }` trong component sẽ tạo object mới mỗi render — nhưng TanStack Query so sánh theo giá trị nên đây KHÔNG phải vấn đề với `queryKey`. Tuy nhiên, dùng memoize `filters` nếu các giá trị trong đó phụ thuộc vào state phức tạp.
- **Key quá chung**: `['data']` hay `['list']` là key quá rộng, dễ gây conflict và khó invalidate chính xác.

---

## 7. Trade-offs

| Pros | Cons |
|---|---|
| Hierarchical invalidation chính xác | Dễ viết sai key (typo, thứ tự sai) và khó debug |
| Serialization theo giá trị, không theo reference | Vấn đề "magic strings": 'todos' lặp lại ở nhiều nơi |
| Object params: thứ tự keys không quan trọng | Có thể inconsistent nếu không có quy ước chung |
| Prefix matching linh hoạt cho invalidation | Càng nhiều params, key càng dài và khó đọc |
| Tương thích với TypeScript inference | Không có compile-time check cho key structure |

---

## 8. Alternatives

| Approach | Type safety | Hierarchical invalidation | Maintenance |
|---|---|---|---|
| Array keys (built-in) | Không | Có | Trung bình — "magic strings" |
| String keys | Không | Không | Kém |
| Query Key Factory (pattern) | Có (manual types) | Có | Tốt |
| @lukemorales/query-key-factory | Có (auto inference) | Có | Rất tốt |

Xem thêm: [[query-key-factory]]

---

## 9. How

### Các pattern key phổ biến

```ts
// Entity đơn giản — tất cả todos
useQuery({ queryKey: ['todos'], queryFn: fetchTodos })

// Entity theo ID — một todo cụ thể
useQuery({ queryKey: ['todos', 1], queryFn: () => fetchTodo(1) })

// List với filters — todos theo trạng thái
useQuery({
  queryKey: ['todos', 'list', { status: 'active' }],
  queryFn: () => fetchTodos({ status: 'active' }),
})

// Nested resource — comments của một todo
useQuery({
  queryKey: ['todos', 1, 'comments'],
  queryFn: () => fetchComments(1),
})

// Complex filters
useQuery({
  queryKey: ['todos', 'list', { status: 'active', assigneeId: 5, page: 2 }],
  queryFn: () => fetchTodos({ status: 'active', assigneeId: 5, page: 2 }),
})

// Xu hướng: đặt 'detail' hoặc 'list' là segment thứ hai để phân biệt rõ
useQuery({ queryKey: ['todos', 'list'], queryFn: fetchTodoList })
useQuery({ queryKey: ['todos', 'detail', todoId], queryFn: () => fetchTodoDetail(todoId) })
```

### Hierarchical invalidation

```ts
const queryClient = useQueryClient()

// Invalidate TẤT CẢ queries có key bắt đầu bằng ['todos']
queryClient.invalidateQueries({ queryKey: ['todos'] })
// => Affect: ['todos'], ['todos', 1], ['todos', 'list', ...], ['todos', 1, 'comments']

// Invalidate chỉ queries của todo ID = 1 (và con cháu của nó)
queryClient.invalidateQueries({ queryKey: ['todos', 1] })
// => Affect: ['todos', 1], ['todos', 1, 'comments']
// => Không affect: ['todos', 'list', ...]

// Invalidate chính xác một key (exact match)
queryClient.invalidateQueries({ queryKey: ['todos', 'list'], exact: true })
// => Chỉ affect ĐÚNG ['todos', 'list'], không affect ['todos', 'list', { status: 'active' }]

// Invalidate theo predicate function — linh hoạt nhất
queryClient.invalidateQueries({
  predicate: (query) => {
    return query.queryKey[0] === 'todos' && query.queryKey[2]?.status === 'active'
  },
})
```

### Serialization — object key order không quan trọng

```ts
// Ba cái sau đây LÀ CÙNG KEY trong TanStack Query:
useQuery({ queryKey: ['todos', { status: 'active', page: 1 }], ... })
useQuery({ queryKey: ['todos', { page: 1, status: 'active' }], ... })  // Thứ tự đảo ngược

// TanStack Query serialize theo deep value comparison, không theo reference:
const filters1 = { status: 'active' }
const filters2 = { status: 'active' }
// filters1 !== filters2 (khác reference)
// Nhưng ['todos', filters1] VÀ ['todos', filters2] là CÙNG CACHE ENTRY

// Tuy nhiên, các giá trị khác nhau là key khác nhau:
['todos', { status: 'active' }]   // key A
['todos', { status: 'done' }]     // key B — khác key A
['todos', { status: 'active', page: 1 }]  // key C — khác A và B
```

### Quy ước tổ chức key cho dự án lớn

```ts
// Quy ước tham khảo (trước khi dùng Query Key Factory):
// [entity]                          -> tất cả data của entity
// [entity, 'list']                  -> tất cả lists
// [entity, 'list', filters]         -> list cụ thể với filters
// [entity, 'detail', id]            -> detail của một item
// [entity, 'detail', id, 'related'] -> nested resource

const QUERY_KEYS = {
  // Todo keys
  todoAll:    () => ['todos'] as const,
  todoList:   () => ['todos', 'list'] as const,
  todoFilter: (filters: TodoFilters) => ['todos', 'list', filters] as const,
  todoDetail: (id: number) => ['todos', 'detail', id] as const,
  todoComments: (id: number) => ['todos', 'detail', id, 'comments'] as const,
}

// Sử dụng:
useQuery({ queryKey: QUERY_KEYS.todoFilter({ status: 'active' }), queryFn: ... })

// Invalidate sau khi update todo ID 5:
queryClient.invalidateQueries({ queryKey: QUERY_KEYS.todoDetail(5) })
// => Cũng invalidate ['todos', 'detail', 5, 'comments'] tự động
```

---

## 10. Production Concerns

### Scaling

- **Quy ước key là thỏa thuận nhóm**: toàn bộ team phải dùng cùng quy ước. Document lại trong codebase hoặc dùng [[query-key-factory]] để enforce programmatically.
- **Tránh key collision**: nếu dùng nhiều entities, đảm bảo phần tử đầu tiên của key là unique string cho entity đó. Không nên dùng số hay boolean làm phần tử đầu.
- **Pagination và filters trong object**: gộp hết params vào một object trong array thay vì flat array: `['todos', 'list', { page, status, search }]` thay vì `['todos', 'list', page, status, search]`. Object dễ mở rộng hơn khi thêm params mới.

### Failure Handling

- **Typo trong query key**: `['tdoos']` thay vì `['todos']` sẽ tạo cache entry mới hoàn toàn — bug rất khó debug. Giải pháp: dùng [[query-key-factory]] hoặc constants để tránh magic strings.
- **Key thay đổi bất ngờ**: nếu `queryKey` phụ thuộc vào biến có thể `undefined`, TanStack Query sẽ fetch lại mỗi khi biến đó thay đổi. Kết hợp với `enabled: false` khi biến chưa sẵn sàng.

### Monitoring

- Dùng TanStack Query Devtools để xem chính xác cache keys và trạng thái của từng key. Trong Devtools, tất cả active queries và keys của chúng hiển thị theo thời gian thực — rất hữu ích khi debug invalidation.

---

## 11. Common Mistakes

- Mistake: Dùng string key thay vì array: `useQuery({ queryKey: 'todos', ... })`.
  Fix: Luôn dùng array: `useQuery({ queryKey: ['todos'], ... })`. String keys vẫn hoạt động nhưng TanStack Query wrap chúng thành array nội bộ — gây confusion và mất hierarchical invalidation.

- Mistake: Quên thêm dynamic param vào queryKey khi queryFn phụ thuộc vào nó, dẫn đến stale data khi param thay đổi.
  Fix: Bất kỳ giá trị nào được dùng trong `queryFn` phải xuất hiện trong `queryKey`. Đây là quy tắc iron-clad:
  ```tsx
  // WRONG: todoId không có trong key, cache sẽ không update khi todoId thay đổi
  useQuery({ queryKey: ['todo'], queryFn: () => fetchTodo(todoId) })
  
  // CORRECT
  useQuery({ queryKey: ['todos', 'detail', todoId], queryFn: () => fetchTodo(todoId) })
  ```

- Mistake: Dùng object reference trực tiếp từ outside scope làm key mà object đó bị tạo mới mỗi render, và lo lắng rằng nó sẽ gây ra fetch liên tục.
  Fix: TanStack Query so sánh keys theo deep value equality, KHÔNG theo reference. `{ status: 'active' }` tạo mới mỗi render vẫn là cùng cache key. Tuy nhiên, vẫn nên dùng `useMemo` cho complex filter objects để rõ ràng về intent và tránh issues với ESLint exhaustive-deps.

- Mistake: Invalidate quá broad: sau khi update một todo, gọi `invalidateQueries({ queryKey: [] })` (invalidate TẤT CẢ queries trong cache).
  Fix: Invalidate chính xác nhất có thể: `invalidateQueries({ queryKey: ['todos', todoId] })` hoặc `invalidateQueries({ queryKey: ['todos'] })`. Invalidate tất cả là last resort.

---

## 12. Sample Project

**Project: "E-commerce Product Catalog với Advanced Filtering"**

**Constraint cứng**: Sau khi admin cập nhật giá của một sản phẩm, CHÍNH XÁC tất cả các views liên quan đến sản phẩm đó phải hiển thị giá mới — bao gồm: product detail page, tất cả list pages có chứa sản phẩm đó, và compare view. Các products khác không bị ảnh hưởng và không phải refetch.

**Yêu cầu kỹ thuật**:
- Key structure: `['products', 'list', filters]`, `['products', 'detail', id]`, `['products', 'compare', idList]`.
- Sau khi admin mutation, invalidate `['products', 'detail', updatedId]`.
- Invalidate `['products', 'list']` (tất cả lists, vì price thay đổi ảnh hưởng đến tất cả filtered views).
- `['products', 'compare']` keys chứa `updatedId` phải được invalidate qua predicate:
  ```ts
  queryClient.invalidateQueries({
    predicate: (query) =>
      query.queryKey[0] === 'products' &&
      query.queryKey[1] === 'compare' &&
      (query.queryKey[2] as number[]).includes(updatedId),
  })
  ```
- Chứng minh rằng products không liên quan đến `updatedId` KHÔNG bị refetch bằng DevTools.

---

## 13. Interview

### Core Q&A

**Q: Tại sao query keys là arrays thay vì strings?**
A: Arrays cho phép hierarchical structure và prefix-based invalidation. `invalidateQueries({ queryKey: ['todos'] })` sẽ invalidate tất cả queries có key bắt đầu bằng `['todos']` — bao gồm `['todos', 1]`, `['todos', 'list', ...]`, v.v. Với strings, bạn không có cách nào làm điều này cleanly mà không dùng string prefix matching thủ công.

**Q: Thứ tự elements trong object của query key có quan trọng không?**
A: Không. TanStack Query serialize objects theo deep structural equality, không phụ thuộc vào thứ tự key. `{ a: 1, b: 2 }` và `{ b: 2, a: 1 }` là cùng cache key. Điều này rất tiện lợi khi bạn build filters từ nhiều sources khác nhau.

**Q: Nếu queryFn dùng biến `userId` nhưng `queryKey` không chứa `userId`, điều gì xảy ra?**
A: Cache entry sẽ không bao giờ thay đổi khi `userId` thay đổi — component luôn lấy data của `userId` đầu tiên từ cache. Đây là bug rất phổ biến. Quy tắc: bất kỳ biến nào được dùng trong `queryFn` phải có mặt trong `queryKey`.

**Q: Prefix matching của `invalidateQueries` có làm việc không khi dùng `exact: true`?**
A: Không. `exact: true` yêu cầu key phải khớp CHÍNH XÁC. `invalidateQueries({ queryKey: ['todos'], exact: true })` chỉ invalidate query có key ĐÚNG LÀ `['todos']`, không affect `['todos', 1]` hay `['todos', 'list']`.

**Q: `queryClient.removeQueries()` khác `invalidateQueries()` thế nào?**
A: `invalidateQueries` đánh dấu queries là stale — chúng vẫn còn trong cache và sẽ được refetch khi có observer. `removeQueries` XÓA hoàn toàn cache entry — data mất đi, lần sau khi component mount sẽ fetch lại từ đầu (show loading state). Dùng `removeQueries` khi user logout để xóa sensitive data, hoặc khi muốn completely reset một query.

### Scenario

**Scenario: User đổi ngôn ngữ (language switch) và mỗi API endpoint trả về nội dung theo ngôn ngữ. Làm sao đảm bảo đặt cả queries được refetch với ngôn ngữ mới?**
A: Thêm `locale` vào tất cả query keys: `['todos', 'list', { locale }]`. Khi locale thay đổi, tất cả keys thay đổi theo, TanStack Query tự động fetch lại. Nếu muốn immediate refetch thay vì chờ component remount, gọi `queryClient.invalidateQueries()` (không tham số) sau khi đổi locale — nhưng sau đó tất cả queries có observer sẽ refetch ngay.

**Scenario: Sau khi user delete một comment, bạn cần invalidate các queries nào?**
A: Cần invalidate ít nhất: `['comments', commentId]` (nếu có detail view) và `['todos', todoId, 'comments']` (list comments của todo đó). Nếu có pagination, tất cả pages của comments list đều bị ảnh hưởng. Dùng `invalidateQueries({ queryKey: ['todos', todoId, 'comments'] })` để invalidate tất cả pages cùng lúc. Nếu comment count hiển thị trên todo card, cũng cần `invalidateQueries({ queryKey: ['todos', 'detail', todoId] })`.

---

## 14. References

- Query Keys Guide: https://tanstack.com/query/v5/docs/framework/react/guides/query-keys
- Query Invalidation: https://tanstack.com/query/v5/docs/framework/react/guides/query-invalidation
- Filters (QueryFilters): https://tanstack.com/query/v5/docs/reference/QueryFilters
- TkDodo: "Effective React Query Keys": https://tkdodo.eu/blog/effective-react-query-keys

---

## 15. Real-world Code

- Bulletproof React — query key patterns trong feature modules: https://github.com/alan2207/bulletproof-react
- Vercel's SWR key patterns (giá trị tham khảo tương đương): https://github.com/vercel/swr
- TanStack Query examples repo: https://github.com/TanStack/query/tree/main/examples/react

---

## 16. Community

- TkDodo: "Effective React Query Keys" — bài viết chuẩn nhất về chủ đề này: https://tkdodo.eu/blog/effective-react-query-keys
- TkDodo: "Using WebSockets with React Query" — thấy hierarchical keys được dùng thế nào với subscriptions: https://tkdodo.eu/blog/using-web-sockets-with-react-query
- Reddit discussion "Best practices for React Query keys": https://www.reddit.com/r/reactjs/
- Stack Overflow: https://stackoverflow.com/questions/tagged/react-query
