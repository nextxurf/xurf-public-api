# XURF 公開 API 串接規格書

## 文件資訊

| 項目 | 內容 |
|---|---|
| 文件名稱 | XURF 公開 API 串接規格書 |
| 文件版號 | v1.0 |
| 文件日期 | 2026-07-23 |
| 正式環境 Base URL | `https://goapi2.xurf.ai` |
| 資料格式 | JSON |
| 編碼 | UTF-8 |

### 文件目的

本文件整合下列資料來源中列出的全部 API，統一說明 Endpoint、認證方式、Request、Response、cURL 與 JavaScript 呼叫範例：

- `AppVIP-AppMember-POS.pdf`
- `Xurf 雲悦串接規格書 1.3.pdf`

### 重要限制

1. 語音點餐廠商 API 的 HMAC 憑證、Nonce 防重放、來源 IP/CIDR 白名單及權限驗證，是正式對外開放前必須完成的驗證層。現有正式路由尚未完成此專用驗證層，完成後才可交付廠商憑證。
2. `add_pos_order` 的來源 PDF 未定義 Base URL、授權 Header、Response Schema 與錯誤碼。本文件只整理來源文件已明確定義的契約；正式串接前須向 XURF 確認缺少的項目。
3. 文件中的 Token、Client ID、簽章、企業與門市代號皆為範例或占位值，不可直接用於正式環境。

### 版號歷程

| 版號 | 日期 | 異動說明 |
|---|---|---|
| v1.0 | 2026-07-23 | 建立公開 API 文件，包含 POS 銷售訂單上傳及語音點餐廠商 API。 |

---

## 1. API 一覽

| 類別 | 功能 | Method | Endpoint | 認證 |
|---|---|:---:|---|---|
| POS 上傳 | 上傳 POS 訂單 | POST | `/v1/services/epos/add_pos_order` | 來源 PDF 未定義 |
| 語音點餐廠商 | 取得外帶菜單 | GET | `/MenuFood/GetFoodList` | HMAC + IP 白名單 |
| 語音點餐廠商 | 取得門市接單狀態 | GET | `/Shop/GetShopInfo` | HMAC + IP 白名單 |
| 語音點餐廠商 | 建立線上訂單 | POST | `/Order/CalculateOrderPreview` | HMAC + IP 白名單 |

---

## 2. 共用規範

### 2.1 Base URL

```text
https://goapi2.xurf.ai
```

`add_pos_order` 的來源 PDF 只提供相對路徑，未明確指定 Base URL。以下範例暫以本文件正式環境 Base URL 組合，正式使用前須由 XURF 確認。

### 2.2 資料格式

- Request 與 Response 使用 UTF-8。
- JSON Request Body 應帶入 `Content-Type: application/json`。
- 建議所有呼叫帶入 `Accept: application/json`。
- Query String 必須進行 URL encoding。

### 2.3 日期與時間

| 用途 | 格式 | 範例 |
|---|---|---|
| 一般日期 | `YYYY-MM-DD` | `2026-07-20` |
| AppMember 日期 | 來源範例為 `YYYYMMDD` | `20251114` |
| 線上點餐時間 | ISO 8601，包含時區 | `2026-06-30T12:30:00+08:00` |
| HMAC Timestamp | Unix timestamp seconds | `1780790400` |

---

## 3. 認證方式

### 3.1 語音點餐廠商 HMAC

#### 開通資料

廠商須提供：

- 廠商名稱及系統識別碼。
- 測試與正式環境的固定來源公網 IP 或 CIDR。
- 技術及資安聯絡人。
- 需要呼叫的企業代號、門市代號及 API 權限。

XURF 審核後提供：

- `Client-Id`：廠商公開識別碼。
- `Client-Secret`：私密金鑰，只在建立或重設時提供一次。
- 測試與正式環境 Base URL。

`Client-Secret` 不得出現在 URL、Query String、Request Body、瀏覽器或 App 前端。

#### 必要 Headers

```http
X-Client-Id: <client_id>
X-Timestamp: <unix_timestamp_seconds>
X-Nonce: <uuid>
X-Signature: <lowercase_hex_hmac_sha256>
Accept: application/json
```

POST JSON 另須帶入：

```http
Content-Type: application/json
```

廠商可選擇帶入自身版本：

```http
X-Client-Version: voice-order-1.0.0
```

#### 簽章原文

```text
HTTP_METHOD
REQUEST_PATH
CANONICAL_QUERY_STRING
X_TIMESTAMP
X_NONCE
SHA256_HEX(REQUEST_BODY)
```

各欄以換行字元 `\n` 串接，再以 `Client-Secret` 計算 HMAC-SHA256，輸出小寫十六進位字串。

- GET 沒有 Body 時，`REQUEST_BODY` 使用空字串。
- Query String 須按參數名稱排序，並使用 RFC 3986 URL encoding。
- `X-Timestamp` 與伺服器時間差必須在 5 分鐘內。
- `X-Nonce` 不可重複使用。
- POST 簽章使用的 JSON 字串必須與實際傳送的 Body 完全相同。

#### 授權檢查順序

1. `Client-Id` 存在且啟用。
2. 廠商已設定 `Client-Secret`。
3. 來源公網 IP 符合 IP/CIDR 白名單。
4. Timestamp 在允許時間內。
5. Nonce 未使用過。
6. HMAC-SHA256 簽章相符。
7. 廠商具有該 API、企業及門市權限。

若經過 Cloudflare、Load Balancer 或 Reverse Proxy，只能採用受信任代理所寫入的真實來源 IP Header，不得以 `Host`、`Origin`、`Referer` 或反向 DNS 代替來源 IP 驗證。

#### 授權錯誤

| HTTP Status | code | 說明 |
|---:|---|---|
| 401 | `INVALID_CLIENT` | Client ID 不存在、停用或未設定私密金鑰。 |
| 401 | `INVALID_SIGNATURE` | 簽章不符。 |
| 401 | `REQUEST_EXPIRED` | Timestamp 超過允許時間。 |
| 401 | `REPLAY_DETECTED` | Nonce 已使用。 |
| 403 | `SOURCE_NOT_ALLOWED` | 來源 IP 不在白名單。 |
| 403 | `SCOPE_DENIED` | 無 API、企業或門市權限。 |

```json
{
  "code": "SOURCE_NOT_ALLOWED",
  "message": "Request source is not allowed."
}
```

### 3.2 HMAC Node.js 共用函式

```javascript
import crypto from 'node:crypto';

const baseUrl = 'https://goapi2.xurf.ai';
const clientId = process.env.XURF_CLIENT_ID;
const clientSecret = process.env.XURF_CLIENT_SECRET;

function encodeRFC3986(value) {
  return encodeURIComponent(value).replace(/[!'()*]/g, (char) =>
    `%${char.charCodeAt(0).toString(16).toUpperCase()}`,
  );
}

function canonicalQuery(params) {
  return Object.entries(params)
    .filter(([, value]) => value !== undefined && value !== null)
    .sort(([a], [b]) => a.localeCompare(b))
    .map(([key, value]) => `${encodeRFC3986(key)}=${encodeRFC3986(String(value))}`)
    .join('&');
}

function sha256Hex(value) {
  return crypto.createHash('sha256').update(value, 'utf8').digest('hex');
}

function createVendorHeaders(method, path, query = '', body = '') {
  if (!clientId || !clientSecret) {
    throw new Error('請設定 XURF_CLIENT_ID 與 XURF_CLIENT_SECRET');
  }

  const timestamp = Math.floor(Date.now() / 1000).toString();
  const nonce = crypto.randomUUID();
  const signingText = [
    method.toUpperCase(),
    path,
    query,
    timestamp,
    nonce,
    sha256Hex(body),
  ].join('\n');

  const signature = crypto
    .createHmac('sha256', clientSecret)
    .update(signingText, 'utf8')
    .digest('hex');

  return {
    'X-Client-Id': clientId,
    'X-Timestamp': timestamp,
    'X-Nonce': nonce,
    'X-Signature': signature,
    'X-Client-Version': 'voice-order-1.0.0',
    Accept: 'application/json',
  };
}
```

---

## 4. POS 銷售訂單上傳

### 4.1 Request

```http
POST /v1/services/epos/add_pos_order
```

Kiosk 或 POS 完成訂單後，將 BearPOS 格式的 `order_info` 傳送至雲端。Request 型別為 `vo.AddPOSOrderRequest`。

| 欄位 | 說明 |
|---|---|
| `account` | POS 授權人員帳號。 |
| `company_id` | 企業總部 ID。 |
| `machine_code` | 裝置授權碼。 |
| `order_info` | 訂單 JSON 字串。 |
| `pos_no` | POS 機代號。 |
| `pstore_id` | 門市或分店編碼。 |

```json
{
  "account": "your_account",
  "company_id": "your_company_id",
  "machine_code": "your_machine_code",
  "order_info": "{\"orderId\":\"20250120-0001\",\"items\":[]}",
  "pos_no": "A",
  "pstore_id": "STORE001"
}
```

### 4.2 cURL

來源 PDF 未定義授權方式；以下只帶入已知的媒體類型 Header。

```bash
curl 'https://goapi2.xurf.ai/v1/services/epos/add_pos_order' \
  -X POST \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  --data '{
    "account": "your_account",
    "company_id": "your_company_id",
    "machine_code": "your_machine_code",
    "order_info": "{\"orderId\":\"20250120-0001\",\"items\":[]}",
    "pos_no": "A",
    "pstore_id": "STORE001"
  }'
```

### 4.3 JavaScript

```javascript
const response = await fetch(
  'https://goapi2.xurf.ai/v1/services/epos/add_pos_order',
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Accept: 'application/json',
    },
    body: JSON.stringify({
      account: 'your_account',
      company_id: 'your_company_id',
      machine_code: 'your_machine_code',
      order_info: JSON.stringify({
        orderId: '20250120-0001',
        items: [],
      }),
      pos_no: 'A',
      pstore_id: 'STORE001',
    }),
  },
);

if (!response.ok) {
  throw new Error(`HTTP ${response.status}: ${await response.text()}`);
}

console.log(await response.json());
```

### 4.4 Response

來源 PDF 未提供 Response Schema、成功範例與錯誤碼，正式串接前須向 XURF 確認。

---

## 5. GetFoodList 外帶菜單

### 5.1 Request

```http
GET /MenuFood/GetFoodList
```

取得指定門市的外帶菜單、商品 ID、名稱、分類、售價、套餐及加料選項。

| 參數 | 必填 | 型別 | 說明 |
|---|:---:|---|---|
| `enterpriseId` | 是 | string | 企業代號。 |
| `shopId` | 是 | string | 門市代號。 |
| `orderType` | 是 | string | 外帶固定為 `takeout`。 |
| `langId` | 否 | string | 語系，預設 `TW`。 |
| `foodId` | 否 | string | 只查指定商品。 |
| `mouldCode` | 否 | string | 指定菜單模板；未傳時由系統取得有效模板。 |
| `showHidenKind` | 否 | boolean | 是否包含隱藏分類，廠商固定使用 `false`。 |

### 5.2 使用規則

- Response 為 JSON Array。
- 空陣列 `[]` 可能表示無可用菜單、參數錯誤或查詢失敗，不得視為正常可下單狀態。
- 整份菜單中的套餐可能不展開 `comboList` 與 `productDetailList`；組合套餐前應再帶入 `foodId` 查詢完整明細。
- `isSoldOut=true`、`isHidden=true` 或 `stop=true` 的商品不可下單。
- 商品 ID、名稱、分類與價格必須使用同一次 API 回傳資料，不得由廠商自行維護售價。

### 5.3 Response 欄位

#### 主商品

| 欄位 | 型別 | 說明 | 下單對應 |
|---|---|---|---|
| `foodCategoryId` | string | 商品分類 ID。 | `Items[].KindID` |
| `foodCategoryName` | string | 商品分類名稱。 | 顯示用 |
| `foodId` | string | 商品 ID。 | `Items[].FoodID` |
| `foodName` | string | 商品名稱。 | `Items[].FoodName` |
| `originFoodName` | string | 原始商品名稱。 | `Items[].FoodName` 或 `OriginFoodName` |
| `price` | number | 商品單價。 | `Items[].Price` |
| `isCombo` | boolean | 是否為套餐商品。 | 判斷是否需帶 `ComboItems` |
| `isSoldOut` | boolean | 是否售完。 | `true` 不可下單 |
| `isHidden` | boolean | 是否隱藏。 | `true` 不可下單 |
| `stop` | boolean | 是否停售。 | `true` 不可下單 |
| `comboList` | array | 套餐附餐群組。 | `Items[].ComboItems` |
| `productDetailList` | array | 主商品選項或加料群組。 | `Items[].OptionItems` |

#### `comboList[]`

| 欄位 | 型別 | 說明 | 下單對應 |
|---|---|---|---|
| `parentId` | string | 所屬主商品 ID。 | 檢核用 |
| `itemId` | string | 附餐群組 ID。 | `ComboItems[].ComboID` |
| `itemName` | string | 附餐群組名稱。 | `ComboItems[].ItemName` |
| `minSelectCount` | integer | 最少必選數量。 | 送單前檢核 |
| `maxSelectCount` | integer | 最多可選數量。 | 送單前檢核 |
| `group` | integer | 排序或群組序號。 | 顯示及排序用 |
| `foodItems` | array | 可選附餐商品。 | `ComboItems[].Items` |

#### `comboList[].foodItems[]`

| 欄位 | 型別 | 說明 | 下單對應 |
|---|---|---|---|
| `foodCategoryId` | string / null | 附餐分類 ID。 | 通常不送出 |
| `foodCategoryName` | string / null | 附餐分類名稱。 | 顯示用 |
| `foodId` | string | 附餐商品 ID。 | `ComboItems[].Items[].FoodID` |
| `foodName` | string | 附餐商品名稱。 | `ComboItems[].Items[].FoodName` |
| `originFoodName` | string | 附餐原始名稱。 | `OriginFoodName` |
| `price` | number | 附餐加價金額。 | `AdditionalPrice` |
| `isSoldOut` | boolean | 是否售完。 | `true` 不可選 |
| `isHidden` | boolean | 是否隱藏。 | `true` 不可選 |
| `stop` | boolean | 是否停售。 | `true` 不可選 |
| `productDetailList` | array | 附餐選項或加料群組。 | 附餐的 `OptionItems` |

#### `productDetailList[]`

| 欄位 | 型別 | 說明 |
|---|---|---|
| `parentId` | string | 所屬主商品或附餐商品 ID。 |
| `itemId` | string | 選項群組 ID。 |
| `itemName` | string | 選項群組名稱。 |
| `minSelectCount` | integer | 最少必選數量。 |
| `maxSelectCount` | integer | 最多可選數量。 |
| `items` | array | 可選品項。 |

`productDetailList[].items[]` 包含 `foodId`、`foodName`、`originFoodName`、`price`、`isSoldOut`、`isHidden`、`stop`，下單時分別對應選項的 `FoodID`、`FoodName`、`OriginFoodName`、`AdditionalPrice`。

### 5.4 Response 範例

```json
[
  {
    "foodCategoryId": "CAT001",
    "foodCategoryName": "套餐分類範例",
    "foodId": "FOOD001",
    "foodName": "套餐商品範例",
    "originFoodName": "套餐商品範例",
    "price": 0,
    "isCombo": true,
    "isSoldOut": false,
    "isHidden": false,
    "stop": false,
    "comboList": [
      {
        "parentId": "FOOD001",
        "itemId": "COMBO001",
        "itemName": "附餐群組範例A",
        "minSelectCount": 1,
        "maxSelectCount": 1,
        "group": 1,
        "foodItems": [
          {
            "foodId": "FOOD002",
            "foodName": "附餐商品範例1",
            "originFoodName": "附餐商品範例1",
            "price": 115,
            "isCombo": false,
            "isSoldOut": false,
            "isHidden": false,
            "stop": false,
            "productDetailList": []
          }
        ]
      }
    ],
    "productDetailList": []
  }
]
```

### 5.5 cURL

```bash
curl -G 'https://goapi2.xurf.ai/MenuFood/GetFoodList' \
  -H 'X-Client-Id: <client_id>' \
  -H 'X-Timestamp: <unix_timestamp_seconds>' \
  -H 'X-Nonce: <uuid>' \
  -H 'X-Signature: <calculated_hmac_sha256>' \
  -H 'Accept: application/json' \
  --data-urlencode 'enterpriseId=your_company_id' \
  --data-urlencode 'foodId=FOOD001' \
  --data-urlencode 'langId=TW' \
  --data-urlencode 'orderType=takeout' \
  --data-urlencode 'shopId=SHOP001'
```

簽章時的 canonical query 必須是：

```text
enterpriseId=your_company_id&foodId=FOOD001&langId=TW&orderType=takeout&shopId=SHOP001
```

### 5.6 JavaScript

使用第 3.2 節的共用函式：

```javascript
const path = '/MenuFood/GetFoodList';
const query = canonicalQuery({
  enterpriseId: 'your_company_id',
  foodId: 'FOOD001',
  langId: 'TW',
  orderType: 'takeout',
  shopId: 'SHOP001',
});

const response = await fetch(`${baseUrl}${path}?${query}`, {
  headers: createVendorHeaders('GET', path, query),
});

if (!response.ok) {
  throw new Error(`HTTP ${response.status}: ${await response.text()}`);
}

const foods = await response.json();
console.log(foods);
```

---

## 6. 門市接單狀態

### 6.1 Request

```http
GET /Shop/GetShopInfo
```

取得後台「數位點餐設定 > 外帶管理 > 接受訂單」狀態。POS 接單開關使用同一門市設定。

| 參數 | 必填 | 型別 | 說明 |
|---|:---:|---|---|
| `enterpriseId` | 是 | string | 企業代號。 |
| `shopId` | 是 | string | 門市代號。 |

既有前端可能傳送 `version`，但三支語音點餐 API 的核心邏輯不依賴該欄位，廠商不需傳送。若需標示自身版本，使用 `X-Client-Version` Header。

### 6.2 Response

API 回傳的 `Result` 是 JSON 字串，必須再解析一次，取得 `takeoutSettings`：

```json
{
  "Result": "{\"shopInfo\":{\"ShopID\":\"SHOP001\",\"ShopName\":\"示範門市\"},\"takeoutSettings\":[{\"key\":\"IsRejectOrder\",\"value\":\"false\"}]}"
}
```

| `IsRejectOrder` | 意義 | 是否可下單 |
|---|---|:---:|
| `"false"` 或 `false` | 接受訂單 | 是 |
| `"true"` 或 `true` | 暫停接單 | 否 |
| 缺少、`null` 或無法解析 | 狀態不明 | 否 |

欄位名稱 `IsRejectOrder` 與畫面「接受訂單」語意相反。呼叫端必須採 fail-closed，狀態不明時不可送單。建議每次下單前即時查詢；若需快取，最長 30 秒。

### 6.3 cURL

```bash
curl -G 'https://goapi2.xurf.ai/Shop/GetShopInfo' \
  -H 'X-Client-Id: <client_id>' \
  -H 'X-Timestamp: <unix_timestamp_seconds>' \
  -H 'X-Nonce: <uuid>' \
  -H 'X-Signature: <calculated_hmac_sha256>' \
  -H 'X-Client-Version: voice-order-1.0.0' \
  -H 'Accept: application/json' \
  --data-urlencode 'enterpriseId=ENTERPRISE001' \
  --data-urlencode 'shopId=SHOP001'
```

### 6.4 JavaScript

```javascript
const path = '/Shop/GetShopInfo';
const query = canonicalQuery({
  enterpriseId: 'ENTERPRISE001',
  shopId: 'SHOP001',
});

const response = await fetch(`${baseUrl}${path}?${query}`, {
  headers: createVendorHeaders('GET', path, query),
});

if (!response.ok) {
  throw new Error(`HTTP ${response.status}: ${await response.text()}`);
}

const outer = await response.json();
const result = JSON.parse(outer.Result);
const rejectSetting = result.takeoutSettings?.find(
  ({ key }) => key === 'IsRejectOrder',
);
const acceptsOrders =
  rejectSetting?.value === false || rejectSetting?.value === 'false';

if (!acceptsOrders) {
  throw new Error('門市暫停接單或接單狀態不明');
}
```

---

## 7. 線上點餐下單

### 7.1 Request

```http
POST /Order/CalculateOrderPreview
Content-Type: application/json
```

建立外帶線上訂單。API 會驗證商品並儲存訂單；店內付款時會通知 POS。

### 7.2 必要規則

- `Order.OrderType` 固定為 `2`。
- `Order.SaleType` 固定為 `"takeout"`。
- `Order.BillType` 固定為 `"website"`。
- `Order.ReceiverMark` 固定為 `"語音點餐"`。
- `Order.ID` 與每筆 `Items[].ID` 使用 UUID。
- 同一訂單重送時必須沿用原本的 `Order.ID`，避免重複下單。
- 商品 ID、名稱與單價必須來自同一門市最新的 `GetFoodList` 結果。
- 下單前必須確認門市接受訂單。
- 時間使用 ISO 8601。

### 7.3 主要欄位

| 欄位 | 必填 | 型別 | 說明 |
|---|:---:|---|---|
| `enterpriseId` | 是 | string | 企業代號。 |
| `shopId` | 是 | string | 門市代號。 |
| `Order.ID` | 是 | UUID | 訂單唯一識別碼及重送冪等鍵。 |
| `Order.SaleType` | 是 | string | 固定 `takeout`。 |
| `Order.ReceiverMark` | 是 | string | 固定 `語音點餐`。 |
| `Order.PayChannel` | 是 | string | 初期建議 `InstorePay`。 |
| `Order.ReceiverName` | 是 | string | 取餐人姓名。 |
| `Order.VIPTel` | 是 | string | 聯絡電話。 |
| `Order.TakeWayTime` | 是 | datetime | 預計取餐時間。 |
| `Order.Total` | 是 | number | 商品總額。 |
| `Order.PayTotal` | 是 | integer | 應付總額。 |
| `Items` | 是 | array | 至少一筆商品。 |
| `Items[].FoodID` | 是 | string | 菜單回傳的商品 ID。 |
| `Items[].Count` | 是 | number | 數量，必須大於 0。 |
| `Items[].Price` | 是 | number | 商品單價。 |
| `Items[].AddCost` | 是 | number | 主商品選項加價合計。 |
| `Items[].Total` | 是 | number | 單筆商品含所有加價的總額。 |
| `Items[].Add` | 否 | string | 主商品加料文字。 |
| `Items[].Taste` | 否 | string | 主商品備註或口味。 |
| `Items[].OptionItems` | 否 | array | 主商品選項；無選項時傳 `[]`。 |
| `Items[].ComboItems` | 否 | array | 套餐附餐群組；非套餐時傳 `[]`。 |
| `ComboItems[].ComboID` | 是 | string | 對應 `comboList[].itemId`。 |
| `ComboItems[].ItemName` | 是 | string | 對應 `comboList[].itemName`。 |
| `ComboItems[].Items` | 是 | array | 實際選取的附餐。 |
| `ComboItems[].Items[].FoodID` | 是 | string | 附餐商品 ID。 |
| `ComboItems[].Items[].Quantity` | 是 | integer | 附餐數量。 |
| `ComboItems[].Items[].AdditionalPrice` | 是 | number | 附餐加價。 |
| `ComboItems[].Items[].OptionItems` | 否 | array | 附餐選項。 |
| `OptionItems[].FoodID` | 是 | string | 選項或加料 ID。 |
| `OptionItems[].Quantity` | 是 | integer | 選項數量。 |
| `OptionItems[].AdditionalPrice` | 是 | number | 選項加價。 |

### 7.4 套餐與金額規則

- 主商品選項來自 `productDetailList[].items[]`，放入 `Items[].OptionItems[]`。
- 套餐附餐來自 `comboList[]`，每個群組放入一筆 `Items[].ComboItems[]`。
- 附餐自身的選項放入 `ComboItems[].Items[].OptionItems[]`。
- `FoodID`、`FoodName`、`OriginFoodName`、`AdditionalPrice` 必須使用同一次菜單查詢結果。
- 沒有套餐或選項時，`ComboItems`、`OptionItems` 傳空陣列，不要省略。
- `Items[].AddCost` 只計主商品自身選項加價；附餐及附餐選項加價另計入 `Items[].Total`、`Order.Total` 與 `Order.PayTotal`。
- 最終應付金額以 API 回傳的 `data.payTotal` 為準。

### 7.5 Request Body 範例

```json
{
  "enterpriseId": "your_company_id",
  "shopId": "SHOP001",
  "memberNo": "",
  "memberName": "王小明",
  "currentOrderId": null,
  "mouldCode": [],
  "expire": "",
  "Order": {
    "ID": "16a0dc4a-cb47-4da7-b223-bc2ff954c91f",
    "OrderNo": "",
    "OrderNo2": "",
    "Machine": "",
    "OrderType": 2,
    "DeskID": "",
    "SaleType": "takeout",
    "BillType": "website",
    "Count": 1,
    "OrderFoodCount": 1,
    "ReceiverMark": "語音點餐",
    "AppFunGID": "7e273165-d445-419b-a66e-aabf64c9a320",
    "Total": 155,
    "PayTotal": 155,
    "ServiceTotal": 0,
    "ServicePercent": 0,
    "AgioDiscount": 0,
    "AgioCost": 0,
    "AgioPercent": 0,
    "InvoiceTotal": 0,
    "TendTotal": 0,
    "PayStatus": 0,
    "PayChannel": "InstorePay",
    "ReceiverName": "王小明",
    "VIPTel": "0912345678",
    "VipAddr": "",
    "TakeWayTime": "2026-06-30T12:30:00+08:00",
    "TakeWayTime2": "2026-06-30T12:30:00+08:00",
    "SaleTime": "2026-06-30T12:00:00+08:00",
    "MeTime": "20260630120000",
    "VIPNo": "",
    "CardID": "",
    "VIPName": "王小明",
    "VendNo": null,
    "CarrierId": null,
    "DonateCode": null,
    "Man": 0,
    "Woman": 0,
    "Child": 0,
    "Baby": 0,
    "PrintStatus": 0,
    "OrderFlag": 0
  },
  "Items": [
    {
      "ID": "dd3c59d1-68aa-4560-bcaf-e2bbad9a1002",
      "FoodID": "FOOD001",
      "FoodName": "套餐商品範例",
      "KindID": "CAT001",
      "Count": 1,
      "Price": 0,
      "AddCost": 0,
      "Total": 155,
      "InputTime": "2026-06-30T12:00:00+08:00",
      "SaleTime": "2026-06-30T12:00:00+08:00",
      "SHOPID": "SHOP001",
      "OrderIndex": 1,
      "ComboItems": [
        {
          "ComboID": "COMBO001",
          "FoodID": "FOOD001",
          "ItemName": "附餐群組範例A",
          "FoodName": "套餐商品範例",
          "Items": [
            {
              "FoodID": "FOOD002",
              "FoodName": "附餐商品範例1",
              "OriginFoodName": "附餐商品範例1",
              "ImagePath": "",
              "Quantity": 1,
              "AdditionalPrice": 115,
              "OptionItems": [
                {
                  "FoodID": "6e8ed17c-38f2-4aa8-ba55-155f3c54da11",
                  "FoodName": "加購選項範例1",
                  "OriginFoodName": "加購選項範例1",
                  "ImagePath": "",
                  "Quantity": 1,
                  "AdditionalPrice": 15
                }
              ],
              "Remark": "",
              "Total": 130
            }
          ]
        },
        {
          "ComboID": "COMBO002",
          "FoodID": "FOOD001",
          "ItemName": "附餐群組範例B",
          "FoodName": "套餐商品範例",
          "Items": [
            {
              "FoodID": "FOOD003",
              "FoodName": "附餐商品範例2",
              "OriginFoodName": "附餐商品範例2",
              "ImagePath": "",
              "Quantity": 1,
              "AdditionalPrice": 10,
              "OptionItems": [
                {
                  "FoodID": "700A0165-2283-45A9-824B-D87E0913D5B1",
                  "FoodName": "加購選項範例2",
                  "OriginFoodName": "加購選項範例2",
                  "ImagePath": "",
                  "Quantity": 1,
                  "AdditionalPrice": 15
                }
              ],
              "Remark": "",
              "Total": 25
            }
          ]
        }
      ],
      "OptionItems": [],
      "Memo": "",
      "Taste": "",
      "Add": "",
      "AgioCost": 0,
      "ServCost": false,
      "Discount": true,
      "TotalDiscount": true,
      "ChangePrice": false
    }
  ],
  "coupons": [],
  "Operator": ""
}
```

### 7.6 cURL

`voice-order.json` 必須與計算簽章時使用的 JSON 字串完全相同。

```bash
curl 'https://goapi2.xurf.ai/Order/CalculateOrderPreview' \
  -X POST \
  -H 'X-Client-Id: <client_id>' \
  -H 'X-Timestamp: <unix_timestamp_seconds>' \
  -H 'X-Nonce: <uuid>' \
  -H 'X-Signature: <calculated_hmac_sha256>' \
  -H 'X-Client-Version: voice-order-1.0.0' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  --data-binary '@voice-order.json'
```

### 7.7 JavaScript

```javascript
const path = '/Order/CalculateOrderPreview';
const body = JSON.stringify(orderRequest);
const headers = {
  ...createVendorHeaders('POST', path, '', body),
  'Content-Type': 'application/json',
};

const response = await fetch(`${baseUrl}${path}`, {
  method: 'POST',
  headers,
  body,
});

const result = await response.json();
if (!response.ok || result.code !== 0) {
  throw new Error(`HTTP ${response.status}: ${JSON.stringify(result)}`);
}

if (result.data?.orderID !== orderRequest.Order.ID) {
  throw new Error('回傳的 orderID 與送出的 Order.ID 不一致');
}

console.log(result);
```

### 7.8 Success Response

成功必須同時符合 HTTP Status `200`、`code` 為 `0`，且 `data.orderID` 與送出的 `Order.ID` 相同。

> 來源 PDF 的 Request 範例金額為 `155`，Success Response 範例的 `payTotal` 為 `135`。兩段為獨立示例；實際應付金額一律以當次 API 回傳的 `data.payTotal` 為準。

```json
{
  "code": 0,
  "message": "計算成功",
  "data": {
    "isOnlinePay": false,
    "orderID": "16a0dc4a-cb47-4da7-b223-bc2ff954c91f",
    "payTotal": 135,
    "discount": 0,
    "serviceFee": 0,
    "deliveryFee": 0,
    "paymentLink": null
  }
}
```

### 7.9 業務錯誤

部分業務錯誤仍使用 HTTP `200`，必須檢查內層 `code`。

| code | 說明 | 處理方式 |
|---:|---|---|
| -6 | 品項無效、停售、售完或不存在。 | 重新呼叫 `GetFoodList`，更新菜單後請顧客重選。 |
| -8 | 取餐時段已額滿。 | 請顧客改選其他取餐時間。 |
| 其他非 0 | 下單失敗。 | 不得視為成功；記錄回應並聯絡 XURF。 |

| HTTP Status | 說明 |
|---:|---|
| 400 | JSON 格式或必要欄位錯誤。 |
| 401 | 憑證無效或已過期。 |
| 500 | 儲存訂單、資料庫或付款服務錯誤。 |

---

## 8. 上線驗收

### 13.1 通用

- 所有 cURL 與程式呼叫皆帶入規格要求的 Header。
- 憑證與密碼不得提交到 Git。
- API 逾時或狀態不明時，不得向使用者宣告成功。
- 問題回報須提供發生時間、Endpoint、HTTP Status 及遮蔽敏感資料後的回應。

### 13.2 語音點餐廠商

- 未建立廠商憑證時，三支 API 均回傳 `401 INVALID_CLIENT`。
- 私密金鑰錯誤、簽章錯誤、逾時或重送 Nonce 均不可通過。
- 非白名單來源回傳 `403 SOURCE_NOT_ALLOWED`。
- 已授權廠商不可跨企業或門市呼叫。
- `GetFoodList` 可取得外帶菜單，下單使用相同商品 ID 與價格。
- 門市關閉接單後不可送單；開啟後可建立外帶訂單。
- POS 可收到訂單，且備註顯示「語音點餐」。
- 同一 `Order.ID` 因逾時重送時不得產生兩張訂單。
- 停售、售完、錯誤商品 ID 與額滿時段均能正確提示。

### 13.3 技術支援

XURF API Support：`service@cloudxurf.com`
