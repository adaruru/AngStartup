# RxJS Observable 完全新手教學（Angular 實戰版）

## Observable
Observable

- 一條會隨時間推送資料的資料流
- 可取消、可組合、有時間序列的流程
- 有時間軸的資料
- 資料直播頻道

  - 你先訂閱（subscribe）。

  - 資料之後可能持續送來（next）。

  - 中途可能出錯（error）。

  - 或在某個時間點結束（complete）。


### HttpClient

Angular api 框架使用 `HttpClient` 回傳的是 Observable，不是 Promise。

- 單次 HTTP 通常會在回應後自動 complete。
- 長期流（例如 `interval`、`fromEvent`）就要注意取消訂閱。

### Rxjs

所以選擇 Angular 就勢必要學 Rxjs，Angular 的需求更接近「資料流」，Promise 適合「一次拿結果」；Observable 適合「可取消、可組合、有時間序列的流程」。

1. 可取消請求
   Observable 可 unsubscribe，搭配 switchMap 能取消舊請求（搜尋特別常用）。
2. 可處理多次事件
   HTTP 不只最終結果，還可能有上傳/下載進度事件；Observable 天然適合事件流。
3. 可做複雜非同步組合
   retry、catchError、debounceTime、throttle、switchMap 這些在 RxJS 很完整。
4. 與 Angular 生態一致
   Router、事件、NgRx Effects 都是流式思維，用 Observable 心智模型一致。

## Promise 比較

| 比較 | Promise | Observable |
|---|---|---|
| 資料次數 | 一次 | 0 次到多次 |
| 是否可取消 | 不好取消 | 可取消（unsubscribe） |
| 資料型態 | 單次結果 | 時間序列資料流 |
| 組合能力 | 有限 | 很強（operators） |

Angular 也可以走 Promise 風格（尤其簡單 CRUD）：

```typescript
import { firstValueFrom } from 'rxjs'; const data = await firstValueFrom(this.http.get<MyDto>('/api/items')); 
```

## Observable 的 3 種通知

- `next(value)`：送資料
- `error(err)`：出錯，流結束
- `complete()`：正常完成

最小例子：
```ts
import { Observable } from 'rxjs';

const stream$ = new Observable<number>((observer) => {
  observer.next(1);
  observer.next(2);
  observer.complete();
});

stream$.subscribe({
  next: (v) => console.log('next:', v),
  error: (e) => console.error('error:', e),
  complete: () => console.log('complete')
});
```

## 你會一直看到的符號：`$`
- 慣例上，變數名後面加 `$` 代表它是 Observable。
- 例如：`items$`、`loading$`、`orderResult$`。

## 建立 Observable 的常用方式

```ts
import { of, from, interval, fromEvent } from 'rxjs';

const a$ = of(1, 2, 3);                     // 直接送固定值
const b$ = from([10, 20, 30]);              // 從陣列/Promise 轉
const c$ = interval(1000);                  // 每秒送值
const d$ = fromEvent(document, 'click');    // 事件流
```

## 核心觀念：Operator（操作子）
- 你不直接改資料，而是在 `pipe(...)` 裡把資料流一步步轉換。

```ts
import { map, filter } from 'rxjs/operators';

items$.pipe(
  filter((x) => x.price > 100),
  map((x) => ({ ...x, label: `${x.name} - $${x.price}` }))
);
```

### 8.1 新手先會這 8 個
- `map`：轉資料
- `filter`：過濾資料
- `tap`：做 side effect（log、debug）
- `debounceTime`：防抖
- `distinctUntilChanged`：值相同就不重送
- `catchError`：錯誤處理
- `take`：只取前 N 筆
- `finalize`：收尾（關 loading）

## 最容易搞混：`switchMap` / `mergeMap` / `concatMap` / `exhaustMap`

| 操作子 | 行為 | 適合場景 |
|---|---|---|
| `switchMap` | 新請求來就取消舊請求 | 搜尋建議、即時查詢 |
| `mergeMap` | 全部並行 | 可並行的批次請求 |
| `concatMap` | 排隊依序 | 必須照順序送出 |
| `exhaustMap` | 忽略重複觸發直到完成 | 防重複提交（下單、付款） |

## Angular 實戰 1：搜尋框（必學）

```ts
this.keywordControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap((keyword) => this.menuService.search(keyword))
).subscribe((items) => {
  this.items = items;
});
```

為什麼是 `switchMap`：
- 使用者一直打字時，舊請求應該取消，避免舊資料覆蓋新資料。

## Angular 實戰 2：下單按鈕防連點

```ts
fromEvent(submitButton, 'click').pipe(
  exhaustMap(() => this.orderService.createOrder(payload))
).subscribe();
```

為什麼是 `exhaustMap`：
- 第一次提交還沒回來前，忽略後續連點，避免重複下單。

## 錯誤與 loading 的標準寫法

```ts
this.loading = true;
this.orderService.createOrder(payload).pipe(
  catchError((err) => {
    this.errorMsg = err?.message ?? '下單失敗';
    return of(null);
  }),
  finalize(() => {
    this.loading = false;
  })
).subscribe((result) => {
  if (!result) return;
  this.orderId = result.id;
});
```

## 取消訂閱：什麼時候要做
- 需要：`interval`、`fromEvent`、自建長期流。
- 通常不需要手動：單次 `HttpClient`。

Angular 常見做法：
- `takeUntil(destroy$)`（傳統）
- `takeUntilDestroyed()`（Angular 新式寫法）

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

this.someLongStream$
  .pipe(takeUntilDestroyed())
  .subscribe();
```

## Observable 在 NgRx Effects 長怎樣

```ts
submitOrder$ = createEffect(() =>
  this.actions$.pipe(
    ofType(OrderActions.submitOrder),
    switchMap(({ payload }) =>
      this.orderService.createOrder(payload).pipe(
        map((res) => OrderActions.submitOrderSuccess({ orderId: res.id })),
        catchError((e) => of(OrderActions.submitOrderFailure({ error: e.message })))
      )
    )
  )
);
```

重點：
- Effects 本質就是在操作 Observable 流。

## 常見錯誤（新手高機率）
- 一直 `subscribe` 但不管理生命週期，造成 memory leak。
- 看到 `map` 就拿來打 API（應該用 `switchMap/mergeMap/...`）。
- `catchError` 後沒回 Observable（會直接壞掉）。
- 在元件裡塞太多資料流邏輯，變成難維護。

## 快速判斷表：我該用哪個
- 需要取消前一次請求：`switchMap`
- 要並行：`mergeMap`
- 要排隊：`concatMap`
- 要防重複提交：`exhaustMap`

## 練習路線（建議照做）
1. 用 `of`、`from`、`interval` 建 3 個 Observable，練 `subscribe`。
2. 做搜尋框：`debounceTime + distinctUntilChanged + switchMap`。
3. 做下單防連點：`exhaustMap`。
4. 補上 `catchError + finalize`。
5. 把 API 流程搬進 NgRx Effect。
