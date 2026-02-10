# TypeScript 陣列與物件陣列資料處理講義（實戰版）

## 1. 這份講義的目標
- 只講「資料處理」：陣列、物件陣列。
- 不談非同步流程。
- 讀完可以自己寫出乾淨、可維護、型別清楚的資料轉換程式碼。

## 2. 先建立正確心法
- 一次只做一件事：先篩選，再轉換，再排序。
- 優先使用「不改原資料」的寫法（immutable）。
- 命名要表達意圖：`activeUsers`、`sortedOrders`、`groupedByStatus`。
- 優先讓編譯器幫你抓錯：輸入、輸出、回傳型別盡量寫清楚。

## 3. 範例資料

```ts
type OrderStatus = 'pending' | 'paid' | 'failed';

interface OrderItem {
  sku: string;
  name: string;
  qty: number;
  price: number;
}

interface Order {
  id: string;
  customer: string;
  status: OrderStatus;
  tableNo: number;
  createdAt: string;
  items: OrderItem[];
  tags: string[];
}

const orders: Order[] = [
  {
    id: 'o001',
    customer: 'Amy',
    status: 'paid',
    tableNo: 3,
    createdAt: '2026-02-10T10:00:00Z',
    items: [
      { sku: 'tea01', name: 'Black Tea', qty: 2, price: 45 },
      { sku: 'snk01', name: 'Fries', qty: 1, price: 80 }
    ],
    tags: ['dine-in']
  },
  {
    id: 'o002',
    customer: 'Bob',
    status: 'pending',
    tableNo: 8,
    createdAt: '2026-02-10T10:05:00Z',
    items: [{ sku: 'cof01', name: 'Latte', qty: 1, price: 120 }],
    tags: ['takeout', 'vip']
  },
  {
    id: 'o003',
    customer: 'Cathy',
    status: 'failed',
    tableNo: 5,
    createdAt: '2026-02-10T10:08:00Z',
    items: [{ sku: 'tea01', name: 'Black Tea', qty: 1, price: 45 }],
    tags: []
  }
];
```

## 4. 查詢類（找資料）

### 4.1 `filter`：找出多筆
```ts
const pendingOrders: Order[] = orders.filter((o) => o.status === 'pending');
```

### 4.2 `find`：找第一筆
```ts
const order002: Order | undefined = orders.find((o) => o.id === 'o002');
```

### 4.3 `some` / `every`：條件檢查
```ts
const hasFailed: boolean = orders.some((o) => o.status === 'failed');
const allHaveItems: boolean = orders.every((o) => o.items.length > 0);
```

## 5. 轉換類（改資料形狀）

### 5.1 `map`：一對一轉換
```ts
type OrderSummary = Pick<Order, 'id' | 'customer' | 'status'>;

const simple: OrderSummary[] = orders.map((o) => ({
  id: o.id,
  customer: o.customer,
  status: o.status
}));
```

### 5.2 `flatMap`：展開後再轉換
```ts
type FlatOrderItem = {
  orderId: string;
  sku: string;
  qty: number;
  amount: number;
};

const allItems: FlatOrderItem[] = orders.flatMap((o) =>
  o.items.map((i) => ({
    orderId: o.id,
    sku: i.sku,
    qty: i.qty,
    amount: i.qty * i.price
  }))
);
```

## 6. 聚合類（算總數、總額、統計）

### 6.1 `reduce`：總額
```ts
const totalRevenue: number = orders
  .filter((o) => o.status === 'paid')
  .reduce((sum, o) => {
    const orderAmount = o.items.reduce((s, i) => s + i.qty * i.price, 0);
    return sum + orderAmount;
  }, 0);
```

### 6.2 `reduce`：計數
```ts
const statusCount = orders.reduce<Record<OrderStatus, number>>(
  (acc, o) => {
    acc[o.status] += 1;
    return acc;
  },
  { pending: 0, paid: 0, failed: 0 }
);
```

## 7. 排序類

### 7.1 數字排序（桌號）
```ts
const byTable: Order[] = [...orders].sort((a, b) => a.tableNo - b.tableNo);
```

### 7.2 時間排序（最新在前）
```ts
const byLatest: Order[] = [...orders].sort(
  (a, b) => new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime()
);
```

### 7.3 多欄位排序（先狀態再時間）
```ts
const rank: Record<OrderStatus, number> = { pending: 1, paid: 2, failed: 3 };
const byStatusThenTime: Order[] = [...orders].sort((a, b) => {
  if (rank[a.status] !== rank[b.status]) return rank[a.status] - rank[b.status];
  return new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime();
});
```

## 8. 群組類（group by）

### 8.1 用 `reduce` 群組（相容性高）
```ts
const groupedByStatus = orders.reduce<Record<OrderStatus, Order[]>>(
  (acc, o) => {
    acc[o.status].push(o);
    return acc;
  },
  { pending: [], paid: [], failed: [] }
);
```

### 8.2 群組後立刻計算小計
```ts
const amountByStatus = orders.reduce<Record<OrderStatus, number>>(
  (acc, o) => {
    const amount = o.items.reduce((s, i) => s + i.qty * i.price, 0);
    acc[o.status] += amount;
    return acc;
  },
  { pending: 0, paid: 0, failed: 0 }
);
```

## 9. 去重類（dedupe）

### 9.1 基本型別去重
```ts
const tags: string[] = orders.flatMap((o) => o.tags);
const uniqueTags: string[] = [...new Set(tags)];
```

### 9.2 物件陣列依 key 去重
```ts
const uniqueById: Order[] = Array.from(
  new Map(orders.map((o) => [o.id, o])).values()
);
```

## 10. 物件陣列 CRUD（最常用）

### 10.1 新增一筆（Create）
```ts
const newOrder: Order = {
  id: 'o004',
  customer: 'Dora',
  status: 'pending',
  tableNo: 2,
  createdAt: '2026-02-10T10:15:00Z',
  items: [],
  tags: []
};
const created: Order[] = [...orders, newOrder];
```

### 10.2 更新一筆（Update）
```ts
const updated: Order[] = orders.map((o) =>
  o.id === 'o002' ? { ...o, status: 'paid' } : o
);
```

### 10.3 刪除一筆（Delete）
```ts
const removed: Order[] = orders.filter((o) => o.id !== 'o003');
```

## 11. 深層更新（巢狀資料）

```ts
const updateItemQty = (
  list: Order[],
  orderId: string,
  sku: string,
  qty: number
): Order[] =>
  list.map((o) =>
    o.id !== orderId
      ? o
      : {
          ...o,
          items: o.items.map((i) => (i.sku === sku ? { ...i, qty } : i))
        }
  );
```

## 12. 組合技：查詢 + 轉換 + 聚合

需求：只看 `paid` 訂單，列出每張訂單金額，最後依金額大到小排序。

```ts
type PaidOrderSummary = {
  id: string;
  customer: string;
  amount: number;
};

const paidOrderSummary: PaidOrderSummary[] = orders
  .filter((o) => o.status === 'paid')
  .map((o) => ({
    id: o.id,
    customer: o.customer,
    amount: o.items.reduce((sum, i) => sum + i.qty * i.price, 0)
  }))
  .sort((a, b) => b.amount - a.amount);
```

## 13. 常見陷阱（一定會踩）

### 13.1 `sort()` 會改原陣列
```ts
// 錯誤風險：會動到 orders 本身
orders.sort((a, b) => a.tableNo - b.tableNo);

// 建議：先複製再排
const safeSorted: Order[] = [...orders].sort((a, b) => a.tableNo - b.tableNo);
```

### 13.2 `map` 不會過濾資料
```ts
// 錯誤寫法：想過濾卻用 map
const wrong: Array<Order | null> = orders.map((o) =>
  o.status === 'paid' ? o : null
);

// 正確：先 filter 再 map
const right: string[] = orders
  .filter((o) => o.status === 'paid')
  .map((o) => o.id);
```

### 13.3 淺拷貝不是深拷貝
```ts
const clone = [...orders]; // 只複製第一層陣列
```

## 14. 可重用的工具函式（建議收進 utils）

```ts
export const sumBy = <T>(arr: T[], pick: (item: T) => number) =>
  arr.reduce((sum, item) => sum + pick(item), 0);

export const groupBy = <T, K extends string | number>(
  arr: T[],
  key: (item: T) => K
) =>
  arr.reduce<Record<K, T[]>>((acc, item) => {
    const k = key(item);
    (acc[k] ||= []).push(item);
    return acc;
  }, {} as Record<K, T[]>);

export const uniqueBy = <T, K extends PropertyKey>(
  arr: T[],
  key: (item: T) => K
) =>
  Array.from(new Map(arr.map((item) => [key(item), item])).values());
```

## 15. 實戰題（課堂可直接用）

1. 只保留 `pending` 與 `paid` 訂單，並依時間新到舊排序。
2. 計算每張訂單總額，回傳 `{ id, customer, amount }[]`。
3. 找出購買過 `tea01` 的客人名單（去重）。
4. 統計每個 `status` 的訂單數量與總金額。
5. 更新某張訂單某個 item 的數量，保持不改原資料。

## 16. 一頁速記
- 找多筆：`filter`
- 找一筆：`find`
- 轉形狀：`map`
- 展平：`flatMap`
- 聚合：`reduce`
- 排序：`sort`（先複製）
- 去重：`Set` / `Map`
- 新增更新刪除：`[...]` / `map` / `filter`
