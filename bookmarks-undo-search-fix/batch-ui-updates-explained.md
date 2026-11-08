# Giải thích `batchUIUpdates`, `beginBatchUpdate`, `endBatchUpdate`

Ba hàm này gom nhiều thay đổi dữ liệu liên tiếp lại để **giao diện chỉ vẽ lại một
lần**. Patch trong `fix-patch-new-base.md` dựa vào cơ chế này để chỉ search lại
một lần cho mỗi lô sự kiện.

Code bên dưới lấy từ Chromium `main`. Bản Samsung cũ cũng có đúng code này, chỉ
bị nén thành một dòng.

Có 3 thành phần:

- **Store**: giữ dữ liệu của trang và báo cho giao diện mỗi khi dữ liệu đổi.
- **`beginBatchUpdate` / `endBatchUpdate`**: công tắc bật/tắt việc báo cho giao
  diện.
- **`batchUIUpdates` + `Debouncer`**: tự bật công tắc khi có sự kiện, và tự tắt
  khi các sự kiện ngừng đến.

## 1. Store: dữ liệu luôn cập nhật, giao diện thì tuỳ chế độ

`ui/webui/resources/js/store.ts`, dòng 119–131:

```ts
protected reduce(action: A|null) {
  ...
  this.data = this.reducer_(this.data, action);       // dữ liệu LUÔN được cập nhật

  if (this.isInitialized() && !this.batchMode_) {     // chỉ báo giao diện khi KHÔNG ở chế độ gom
    this.notifyObservers_(this.data);
  }
}
```

Mỗi lần `dispatch(action)`, reducer chạy và `this.data` đổi ngay. Nhưng nếu
`batchMode_` đang bật thì các component giao diện **không được báo**, nên không vẽ
lại.

## 2. `beginBatchUpdate()`: bật chế độ gom

`ui/webui/resources/js/store.ts`, dòng 76–78:

```ts
beginBatchUpdate() {
  this.batchMode_ = true;
}
```

Hàm chỉ bật cờ. Từ lúc này các `dispatch` vẫn cập nhật dữ liệu nhưng không làm
giao diện vẽ lại. Comment trong code lấy ví dụ: xoá 100 bookmark thì không nên vẽ
lại 100 lần.

## 3. `endBatchUpdate()`: tắt chế độ gom và vẽ lại một lần

`ui/webui/resources/js/store.ts`, dòng 84–87:

```ts
endBatchUpdate() {
  this.batchMode_ = false;
  this.notifyObservers_(this.data);   // báo giao diện 1 lần với dữ liệu cuối cùng
}
```

Hàm tắt cờ rồi báo cho giao diện **đúng một lần**, với dữ liệu đã gồm mọi thay đổi
trong lúc gom.

## 4. `Debouncer`: hẹn giờ có thể dời lại

`chrome/browser/resources/bookmarks/debouncer.ts`:

```ts
restartTimeout(delay?: number) {            // dòng 35
  assert(!this.isDone_);
  this.cancelTimeout_();                     // huỷ hẹn giờ cũ
  this.timer_ = this.timerProxy_.setTimeout(this.boundTimerCallback_, delay || 0);  // hẹn lại, mặc định 0ms
}

reset() {                                    // dòng 54: dùng lại như mới
  this.isDone_ = false;
  this.promiseResolver_ = new PromiseResolver();
  this.cancelTimeout_();
}

private timerCallback_() {                   // dòng 70: khi hết giờ
  this.isDone_ = true;
  this.callback_.call(this);                 // gọi callback (ở đây là endBatchUpdate)
  this.promiseResolver_.resolve();           // báo cho ai đang chờ .promise
}
```

`setTimeout(..., 0)` có nghĩa là "chạy sau khi lượt xử lý hiện tại kết thúc". Mỗi
lần gọi `restartTimeout()`, hẹn giờ cũ bị huỷ và hẹn lại từ đầu. Vì vậy callback
chỉ chạy khi các sự kiện **ngừng đến**.

## 5. `batchUIUpdates()`: nối mọi thứ lại

`chrome/browser/resources/bookmarks/api_listener.ts`, dòng 31–42:

```ts
function batchUIUpdates() {
  if (debouncer === null) {
    debouncer = new Debouncer(() => Store.getInstance().endBatchUpdate());  // (a)
  }

  if (debouncer.done()) {                    // (b) lô trước đã xong → mở lô mới
    Store.getInstance().beginBatchUpdate();
    debouncer.reset();
  }

  debouncer.restartTimeout();                // (c) dời thời điểm kết thúc lô
}
```

- **(a)** Tạo Debouncer một lần duy nhất. Callback khi hết giờ là
  `endBatchUpdate()`.
- **(b)** Nếu lô trước đã kết thúc thì bật chế độ gom và reset Debouncer để mở lô
  mới.
- **(c)** Hẹn lại giờ kết thúc lô. Còn sự kiện đến thì lô còn kéo dài.

Hàm này được gọi ở đầu `onBookmarkCreated`, `onBookmarkRemoved` và
`onBookmarkMoved`. `onBookmarkChanged` và `onChildrenReordered` không gọi nó, nên
hai sự kiện đó cập nhật giao diện ngay.

## 6. Ví dụ: một lô 3 sự kiện "bookmark mới được tạo"

Với import, cả 3 sự kiện đến trong cùng một lượt xử lý:

```
sự kiện 1: batchUIUpdates() → done() = true → beginBatchUpdate(), reset(), hẹn giờ T1
           dispatch(createBookmark) → data đổi, KHÔNG vẽ lại
sự kiện 2: batchUIUpdates() → done() = false → huỷ T1, hẹn T2
           dispatch(createBookmark) → data đổi, KHÔNG vẽ lại
sự kiện 3: batchUIUpdates() → huỷ T2, hẹn T3
           dispatch(createBookmark) → data đổi, KHÔNG vẽ lại
--- hết lượt xử lý ---
T3 chạy: isDone_ = true → endBatchUpdate() → tắt gom, vẽ lại 1 LẦN
         → promise resolve → highlightUpdatedItems() (nếu đang chờ) chạy tiếp
```

- Các sự kiện xử lý **trong cùng một lượt** (lô import gửi từ C++) chắc chắn nằm
  chung một lô.
- Các sự kiện đến **bằng tin nhắn riêng** (Undo nhiều mục) chỉ nằm chung lô nếu
  chúng chạy trước khi hẹn giờ 0ms hết. Nếu đến rải rác thì bị tách thành vài lô.

## 7. Một điểm lạ: lô đầu tiên không được gom

Khi `Debouncer` mới tạo, `isDone_` bằng `false`. Vì vậy ở lần gọi đầu tiên, điều
kiện **(b)** sai và `beginBatchUpdate()` **không được gọi**. Lô đầu tiên sau khi
mở trang chạy không gom: mỗi `dispatch` đều làm vẽ lại, và cuối lô
`endBatchUpdate()` vẽ thêm một lần nữa. Từ lô thứ hai trở đi mới gom bình thường.

## 8. Patch cho base mới gắn vào đâu

Patch trong `fix-patch-new-base.md` thay callback ở **(a)** bằng
`endBatchUIUpdates`:

```ts
function endBatchUIUpdates() {
  if (searchRefreshNeeded) {
    searchRefreshNeeded = false;
    dispatch(refreshSearchResults());   // vẫn đang batchMode → chỉ đổi data, chưa vẽ
  }
  Store.getInstance().endBatchUpdate(); // vẽ lại 1 lần: node mới + kết quả search mới
}
```

Search lại được chạy ngay **trước** `endBatchUpdate()`, tức là lúc trang vẫn còn
đang gom. Vì vậy cả lô chỉ search một lần, và dữ liệu mới cùng kết quả search mới
được vẽ lên trong cùng một lần.
