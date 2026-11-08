# Patch sửa lỗi: Undo xoá bookmark không hiện lại trong kết quả search

File này chứa đầy đủ patch, có thể copy sang máy có source bản fork để áp dụng.
Phân tích nguyên nhân chi tiết: xem `root-cause-analysis.md`.

> **Base Chromium mới:** cách A trong file này search lại ở mỗi sự kiện tạo
> bookmark, nên nếu import lúc đang search sẽ tốn gần O(n²). Với base mới, dùng
> `fix-patch-new-base.md` (search lại 1 lần mỗi lô sự kiện).

## 1. Tóm tắt lỗi

- Mở `chrome://bookmarks/?q=chat`, xoá "ChatGPT" rồi Undo. Bookmark được khôi
  phục nhưng không hiện lại trong kết quả search cho đến khi reload.
- Nguyên nhân: trong `chrome/browser/resources/bookmarks/reducers.ts`, hàm
  `updateSearch()` có xử lý `'remove-bookmark'` nhưng **thiếu
  `'create-bookmark'`**. Undo khôi phục bookmark qua sự kiện `onCreated` →
  `'create-bookmark'`, nên `state.search.results` không được cập nhật.
- Cách sửa (giống Edge): khi có bookmark được tạo trong lúc đang search thì
  **tính lại kết quả search**.

## 2. Chọn cách sửa

Mở `chrome/browser/resources/bookmarks/app.ts` của bản fork, xem hàm
`searchTermChanged_()`:

| Hàm đang gọi | Dùng |
| --- | --- |
| `searchBookmarks(this.getState().nodes, searchTerm)` (search cục bộ, đồng bộ; Chromium mới, Chrome 157) | **Cách A** (mục 3) |
| `BookmarksApiProxyImpl.getInstance().search(searchTerm)` hoặc `chrome.bookmarks.search(...)` (bất đồng bộ; Chromium ≤ ~145) | **Cách B** (mục 4) |

Mỗi cách gồm 2 phần: **các bước sửa tay** (dùng khi diff không apply được vì
code bản fork bị lệch) và **diff đầy đủ** (dùng với `git apply`).

---

## 3. Cách A: tính lại kết quả trong reducer

**File sửa:** `chrome/browser/resources/bookmarks/reducers.ts`
**File test:** `chrome/test/data/webui/bookmarks/reducers_test.ts`

**Hành vi sau khi sửa:** khi có action `'create-bookmark'` và đang hiển thị kết
quả search, chạy lại `searchBookmarks(nodes, term)` trên cây node đã cập nhật.
Đây là hàm thuần (pure), chạy đồng bộ, và **giữ nguyên các mục đang được chọn**.

### 3.1. Các bước sửa tay

**Bước 1.** Thêm `searchBookmarks` vào dòng import từ `./util.js`:

```ts
import {/* ...các tên đang có... */, searchBookmarks} from './util.js';
```

**Bước 2.** Thêm hàm sau, đặt ngay trên `updateSearch()`:

```ts
/**
 * Re-runs the active search against |nodes| so that nodes added while a
 * search is shown (undo of a delete, paste, a bookmark created from another
 * window, sync) appear in the results, mirroring how 'remove-bookmark'
 * drops deleted nodes from them.
 */
function refreshSearchResults(search: SearchState, nodes: NodeMap):
    SearchState {
  if (!search.term || search.results === null) {
    return search;
  }

  const results = searchBookmarks(nodes, search.term).map(node => node.id);
  return Object.assign({}, search, {results});
}
```

Hàm chỉ chạy khi đang có từ khoá **và** đã có kết quả search. Ngoài chế độ
search thì không làm gì.

**Bước 3.** Thêm tham số `nodes` cho `updateSearch()` và thêm nhánh
`'create-bookmark'`:

```ts
function updateSearch(
    search: SearchState, action: Action, nodes: NodeMap): SearchState {
  switch (action.name) {
    // ...các case đang có giữ nguyên...
    case 'remove-bookmark':
      return removeDeletedResults(
          search, (action as RemoveBookmarkAction).descendants);
    case 'create-bookmark':
      return refreshSearchResults(search, nodes);
    default:
      return search;
  }
}
```

**Bước 4.** Trong `reduceAction()`, tính `nodes` mới trước rồi truyền vào
`updateSearch()`. Search phải chạy trên cây **đã có** node vừa tạo:

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

**Bước 5.** Thêm 3 test trong diff ở mục 3.2 vào cuối suite `'search state'`
của `reducers_test.ts`:

| Test | Kiểm tra |
| --- | --- |
| `shows a matching node again after delete is undone` | Search "chat" → xoá → tạo lại (Undo) → bookmark hiện lại |
| `ignores created nodes that do not match the search` | Tạo bookmark không khớp → kết quả không đổi |
| `does not start a search when a node is created` | Không search thì tạo bookmark không làm bật chế độ search |

Nếu kiểu `BookmarkNode` của bản fork không có field `legacyId` thì bỏ field
này trong các object test.

### 3.2. Diff đầy đủ (base: Chromium `main`, khớp Chrome 157)

```diff
diff --git a/chrome/browser/resources/bookmarks/reducers.ts b/chrome/browser/resources/bookmarks/reducers.ts
index 9d00c5a..9a5cddc 100644
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
@@ -142,7 +142,24 @@ function removeDeletedResults(
   }) as SearchState);
 }
 
-function updateSearch(search: SearchState, action: Action): SearchState {
+/**
+ * Re-runs the active search against |nodes| so that nodes added while a
+ * search is shown (undo of a delete, paste, a bookmark created from another
+ * window, sync) appear in the results, mirroring how 'remove-bookmark'
+ * drops deleted nodes from them.
+ */
+function refreshSearchResults(search: SearchState, nodes: NodeMap):
+    SearchState {
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
@@ -154,6 +171,8 @@ function updateSearch(search: SearchState, action: Action): SearchState {
     case 'remove-bookmark':
       return removeDeletedResults(
           search, (action as RemoveBookmarkAction).descendants);
+    case 'create-bookmark':
+      return refreshSearchResults(search, nodes);
     default:
       return search;
   }
@@ -548,14 +567,15 @@ function updatePrefs(
 
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
diff --git a/chrome/test/data/webui/bookmarks/reducers_test.ts b/chrome/test/data/webui/bookmarks/reducers_test.ts
index c903672..636e669 100644
--- a/chrome/test/data/webui/bookmarks/reducers_test.ts
+++ b/chrome/test/data/webui/bookmarks/reducers_test.ts
@@ -688,4 +688,62 @@ suite('search state', function() {
     // 2.
     assertDeepEquals(['1'], state.search.results);
   });
+
+  test('shows a matching node again after delete is undone', function() {
+    state.nodes = testTree(createFolder('1', [
+      createItem('2', {title: 'ChatGPT', url: 'https://chatgpt.com/'}),
+      createItem('3', {title: 'Example', url: 'https://example.com/'}),
+    ]));
+
+    state = reduceAction(state, setSearchTerm('chat'));
+    state = reduceAction(state, setSearchResults(['2']));
+
+    state = reduceAction(state, removeBookmark('2', '1', 0, state.nodes));
+    assertDeepEquals([], getDisplayedList(state));
+
+    // Undo restores the node, which reaches the page as an onCreated event.
+    const restored = {
+      id: '2',
+      parentId: '1',
+      title: 'ChatGPT',
+      url: 'https://chatgpt.com/',
+      legacyId: 2,
+    };
+    state = reduceAction(state, createBookmark('1', 0, restored));
+
+    assertTrue(isShowingSearch(state));
+    assertEquals('chat', state.search.term);
+    assertDeepEquals(['2'], getDisplayedList(state));
+  });
+
+  test('ignores created nodes that do not match the search', function() {
+    state = reduceAction(state, setSearchTerm('chat'));
+    state = reduceAction(state, setSearchResults([]));
+
+    const other = {
+      id: '4',
+      parentId: '1',
+      title: 'Other',
+      url: 'https://other.com/',
+      legacyId: 4,
+    };
+    state = reduceAction(state, createBookmark('1', 1, other));
+
+    assertDeepEquals([], state.search.results);
+  });
+
+  test('does not start a search when a node is created', function() {
+    const item = {
+      id: '4',
+      parentId: '1',
+      title: 'ChatGPT',
+      url: 'https://chatgpt.com/',
+      legacyId: 4,
+    };
+    state = reduceAction(state, createBookmark('1', 1, item));
+
+    assertEquals('', state.search.term);
+    assertEquals(null, state.search.results);
+    assertFalse(isShowingSearch(state));
+  });
 });
```

---

## 4. Cách B: search lại trong API listener

**File sửa:** `chrome/browser/resources/bookmarks/api_listener.ts`

**Hành vi sau khi sửa:**

- Sau sự kiện `onCreated`, nếu đang hiển thị kết quả search thì đợi hết đợt cập
  nhật UI hiện tại (`debouncer.promise`) rồi gọi lại đúng API search mà trang
  đang dùng.
- Undo xoá nhiều mục sẽ sinh nhiều sự kiện `onCreated` nhưng **chỉ gọi search
  1 lần**.
- Nếu người dùng đã đổi từ khoá trong lúc chờ thì bỏ qua kết quả cũ.
- `setSearchResults()` sẽ bỏ chọn các mục đang chọn (Edge cũng làm vậy).

### 4.1. Các bước sửa tay

**Bước 1.** Thêm import:

```ts
import {/* ...các tên đang có... */, setSearchResults} from './actions.js';
import {BookmarksApiProxyImpl} from './bookmarks_api_proxy.js';
```

Nếu `app.ts` của bản fork gọi thẳng `chrome.bookmarks.search()` thì ở bước 3
dùng lời gọi đó thay cho `BookmarksApiProxyImpl`.

**Bước 2.** Thêm biến trạng thái, đặt cạnh `let debouncer`:

```ts
let searchRefreshPending: boolean = false;
```

**Bước 3.** Thêm hàm search lại:

```ts
/**
 * Re-runs the active search once the current batch of bookmark events has
 * been applied, so that nodes created while search results are shown (undo of
 * a delete, paste, a bookmark added from another window) appear in them.
 * Bursts of events, such as undoing a multi-item delete, share one query.
 */
function scheduleSearchRefresh() {
  const search = Store.getInstance().data.search;
  if (searchRefreshPending || !search.term || search.results === null) {
    return;
  }

  assert(debouncer);
  searchRefreshPending = true;
  debouncer.promise.then(() => {
    searchRefreshPending = false;
    const term = Store.getInstance().data.search.term;
    if (!term) {
      return;
    }
    BookmarksApiProxyImpl.getInstance().search(term).then(results => {
      // Drop the response if the user changed the search in the meantime.
      if (Store.getInstance().data.search.term !== term) {
        return;
      }
      dispatch(setSearchResults(results.map(node => node.id)));
    });
  });
}
```

**Bước 4.** Gọi hàm này ở cuối `onBookmarkCreated()`, sau `batchUIUpdates()` và
sau khi dispatch `createBookmark` (khi đó `debouncer` chắc chắn đã được tạo):

```ts
function onBookmarkCreated(
    id: string, treeNode: chrome.bookmarks.BookmarkTreeNode) {
  batchUIUpdates();
  if (trackUpdates) {
    updatedItems.push(id);
  }
  dispatch(createBookmark(id, treeNode));
  scheduleSearchRefresh();
}
```

### 4.2. Diff đầy đủ (base: Chromium 140.0.7339.80)

```diff
diff --git a/chrome/browser/resources/bookmarks/api_listener.ts b/chrome/browser/resources/bookmarks/api_listener.ts
index f3e1f92..89f195d 100644
--- a/chrome/browser/resources/bookmarks/api_listener.ts
+++ b/chrome/browser/resources/bookmarks/api_listener.ts
@@ -6,7 +6,8 @@ import {assert} from 'chrome://resources/js/assert.js';
 import {addWebUiListener, removeWebUiListener} from 'chrome://resources/js/cr.js';
 import type {Action} from 'chrome://resources/js/store.js';
 
-import {createBookmark, editBookmark, moveBookmark, refreshNodes, removeBookmark, reorderChildren, setCanEditBookmarks, setIncognitoAvailability} from './actions.js';
+import {createBookmark, editBookmark, moveBookmark, refreshNodes, removeBookmark, reorderChildren, setCanEditBookmarks, setIncognitoAvailability, setSearchResults} from './actions.js';
+import {BookmarksApiProxyImpl} from './bookmarks_api_proxy.js';
 import {BrowserProxyImpl} from './browser_proxy.js';
 import type {IncognitoAvailability} from './constants.js';
 import {Debouncer} from './debouncer.js';
@@ -23,6 +24,8 @@ let updatedItems: string[] = [];
 
 let debouncer: Debouncer|null = null;
 
+let searchRefreshPending: boolean = false;
+
 /**
  * Batches UI updates so that no changes will be made to UI until the next
  * task after the last call to this method. This is useful for listeners which
@@ -81,6 +84,36 @@ function onBookmarkChanged(
   dispatch(editBookmark(id, changeInfo));
 }
 
+/**
+ * Re-runs the active search once the current batch of bookmark events has
+ * been applied, so that nodes created while search results are shown (undo of
+ * a delete, paste, a bookmark added from another window) appear in them.
+ * Bursts of events, such as undoing a multi-item delete, share one query.
+ */
+function scheduleSearchRefresh() {
+  const search = Store.getInstance().data.search;
+  if (searchRefreshPending || !search.term || search.results === null) {
+    return;
+  }
+
+  assert(debouncer);
+  searchRefreshPending = true;
+  debouncer.promise.then(() => {
+    searchRefreshPending = false;
+    const term = Store.getInstance().data.search.term;
+    if (!term) {
+      return;
+    }
+    BookmarksApiProxyImpl.getInstance().search(term).then(results => {
+      // Drop the response if the user changed the search in the meantime.
+      if (Store.getInstance().data.search.term !== term) {
+        return;
+      }
+      dispatch(setSearchResults(results.map(node => node.id)));
+    });
+  });
+}
+
 function onBookmarkCreated(
     id: string, treeNode: chrome.bookmarks.BookmarkTreeNode) {
   batchUIUpdates();
@@ -88,6 +121,7 @@ function onBookmarkCreated(
     updatedItems.push(id);
   }
   dispatch(createBookmark(id, treeNode));
+  scheduleSearchRefresh();
 }
 
 function onBookmarkRemoved(
```

---

## 5. Cách áp diff

1. Copy nội dung khối diff (không lấy dòng ```` ```diff ```` và ```` ``` ````)
   ra một file, ví dụ `bookmarks-search-refresh.patch`.
2. Ở thư mục `src/` của bản fork, chạy:

```
git apply --3way --check bookmarks-search-refresh.patch
git apply --3way bookmarks-search-refresh.patch
```

Nếu báo lỗi vì code bản fork lệch context thì làm theo các bước sửa tay ở mục
3.1 hoặc 4.1.

## 6. Tuỳ chọn: giống Edge hoàn toàn (chưa có trong diff)

**Search lại khi đổi tên/sửa bookmark.** Edge search lại khi có `onChanged`, nên
mục đổi tên không còn khớp sẽ biến khỏi kết quả. Chromium hiện vẫn giữ mục đó.
Nếu muốn giống Edge:

- Cách A: thêm `case 'edit-bookmark':` ngay trên `case 'create-bookmark':` để
  dùng chung `refreshSearchResults(search, nodes)`.
- Cách B: trong `onBookmarkChanged()`, gọi `batchUIUpdates()` rồi
  `scheduleSearchRefresh()` (hàm này cần debouncer).

**Chi phí của cách A khi Undo số lượng lớn.** Cách A tính lại search cho mỗi
bookmark được tạo. Undo xoá N mục sẽ duyệt cả cây bookmark N lần. Với số
lượng bookmark thông thường thì không đáng kể. Nếu thư viện bookmark rất lớn thì
chuyển việc tính lại ra cuối đợt cập nhật như cách B.

## 7. Kiểm tra

**Test tự động** (cần build Chromium):

```
autoninja -C out/Default browser_tests
out/Default/browser_tests --gtest_filter="BookmarksTest.Reducers"
```

Cách B không có test tự động sẵn (Chromium 140 không có `api_listener_test`),
nên chỉ kiểm tra bằng tay.

**Kiểm tra bằng tay:**

| # | Thao tác | Kết quả mong đợi |
| --- | --- | --- |
| 1 | `chrome://bookmarks/?q=chat` → xoá 1 bookmark khớp → Undo | Bookmark hiện lại ngay, không cần reload |
| 2 | Chọn nhiều kết quả → Delete → Undo | Tất cả hiện lại |
| 3 | Đang mở `?q=chat`, thêm bookmark tên "chat test" từ cửa sổ khác | Bookmark mới hiện trong kết quả |
| 4 | Thêm bookmark không khớp từ khoá | Kết quả không đổi |
| 5 | Xoá ô search | Quay về hiển thị folder bình thường |
| 6 | Không search, xoá rồi Undo trong folder | Hoạt động như trước (không bị ảnh hưởng) |

## 8. Trạng thái

- Cả 2 diff đã kiểm tra `git apply` sạch trên đúng base của chúng: Chromium
  `main` cho cách A, 140.0.7339.80 cho cách B.
- **Chưa build và chưa chạy test** vì máy phân tích không có source Chromium.
