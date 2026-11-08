# Phân tích nguyên nhân: Undo xoá bookmark không hiện lại trong kết quả search

## 1. Hiện tượng

Các bước tái hiện:

1. Mở `chrome://bookmarks/?q=chat`. Kết quả search có bookmark "ChatGPT".
2. Chọn "ChatGPT" rồi bấm **Delete**. Bookmark biến mất khỏi danh sách.
3. Bấm **Undo** (nút trên toast hoặc Ctrl+Z).

Kết quả hiện tại:

- Thanh địa chỉ vẫn là `chrome://bookmarks/?q=chat`, ô search vẫn là `chat`.
- Bookmark đã được khôi phục thật (mở folder chứa nó sẽ thấy, reload trang cũng thấy).
- Nhưng **danh sách kết quả search không hiện lại "ChatGPT"**. Phải reload hoặc
  gõ lại từ khoá thì mới hiện.

Kết quả mong muốn (Edge đang làm đúng): sau Undo, "ChatGPT" hiện lại ngay trong
kết quả search.

## 2. Kết luận ngắn

Đây là lỗi ở **WebUI của trang bookmarks** (TypeScript, thư mục
`chrome/browser/resources/bookmarks/`), **không phải ở phần C++**
(BookmarkModel / BookmarkUndoService vẫn khôi phục đúng).

Trang bookmarks lưu kết quả search thành một **danh sách id cố định**, chỉ tính
lại khi **từ khoá thay đổi**. Khi xoá, có đoạn code bỏ id bị xoá khỏi danh sách đó.
Nhưng khi bookmark được **tạo lại** (Undo chính là tạo lại), không có đoạn code
nào thêm nó vào danh sách kết quả. Vì vậy bookmark có trong dữ liệu nhưng không
nằm trong danh sách đang hiển thị.

## 3. Cách trang bookmarks hoạt động

Trang dùng mô hình kiểu Redux: một `Store` chứa `state`, mọi thay đổi đi qua
`action` → `reducer` → `state` mới → UI vẽ lại. Hai phần của state liên quan
đến lỗi này:

| Phần state | Ý nghĩa |
| --- | --- |
| `state.nodes` | Toàn bộ cây bookmark (map `id → node`) |
| `state.search` | `{term, inProgress, results}`. `results` là **mảng id** kết quả search |

Khi đang search, danh sách hiển thị **chỉ lấy từ `state.search.results`**,
không đọc từ `state.nodes`:

```ts
// util.ts
export function getDisplayedList(state: BookmarksPageState): string[] {
  if (isShowingSearch(state)) {
    return state.search.results;   // chỉ hiện những id có trong mảng này
  }
  ...
}
```

### 3.1. Luồng search

Search chỉ được chạy khi **từ khoá thay đổi** (gõ vào ô search, hoặc mở URL
`?q=...`):

```ts
// app.ts
private searchTermChanged_(newValue?: string, oldValue?: string) {
  ...
  const results = searchBookmarks(this.getState().nodes, searchTerm);
  this.dispatch(setSearchResults(results.map(node => node.id)));
}
```

Action `setSearchResults` (`'finish-search'`) ghi mảng id vào
`state.search.results`. Từ lúc này mảng kết quả **cố định**, chỉ thay đổi khi
có reducer khác sửa nó.

> Lưu ý: bản Chromium cũ (≤ ~145) gọi `chrome.bookmarks.search(term)` bất
> đồng bộ thay vì `searchBookmarks()` cục bộ, nhưng cơ chế giống hệt: chỉ chạy
> khi từ khoá đổi.

### 3.2. Luồng xoá: có xử lý search

1. Bấm Delete → C++ xoá node → trang nhận sự kiện `chrome.bookmarks.onRemoved`.
2. `api_listener.ts` → `onBookmarkRemoved()` → dispatch `removeBookmark(...)`
   (`'remove-bookmark'`).
3. Reducer:
   - `updateNodes()` xoá node khỏi `state.nodes`.
   - `updateSearch()` **có nhánh `'remove-bookmark'`**, gọi
     `removeDeletedResults()` để bỏ id đã xoá khỏi `state.search.results`.

```ts
// reducers.ts
function updateSearch(search: SearchState, action: Action): SearchState {
  switch (action.name) {
    case 'start-search':    return startSearch(...);
    case 'select-folder':
    case 'clear-search':    return clearSearch();
    case 'finish-search':   return finishSearch(...);
    case 'remove-bookmark': return removeDeletedResults(...);  // xoá: có xử lý
    default:                return search;                     // tạo: rơi vào đây
  }
}
```

Vì vậy bước xoá hiển thị đúng: "ChatGPT" biến khỏi danh sách.

### 3.3. Luồng Undo: thiếu xử lý search, đây là chỗ gây lỗi

1. Bấm Undo → `command_manager.ts` gọi `chrome.bookmarkManagerPrivate.undo()`.
2. C++ (BookmarkUndoService) khôi phục node vào BookmarkModel → trang nhận sự
   kiện **`chrome.bookmarks.onCreated`**.
3. `api_listener.ts` → `onBookmarkCreated()` → dispatch `createBookmark(...)`
   (`'create-bookmark'`).
4. Reducer:
   - `updateNodes()` thêm node vào `state.nodes`. **Dữ liệu đã đúng.**
   - `updateSearch()` **không có nhánh `'create-bookmark'`**, rơi vào
     `default: return search`, nên `state.search.results` giữ nguyên mảng cũ
     (không có id của "ChatGPT").
5. `getDisplayedList()` vẫn trả về mảng cũ, nên UI không hiện "ChatGPT".

Tóm tắt:

```
Delete:  onRemoved → 'remove-bookmark' → nodes: bỏ node   | search.results: bỏ id   ✔
Undo:    onCreated → 'create-bookmark' → nodes: thêm node | search.results: KHÔNG ĐỔI ✘
```

Ngoài ra, phía undo không có đoạn code nào gọi lại search, và `search.term`
không đổi (vẫn là `chat`) nên `searchTermChanged_()` cũng không chạy lại.

### 3.4. Vì sao reload hoặc gõ lại thì hết lỗi

- **Reload trang:** state được tạo mới, đọc lại `?q=chat` từ URL và chạy search
  trên cây bookmark mới (đã có "ChatGPT").
- **Gõ lại từ khoá:** `search.term` thay đổi nên `searchTermChanged_()` chạy lại.

Cả hai cách đều buộc search chạy lại. Điều đó xác nhận dữ liệu đúng, chỉ có
**mảng kết quả search bị cũ**.

### 3.5. Vì sao URL vẫn là `?q=chat`

URL được đồng bộ từ `state.search.term` (qua router). Undo không đổi `term`, nên
URL giữ nguyên. Điều này đúng như thiết kế, không phải một phần của lỗi.

## 4. Phạm vi ảnh hưởng

Lỗi không chỉ xảy ra với Undo. Mọi trường hợp **bookmark được tạo khi đang ở chế
độ search** đều bị, vì đều đi qua `onCreated` → `'create-bookmark'`:

| Trường hợp | Bị lỗi? |
| --- | --- |
| Undo sau khi xoá 1 bookmark trong kết quả search | Có |
| Undo sau khi xoá nhiều bookmark cùng lúc | Có (không mục nào hiện lại) |
| Paste (Ctrl+V) bookmark khớp từ khoá khi đang search | Có |
| Thêm bookmark khớp từ khoá từ cửa sổ/tab khác | Có |
| Sync kéo về bookmark khớp từ khoá | Có |
| Xoá bookmark | Không (đã có `removeDeletedResults`) |

Các phiên bản đã kiểm tra đều có lỗi:

- Chrome 157.0.8103.38 (code thật trong `resources.pak` đang cài trên máy).
- Chromium `main` (tháng 11/2026).
- Chromium 140.0.7339.80.

## 5. So sánh với Edge (Edge 156.0.4314.58)

Trang `edge://favorites` **không dùng code bookmarks của Chromium**. Đây là app
React/Redux riêng của Microsoft, đóng gói trong `resources.pak` của Edge (Edge
dùng định dạng `.pak` riêng với resource id 32-bit, khác Chromium 16-bit).

Edge cũng nghe các sự kiện `chrome.bookmarks.*`. Điểm khác là khi đang ở chế độ
search, Edge **chạy lại search** thay vì sửa tay mảng kết quả:

| Sự kiện | Edge xử lý | Chromium xử lý |
| --- | --- | --- |
| `onCreated` (Undo, paste, thêm mới) | Đang search → gọi lại `chrome.bookmarks.search(term)` và thay toàn bộ kết quả | **Không làm gì với kết quả search** (gây lỗi) |
| `onRemoved` | Đang search → search lại. Nếu xoá nhiều mục thì đợi mục cuối rồi mới search 1 lần | Bỏ id đã xoá khỏi mảng kết quả |
| `onChanged` (đổi tên/URL) | Đang search → search lại (mục không còn khớp sẽ biến mất) | Không search lại, mục vẫn nằm trong kết quả |
| `onMoved`, `onChildrenReordered` | Chỉ cập nhật dữ liệu | Chỉ cập nhật dữ liệu |

Riêng hàm Undo của Edge không đụng gì đến search. Bookmark hiện lại được là nhờ
quy tắc `onCreated` ở trên. Đây là điểm mấu chốt cần áp dụng cho bản fork.

## 6. Hướng sửa

Bổ sung cho `'create-bookmark'` cách xử lý tương ứng với `'remove-bookmark'`:
khi có bookmark được tạo trong lúc đang search thì **tính lại kết quả search**.

Có hai cách, tuỳ base Chromium của bản fork (xem `searchTermChanged_()` trong
`app.ts`):

| Bản fork search bằng | Cách sửa |
| --- | --- |
| `searchBookmarks(nodes, term)` (cục bộ, đồng bộ, bản mới) | Trong `reducers.ts`, thêm nhánh `case 'create-bookmark'` vào `updateSearch()` để chạy lại `searchBookmarks()` trên cây node mới. Giữ được các mục đang chọn. |
| `chrome.bookmarks.search(term)` (bất đồng bộ, bản ≤ ~145) | Trong `api_listener.ts`, sau `onBookmarkCreated()` thì gọi lại API search (giống Edge), gộp nhiều sự kiện thành 1 lần gọi khi undo nhiều mục. |

Chi tiết patch: xem file `fix-patch.md` trong cùng thư mục.

## 7. Vị trí code liên quan (Chromium `main`)

| File | Vị trí | Vai trò |
| --- | --- | --- |
| `chrome/browser/resources/bookmarks/reducers.ts` | `updateSearch()` | **Chỗ thiếu nhánh `'create-bookmark'`** |
| `chrome/browser/resources/bookmarks/reducers.ts` | `removeDeletedResults()` | Xử lý search khi xoá (để đối chiếu) |
| `chrome/browser/resources/bookmarks/reducers.ts` | `reduceAction()` | Gọi `updateNodes()` và `updateSearch()` |
| `chrome/browser/resources/bookmarks/api_listener.ts` | `onBookmarkCreated()` | Nhận `onCreated`, dispatch `createBookmark` |
| `chrome/browser/resources/bookmarks/app.ts` | `searchTermChanged_()` | Chỗ duy nhất chạy search |
| `chrome/browser/resources/bookmarks/util.ts` | `getDisplayedList()` | Đang search thì chỉ hiện `search.results` |
| `chrome/browser/resources/bookmarks/command_manager.ts` | `Command.UNDO` | Gọi `chrome.bookmarkManagerPrivate.undo()` |
| `chrome/test/data/webui/bookmarks/reducers_test.ts` | suite `'search state'` | Nơi thêm test cho bản sửa |

Tên hàm giữ nguyên qua các phiên bản. Số dòng có thể lệch tuỳ base của bản fork.
