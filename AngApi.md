# Angular API 講義整理

## 1. 這份講義的用途
- 目標：整理課綱中所有 API 相關知識，讓學員從「會呼叫 API」到「能維護 API 架構」。
- 適用章節：主要對應 `L11-L22`，並延伸到 `L26-L29` 的測試與部署驗收。
- 專案情境：餐飲業者點餐機（菜單、購物車、結帳、付款、後台管理）。

## 2. 為什麼 Angular 使用 HttpClient（不是 fetch）
- Angular 官方整合：`HttpClient` 來自 `@angular/common/http`，和 DI、攔截器、測試工具完整整合。
- 可測試性高：可用 `HttpTestingController` 模擬與驗證請求。
- 介面一致：所有 API 透過 service 層管理，降低元件直接打 API 的耦合。
- 易於擴充：token、錯誤處理、重試、統一 headers 可由 interceptor 集中處理。

> 結論：`fetch` 能用，但在 Angular 教學與實務中，`HttpClient` 更適合作為主方案。

## 3. 需要的套件與模組
- 內建主要套件：
  - `@angular/common/http`
  - `rxjs`
- 開發用 mock（可選）：
  - `json-server`

### 3.1 基本匯入
```ts
// app.config.ts (standalone) 或 app.module.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export const appConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([
        // 這裡可放 functional interceptor
      ])
    )
  ]
};
```

## 4. API 架構原則（本課程標準）
- 元件只負責顯示與互動，不直接處理 API 細節。
- API 由 service 統一管理，回傳 `Observable<T>`。
- 所有 URL 集中在環境變數與 endpoint 常數，不在元件內硬寫。
- 請求錯誤統一交給 interceptor + UI 錯誤提示元件處理。

## 5. 領域型別（API I/O）
```ts
export interface MenuCategory {
  id: string;
  name: string;
  sort: number;
}

export interface OptionItem {
  id: string;
  name: string;
  priceDelta: number;
}

export interface OptionGroup {
  id: string;
  name: string;
  required: boolean;
  maxSelect: number;
  options: OptionItem[];
}

export interface MenuItem {
  id: string;
  categoryId: string;
  name: string;
  price: number;
  options: OptionGroup[];
  isAvailable: boolean;
}

export interface CartItem {
  itemId: string;
  qty: number;
  selectedOptions: string[];
  subtotal: number;
}

export interface Order {
  id: string;
  items: CartItem[];
  amount: number;
  tax: number;
  serviceFee: number;
  total: number;
  status: 'pending' | 'paid' | 'failed';
}

export interface PaymentResult {
  orderId: string;
  status: 'success' | 'failed';
  transactionId: string;
  paidAt: string;
}
```

## 6. Service 設計與實作範例

### 6.1 MenuService
```ts
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class MenuService {
  private http = inject(HttpClient);
  private baseUrl = '/api';

  getCategories(): Observable<MenuCategory[]> {
    return this.http.get<MenuCategory[]>(`${this.baseUrl}/categories`);
  }

  getItems(): Observable<MenuItem[]> {
    return this.http.get<MenuItem[]>(`${this.baseUrl}/items`);
  }

  saveItem(payload: MenuItem): Observable<MenuItem> {
    return this.http.post<MenuItem>(`${this.baseUrl}/items`, payload);
  }

  deleteItem(id: string): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/items/${id}`);
  }
}
```

### 6.2 CartService（前端狀態 + API 可擴充）
```ts
import { Injectable, signal, computed } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class CartService {
  private items = signal<CartItem[]>([]);

  cartItems = computed(() => this.items());
  totalQty = computed(() => this.items().reduce((sum, i) => sum + i.qty, 0));
  totalAmount = computed(() => this.items().reduce((sum, i) => sum + i.subtotal, 0));

  addItem(item: CartItem): void {
    this.items.update(curr => [...curr, item]);
  }

  updateQty(itemId: string, qty: number): void {
    this.items.update(curr =>
      curr.map(i => (i.itemId === itemId ? { ...i, qty } : i))
    );
  }

  removeItem(itemId: string): void {
    this.items.update(curr => curr.filter(i => i.itemId !== itemId));
  }

  clear(): void {
    this.items.set([]);
  }

  getSummary() {
    return {
      totalQty: this.totalQty(),
      totalAmount: this.totalAmount()
    };
  }
}
```

### 6.3 OrderService
```ts
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class OrderService {
  private http = inject(HttpClient);
  private baseUrl = '/api/orders';

  createOrder(payload: Omit<Order, 'id' | 'status'>): Observable<Order> {
    return this.http.post<Order>(this.baseUrl, payload);
  }

  payOrder(orderId: string): Observable<PaymentResult> {
    return this.http.post<PaymentResult>(`${this.baseUrl}/${orderId}/pay`, {});
  }

  getOrderResult(orderId: string): Observable<Order> {
    return this.http.get<Order>(`${this.baseUrl}/${orderId}`);
  }
}
```

## 7. RxJS 在 API 流程中的標準用法

### 7.1 搜尋與分類串流
```ts
import { combineLatest, map, startWith } from 'rxjs';

const filteredItems$ = combineLatest([
  menuItems$,
  keywordControl.valueChanges.pipe(startWith('')),
  categoryControl.valueChanges.pipe(startWith('all'))
]).pipe(
  map(([items, keyword, category]) => {
    return items.filter(item => {
      const okKeyword = item.name.toLowerCase().includes(String(keyword).toLowerCase());
      const okCategory = category === 'all' || item.categoryId === category;
      return okKeyword && okCategory;
    });
  })
);
```

### 7.2 常用 operators 建議
- `map`：回應資料轉換（DTO -> ViewModel）
- `switchMap`：使用者快速重複觸發請求時，保留最新一次
- `catchError`：局部錯誤降級（例如回空陣列）
- `tap`：非業務邏輯 side-effect（如記錄 log）
- `shareReplay(1)`：共用同一份請求結果，避免重複打 API

## 8. Interceptor 與全域錯誤處理

### 8.1 Token + Error Interceptor 範例
```ts
import { HttpErrorResponse, HttpInterceptorFn } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';

export const apiInterceptor: HttpInterceptorFn = (req, next) => {
  const authReq = req.clone({
    setHeaders: {
      Authorization: 'Bearer demo-token'
    }
  });

  return next(authReq).pipe(
    catchError((error: HttpErrorResponse) => {
      // 統一轉換錯誤訊息，UI 可直接顯示
      const message =
        error.status === 0
          ? '網路連線異常，請稍後再試'
          : `API 錯誤（${error.status}）`;

      return throwError(() => new Error(message));
    })
  );
};
```

### 8.2 錯誤分級建議
- `4xx`：顯示可修正提示（輸入錯誤、權限不足）
- `5xx`：顯示系統忙碌，提供重試
- `0`：網路錯誤，提示檢查連線

## 9. Mock API 教學流程（json-server）

### 9.1 建議資料檔 `db.json`
```json
{
  "categories": [],
  "items": [],
  "orders": []
}
```

### 9.2 啟動命令（範例）
```bash
npx json-server --watch db.json --port 3000
```

### 9.3 前端代理（避免 CORS）
- 設定 `proxy.conf.json`，將 `/api` 轉發到 `http://localhost:3000`。
- 開發啟動時帶 `--proxy-config proxy.conf.json`。

## 10. API 測試講義（L26-L27）

### 10.1 Service 單元測試重點
- `getItems()` 是否打到正確 endpoint。
- `createOrder()` 請求 body 是否完整。
- 錯誤時是否回傳預期錯誤訊息。

### 10.2 `HttpTestingController` 範例
```ts
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';

it('should request menu items', () => {
  TestBed.configureTestingModule({
    providers: [provideHttpClient(), provideHttpClientTesting(), MenuService]
  });

  const service = TestBed.inject(MenuService);
  const http = TestBed.inject(HttpTestingController);

  service.getItems().subscribe(items => {
    expect(items.length).toBe(1);
  });

  const req = http.expectOne('/api/items');
  expect(req.request.method).toBe('GET');
  req.flush([{ id: 'm1', name: 'Tea', categoryId: 'drink', price: 50, options: [], isAvailable: true }]);
  http.verify();
});
```

## 11. 部署前 API 驗收清單（L29-L30）
- 所有 API URL 使用環境變數，不硬編碼。
- 失敗流程（超時、500、斷線）都有可見提示。
- 付款失敗可重試，且不會重複建立訂單。
- 後台 CRUD 的新增/編輯/刪除都能正確刷新畫面。
- 主流程 E2E 測試可穩定通過。

## 12. 對應課堂節點（快速索引）
- `L11`：Service + DI
- `L12`：HttpClient + Mock API
- `L13`：RxJS 串流與 operator
- `L16-L20`：訂單與付款 API 流程
- `L21`：Interceptor + 全域錯誤處理
- `L23-L24`：管理端 CRUD API
- `L26-L27`：API 單元測試與 E2E 流程測試
- `L29`：部署前 API 檢查

## 13. 常見錯誤與修正
- 問題：元件內直接 `http.get()`，導致難測試。
  - 修正：抽到 service，元件僅訂閱 observable。
- 問題：多處重複處理 token 與錯誤。
  - 修正：集中到 interceptor。
- 問題：沒有型別，API 回應一變就壞。
  - 修正：先定義 interface，再串 API。
- 問題：重複點擊送出造成重複訂單。
  - 修正：按鈕防連點 + `switchMap` + 後端冪等設計（若可配合）。

## 14. Order API 完整流程：Component、Service、Effects

### 14.1 先講角色分工（為什麼要拆）
- `Component`：收集輸入、觸發事件、顯示畫面，不直接打 API。
- `Service`：封裝 `HttpClient`，專職和後端溝通。
- `Effects`：接住 action 後呼叫 service，處理成功/失敗，再回傳 action。
- `Reducer/Selector`：把 effects 結果寫進 state，並提供畫面讀取。

> 重點：`Component + Service + Effects` 是主幹；若要完整資料流，還需要 `Action/Reducer/Selector`。

### 14.2 Order 功能最小檔案清單（NgRx + Effects）

| 檔案 | 用途 | 必要性 |
|---|---|---|
| `models`/order.models.ts | 定義 `Order`、`CreateOrderPayload`、`PaymentResult` 型別 | 必要 |
| `api`/order.service.ts | `HttpClient` 呼叫 `/api/orders` | 必要 |
| `store`/order.actions.ts | 定義 `submitOrder / success / failure` 事件 | 必要 |
| `store`/order.reducer.ts | 管理 `loading/error/order` 狀態變化 | 必要 |
| `store`/order.selectors.ts | 提供元件讀取 `loading/error/orderId` | 必要 |
| `store`/order.effects.ts | 呼叫 `OrderService`，分發成功失敗 action | 必要 |
| `pages`/checkout/checkout.component.ts | 觸發下單、顯示 loading/error | 必要 |
| shared/http/api.interceptor.ts | token、錯誤訊息、重試策略 | 建議 |
| app.config.ts | 註冊 provideStore/provideState/provideEffects/provideHttpClient | 必要 |
| environments/*.ts | 管理 API base URL | 建議 |
| **/*.spec.ts | service/effects/reducer 測試 | 建議 |

建議目錄（Order feature）：
```text
src/app/features/order/
  models/order.models.ts
  api/order.service.ts
  store/order.actions.ts
  store/order.reducer.ts
  store/order.selectors.ts
  store/order.effects.ts
  pages/checkout/checkout.component.ts
```

### 14.3 一筆「送出訂單」的完整時序

1. `CheckoutComponent` 觸發 `dispatch(OrderActions.submitOrder(payload))`
2. `OrderEffects` 監聽 `submitOrder`
3. `OrderEffects` 呼叫 `OrderService.createOrder(payload)`
4. API 成功：dispatch `submitOrderSuccess(orderId)`；失敗：dispatch `submitOrderFailure(error)`
5. `orderReducer` 更新 `loading/error/currentOrderId`
6. `CheckoutComponent` 透過 selector 讀取最新狀態並更新 UI

### 14.4 最小可教學骨架（Order）

```ts
// features/order/models/order.models.ts
export interface CreateOrderPayload {
  items: Array<{ itemId: string; qty: number }>;
}

export interface Order {
  id: string;
}
```

```ts
// features/order/store/order.actions.ts
import { createActionGroup, props } from '@ngrx/store';
import { CreateOrderPayload } from '../models/order.models';

export const OrderActions = createActionGroup({
  source: 'Order',
  events: {
    'Submit Order': props<{ payload: CreateOrderPayload }>(),
    'Submit Order Success': props<{ orderId: string }>(),
    'Submit Order Failure': props<{ error: string }>()
  }
});
```

```ts
// features/order/store/order.effects.ts
import { Injectable, inject } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { catchError, map, of, switchMap } from 'rxjs';
import { OrderActions } from './order.actions';
import { OrderService } from '../api/order.service';

@Injectable()
export class OrderEffects {
  private actions$ = inject(Actions);
  private orderService = inject(OrderService);

  submitOrder$ = createEffect(() =>
    this.actions$.pipe(
      ofType(OrderActions.submitOrder),
      switchMap(({ payload }) =>
        this.orderService.createOrder(payload).pipe(
          map((order) => OrderActions.submitOrderSuccess({ orderId: order.id })),
          catchError((e) => of(OrderActions.submitOrderFailure({ error: e.message ?? '下單失敗' })))
        )
      )
    )
  );
}
```

```ts
// features/order/api/order.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';
import { CreateOrderPayload, Order } from '../models/order.models';

@Injectable({ providedIn: 'root' })
export class OrderService {
  private http = inject(HttpClient);
  private baseUrl = '/api/orders';

  createOrder(payload: CreateOrderPayload): Observable<Order> {
    return this.http.post<Order>(this.baseUrl, payload);
  }
}
```

```ts
// features/order/pages/checkout/checkout.component.ts
import { Component, inject } from '@angular/core';
import { Store } from '@ngrx/store';
import { OrderActions } from '../../store/order.actions';
import { selectOrderLoading, selectOrderError } from '../../store/order.selectors';

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

### 14.5 如果你不用 NgRx（只有 Component + Service）
- 可行，但你需要自行處理：`loading`、`error`、流程追蹤、重複提交防護、測試隔離。
- 專案小時可以先這樣做；流程複雜後建議升級到 `Store + Effects`。
