# Angular 初學入門 30 堂課課綱規劃（餐飲業者點餐機）

## 1. 課程總覽（目標、對象、工具、評量方式）

### 目標
- 以 `30` 堂課、每堂 `75 分鐘`，帶初學者完成一個可展示的 Angular「餐飲點餐機」前端專題。
- 教學設定：`基礎 + 小專題`、`僅課堂實作`、學員已有其他框架經驗。
- 主流程目標：可完成「選餐 -> 加入購物車 -> 結帳 -> 模擬付款 -> 訂單結果」。

### 對象
- 已有基本程式能力，且有其他框架經驗的初學者。

### 工具
- Angular CLI、TypeScript、RxJS、Angular Router、Reactive Forms、HttpClient。
- 測試工具：Karma/Jasmine（單元測試）與 E2E 測試工具（流程測試）。
- 版本控制與部署：Git、production build、靜態主機部署。

### 評量方式
- 以課堂實作驗收為主，不安排課後作業。
- 每 5 堂課一個 checkpoint，共 6 個里程碑。
- 後段加入單元測試與端到端主流程測試，作為品質門檻。

## 2. 每堂課固定時間模板（75 分鐘）

- `10 分鐘`：複習與暖身（上堂重點、錯誤修正）
- `20 分鐘`：新知講解（本堂核心概念）
- `35 分鐘`：課堂實作（跟做 + 小段落驗收）
- `10 分鐘`：驗收與 Q&A（成果展示、問題收斂）

> 所有堂次皆使用同一時間模板：`10/20/35/10`，總長 `75 分鐘`。

## 3. 30 堂課明細表（堂次、主題、重點、課堂產出）

| 堂次 | 時間 | 主題 | 重點 | 課堂實作產出 |
|---|---|---|---|---|
| 01 | 75 分鐘 | 課程啟動與 Angular 架構 | 專題需求拆解、CLI、專案初始化 | 建立專案骨架與首頁 |
| 02 | 75 分鐘 | TypeScript 快速補齊 | 型別、介面、泛型、嚴格模式 | 建立點餐領域型別檔 |
| 03 | 75 分鐘 | 元件與模板基礎 | interpolation、binding、事件 | Menu 清單元件初版 |
| 04 | 75 分鐘 | 控制流程與清單渲染 | `@if`、`@for`、條件顯示 | 分類切換與商品列表 |
| 05 | 75 分鐘 | 元件溝通 | Input/Output、事件流 | 商品卡與數量選擇器串接 |
| 06 | 75 分鐘 | Kiosk UI 版型 | 觸控友善設計、RWD | 點餐機主畫面排版完成 |
| 07 | 75 分鐘 | Reactive Forms 入門 | FormGroup/FormControl | 客製選項表單 |
| 08 | 75 分鐘 | 表單驗證 | 同步驗證、錯誤訊息 UX | 必填/數量限制驗證 |
| 09 | 75 分鐘 | 共用顯示邏輯 | Pipe、Directive 基礎 | 價格格式 Pipe、高亮指令 |
| 10 | 75 分鐘 | 路由基礎 | 頁面切換、route 結構 | Menu/Cart/Checkout 三頁路由 |
| 11 | 75 分鐘 | Service 與 DI | 資料存取層、可測性 | MenuService/CartService |
| 12 | 75 分鐘 | HttpClient 與假資料 | 非同步資料流、mock API | 菜單改由 API 載入 |
| 13 | 75 分鐘 | RxJS 入門 | Observable、常用 operators | 搜尋/分類過濾串流 |
| 14 | 75 分鐘 | 購物車狀態管理 | 單向資料流、即時計算 | Header 購物車 badge 與總額 |
| 15 | 75 分鐘 | 本地持久化 | localStorage、還原流程 | 重整後保留購物車 |
| 16 | 75 分鐘 | 訂單領域建模 | 訂單、稅金、服務費規則 | Order model 與計價模組 |
| 17 | 75 分鐘 | 促銷與商業規則 | 套餐、折扣條件 | PromotionService 初版 |
| 18 | 75 分鐘 | 結帳流程 | 多步驟流程、資料整合 | 結帳頁完成（內用/外帶） |
| 19 | 75 分鐘 | 模擬付款 | 狀態機、成功/失敗處理 | 假付款流程與結果畫面 |
| 20 | 75 分鐘 | 訂單確認與憑證 | 訂單摘要、列印視圖 | 完整訂單確認頁 |
| 21 | 75 分鐘 | 例外處理 | interceptor、全域錯誤 UX | API 錯誤統一提示 |
| 22 | 75 分鐘 | 路由守衛 | 模式切換（點餐/管理） | 管理模式 PIN 守衛 |
| 23 | 75 分鐘 | 管理端 CRUD（清單） | 菜單維護列表 | 後台商品清單頁 |
| 24 | 75 分鐘 | 管理端 CRUD（表單） | 新增/編輯/刪除流程 | 商品維護表單完成 |
| 25 | 75 分鐘 | 可用性與無障礙 | 大按鈕、焦點、ARIA | 點餐流程可鍵盤操作 |
| 26 | 75 分鐘 | 單元測試 | Component/Service 測試 | Cart/計價核心測試 |
| 27 | 75 分鐘 | 流程測試 | 端到端主流程驗證 | 點餐到成立訂單測試腳本 |
| 28 | 75 分鐘 | 效能優化 | OnPush、trackBy、lazy load | 首屏與列表效能改善 |
| 29 | 75 分鐘 | 打包與部署 | build config、環境變數 | 可部署的 production build |
| 30 | 75 分鐘 | 期末展示與總驗收 | Demo、回顧、補強地圖 | 可展示版點餐機專題 |

## 4. 里程碑與驗收點（每 5 堂一個 checkpoint）

1. `L05`：可瀏覽菜單並選擇品項。
2. `L10`：三大頁面可路由切換。
3. `L15`：購物車可完整操作且可持久化。
4. `L20`：可完成下單與付款模擬。
5. `L25`：具管理端與基本可用性品質。
6. `L30`：完成可部署、可演示版本。

## 5. 評量規則（課堂驗收為主）

- 結構驗收：`plan.md` 必須有且僅有 `30` 堂課。
- 時間驗收：每堂均標示 `75 分鐘`，並採固定模板 `10/20/35/10`。
- 主流程驗收：可完成「選餐 -> 加入購物車 -> 結帳 -> 模擬付款 -> 訂單結果」。
- 邊界情境：缺貨商品、付款失敗、網路錯誤、無效輸入。
- 品質驗收：至少具單元測試與一條端到端主流程測試。
- 發布驗收：可輸出 production build 並可在靜態主機部署。

## 6. 風險與補救策略

### 風險 1：學員 Angular 熟悉度落差
- 補救：前 5 堂維持高密度小迭代，每 10-15 分鐘做一次小驗收。

### 風險 2：RxJS 與狀態流概念不易吸收
- 補救：先用服務層狀態管理，再引入 operators；以購物車場景反覆演練。

### 風險 3：專題範圍膨脹導致進度失控
- 補救：嚴格以里程碑管控，優先完成點餐主流程，管理端功能保持入門 CRUD。

### 風險 4：測試與效能章節被擠壓
- 補救：在 L26-L28 鎖定為不可刪章節，若前段延誤以縮減 UI 細節換取品質內容。

### 風險 5：後端契約不穩定
- 補救：先以 mock API 教學，並將資料契約抽象為型別與 service 介面，降低替換成本。

## 重要介面 / 型別規劃（Public APIs / Interfaces / Types）

### 前端核心型別
- `MenuCategory { id, name, sort }`
- `MenuItem { id, categoryId, name, price, options, isAvailable }`
- `OptionGroup { id, name, required, maxSelect, options[] }`
- `CartItem { itemId, qty, selectedOptions, subtotal }`
- `Order { id, items, amount, tax, serviceFee, total, status }`
- `PaymentResult { orderId, status, transactionId, paidAt }`

### 前端服務介面
- `MenuService`: `getCategories()`, `getItems()`, `saveItem()`, `deleteItem()`
- `CartService`: `addItem()`, `updateQty()`, `removeItem()`, `clear()`, `getSummary()`
- `OrderService`: `createOrder()`, `payOrder()`, `getOrderResult()`

### 路由介面
- `/menu`
- `/cart`
- `/checkout`
- `/result/:orderId`
- `/admin/*`

## 明確假設與預設

- 學員已具基本程式能力與其他框架經驗，不需大量基礎程式補課。
- 不安排課後作業，學習成果以課內實作與里程碑驗收為主。
- 後端以 mock API 起步，後段再對接真實 API 契約（若有）。
- 優先完成前台點餐核心流程，管理端僅做入門 CRUD 範圍。
- 每堂課時間固定 75 分鐘，避免單堂過長造成吸收落差。
