# Webot US Institution Open API Documentation

HTTP API reference for institutional service providers (`/api/v1/institution/*`). An institution uses **its own master-account API Key** to query and operate all Webot sub-accounts bound to it. The target sub-account of each request is specified by the `uid` parameter.

> This is part of the Webot US Open API. **General information and authentication are shared** — see the main reference [webot-openapi.md](./webot-openapi.md).

## General Information

Base URL (`https://api.webot.com`), HTTPS, JSON format, camelCase fields, the success/error response envelope, and data-type conventions are shared across all Webot US APIs. See the [General Information](./webot-openapi.md#general-information) section of the main reference.

### Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/institution/accounts` | List bound sub-accounts |
| GET | `/api/v1/institution/balances` | Get sub-account balances |
| GET | `/api/v1/institution/transactions` | Aggregated fund records (fiat + on-chain) |
| GET | `/api/v1/institution/orders` | Aggregated order history |
| POST | `/api/v1/institution/fiat/fvb/withdraw/create` | Create fiat withdrawal |
| GET | `/api/v1/institution/fiat/banks/list` | List payout banks |
| GET | `/api/v1/institution/fiat/common/getWithdraws` | Query withdrawal records |
| GET | `/api/v1/institution/fiat/deposit/getVirtualAccount` | Get deposit virtual account |
| GET | `/api/v1/institution/fiat/common/getDeposits` | Query deposit records |
| POST | `/api/v1/institution/fiat/convert/create` | Create stablecoin conversion |
| GET | `/api/v1/institution/fiat/convert/record` | Query conversion result |
| GET | `/api/v1/institution/asset/currencies` | List currencies (public, no `uid`) |
| GET | `/api/v1/institution/asset/address` | Get deposit address |
| POST | `/api/v1/institution/asset/withdraw` | Create crypto withdrawal |
| GET | `/api/v1/institution/asset/withdraw` | Query a single withdrawal |
| GET | `/api/v1/institution/asset/records` | Query deposit/withdrawal history |
| GET | `/api/v1/institution/account/balances` | Get trading account balances |
| POST | `/api/v1/institution/trade/order` | Place order |
| GET | `/api/v1/institution/trade/order` | Query order |
| DELETE | `/api/v1/institution/trade/order` | Cancel order |
| POST | `/api/v1/institution/trade/massOrder` | Place orders in batch |
| GET | `/api/v1/institution/trade/orderByClientOrderId` | Query order by client order ID |
| GET | `/api/v1/institution/trade/openOrders` | Query open orders |
| GET | `/api/v1/institution/trade/allOrders` | Query all orders |
| DELETE | `/api/v1/institution/trade/allOrders` | Cancel all orders |
| GET | `/api/v1/institution/trade/fills` | Query fills |
| GET | `/api/v1/institution/trade/fillsByOrderId` | Query fills by order ID |

> Internal transfer endpoints (create / query / history) are **coming soon**.

---

## Authentication

All Webot US APIs share the same authentication (API Key + HMAC SHA256 signature, `PIONEX-KEY` / `PIONEX-SIGNATURE` headers, and the `timestamp` query parameter). See the [Authentication](./webot-openapi.md#authentication) section of the main reference for the full signature construction steps.

The key used here is your **institution master-account API Key**.

---

## The `uid` Parameter

Except for public endpoints (currently only *List currencies*), **every endpoint requires `uid`** — the UID of the target sub-account:

- **GET requests:** pass `uid` in the query string;
- **POST/DELETE requests:** pass `uid` in the JSON request body.

Obtain `uid` values from [List bound sub-accounts](#1-list-bound-sub-accounts). Only sub-accounts bound to your institution with status `ACTIVE` can be operated; any other `uid` returns `FORBIDDEN`.

---

## Error Codes

The following error codes may be returned by any endpoint:

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| UNAUTHENTICATED | 401 | Authentication failed (missing token, invalid signature, etc.) |
| FORBIDDEN | 403 | Caller is not a registered institution, `uid` is not a sub-account bound to your institution, or the account is suspended |
| INVALID_ARGUMENTS | 400 | Request parameter is missing or invalid |
| INTERNAL_ERROR | 500 | Internal error |
| DOWNSTREAM_ERROR | 502 | The underlying Webot API returned a business error; the original error code is returned as `code` (e.g. `TAPI_*`, `BOT_*`), with `message` as its human-readable description |

---

## Institution Endpoints

### 1. List Bound Sub-Accounts

Query all Webot sub-accounts bound to your institution. Suspended (`SUSPENDED`) accounts are also returned, so that a disabled account is visible rather than silently disappearing.

```
GET /api/v1/institution/accounts
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| timestamp | integer | Yes | Current time in milliseconds |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1785706000000,
  "data": {
    "accounts": [
      { "uid": "88001234", "note": "Client A", "status": "ACTIVE", "createdAt": 1785700000000 },
      { "uid": "88005678", "note": "Client B", "status": "SUSPENDED", "createdAt": 1785700100000 }
    ]
  }
}
```

**Response Fields:**

| Field | | Type | Description |
|-------|---|------|-------------|
| accounts | | object[] | Bound sub-accounts, one element per binding |
| | uid | string | Sub-account UID, used as the `uid` parameter of every other endpoint |
| | note | string | Note recorded at binding time (e.g. end-client name) |
| | status | string | `ACTIVE` / `SUSPENDED` |
| | createdAt | int64 | Binding creation time (millisecond timestamp) |

### 2. Get Sub-Account Balances

Query the trading account balances of a single sub-account. Bot accounts and earn accounts are not included.

```
GET /api/v1/institution/balances
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| timestamp | integer | Yes | Current time in milliseconds |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1785706000000,
  "data": {
    "uid": "88001234",
    "balances": [
      { "coin": "BTC", "free": "0.90000000", "frozen": "0.00000000" },
      { "coin": "USDT", "free": "100.00000000", "frozen": "900.00000000" }
    ]
  }
}
```

**Response Fields:**

| Field | | Type | Description |
|-------|---|------|-------------|
| uid | | string | Sub-account UID of this query |
| balances | | object[] | Balance list, one element per currency |
| | coin | string | Currency identifier, e.g. BTC |
| | free | string | Available balance (8 decimal places) |
| | frozen | string | Frozen balance (8 decimal places) |

### 3. Aggregated Fund Records

Query wallet fund records — **fiat withdrawals, fiat deposits, and on-chain deposits/withdrawals** merged into a single list, sorted by time in **descending** order (newest first). Records from different accounts and different sources are interleaved on one global timeline; each record keeps the original structure of its source endpoint.

```
GET /api/v1/institution/transactions
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | No | Sub-account UID (query string). **When omitted, all `ACTIVE` sub-accounts of the institution are traversed** |
| startTime | int64 | No | Start time (millisecond timestamp), defaults to `endTime - 7 days` |
| endTime | int64 | No | End time (millisecond timestamp), defaults to now |
| limit | int | No | Page size, default 100, max 500 |
| cursor | int64 | No | Pagination cursor; pass `nextCursor` from the previous page. When present it overrides `endTime` |
| timestamp | integer | Yes | Current time in milliseconds |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1785706000000,
  "data": {
    "items": [
      {
        "uid": "88005678",
        "source": "CHAIN",
        "record": { "id": "123456789", "currency": "USDT", "chain": "TRC20", "type": "DEPOSIT", "amount": "5000", "status": "COMPLETED", "createTime": 1700000500000 }
      },
      {
        "uid": "88001234",
        "source": "FIAT_DEPOSIT",
        "record": { "orderId": "d-9", "amount": "100", "currency": "USD", "status": "COMPLETED", "createTime": 1700000400000 }
      },
      {
        "uid": "88001234",
        "source": "FIAT_WITHDRAW",
        "record": { "orderId": "order-001", "amount": "100", "realAmount": "99", "fee": "1", "status": "success", "createTime": 1700000000000 }
      }
    ],
    "hasMore": true,
    "nextCursor": 1700000000000
  }
}
```

**Response Fields:**

| Field | | Type | Description |
|-------|---|------|-------------|
| items | | object[] | Fund records of all accounts and all three sources, merged and globally sorted by `record.createTime` descending |
| | uid | string | Sub-account this record belongs to |
| | source | string | Record source: `FIAT_WITHDRAW` / `FIAT_DEPOSIT` / `CHAIN` (for `CHAIN`, direction is `record.type` = DEPOSIT / WITHDRAW) |
| | record | object | The record in the structure of its source endpoint: *Query withdrawal records*, *Query deposit records* ([Fiat Open API](./webot-fiat-openapi.md)) or *Query deposit/withdrawal history* ([Wallet Open API](./webot-wallet-openapi.md)) |
| hasMore | | boolean | Whether earlier records exist within the time window |
| nextCursor | | int64 | Returned when `hasMore=true`; equals the time of the last item on this page |

**Pagination:** when `hasMore=true`, request again with `cursor=<nextCursor>` (other parameters unchanged) until `hasMore=false`.

### 4. Aggregated Order History

Query order history (open and closed orders) sorted by time in descending order, with multi-account aggregation.

```
GET /api/v1/institution/orders
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| symbol | string | Yes | Trading pair, e.g. `BTC_USDT` |
| uid | string | No | Sub-account UID (query string). When omitted, all `ACTIVE` sub-accounts of the institution are traversed |
| startTime | int64 | No | Start time (millisecond timestamp), defaults to `endTime - 7 days` |
| endTime | int64 | No | End time (millisecond timestamp), defaults to now |
| limit | int | No | Page size, default 100, max 500 |
| cursor | int64 | No | Pagination cursor, same as *Aggregated Fund Records* |
| timestamp | integer | Yes | Current time in milliseconds |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1785707000000,
  "data": {
    "items": [
      {
        "uid": "88005678",
        "order": { "orderId": 1234567, "symbol": "BTC_USDT", "type": "LIMIT", "side": "BUY", "price": "60000", "size": "0.01", "status": "CLOSED", "clientOrderId": "c-001", "createTime": 1700000500000, "updateTime": 1700000600000 }
      }
    ],
    "hasMore": true,
    "nextCursor": 1700000100000
  }
}
```

**Response Fields:**

| Field | | Type | Description |
|-------|---|------|-------------|
| items | | object[] | Order entries, sorted by time descending |
| | uid | string | Sub-account this order belongs to |
| | order | object | The order object; field structure is defined by the order fields of the [main reference](./webot-openapi.md) trading section |
| hasMore | | boolean | Whether earlier orders exist within the time window |
| nextCursor | | int64 | Returned when `hasMore=true`; equals the time of the last item on this page |

---

## Fiat Endpoints

Fiat deposit/withdrawal and stablecoin conversion, executed on the sub-account specified by `uid`.

### 1. Create Fiat Withdrawal

Create a fiat withdrawal. Currently only the bound-bank mode (`bankId`) is supported; a new bank account must complete one successful withdrawal via the App/Web first, after which its `bankId` is returned by *List Payout Banks*.

> ⚠️ **Important: this endpoint has no idempotency mechanism — never retry blindly.** If the request times out or no response is received, do **not** resend directly. First call *Query Withdrawal Records* (`getWithdraws`) with a time window to check whether an order was already created; resend only after confirming no order exists. Otherwise a duplicate withdrawal may occur.

```
POST /api/v1/institution/fiat/fvb/withdraw/create
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| timestamp | integer | Yes | Current time in milliseconds (query string) |

**Request Body (JSON):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (request body) |
| amount | string | Yes | Withdrawal amount, must be greater than 50 |
| bankId | string | No | Bound bank ID (bound-bank mode) |
| phone | string | No | Contact phone number |
| fileIds | string[] | No | Supporting document file IDs |
| bank | object | No | Payee bank information (new-bank mode, not supported in this phase); structure defined in the [Fiat Open API](./webot-fiat-openapi.md) |

**Request Example:**

```json
{
  "uid": "88001234",
  "bankId": "8c66e62274375facda91ecffef83321c",
  "amount": "100",
  "phone": "+14235735382"
}
```

**Response Example:**

```json
{ "result": true, "timestamp": 1751000000000, "data": { "orderId": "o-9" } }
```

**Response Fields:**

| Field | | Type | Description |
|-------|---|------|-------------|
| orderId | | string | Withdrawal order ID |

**Errors:**

`BOT_INVALID_ARGUMENT` is returned when the amount is ≤ 50 or a parameter is missing/invalid; see `message` for the specific reason. Downstream business errors are returned with `code` in the form `TAPI_FBO_*`. Common ones (non-exhaustive):

| Error Code | Description |
|------------|-------------|
| TAPI_FBO_PARAMETER_ERROR | Parameter validation failed |
| TAPI_FBO_BALANCE_ERROR | Insufficient balance |
| TAPI_FBO_BANK_ID_NOT_EXIST_ERROR | `bankId` does not exist or is unavailable |
| TAPI_FBO_CHANNEL_CLOSE_ERROR | Withdrawal channel is closed |
| TAPI_FBO_KYC_INFO_ERROR | KYC verification failed |
| TAPI_FBO_IN_BLACK_LIST_ERROR | User is blacklisted |
| TAPI_FBO_QUESTIONNAIRE_NOT_EXIST_ERROR | Questionnaire required for large withdrawals not filled |
| TAPI_FBO_SYSTEM_ERROR | Downstream system error |

### 2. List Payout Banks

Query the payout banks bound to the sub-account, filtered by `transferType`.

```
GET /api/v1/institution/fiat/banks/list
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| transferType | string | Yes | Transfer type, currently only `wire` |
| timestamp | integer | Yes | Current time in milliseconds |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1751000000000,
  "data": {
    "banks": [
      {
        "bankId": "8c66e62274375facda91ecffef83321c",
        "transferType": "wire",
        "bankAccountUs": { "accountNumber": "111111", "routingNumber": "123456789" },
        "bankAddress": { "bankName": "11", "city": "", "country": "US", "district": "", "line1": "", "line2": "" },
        "billingDetails": { "name": "Jamesaweg Tayloraega", "line1": "11", "city": "11", "district": "AL", "country": "US", "postalCode": "12321411" }
      }
    ],
    "uploadFiles": [ { "fileId": "bank-cert/webot.us/xxx.png", "uploadUrl": "https://..." } ],
    "phone": "+14235735382"
  }
}
```

**Response Fields:**

| Field | | Type | Description |
|-------|---|------|-------------|
| banks | | object[] | Bank list; element structure is the `bank` object of *Create Fiat Withdrawal* in the [Fiat Open API](./webot-fiat-openapi.md), with `bankId` populated |
| uploadFiles | | object[] | Supporting documents |
| | fileId | string | File ID |
| | uploadUrl | string | Upload URL |
| phone | | string | Contact phone number |

### 3. Query Withdrawal Records

Query withdrawal records filtered by time window; all matching records are returned. Pass `orderId` to query a single order (poll its status after creating a withdrawal).

```
GET /api/v1/institution/fiat/common/getWithdraws
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| channel | string | Yes | Withdrawal channel, currently only `fvbank` |
| transferType | string | Yes | Transfer type: `wire` or `ach` |
| startTime | int64 | No | Start time (millisecond timestamp) |
| endTime | int64 | No | End time (millisecond timestamp) |
| orderId | string | No | Withdrawal order ID; when present only that order is returned |
| timestamp | integer | Yes | Current time in milliseconds |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1751000000000,
  "data": {
    "records": [
      { "orderId": "order-001", "amount": "100", "realAmount": "99", "fee": "1", "status": "success", "createTime": 1700000000000, "remitTime": 1700000200000 }
    ]
  }
}
```

**Response Fields:**

| Field | | Type | Description |
|-------|---|------|-------------|
| records | | object[] | Withdrawal records |
| | orderId | string | Order ID |
| | amount | string | Requested amount |
| | realAmount | string | Actual amount received |
| | fee | string | Fee |
| | status | string | `success` (final) / `fail` (final) / `pending` (processing, keep polling) |
| | createTime | int64 | Creation time (millisecond timestamp) |
| | remitTime | int64 | Remittance time (millisecond timestamp) |

### 4. Get Deposit Virtual Account

Query the deposit virtual bank account. `channel` is required: `stripe_bank_transfer_wire` (non-same-name VA), `fvbank_wire` (same-name VA), and `bridge_wire` (same-name VA) are supported.

A single virtual account may contain multiple payment methods (`methods`): for example, domestic wire (`wire`) and international wire (`international_wire`) may coexist. Iterate over `methods` and decide which fields to display based on `rail`. Every field is returned only when it has a value.

The `fvbank_wire` virtual account must be applied for — if the call returns an error, contact your integration representative. The `bridge_wire` channel currently provides only the domestic wire (`wire`) method; the virtual account must be enabled, otherwise `BOT_BRIDGE_VA_NOT_FOUND` is returned and you must contact support.

```
GET /api/v1/institution/fiat/deposit/getVirtualAccount
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| channel | string | Yes | Deposit channel: `stripe_bank_transfer_wire`, `fvbank_wire`, or `bridge_wire` |
| timestamp | integer | Yes | Current time in milliseconds |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1751000000000,
  "data": {
    "channel": "stripe_bank_transfer_wire",
    "methods": [
      {
        "rail": "wire",
        "accountNumber": "123456789",
        "routingNumber": "021000021",
        "bankName": "Test Bank",
        "bankAddress": "185 Berry St, San Francisco, CA, 94107, US",
        "beneficiaryName": "John Doe",
        "beneficiaryAddress": "1 Main St, New York, NY, 10001, US",
        "minDepositAmount": "50",
        "fee": { "feePerOrder": "20" }
      }
    ]
  }
}
```

**Response Fields:**

| Field | | Type | Description |
|-------|---|------|-------------|
| channel | | string | Channel identifier |
| methods | | object[] | Payment methods. Element fields (`rail` / `accountNumber` / `routingNumber` / `swiftCode` / `bankName` / `bankAddress` / intermediary-bank fields / beneficiary fields / `minDepositAmount` / `fee` / `extraFields`, etc.), the full per-channel examples, and the VA status error codes (`BOT_FVBANK_VA_*`, `BOT_BRIDGE_VA_NOT_FOUND`) are documented in the [Fiat Open API](./webot-fiat-openapi.md) |

### 5. Query Deposit Records

Query deposit records with pagination and time-window filters.

```
GET /api/v1/institution/fiat/common/getDeposits
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| channel | string | Yes | Deposit channel: `stripe_bank_transfer_wire`, `fvbank_wire`, or `bridge_wire`, matching *Get Deposit Virtual Account* |
| page | int | No | Page number starting from 1; empty or 0 means page 1 |
| limit | int | No | Page size in `[1,50]`; out of range or 0 means 50 |
| startTime | int64 | No | Start time (millisecond timestamp, by creation time, inclusive) |
| endTime | int64 | No | End time (millisecond timestamp, exclusive) |
| timestamp | integer | Yes | Current time in milliseconds |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1751000000000,
  "data": {
    "records": [
      {
        "orderId": "d-1", "amount": "100", "coin": "USDT", "currency": "USD",
        "feeAmount": "1", "transferAmount": "99", "status": "COMPLETED",
        "channel": "stripe_bank_transfer_wire", "paymentBankName": "Test Bank",
        "errDesc": "", "createTime": 1700000000000, "payTime": 1700000100000, "transferTime": 1700000200000
      }
    ],
    "total": 1
  }
}
```

**Response Fields:**

| Field | | Type | Description |
|-------|---|------|-------------|
| records | | object[] | Deposit records |
| | orderId | string | Order ID |
| | amount | string | Deposit amount |
| | coin | string | Credited currency |
| | currency | string | Fiat currency |
| | feeAmount | string | Fee |
| | transferAmount | string | Actual amount credited |
| | status | string | `COMPLETED` (final) / `FAILED` (final) / `CANCELED` (final) / `PENDING` (processing, keep polling) |
| | channel | string | Actual channel of the order |
| | paymentBankName | string | Paying bank name |
| | errDesc | string | Error description |
| | createTime | int64 | Creation time (millisecond timestamp) |
| | payTime | int64 | Payment time (millisecond timestamp) |
| | transferTime | int64 | Credit time (millisecond timestamp) |
| total | | int | Total number of records |

### 6. Create Stablecoin Conversion

Create a stablecoin conversion. Asynchronous — only the order ID is returned. It is essentially a market order: the `sourceCoin` balance must be sufficient and the trading pair must exist; the effect is the same as placing an order via the trading endpoint.

```
POST /api/v1/institution/fiat/convert/create
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| timestamp | integer | Yes | Current time in milliseconds (query string) |

**Request Body (JSON):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (request body) |
| sourceCoin | string | Yes | Source currency |
| targetCoin | string | Yes | Target currency |
| sourceAmount | string | Yes | Source amount |
| clientOrderId | string | No | UUID, idempotency key; when present, duplicate retries are deduplicated |

**Request Example:**

```json
{ "uid": "88001234", "sourceCoin": "USDC", "targetCoin": "USDT", "sourceAmount": "50", "clientOrderId": "5ab3cb7e-e3f4-4417-83d3-339ba101002a" }
```

**Response Example:**

```json
{ "result": true, "timestamp": 1751000000000, "data": { "orderId": "ord-1" } }
```

**Errors:** parameter validation failures (missing field, malformed `clientOrderId`, non-existent or closed pair) return `BOT_INVALID_ARGUMENT`; downstream business errors are passed through (e.g. `TAPI_WALLET_BALANCE_NOT_ENOUGH` for insufficient source balance).

### 7. Query Conversion Result

Query a conversion result; the currency pair must be passed back. For conversion history, use *Query All Orders* in the trading section.

```
GET /api/v1/institution/fiat/convert/record
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| orderId | string | Yes | Order ID |
| sourceCoin | string | Yes | Source currency |
| targetCoin | string | Yes | Target currency |
| timestamp | integer | Yes | Current time in milliseconds |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1751000000000,
  "data": {
    "orderId": "ord-1", "sourceCoin": "USDC", "sourceAmount": "50",
    "targetCoin": "USDT", "targetAmount": "50.2", "avgPrice": "1.004",
    "feeCoin": "USDT", "feeAmount": "0.05", "status": "SUCCESS",
    "timestamp": 1700000000000, "clientOrderId": "merchant-abc-1"
  }
}
```

**Response Fields:**

| Field | | Type | Description |
|-------|---|------|-------------|
| orderId | | string | Order ID |
| sourceCoin / sourceAmount | | string | Source currency / amount |
| targetCoin / targetAmount | | string | Target currency / amount |
| avgPrice | | string | Average execution price |
| feeCoin / feeAmount | | string | Fee currency / amount |
| status | | string | `SUCCESS` (final) / `PENDING` (matching, keep polling) / `FAILED` (canceled / rejected / expired, final). When polling, any non-`PENDING` status is final |
| timestamp | | int64 | Timestamp |
| clientOrderId | | string | Idempotency key passed at creation; empty if not provided |

---

## Wallet Endpoints

Crypto asset currency information, deposit addresses, withdrawals, and deposit/withdrawal history for the sub-account specified by `uid`.

### 1. List Currencies

Query supported currencies and their chain information.

> This endpoint does not require `uid`. Note: unlike the base API where it is public, calling it via the Institution API **still requires institution signature authentication**.

```
GET /api/v1/institution/asset/currencies
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| currency | string | No | Currency name; all currencies are returned when omitted. Max length 60 |
| timestamp | integer | Yes | Current time in milliseconds |

**Response Fields:**

| Field | | Type | Description |
|-------|---|------|-------------|
| currencies | | object[] | Currency list |
| | currency | string | Currency identifier |
| | displayName | string | Display name |
| | fullName | string | Full name |
| | chainList | object[] | Supported chains; element fields (`chain` / `depositEnable` / `withdrawEnable` / `depositMin` / `withdrawMin` / `withdrawMax` / `withdrawFee` / `hasTag` / `confirm` / `withdrawPrecision` / `contractAddress`, etc.) are documented in the [Wallet Open API](./webot-wallet-openapi.md) |

### 2. Get Deposit Address

Query the deposit address for a given currency and chain.

```
GET /api/v1/institution/asset/address
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| currency | string | Yes | Currency name, max length 60 |
| chain | string | Yes | Chain name, max length 60 |
| timestamp | integer | Yes | Current time in milliseconds |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1679900000000,
  "data": { "address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh", "tag": "" }
}
```

**Errors:**

| Error Code | Description |
|------------|-------------|
| FROZEN_FORBIDDEN | Deposits are frozen for this account |
| KYC_FORBIDDEN | KYC verification required |

### 3. Create Crypto Withdrawal

Submit a withdrawal request. The address whitelist must be enabled and the withdrawal address must be on the whitelist.

```
POST /api/v1/institution/asset/withdraw
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| timestamp | integer | Yes | Current time in milliseconds (query string) |

**Request Body (JSON):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (request body) |
| currency | string | Yes | Currency name, max length 60 |
| chain | string | Yes | Chain name, max length 60 |
| clientId | string | Yes | Client-defined ID (idempotency key), max length 64 |
| address | string | Yes | Withdrawal address, max length 300 |
| tag | string | No | Address tag/memo, max length 100 |
| amount | string | Yes | Withdrawal amount, max length 60 |
| note | string | No | Note, max length 100 |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1679900000000,
  "data": { "id": "123456789", "clientId": "my-withdraw-001" }
}
```

**Errors:**

| Error Code | Description |
|------------|-------------|
| FROZEN_FORBIDDEN | Withdrawals are frozen for this account |
| KYC_FORBIDDEN | KYC verification required |
| FORBIDDEN | Operation blocked after high-risk behavior; `data` contains `restrict_expired_on`, `restrict_ttl` |
| SIGN_FORBIDDEN | Security signature failed |
| WITHDRAW_WITHELIST_ADDRESS_CLOSED | Address whitelist is not enabled (mandatory) |
| WITHDRAW_ADDRESS_NOT_WHITELISTED | Withdrawal address is not on the whitelist |
| WITHDRAW_ADDRESS_NOT_IN_ALLOWED_LIST | Withdrawal address is not on the configured allow list |

### 4. Query a Single Withdrawal

Query a single withdrawal record by system order ID.

```
GET /api/v1/institution/asset/withdraw
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| id | string | Yes | System order ID, max length 64 |
| timestamp | integer | Yes | Current time in milliseconds |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1679900000000,
  "data": {
    "id": "123456789", "clientId": "my-withdraw-001", "currency": "USDT", "chain": "TRC20",
    "type": "withdraw", "internal": false, "address": "TXxxxx...", "tag": "",
    "amount": "100.5", "fee": "1", "status": "completed", "hash": "abc123...",
    "confirmations": 20, "note": "", "createTime": 1679900000000, "updateTime": 1679900100000
  }
}
```

**Response Fields:** `id` / `clientId` / `currency` / `chain` / `type` (deposit/withdraw) / `internal` (internal transfer or not) / `address` / `tag` / `addressFrom` / `tagFrom` / `amount` / `fee` / `status` / `hash` / `confirmations` / `note` / `approvalReason` / `createTime` / `updateTime` — documented in the [Wallet Open API](./webot-wallet-openapi.md).

### 5. Query Deposit/Withdrawal History

Query the deposit and withdrawal history.

```
GET /api/v1/institution/asset/records
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| currency | string | No | Currency name; all when omitted. Max length 60 |
| type | string | No | Transaction type (`DEPOSIT` / `WITHDRAW`); all when omitted. Max length 30 |
| startTime | int64 | No | Start time (millisecond timestamp) |
| endTime | int64 | No | End time (millisecond timestamp) |
| timestamp | integer | Yes | Current time in milliseconds |

**Response:** `records[]`, element fields identical to the `data` of *Query a Single Withdrawal*.

---

## Trading Endpoints

Trading account balances, order management and fills for the sub-account specified by `uid`.

### 1. Get Trading Account Balances

Query the trading account balances. Bot accounts and earn accounts are not included.

```
GET /api/v1/institution/account/balances
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| timestamp | integer | Yes | Current time in milliseconds |

**Response:** the same `balances` structure as *Get Sub-Account Balances* above (this variant's `data` does not echo `uid`).

### 2. Place Order

```
POST /api/v1/institution/trade/order
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| timestamp | integer | Yes | Current time in milliseconds (query string) |

**Request Body (JSON):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (request body) |
| symbol | string | Yes | Trading pair, e.g. `BTC_USDT` |
| side | string | Yes | `BUY` / `SELL` |
| type | string | Yes | `LIMIT` / `MARKET` |
| clientOrderId | string | No | Client order ID (alphanumeric and hyphens, max 64 chars) |
| size | string | Conditional | Quantity. Required for LIMIT orders and MARKET sells |
| price | string | Conditional | Price. Required for LIMIT orders |
| amount | string | Conditional | Amount. Required for MARKET buys |
| IOC | boolean | No | Immediate-or-cancel flag, default `false` |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1566691672311,
  "data": { "orderId": 123456789, "clientOrderId": "my-order-001" }
}
```

### 3. Query Order

Query order details by order ID.

```
GET /api/v1/institution/trade/order
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| orderId | integer | Yes | Order ID |
| timestamp | integer | Yes | Current time in milliseconds |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1566691692311,
  "data": {
    "orderId": 123456789, "symbol": "BTC_USDT", "type": "LIMIT", "side": "BUY",
    "price": "68000.00", "size": "0.001000", "amount": "0",
    "filledSize": "0.001000", "filledAmount": "68.00000000",
    "fee": "0.00000100", "feeCoin": "BTC", "status": "CLOSED", "IOC": false,
    "clientOrderId": "my-order-001", "source": "API",
    "createTime": 1566691672311, "updateTime": 1566691682311
  }
}
```

**Common Order Fields** (the order object of every query endpoint in this section):

| Field | | Type | Description |
|-------|---|------|-------------|
| orderId | | integer | Order ID |
| symbol | | string | Trading pair |
| type | | string | Order type: `LIMIT` / `MARKET` |
| side | | string | `BUY` / `SELL` |
| price | | string | Order price |
| size | | string | Order quantity |
| amount | | string | MARKET buy amount |
| filledSize | | string | Filled quantity |
| filledAmount | | string | Filled amount |
| fee | | string | Fee |
| feeCoin | | string | Fee currency |
| status | | string | Order status: `OPEN` (active) / `CLOSED` (final) |
| IOC | | boolean | Immediate-or-cancel flag |
| clientOrderId | | string | Client order ID |
| source | | string | Order source: `MANUAL` / `API` |
| createTime | | integer | Creation time (millisecond timestamp) |
| updateTime | | integer | Update time (millisecond timestamp) |

### 4. Cancel Order

Cancel the specified order.

```
DELETE /api/v1/institution/trade/order
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| timestamp | integer | Yes | Current time in milliseconds (query string) |

**Request Body (JSON):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (request body) |
| symbol | string | Yes | Trading pair, e.g. `BTC_USDT` |
| orderId | integer | Yes | Order ID to cancel |

**Response Example:**

```json
{ "result": true, "timestamp": 1566691672311 }
```

### 5. Place Orders in Batch

Submit multiple orders at once (max 20, LIMIT only).

```
POST /api/v1/institution/trade/massOrder
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| timestamp | integer | Yes | Current time in milliseconds (query string) |

**Request Body (JSON):**

| Field | | Type | Required | Description |
|-------|---|------|----------|-------------|
| uid | | string | Yes | Sub-account UID (request body) |
| symbol | | string | Yes | Trading pair, e.g. `BTC_USDT` |
| orders | | array | Yes | Order collection (max 20) |
| | side | string | Yes | `BUY` / `SELL` |
| | type | string | Yes | Only `LIMIT` |
| | clientOrderId | string | No | Client order ID (max 64 chars) |
| | size | string | Yes | Order quantity |
| | price | string | Yes | Order price |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1566691672311,
  "data": {
    "orderIds": [
      { "orderId": 123456789, "clientOrderId": "order-1" },
      { "orderId": 123456790, "clientOrderId": "order-2" }
    ]
  }
}
```

### 6. Query Order by Client Order ID

```
GET /api/v1/institution/trade/orderByClientOrderId
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| clientOrderId | string | Yes | Client order ID |
| timestamp | integer | Yes | Current time in milliseconds |

**Response:** same as *Query Order*.

### 7. Query Open Orders

Query all open orders of a trading pair; max 200 per pair.

```
GET /api/v1/institution/trade/openOrders
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| symbol | string | Yes | Trading pair, e.g. `BTC_USDT` |
| timestamp | integer | Yes | Current time in milliseconds |

**Response:** `data.orders[]`, element structure as *Common Order Fields*.

### 8. Query All Orders

Query all orders of a trading pair (open and closed).

```
GET /api/v1/institution/trade/allOrders
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| symbol | string | Yes | Trading pair, e.g. `BTC_USDT` |
| startTime | integer | No | Start time (millisecond timestamp) |
| endTime | integer | No | End time (millisecond timestamp) |
| limit | integer | No | Number of records, range 1-200, default 50 |
| timestamp | integer | Yes | Current time in milliseconds |

**Response:** `data.orders[]`, element structure as *Common Order Fields*.

### 9. Cancel All Orders

Cancel all open orders of a trading pair.

```
DELETE /api/v1/institution/trade/allOrders
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| timestamp | integer | Yes | Current time in milliseconds (query string) |

**Request Body (JSON):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (request body) |
| symbol | string | Yes | Trading pair, e.g. `BTC_USDT` |

**Response Example:**

```json
{ "result": true, "timestamp": 1566691672311 }
```

### 10. Query Fills

Query fills of a trading pair.

```
GET /api/v1/institution/trade/fills
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| symbol | string | Yes | Trading pair, e.g. `BTC_USDT` |
| startTime | integer | No | Start time (millisecond timestamp) |
| endTime | integer | No | End time (millisecond timestamp) |
| timestamp | integer | Yes | Current time in milliseconds |

**Response Example:**

```json
{
  "result": true,
  "timestamp": 1566691672311,
  "data": {
    "fills": [
      {
        "id": 987654321, "orderId": 123456789, "symbol": "BTC_USDT",
        "side": "BUY", "role": "TAKER", "price": "68000.00", "size": "0.001000",
        "fee": "0.00000100", "feeCoin": "BTC", "timestamp": 1566691672311
      }
    ]
  }
}
```

**Common Fill Fields:**

| Field | | Type | Description |
|-------|---|------|-------------|
| fills | | object[] | Fill list |
| | id | integer | Fill ID |
| | orderId | integer | Order ID |
| | symbol | string | Trading pair |
| | side | string | `BUY` / `SELL` |
| | role | string | `TAKER` / `MAKER` |
| | price | string | Fill price |
| | size | string | Fill quantity |
| | fee | string | Fee |
| | feeCoin | string | Fee currency |
| | timestamp | integer | Fill time (millisecond timestamp) |

### 11. Query Fills by Order ID

Query fills of a specific order.

```
GET /api/v1/institution/trade/fillsByOrderId
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| uid | string | Yes | Sub-account UID (query string) |
| orderId | integer | Yes | Order ID; an empty list is returned when it does not exist |
| fromId | integer | No | Return the 100 fills older than this fill ID; latest fills when omitted |
| timestamp | integer | Yes | Current time in milliseconds |

**Response:** same as *Query Fills*.
