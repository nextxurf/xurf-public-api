# XURF 公開 API

本 repository 提供 XURF POS 與語音點餐廠商 API 串接文件。

完整的參數、回應欄位、cURL、JavaScript 及錯誤範例請參閱 [XURF 公開 API 串接規格書](./XURF公開API串接規格書.md)。

## API 基本要求

| 項目 | 要求 |
|---|---|
| Base URL | `https://goapi2.xurf.ai` |
| 傳輸協定 | HTTPS |
| HTTP Method | GET、POST |
| 資料格式 | JSON |
| 字元編碼 | UTF-8 |
| 語音點餐廠商認證 | HMAC-SHA256、Timestamp、Nonce、來源 IP/CIDR 白名單 |
| 日期時間 | 依 API 使用 `YYYYMMDD`、`YYYY-MM-DD` 或 ISO 8601 |

## API 清單

| API | Endpoint |
|---|---|
| 會員點數與票券 | `POST /api/AppVIP` |
| 消費者訂單明細 | `POST /api/appmember` |
| POS 銷售訂單上傳 | `POST /v1/services/epos/add_pos_order` |
| 外帶菜單 | `GET /MenuFood/GetFoodList` |
| 門市接單狀態 | `GET /Shop/GetShopInfo` |
| 線上點餐下單 | `POST /Order/CalculateOrderPreview` |

## 語音點餐必要 Header

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

> HMAC 憑證、Nonce 防重放、來源白名單及權限驗證層尚待正式完成，完成後才可交付廠商使用。

`AppVIP`、`appmember`、`add_pos_order` 尚未定義授權 Header，正式串接前須向 XURF 確認。

## 最小請求範例

以下簽章須依規格書以 `Client-Secret` 計算：

```bash
curl -G 'https://goapi2.xurf.ai/MenuFood/GetFoodList' \
  -H 'X-Client-Id: <client_id>' \
  -H 'X-Timestamp: <unix_timestamp_seconds>' \
  -H 'X-Nonce: <uuid>' \
  -H 'X-Signature: <calculated_hmac_sha256>' \
  -H 'Accept: application/json' \
  --data-urlencode 'enterpriseId=ENTERPRISE001' \
  --data-urlencode 'langId=TW' \
  --data-urlencode 'orderType=takeout' \
  --data-urlencode 'shopId=SHOP001'
```

## 串接注意事項

1. 憑證、密碼及 `Client-Secret` 不得寫入前端、網址、日誌或 Git。
2. Query String 必須排序並使用 RFC 3986 URL encoding 後再計算簽章。
3. Timestamp 與伺服器時間差必須在 5 分鐘內，每次請求使用新的 Nonce。
4. POST 簽章使用的 JSON 字串必須與實際傳送內容完全相同。
5. 門市狀態不明、API 逾時或下單回傳 `code` 非 `0` 時，不得宣告下單成功。

## 技術支援

XURF API Support：`service@cloudxurf.com`
