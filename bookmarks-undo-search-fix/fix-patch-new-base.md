# Patch cho base Chromium mới: search lại 1 lần mỗi lô sự kiện

Áp dụng khi Samsung Browser đã lên **base Chromium mới**. Nhận biết base mới:

- `chrome/browser/resources/bookmarks/app.ts` → `searchTermChanged_()` gọi
  `searchBookmarks(this.getState().nodes, searchTerm)`: search chạy ngay trên
  trang.
- `chrome/browser/resources/bookmarks/bookmarks_api_proxy.ts` có
  `handleMojoEvents()`: sự kiện bookmark đến từ C++ qua Mojo, theo từng lô.

Diff dựa trên Chromium `main` (11/2026). Các đoạn code bị sửa đã được đối chiếu
với JS trong Chrome 157.0.8103.38 và khớp nhau.

Nguyên nhân lỗi và cách Edge làm: xem `root-cause-analysis.md`.

## 1. Vấn đề cần giải quyết

- **Lỗi:** đang search thì xoá một bookmark rồi Undo, bookmark được khôi phục
  nhưng không hiện lại trong kết quả cho đến khi reload. Lý do là kết quả search
  chỉ được tính lại khi đổi từ khoá.
- **Cách Edge sửa:** mỗi khi có bookmark mới được tạo thì search lại.
- **Nếu làm y như Edge trên base mới thì quá tốn:**
  - Search trên base mới quét toàn bộ cây bookmark, mỗi lần tốn O(n).
  - Khi import, C++ gom hết sự kiện và gửi **một lô N sự kiện** "bookmark mới
    được tạo" (`chrome/browser/bookmarks/combined_bookmarks_view.cc`, hàm
    `Notify` và `ExtensiveBookmarkChangesEnded`).
  - Search lại ở mỗi sự kiện thì sẽ quét cây N lần, tổng O(N·n), gần O(n²).

## 2. Ý tưởng

- Khi nhận sự kiện "bookmark mới được tạo", trang **chỉ bật một cờ** "cần search
  lại". Trang không search ở bước này.
- Trang đã có sẵn cơ chế gom: `batchUIUpdates()` cùng `Debouncer` trong
  `api_listener.ts`. Mọi sự kiện đến liên tiếp được gom vào một lô, và giao diện
  chỉ vẽ lại khi lô kết thúc.
- **Khi lô kết thúc**, nếu cờ đang bật thì trang gửi một action riêng là
  `refresh-search-results`. Reducer chạy lại search trên dữ liệu mới **một lần**.
  Nếu không đang search thì reducer không làm gì. Sau đó lô mới kết thúc và giao
  diện vẽ lại một lần.
- Phải dùng **action riêng** vì action `finish-search` có sẵn sẽ bỏ chọn tất cả
  các mục. Action mới giữ nguyên các mục đang chọn.

Luồng xử lý sau khi sửa:

```
Undo / import / dán / thêm bookmark từ cửa sổ khác
  → C++ gửi sự kiện "bookmark mới được tạo"
      (import: cả N sự kiện trong 1 lô; Undo: mỗi bookmark một tin nhắn)
  → onBookmarkCreated(), chạy cho từng sự kiện:
      - thêm node vào dữ liệu (như cũ)
      - bật cờ searchRefreshNeeded (mới)
  → hết lượt xử lý → Debouncer gọi endBatchUIUpdates(), chỉ 1 lần mỗi lô:
      - cờ bật → dispatch refresh-search-results
               → reducer rerunSearch(): quét cây 1 lần nếu đang search
      - endBatchUpdate() → giao diện vẽ lại 1 lần
```

## 3. Thay đổi chi tiết

Sửa 3 file trong `chrome/browser/resources/bookmarks/`.

### 3.1. `actions.ts`: thêm action mới

Thêm ngay sau hàm `setSearchResults()`:

```ts
/**
 * Re-runs the active search against the current nodes, e.g. after nodes were
 * created while search results are shown. Unlike 'finish-search', this keeps
 * the current selection.
 */
export function refreshSearchResults(): Action {
  return {
    name: 'refresh-search-results',
  };
}
```

Action không mang dữ liệu gì. Reducer tự tính kết quả từ dữ liệu bookmark hiện
có, nên kết quả luôn khớp với dữ liệu tại thời điểm xử lý.

### 3.2. `reducers.ts`: xử lý action mới

**Bước 1.** Thêm `searchBookmarks` vào import từ `./util.js`. Đây cũng là hàm mà
`app.ts` dùng cho lần search đầu, nên điều kiện khớp và thứ tự kết quả giống hệt:

```ts
import {/* ...các tên đang có... */, searchBookmarks} from './util.js';
```

**Bước 2.** Thêm hàm chạy lại search, đặt ngay trên `updateSearch()`:

```ts
function rerunSearch(search: SearchState, nodes: NodeMap): SearchState {
  if (!search.term || search.results === null) {
    return search;
  }

  const results = searchBookmarks(nodes, search.term).map(node => node.id);
  return Object.assign({}, search, {results});
}
```

- Nếu không có từ khoá, hoặc chưa có kết quả search nào, hàm trả về nguyên trạng
  ngay. Chi phí O(1).
- Nếu đang search, hàm quét cây một lần. Chi phí O(n).

**Bước 3.** Thêm tham số `nodes` cho `updateSearch()` và thêm nhánh cho action
mới:

```ts
function updateSearch(
    search: SearchState, action: Action, nodes: NodeMap): SearchState {
  switch (action.name) {
    // ...các case đang có giữ nguyên...
    case 'remove-bookmark':
      return removeDeletedResults(
          search, (action as RemoveBookmarkAction).descendants);
    case 'refresh-search-results':
      return rerunSearch(search, nodes);
    default:
      return search;
  }
}
```

**Bước 4.** Trong `reduceAction()`, tính `nodes` trước rồi truyền vào
`updateSearch()`:

```ts
export function reduceAction(
    state: BookmarksPageState, action: Action): BookmarksPageState {
  const nodes = updateNodes(state.nodes, action);
  return {
    nodes,
    // ...các field khác giữ nguyên...
    search: updateSearch(state.search, action, nodes),
    selection: updateSelection(state.selection, action),
  };
}
```

`updateSelection()` không có nhánh cho `refresh-search-results`, nên các mục
đang chọn được giữ nguyên.

### 3.3. `api_listener.ts`: gom lại và chỉ search 1 lần mỗi lô

**Bước 1.** Thêm `refreshSearchResults` vào import từ `./actions.js`:

```ts
import {createBookmark, editBookmark, moveBookmark, refreshSearchResults, removeBookmark, reorderChildren, setCanEditBookmarks, setIncognitoAvailability} from './actions.js';
```

**Bước 2.** Thêm cờ, đặt ngay dưới `let debouncer`:

```ts
// Set when nodes were created during the current batch of UI updates, so the
// active search (if any) is re-run once when the batch ends.
let searchRefreshNeeded: boolean = false;
```

**Bước 3.** Thêm hàm kết thúc lô, đặt ngay dưới `batchUIUpdates()`:

```ts
/**
 * Ends the current batch of UI updates. Nodes created while search results
 * are shown (undo of a delete, paste, import, a bookmark added from another
 * window) must appear in them. The search scans every node, so it runs once
 * here rather than once per created node: an import delivers all of its nodes
 * in a single batch, and a per-node search would be quadratic.
 */
function endBatchUIUpdates() {
  if (searchRefreshNeeded) {
    searchRefreshNeeded = false;
    dispatch(refreshSearchResults());
  }
  Store.getInstance().endBatchUpdate();
}
```

Action được gửi **trước** `endBatchUpdate()`, tức là lúc trang vẫn đang gom.
Nhờ vậy dữ liệu mới và kết quả search mới được vẽ lên trong cùng một lần.

**Bước 4.** Trong `batchUIUpdates()`, cho `Debouncer` gọi hàm mới:

```ts
// Trước:
debouncer = new Debouncer(() => Store.getInstance().endBatchUpdate());
// Sau:
debouncer = new Debouncer(endBatchUIUpdates);
```

**Bước 5.** Ở cuối `onBookmarkCreated()`, bật cờ:

```ts
function onBookmarkCreated(
    parentId: string, index: number, node: BookmarkNode) {
  batchUIUpdates();
  if (trackUpdates) {
    updatedItems.push(node.id);
  }
  dispatch(createBookmark(parentId, index, node));
  searchRefreshNeeded = true;
}
```

Hàm này không kiểm tra có đang search hay không. Việc đó để `rerunSearch()` lo
lúc cuối lô, vì đến lúc đó người dùng có thể đã đổi hoặc xoá từ khoá.

## 4. Diff đầy đủ

Base: Chromium `main`, khớp Chrome 157.0.8103.38. Chạy từ thư mục `src/`:

```
git apply --3way --check new-base-search-refresh.patch
git apply --3way new-base-search-refresh.patch
```

Nếu báo lỗi do context của bản fork bị lệch thì sửa tay theo mục 3.

```diff
diff --git a/chrome/browser/resources/bookmarks/actions.ts b/chrome/browser/resources/bookmarks/actions.ts
index 1bd5126..cdb99f8 100644
--- a/chrome/browser/resources/bookmarks/actions.ts
+++ b/chrome/browser/resources/bookmarks/actions.ts
@@ -256,6 +256,17 @@ export function setSearchResults(ids: string[]): FinishSearchAction {
   };
 }
 
+/**
+ * Re-runs the active search against the current nodes, e.g. after nodes were
+ * created while search results are shown. Unlike 'finish-search', this keeps
+ * the current selection.
+ */
+export function refreshSearchResults(): Action {
+  return {
+    name: 'refresh-search-results',
+  };
+}
+
 export type SetPrefAction = Action&{
   value: IncognitoAvailability | boolean,
 };
diff --git a/chrome/browser/resources/bookmarks/api_listener.ts b/chrome/browser/resources/bookmarks/api_listener.ts
index b526834..0c068c6 100644
--- a/chrome/browser/resources/bookmarks/api_listener.ts
+++ b/chrome/browser/resources/bookmarks/api_listener.ts
@@ -5,7 +5,7 @@
 import {addWebUiListener, removeWebUiListener} from 'chrome://resources/js/cr.js';
 import type {Action} from 'chrome://resources/js/store.js';
 
-import {createBookmark, editBookmark, moveBookmark, removeBookmark, reorderChildren, setCanEditBookmarks, setIncognitoAvailability} from './actions.js';
+import {createBookmark, editBookmark, moveBookmark, refreshSearchResults, removeBookmark, reorderChildren, setCanEditBookmarks, setIncognitoAvailability} from './actions.js';
 import {BookmarksApiProxyImpl} from './bookmarks_api_proxy.js';
 import {BrowserProxyImpl} from './browser_proxy.js';
 import type {IncognitoAvailability} from './constants.js';
@@ -23,6 +23,10 @@ let updatedItems: string[] = [];
 
 let debouncer: Debouncer|null = null;
 
+// Set when nodes were created during the current batch of UI updates, so the
+// active search (if any) is re-run once when the batch ends.
+let searchRefreshNeeded: boolean = false;
+
 /**
  * Batches UI updates so that no changes will be made to UI until the next
  * task after the last call to this method. This is useful for listeners which
@@ -30,7 +34,7 @@ let debouncer: Debouncer|null = null;
  */
 function batchUIUpdates() {
   if (debouncer === null) {
-    debouncer = new Debouncer(() => Store.getInstance().endBatchUpdate());
+    debouncer = new Debouncer(endBatchUIUpdates);
   }
 
   if (debouncer.done()) {
@@ -41,6 +45,21 @@ function batchUIUpdates() {
   debouncer.restartTimeout();
 }
 
+/**
+ * Ends the current batch of UI updates. Nodes created while search results
+ * are shown (undo of a delete, paste, import, a bookmark added from another
+ * window) must appear in them. The search scans every node, so it runs once
+ * here rather than once per created node: an import delivers all of its nodes
+ * in a single batch, and a per-node search would be quadratic.
+ */
+function endBatchUIUpdates() {
+  if (searchRefreshNeeded) {
+    searchRefreshNeeded = false;
+    dispatch(refreshSearchResults());
+  }
+  Store.getInstance().endBatchUpdate();
+}
+
 /**
  * Tracks any items that are created or moved.
  */
@@ -90,6 +109,7 @@ function onBookmarkCreated(
     updatedItems.push(node.id);
   }
   dispatch(createBookmark(parentId, index, node));
+  searchRefreshNeeded = true;
 }
 
 function onBookmarkRemoved(id: string, parentId: string, index: number) {
diff --git a/chrome/browser/resources/bookmarks/reducers.ts b/chrome/browser/resources/bookmarks/reducers.ts
index 9d00c5a..8296cff 100644
--- a/chrome/browser/resources/bookmarks/reducers.ts
+++ b/chrome/browser/resources/bookmarks/reducers.ts
@@ -16,7 +16,7 @@ import type {ChangeFolderOpenAction, CreateBookmarkAction, EditBookmarkAction, F
 import {PermanentFolderType} from './bookmarks_api.mojom-webui.js';
 import {ACCOUNT_HEADING_NODE_ID, LOCAL_HEADING_NODE_ID, ROOT_NODE_ID} from './constants.js';
 import type {BookmarkNode, BookmarksPageState, FolderOpenState, NodeMap, PreferencesState, SearchState, SelectionState} from './types.js';
-import {buildAccountHeadingNode, buildLocalHeadingNode, getDefaultSelectedFolder, removeIdsFromMap, removeIdsFromObject, removeIdsFromSet} from './util.js';
+import {buildAccountHeadingNode, buildLocalHeadingNode, getDefaultSelectedFolder, removeIdsFromMap, removeIdsFromObject, removeIdsFromSet, searchBookmarks} from './util.js';
 
 function selectItems(
     selectionState: SelectionState, action: SelectItemsAction): SelectionState {
@@ -142,7 +142,17 @@ function removeDeletedResults(
   }) as SearchState);
 }
 
-function updateSearch(search: SearchState, action: Action): SearchState {
+function rerunSearch(search: SearchState, nodes: NodeMap): SearchState {
+  if (!search.term || search.results === null) {
+    return search;
+  }
+
+  const results = searchBookmarks(nodes, search.term).map(node => node.id);
+  return Object.assign({}, search, {results});
+}
+
+function updateSearch(
+    search: SearchState, action: Action, nodes: NodeMap): SearchState {
   switch (action.name) {
     case 'start-search':
       return startSearch(search, action as StartSearchAction);
@@ -154,6 +164,8 @@ function updateSearch(search: SearchState, action: Action): SearchState {
     case 'remove-bookmark':
       return removeDeletedResults(
           search, (action as RemoveBookmarkAction).descendants);
+    case 'refresh-search-results':
+      return rerunSearch(search, nodes);
     default:
       return search;
   }
@@ -548,14 +560,15 @@ function updatePrefs(
 
 export function reduceAction(
     state: BookmarksPageState, action: Action): BookmarksPageState {
+  const nodes = updateNodes(state.nodes, action);
   return {
-    nodes: updateNodes(state.nodes, action),
+    nodes,
     selectedFolder:
         updateSelectedFolder(state.selectedFolder, action, state.nodes),
     folderOpenState:
         updateFolderOpenState(state.folderOpenState, action, state.nodes),
     prefs: updatePrefs(state.prefs, action),
-    search: updateSearch(state.search, action),
+    search: updateSearch(state.search, action, nodes),
     selection: updateSelection(state.selection, action),
   };
 }
```

## 5. Chi phí sau khi sửa

| Tình huống | Số lần search | Chi phí thêm |
| --- | --- | --- |
| Không đang search (bất kỳ thao tác nào) | 0, reducer thoát ngay | O(1) mỗi lô |
| Đang search, import N bookmark | **1** (cả N sự kiện về trong một lô) | O(n) |
| Đang search, dán hoặc thêm bookmark từ cửa sổ khác | 1 | O(n) |
| Đang search, Undo xoá N mục | Thường 1. Mỗi bookmark khôi phục là một tin nhắn riêng, nếu đến rải rác thì có thể bị tách thành vài lô | O(n) mỗi lô |

Chi phí có sẵn của Chromium mới, không do patch gây ra và patch cũng không xử lý:
khi import lúc trang bookmarks đang mở, mỗi sự kiện vẫn chép lại toàn bộ bảng
bookmark (`reducers.ts` → `createBookmark()` →
`return Object.assign({}, nodes, nodeModifications)`). Phần này tốn O(N·n)
dù có search hay không.

## 6. Hành vi khác cần biết

- **Các mục đang chọn được giữ nguyên.** Tạo bookmark chỉ có thể thêm kết quả,
  không làm mất kết quả cũ, nên những mục đang chọn vẫn còn trong danh sách.
- **Highlight sau khi dán vẫn đúng.** `highlightUpdatedItems()` chờ
  `debouncer.promise`, mà promise này chỉ xong sau `endBatchUIUpdates()`. Vì vậy
  highlight luôn chạy sau khi kết quả đã được cập nhật.
- **Thứ tự kết quả không đổi.** Lần search đầu và lần search lại dùng chung hàm
  `searchBookmarks()`.
- **Xoá giữ nguyên cơ chế cũ:** id bị xoá được bỏ khỏi kết quả, không cần search
  lại.
- **Đổi tên và di chuyển không search lại**, giống Chromium hiện tại. Edge thì
  search lại khi đổi tên. Nếu muốn giống Edge: trong `onBookmarkChanged()` gọi
  `batchUIUpdates()` rồi đặt `searchRefreshNeeded = true`.
- **Lô đầu tiên sau khi mở trang:** do cách `Debouncer` khởi tạo, lô đầu tiên
  không bật chế độ gom. Giao diện có thể vẽ lại thêm một lần, không ảnh hưởng gì.

## 7. Kiểm tra bằng tay

| # | Thao tác | Kết quả mong đợi |
| --- | --- | --- |
| 1 | `chrome://bookmarks/?q=chat` → xoá 1 bookmark khớp → Undo | Bookmark hiện lại ngay, không cần reload |
| 2 | Chọn nhiều kết quả → Delete → Undo | Tất cả hiện lại |
| 3 | Đang mở `?q=chat`, import file bookmark HTML có nhiều mục chứa "chat" | Sau khi import, các mục đó hiện trong kết quả, trang không bị đơ thêm so với trước khi sửa |
| 4 | Đang mở `?q=chat`, chọn 1 kết quả, rồi thêm bookmark "chat test" từ cửa sổ khác | Mục mới hiện ra, mục đang chọn vẫn được chọn |
| 5 | Đang search, dán bookmark có tên khớp từ khoá | Bookmark hiện trong kết quả và được highlight |
| 6 | Không search, xoá rồi Undo trong một folder | Hoạt động như trước khi sửa |

## 8. Trạng thái

- Diff apply sạch trên Chromium `main`. Các đoạn bị sửa khớp với Chrome
  157.0.8103.38.
- Chưa build, chưa chạy, không kèm test.
