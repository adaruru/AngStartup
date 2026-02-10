# AngNgRx：從零開始搞懂 Angular 狀態管理

## 1. 先用白話理解「狀態管理」是什麼

你在做點餐機時，畫面上有很多「會變的資料」：
- 菜單資料
- 購物車內容
- 使用者輸入
- 付款結果
- 後台商品管理資料

這些「會變的資料」就叫 **state（狀態）**。

狀態管理要解決的核心問題是：
- 資料放哪裡最合理？
- 誰可以改它？
- 改完後，哪些畫面要同步更新？
- API 錯誤、重試、loading 怎麼一致處理？



## 2. Angular 裡你會遇到的 3 種狀態

### 2.1 元件本地狀態（Local State）
- 只在單一元件使用，例如 modal 開關、表單欄位錯誤。
- 通常用 Angular `signal`、`FormControl` 即可。

### 2.2 功能模組狀態（Feature State）
- 例如購物車、菜單篩選條件。
- 多個元件會用到，但範圍還不算全站。
- 非常適合用 `NgRx SignalStore`（`@ngrx/signals`）。

### 2.3 全站共享狀態（Global App State）
- 例如登入資訊、權限、全站通知、跨頁面流程。
- 適合用 `NgRx Store + Effects`（`@ngrx/store`, `@ngrx/effects`）。



## 3. 名詞對照（你會看到的套件）

### 3.1 NgRx 家族
- `@ngrx/store`：Redux 風格全域 store（state + action + reducer + selector）。
- `@ngrx/effects`：把 API、side effects 從元件抽離。
- `@ngrx/signals`：SignalStore，語法更直覺、上手快。
- `@ngrx/store-devtools`：開發除錯（看 action 與 state 時序）。

### 3.2 NGXS
- `@ngxs/store`：另一套 Angular 狀態管理方案，語法較接近 OOP/class。



## 4. 你可以先記這個選型結論

- 想像 Pinia/Zustand 那種好上手體驗：先用 **NgRx SignalStore**。
- 要大型專案、可追蹤流程、嚴格規範：用 **NgRx Store + Effects**。
- 想要 class decorator 風格：可考慮 **NGXS**。

最常見現實做法：
- 小到中型功能：`@ngrx/signals`
- 全站核心流程：`@ngrx/store + @ngrx/effects`
- 同專案混用是正常的



## 5. Part A： `@ngrx/signals`（SignalStore）

### 5.1 安裝
```bash
npm i @ngrx/signals
```

### 5.2 你要先懂的 4 個積木
- `withState`：定義 state 結構與初始值。
- `withComputed`：衍生狀態（例如總金額）。
- `withMethods`：定義改 state 的方法。
- `patchState`：更新 state。

### 5.3 購物車 SignalStore 範例（可直接理解）
```ts
// cart.signal-store.ts
import { computed } from '@angular/core';
import {
  signalStore,
  withState,
  withComputed,
  withMethods,
  patchState
} from '@ngrx/signals';

type CartItem = {
  itemId: string;
  name: string;
  price: number;
  qty: number;
};

type CartState = {
  items: CartItem[];
};

const initialState: CartState = {
  items: []
};

export const CartSignalStore = signalStore(
  { providedIn: 'root' },
  withState(initialState),
  withComputed(({ items }) => ({
    totalQty: computed(() => items().reduce((sum, i) => sum + i.qty, 0)),
    totalAmount: computed(() =>
      items().reduce((sum, i) => sum + i.price * i.qty, 0)
    )
  })),
  withMethods((store) => ({
    addItem(item: CartItem) {
      const existing = store.items().find((i) => i.itemId === item.itemId);
      if (existing) {
        patchState(store, {
          items: store.items().map((i) =>
            i.itemId === item.itemId ? { ...i, qty: i.qty + 1 } : i
          )
        });
        return;
      }
      patchState(store, { items: [...store.items(), item] });
    },

    updateQty(itemId: string, qty: number) {
      patchState(store, {
        items: store.items().map((i) =>
          i.itemId === itemId ? { ...i, qty: Math.max(qty, 1) } : i
        )
      });
    },

    removeItem(itemId: string) {
      patchState(store, {
        items: store.items().filter((i) => i.itemId !== itemId)
      });
    },

    clear() {
      patchState(store, { items: [] });
    }
  }))
);
```

### 5.4 元件怎麼用
```ts
// cart.component.ts
import { Component, inject } from '@angular/core';
import { CartSignalStore } from './cart.signal-store';

@Component({
  selector: 'app-cart',
  template: `
    <h3>購物車</h3>
    <p>總數量：{{ cartStore.totalQty() }}</p>
    <p>總金額：{{ cartStore.totalAmount() }}</p>
  `
})
export class CartComponent {
  cartStore = inject(CartSignalStore);
}
```

### 5.5 SignalStore 你會得到什麼好處
- 比傳統 reducer 寫法更直覺。
- TypeScript 型別自然推導。
- 非常適合 feature scope（像購物車、篩選器、結帳步驟狀態）。



## 6. Part B：`NgRx Store + Effects`（企業級主流）

當專案變大、流程變複雜，這套非常有價值。

### 6.1 安裝
```bash
npm i @ngrx/store @ngrx/effects @ngrx/store-devtools
```

### 6.2 核心觀念（先看這段）
- **Action**：事件，代表「發生了什麼」。
- **Reducer**：純函式，根據 action 產生新 state。
- **Selector**：從 state 取資料、做衍生計算。
- **Effect**：處理 API 等副作用，成功/失敗再 dispatch 新 action。

一句話版：
- 元件 dispatch action
- reducer 改 state
- effects 打 API
- selectors 給畫面讀資料

### 6.3 範例資料結構
```ts
// order.state.ts
export interface OrderState {
  loading: boolean;
  currentOrderId: string | null;
  error: string | null;
}

export const initialOrderState: OrderState = {
  loading: false,
  currentOrderId: null,
  error: null
};
```

### 6.4 Action
```ts
// order.actions.ts
export const OrderActions = createActionGroup({
  source: 'Order',
  events: {
    'Submit Order': props<{ payload: { items: Array<{ itemId: string; qty: number }> } }>(),
    'Submit Order Success': props<{ orderId: string }>(),
    'Submit Order Failure': props<{ error: string }>(),
    'Clear Order Error': emptyProps()
  }
});
```

### 6.5 Reducer
```ts
// order.reducer.ts
export const orderReducer = createReducer(
  initialOrderState,
  on(OrderActions.submitOrder, (state) => ({
    ...state,
    loading: true,
    error: null
  })),
  on(OrderActions.submitOrderSuccess, (state, { orderId }) => ({
    ...state,
    loading: false,
    currentOrderId: orderId
  })),
  on(OrderActions.submitOrderFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error
  })),
  on(OrderActions.clearOrderError, (state) => ({
    ...state,
    error: null
  }))
);
```

### 6.6 Selector
```ts
// order.selectors.ts
import { createFeatureSelector, createSelector } from '@ngrx/store';
import { OrderState } from './order.state';

export const selectOrderState = createFeatureSelector<OrderState>('order');

export const selectOrderLoading = createSelector(
  selectOrderState,
  (state) => state.loading
);

export const selectCurrentOrderId = createSelector(
  selectOrderState,
  (state) => state.currentOrderId
);

export const selectOrderError = createSelector(
  selectOrderState,
  (state) => state.error
);
```

### 6.7 Effect（打 API 的關鍵）
```ts
// order.effects.ts
import { Injectable, inject } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { catchError, map, of, switchMap } from 'rxjs';
import { OrderActions } from './order.actions';
import { OrderApiService } from './order-api.service';

@Injectable()
export class OrderEffects {
  private actions$ = inject(Actions);
  private api = inject(OrderApiService);

  submitOrder$ = createEffect(() =>
    this.actions$.pipe(
      ofType(OrderActions.submitOrder),
      switchMap(({ payload }) =>
        this.api.submit(payload).pipe(
          map((res) => OrderActions.submitOrderSuccess({ orderId: res.orderId })),
          catchError((err) =>
            of(OrderActions.submitOrderFailure({ error: err?.message ?? '下單失敗' }))
          )
        )
      )
    )
  );
}
```

### 6.8 註冊 Store 與 Effects（Standalone）
```ts
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideStore(),
    provideState('order', orderReducer),
    provideEffects([OrderEffects]),
    provideStoreDevtools({ maxAge: 25 })
  ]
};
```

### 6.9 元件怎麼用（dispatch + select）
```ts
@Component({
  selector: 'app-checkout',
  template: `
    <button [disabled]="loading()" (click)="submit()">送出訂單</button>
    <p *ngIf="error()">{{ error() }}</p>
  `
})
export class CheckoutComponent {
  private store = inject(Store);

  loading = this.store.selectSignal(selectOrderLoading);
  error = this.store.selectSignal(selectOrderError);

  submit() {
    this.store.dispatch(
      OrderActions.submitOrder({
        payload: { items: [{ itemId: 'tea', qty: 1 }] }
      })
    );
  }
}
```

## 7. Part C：`@ngxs/store` 是什麼

`NGXS` 是另一個 Angular 狀態管理解法，寫法偏 class decorator。

### 7.1 安裝
```bash
npm i @ngxs/store
```

### 7.2 最小概念
- `@State`：宣告 state。
- `@Action`：處理事件（可含同步/非同步邏輯）。
- `@Selector`：讀取資料。

### 7.3 簡化範例
```ts
import { State, Action, StateContext, Selector } from '@ngxs/store';

export class AddItem {
  static readonly type = '[Cart] Add Item';
  constructor(public payload: { itemId: string; qty: number }) {}
}

interface CartStateModel {
  items: Array<{ itemId: string; qty: number }>;
}

@State<CartStateModel>({
  name: 'cart',
  defaults: { items: [] }
})
export class CartState {
  @Selector()
  static totalQty(state: CartStateModel) {
    return state.items.reduce((sum, i) => sum + i.qty, 0);
  }

  @Action(AddItem)
  add(ctx: StateContext<CartStateModel>, action: AddItem) {
    const state = ctx.getState();
    ctx.patchState({
      items: [...state.items, action.payload]
    });
  }
}
```



## 8. `@ngrx/signals` vs `@ngrx/store` vs `@ngxs/store`

| 面向 | `@ngrx/signals` (SignalStore) | `@ngrx/store` + effects | `@ngxs/store` |
|---|---|---|---|
| 上手難度 | 低到中 | 中到高 | 中 |
| 架構嚴謹度 | 中 | 高 | 中 |
| 大型專案可擴充 | 中到高 | 很高 | 中到高 |
| 心智模型 | 直覺（signals） | Redux 事件流 | Class/Decorator |
| 最適合 | feature state | 全站核心流程 | 偏好 OOP 寫法團隊 |
| 資料存取位置（預設） | 記憶體（SignalStore instance） | 記憶體（Store state tree） | 記憶體（NGXS state container） |

### 8.1 Persistence

重要觀念：`State != Persistence`

- 狀態管理（NgRx/NGXS/SignalStore）主要解決「執行中的一致性」。
- 持久化（Persistence）主要解決「跨重整、跨重啟仍保留資料」。
- 所以兩者不是互斥，而是分工：
  - `狀態管理`：管資料流、更新規則、畫面同步。
  - `持久化策略`：決定哪些 state 要寫到 `localStorage/sessionStorage/IndexedDB` 或回寫後端。

### 8.2 State 分類

| 資料類型     | 例子                            |
| ------------ | ------------------------------- |
| Server State | 菜單、訂單、付款結果            |
| UI State     | loading、error、modal、目前 tab |
| Draft State  | 購物車草稿、結帳步驟進度        |

### 8.2.1 分類完「下一步」要做的 5 個資料流決策決策

1. `Source of Truth`：最終以後端還是前端為準？
2. `讀取策略`：進頁面時重抓 API，還是先用快取再背景更新？
3. `寫入策略`：先改本地再同步，還是成功回應後才更新畫面？
4. `Persistence`決策
   1. 重整後要不要保留？
      - 這份資料「重整後還需要」嗎？
      - 資料保留行為涉及同分頁內`路由切換`，不會重整：留在記憶體 state。
      - 資料保留行為涉及`重整`：做持久化( 可選 localStorage )，但只持久化必要欄位，並設計過期/版本策略。
      - 資料保留行為涉及`跨分頁、跨裝置`：只靠 localStorage 不夠，通常要以後端資料為準。

   2. 不可持久化
      - 會產生過期資料與版本不相容問題（部署新版本後舊資料格式可能壞掉）。
      - 可能有資安/隱私風險（共用裝置留下敏感資訊）。
      - 多分頁與跨裝置同步複雜度上升。
      - 某些狀態本來就短生命週期（例如 loading），持久化沒有價值。

5. `失效策略`：多久過期、版本不相容時怎麼清除？

### 8.2.2 點餐機可直接套用的決策範本

- 先分類，再做 5 個資料流決策，最後才選工具（SignalStore / Store + Effects / NGXS）。

- 若資料跨重整需要保留，才加持久化；不要預設全部寫進 localStorage。


| 資料類型 | Source of Truth | 讀取策略 | 寫入策略 | Persistence 決策 | 失效策略 | 最終工具（建議） |
|---|---|---|---|---|---|---|
| Server State（菜單、訂單、付款結果） | 後端資料庫 | 進頁先顯示快取，再背景重抓 API | 以 API 成功回應為準再更新 store | 通常不強制持久化；若要快啟動可做快取 | TTL + 版本號；版本變更時清快取 | `@ngrx/store + @ngrx/effects` + `HttpClient` |
| UI State（loading、error、modal） | 前端執行期 | 元件啟動時初始化 | 事件觸發即時更新 | 不持久化 | route 離開或流程結束即清空 | Angular component `signal`（必要時 `@ngrx/signals`） |
| Draft State（購物車草稿、結帳進度） | 前端暫存 | 先讀本地草稿，再與菜單有效性比對 | 變更後節流寫入本地 | 建議持久化必要欄位提升 UX（例如 itemId、qty） | 設到期時間；提交訂單後清空 | `@ngrx/signals` + `localStorage`（hydration） |

### 8.5 Boilerplate

Boilerplate（樣板程式碼）

- `Boilerplate` 指的是每個功能都要重複寫、結構很像的基礎程式碼。
- 在狀態管理裡，常見重複是：action、reducer、selector、effect、測試檔案。
- 所以很多人會說 `@ngrx/store + effects` 「boilerplate 比較多」。

| 套件 | Boilerplate 感受 | 主要原因 |
|---|---|---|
| `@ngrx/signals` | 低到中 | 同一個 store 檔案就能放 state/computed/methods |
| `@ngrx/store + effects` | 中到高 | 事件流切分明確，通常要拆多檔案 |
| `@ngxs/store` | 中 | class + decorator 寫法較集中，但大型專案仍會分層 |

Boilerplate 多不一定是壞事：
- 成本：初期看起來慢、檔案多。
- 收益：責任分離清楚、流程可追蹤、多人協作穩定。

如何降低 NgRx Boilerplate（實務做法）：
- 優先用 `createActionGroup`、`createFeature` 等 API 減少重複樣板。
- 以功能切分資料夾（feature-first），不要把所有 action/reducer 混在一起。
- 中小型功能先用 `SignalStore`，只在複雜跨頁流程使用 `Store + Effects`。
- 用 NgRx schematics 產生骨架，再補商業邏輯，避免手刻大量樣板。



## 9. 常見誤解（新手最容易卡）

- 誤解 1：一定要全站都用 store。
  - 正解：不是。簡單局部狀態用 component signal 就夠。

- 誤解 2：SignalStore 比 Store 高級，所以應該全部改成 SignalStore。
  - 正解：不是高低，是適用情境不同。大型流程追蹤仍常用 Store + Effects。

- 誤解 3：Effects 很麻煩，所以 API 放元件裡比較快。
  - 正解：短期快，長期難維護。API 邏輯集中在 effects/service 會更穩。

- 誤解 4：學了 NgRx 就不能用 service。
  - 正解：NgRx 和 service 會一起用。service 專職封裝 API。



## 10. 給你的實作落地建議（對應點餐機專題）

### 第 1 階段（先會做）
- 菜單查詢：`MenuService + HttpClient`
- 購物車：`SignalStore`
- 結帳表單：元件本地 signal + Reactive Forms

### 第 2 階段（進入中大型）
- 訂單送出與付款流程：`@ngrx/store + @ngrx/effects`
- 錯誤、loading、重試：透過 actions/effects/reducer 管理

### 第 3 階段（品質）
- Store Devtools 驗證 action 流程
- 補 selector 單元測試、effect 單元測試



## 11. 學習路線（完全新手版）

1. 先懂 state 與資料流（本講義第 1-4 節）
2. 實作一個 SignalStore（購物車）
3. 把「下單 API」改成 Store + Effects
4. 加入錯誤處理與 loading
5. 最後再比較 NGXS（知道差異即可）



## 12. 面試/實務常問的 6 題

1. 什麼時候用 SignalStore？
   - 功能區域狀態、希望開發快、保持可維護。


2. 什麼時候用 Store + Effects？
   - 跨頁流程、需要事件追蹤、團隊協作大、規模大。

3. SignalStore 和 Zustand/Pinia 像嗎？
   - 使用體驗上比較像，心智負擔較低。

4. Effects 的價值是什麼？
   - 把副作用（API）從元件拆出去，狀態流更乾淨、可測試。

5. NGXS 跟 NgRx 誰比較好？
   - 沒有絕對；目前 Angular 主流與生態整合通常偏 NgRx。

6. 可以混用嗎？
   - 可以，而且很常見。




## 13. 最短可執行練習

1. 建 `CartSignalStore`，做到新增/刪除/計算總額。
2. 建 `Order` feature（action/reducer/effect）做下單 API。
3. 畫面上同時顯示 `loading`、`error`、`orderId`。
4. 用 devtools 檢查 action 時序是否正確。

做到這 4 步，你就已經超過「只會背名詞」的階段。



## 14. 一句話收斂

- `@ngrx/signals`：先上手、快交付。  
- `@ngrx/store + @ngrx/effects`：大規模、可追蹤、團隊友善。  
- `@ngxs/store`：另一種可行路線，但你現在先把 NgRx 兩條主線學會最划算。
