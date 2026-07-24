# Webot US Fiat Open API Documentation

HTTP API reference for fiat deposit/withdrawal and stablecoin conversion (`/api/v1/fiat/*`).

> This is part of the Webot US Open API. **General information and authentication are shared** — see the main reference [webot-openapi.md](./webot-openapi.md).

## General Information

Base URL (`https://api.webot.com`), HTTPS, JSON format, camelCase fields, the success/error response envelope, and data-type conventions are shared across all Webot US APIs. See the [General Information](./webot-openapi.md#general-information) section of the main reference.

### Endpoint Summary

| Method | Path | Description | Permission |
|--------|------|-------------|------------|
| GET | `/api/v1/fiat/deposit/getVirtualAccount` | Get deposit virtual account | Enable reading |
| GET | `/api/v1/fiat/common/getDeposits` | Query deposit records | Enable reading |
| POST | `/api/v1/fiat/convert/create` | Create stablecoin conversion | Enable trading |
| GET | `/api/v1/fiat/convert/record` | Query conversion result | Enable reading |
| POST | `/api/v1/fiat/fvb/withdraw/create` | Create fiat withdrawal | Enable trading |
| GET | `/api/v1/fiat/banks/list` | List payout banks | Enable reading |
| GET | `/api/v1/fiat/common/getWithdraws` | Query withdrawal records | Enable reading |

---

## Authentication

All Webot US APIs share the same authentication (API Key + HMAC SHA256 signature, `PIONEX-KEY` / `PIONEX-SIGNATURE` headers, and the `timestamp` query parameter). See the [Authentication](./webot-openapi.md#authentication) section of the main reference for the full signature construction steps.

---

## Error Codes

The following error codes may be returned by any endpoint:

| Error Code | Description |
|------------|-------------|
| BOT_INVALID_TOKEN | Authentication failed (missing token, malformed, or invalid token), HTTP 401 |
| BOT_INVALID_ARGUMENT | Request parameter is missing or invalid |
| BOT_INTERNAL_ERROR | Internal error or an unclassified downstream error; see `message` for the specific reason |

When a downstream service returns an explicit business error, that error code (in the form `TAPI_*`) is passed through as `code`, with `message` as its human-readable description. Such codes are listed under each endpoint's **Errors** section below.

---

## Fiat Deposit

### Deposit Flow

1. Call `getVirtualAccount` to obtain the virtual account (VA). You must know which deposit channel to use (e.g. `fvbank_wire`) and pass it as a parameter.
2. After obtaining the VA, transfer funds to it.
3. Poll `getDeposits` to check whether a matching new order has appeared.

### 1. Get Deposit Virtual Account

Query the deposit virtual bank account. `channel` is required to select the deposit channel; `stripe_bank_transfer_wire` (non-same-name VA), `fvbank_wire` (same-name VA), and `bridge_wire` (same-name VA) are supported.

A single virtual account may contain multiple payment methods (`methods`): for example, domestic wire (`wire`) and international wire (`international_wire`) may coexist. The client iterates over `methods` and decides which fields to display based on `rail`. Every field is returned only when it has a value.

The `fvbank_wire` virtual account must be applied for. If the call returns an error, contact your integration representative.

The `bridge_wire` channel currently provides only the domestic wire (`wire`) payment method. The virtual account must be enabled; when it is not enabled, `BOT_BRIDGE_VA_NOT_FOUND` is returned and you must contact support.

**Permission required:** `Enable reading`

```
GET /api/v1/fiat/deposit/getVirtualAccount
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| channel | string | Yes | Deposit channel: `stripe_bank_transfer_wire` (non-same-name VA), `fvbank_wire` (same-name VA), or `bridge_wire` (same-name VA) |

**Response Example:**

```json
{
  "result": true,
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
  },
  "timestamp": 1751000000000
}
```

**Response Example (`channel=fvbank_wire`)** — may return both domestic wire (`wire`) and international wire (`international_wire`) payment methods:

```json
{
  "result": true,
  "data": {
    "channel": "fvbank_wire",
    "methods": [
      {
        "rail": "international_wire",
        "accountNumber": "780009000572",
        "swiftCode": "ITTLPRS2XXX",
        "bankName": "FV Bank International Inc.",
        "bankAddress": "270 Muñoz Rivera Avenue, Suite 1120, San Juan, PR 00918",
        "intermediaryBankName": "Convera USA, LLC",
        "intermediaryBankSwiftCode": "TGBPUS3WXXX",
        "beneficiaryName": "Example Company Name",
        "beneficiaryAddress": "1437 VIP Road, Agra, UP 282001, IN",
        "minDepositAmount": "50",
        "fee": { "feePerOrder": "75" },
        "extraFields": {
          "field 70 (remittance information)": "[Any additional reference or payment details]",
          "field 71a (details of charges)": "Specify OUR, sender pays fees",
          "field 56a (intermediary institution)": "TGBPUS3WXXX (Convera USA, LLC)",
          "field 57a (account with institution)": "ITTLPRS2XXX (FV Bank International)",
          "field 59 (beneficiary customer)": "Example Company Name 780009000572"
        }
      }
    ]
  },
  "timestamp": 1783494450758
}
```

**Response Example (`channel=bridge_wire`)** — currently returns only the domestic wire (`wire`) payment method:

```json
{
  "result": true,
  "timestamp": 1751000000000,
  "data": {
    "channel": "bridge_wire",
    "methods": [
      {
        "bankName": "Bank of Nowhere",
        "bankAddress": "1800 North Pole St., Orlando, FL 32801",
        "beneficiaryName": "Sandbox Business",
        "beneficiaryAddress": "123 Test Street, San Francisco, CA 94105, US",
        "fee": {
          "feePerOrder": "20"
        },
        "rail": "wire",
        "accountNumber": "2659572793",
        "routingNumber": "101019644"
      }
    ]
  }
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| channel | string | Upstream channel this virtual account belongs to (e.g. `stripe_bank_transfer_wire`, `fvbank_wire`, `bridge_wire`) |
| methods | object[] | Payment method list, at least 1 element, see below |

`methods[]` element (all fields returned only when they have a value):

| Field | Type | Description |
|-------|------|-------------|
| rail | string | Payment method: `wire` (domestic wire) / `international_wire` (international wire) |
| accountNumber | string | Payout account number (for international wire, IBAN uses this field too) |
| accountType | string | Account type |
| routingNumber | string | Routing number (ABA, domestic wire) |
| swiftCode | string | SWIFT/BIC (international wire) |
| bankName | string | Payout bank name |
| bankAddress | string | Payout bank address (pre-formatted single line) |
| intermediaryBankName | string | Intermediary bank name (international wire) |
| intermediaryBankSwiftCode | string | Intermediary bank SWIFT (international wire) |
| beneficiaryName | string | Beneficiary name |
| beneficiaryAddress | string | Beneficiary address (pre-formatted single line) |
| reference | string | Wire remittance note / reference number |
| minDepositAmount | string | Minimum deposit amount |
| fee | object | Fee: `{ feePerOrder }` (fee per order) |
| extraFields | object | Upstream dynamic additional fields (e.g. fvbank international wire MT103), key-value form |

**Errors**

The following error codes are returned only when `channel=fvbank_wire` and the account is not ready:

| code | Description |
|------|-------------|
| BOT_FVBANK_VA_NOT_FOUND | Virtual account does not exist; it must be applied for first |
| BOT_FVBANK_VA_CREATING | Virtual account is being created; retry later |
| BOT_FVBANK_VA_CREATE_FAILED | Virtual account creation failed; contact support |

**Bridge Virtual Account Status Errors** (returned only when `channel=bridge_wire` and the account is not ready)

| code | Description |
|------|-------------|
| BOT_BRIDGE_VA_NOT_FOUND | Virtual account does not exist; contact support |

---

### 2. Query Deposit Records

Query deposit records, with pagination and time-window filtering.

**Permission required:** `Enable reading`

```
GET /api/v1/fiat/common/getDeposits
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| channel | string | Yes | Deposit channel: `stripe_bank_transfer_wire`, `fvbank_wire`, or `bridge_wire`, matching `getVirtualAccount` |
| page | int | No | Page number, starting from 1; empty or 0 returns page 1 |
| limit | int | No | Page size, range `[1,50]`; out-of-range or 0 uses 50 |
| startTime | int64 | No | Start time (millisecond timestamp, filtered by creation time, inclusive) |
| endTime | int64 | No | End time (millisecond timestamp, exclusive) |

**Response Example:**

```json
{
  "result": true,
  "data": {
    "records": [
      {
        "orderId": "d-1",
        "amount": "100",
        "coin": "USDT",
        "currency": "USD",
        "feeAmount": "1",
        "transferAmount": "99",
        "status": "COMPLETED",
        "channel": "stripe_bank_transfer_wire",
        "paymentBankName": "Test Bank",
        "errDesc": "",
        "createTime": 1700000000000,
        "payTime": 1700000100000,
        "transferTime": 1700000200000
      }
    ],
    "total": 1
  },
  "timestamp": 1751000000000
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| records | object[] | Deposit records, see element fields below |
| total | int | Total count |

`records[]` element:

| Field | Type | Description |
|-------|------|-------------|
| orderId | string | Order ID |
| amount | string | Deposit amount |
| coin | string | Received coin |
| currency | string | Fiat currency |
| feeAmount | string | Fee |
| transferAmount | string | Actual amount received |
| status | string | Order status: `COMPLETED` (credited, terminal) / `FAILED` (failed, terminal) / `CANCELED` (canceled, terminal) / `PENDING` (processing, keep polling). The wire deposit path is mainly `PENDING → COMPLETED / FAILED` |
| channel | string | Actual channel identifier of the order |
| paymentBankName | string | Payer bank name |
| errDesc | string | Error description |
| createTime | int64 | Creation time (millisecond timestamp) |
| payTime | int64 | Payment time (millisecond timestamp) |
| transferTime | int64 | Credit time (millisecond timestamp) |

---

## Conversion

### 3. Create Conversion

Create a stablecoin conversion. The operation is asynchronous and only returns an order ID. Ensure the `sourceCoin` balance is sufficient and the conversion pair exists.

**Permission required:** `Enable trading`

```
POST /api/v1/fiat/convert/create
```

**Request Body:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| sourceCoin | string | Yes | Source coin |
| targetCoin | string | Yes | Target coin |
| sourceAmount | string | Yes | Source coin amount |

**Request Example:**

```json
{ "sourceCoin": "USDC", "targetCoin": "USDT", "sourceAmount": "50" }
```

**Response Example:**

```json
{ "result": true, "data": { "orderId": "ord-1" }, "timestamp": 1751000000000 }
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| orderId | string | Conversion order ID |

**Errors**

If parameter validation fails (missing field, or the coin pair does not exist / is not open for conversion), `BOT_INVALID_ARGUMENT` is returned before the downstream request; see `message` for the specific reason.

Downstream business errors are passed through with `code` in the form `TAPI_*` and `message` as the human-readable description. The table below lists common codes (non-exhaustive); any downstream error not listed maps to `BOT_INTERNAL_ERROR`:

| code | Description |
|------|-------------|
| TAPI_WALLET_BALANCE_NOT_ENOUGH | Insufficient source coin balance |

---

### 4. Query Conversion Result

Query a conversion result by order ID. Each result includes its coin pair and direction. To query trade history, call `/api/v1/trade/allOrders`.

**Permission required:** `Enable reading`

```
GET /api/v1/fiat/convert/record
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| orderId | string | Yes | Order ID |

**Response Example:**

```json
{
  "result": true,
  "data": {
    "orderId": "ord-1",
    "sourceCoin": "USDC",
    "sourceAmount": "50",
    "targetCoin": "USDT",
    "targetAmount": "50.2",
    "avgPrice": "1.004",
    "status": "SUCCESS",
    "timestamp": 1700000000000
  },
  "timestamp": 1751000000000
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| orderId | string | Order ID |
| sourceCoin | string | Source coin |
| sourceAmount | string | Source coin amount |
| targetCoin | string | Target coin |
| targetAmount | string | Target coin amount |
| avgPrice | string | Average fill price |
| status | string | Order status: `SUCCESS` (completed, terminal) / `PENDING` (processing, keep polling) / `FAILED` (failed, terminal). When polling, "not PENDING means terminal" |
| timestamp | int64 | Timestamp |

**Errors**

If `orderId` is missing or the conversion order does not exist, `BOT_INVALID_ARGUMENT` is returned; see `message` for the specific reason.

If the downstream query fails, `BOT_INTERNAL_ERROR` is returned; see `message` for details.

---

## Fiat Withdrawal

### 5. Create Withdrawal

Create a fiat withdrawal. Currently only the `bankId` mode is supported. For a new bank card, complete a successful withdrawal via the App/Web once, then obtain its `bankId` through `/api/v1/fiat/banks/list`.

Two modes are supported:

- **Existing bank:** pass `bankId` to reuse a bound payout bank; leave `bank` empty.
- **New bank:** pass the `bank` object with the payout bank details directly.

**Permission required:** `Enable trading`

```
POST /api/v1/fiat/fvb/withdraw/create
```

**Request Body:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| amount | string | Yes | Withdrawal amount, must be greater than 50 |
| bankId | string | No | Bound bank ID (existing bank mode) |
| phone | string | No | Contact phone number |
| fileIds | string[] | No | List of proof file IDs |
| bank | object | No | Payout bank details (new bank mode), see below |

`bank` object:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| transferType | string | Yes | Transfer type, currently only `wire` is supported |
| bankAccountUs | object | Yes | US payout account, see below |
| bankAddress | object | Yes | Payout bank address, see below |
| billingDetails | object | Yes | Account holder details, see below |

`bank.bankAccountUs` — US payout account:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| accountNumber | string | Yes | Bank account number |
| routingNumber | string | Yes | ABA routing number (9 digits) |

`bank.bankAddress` — payout bank address:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| bankName | string | Yes | Bank name. Max 35 characters; the overflow is written into `line1` then `line2` in order |
| country | string | Yes | Bank country, ISO 3166-1 alpha-2 code, fixed to `US` |
| city | string | No | Bank city. May be empty `""` for the ABA method |
| district | string | No | State/province |
| line1 | string | No | Bank name characters 36–70 |
| line2 | string | No | Bank name characters 71 onward |

`bank.billingDetails` — account holder details:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| name | string | Yes | Holder name, must match the verified identity |
| country | string | Yes | Holder country, ISO 3166-1 alpha-2 code |
| city | string | Yes | City |
| line1 | string | Yes | Street address. Max 35 characters; the overflow is written into `line2` |
| line2 | string | No | Street address characters 36–70 |
| postalCode | string | Yes | Postal code |
| district | string | No | State/province. Required when country is `US` or `CA` |
| addressUrl | string | No | Path to the address proof file |

**Request Example — existing bank:**

```json
{
  "bankId": "8c66e62274375facda91ecffef83321c",
  "amount": "100",
  "fileIds": null,
  "phone": "+14235735382"
}
```

**Request Example — new bank:**

```json
{
  "bankName": "fv_bank",
  "amount": "100",
  "phone": "+14235735382",
  "fileIds": ["bank-cert/webot.us/c7d12f18e9e14f55b8e29950dd8c007b.png"],
  "bank": {
    "transferType": "wire",
    "bankAccountUs": { "routingNumber": "123456789", "accountNumber": "111111" },
    "bankAddress": { "country": "US", "city": "", "bankName": "11" },
    "billingDetails": {
      "name": "Jamesaweg Tayloraega", "city": "11", "country": "US",
      "line1": "11", "postalCode": "12321411", "district": "AL"
    }
  }
}
```

**Response Example:**

```json
{ "result": true, "data": { "orderId": "o-9" }, "timestamp": 1751000000000 }
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| orderId | string | Withdrawal order ID |

**Errors**

If the amount is ≤ 50, or a parameter is missing/invalid, `BOT_INVALID_ARGUMENT` is returned before the downstream request; see `message` for the specific reason.

Downstream business errors are passed through with `code` in the form `TAPI_FBO_*` and `message` as the human-readable description. The table below lists common codes (non-exhaustive); any downstream error not listed maps to `BOT_INTERNAL_ERROR`:

| code | Description |
|------|-------------|
| TAPI_FBO_PARAMETER_ERROR | Parameter validation failed (e.g. missing bank/fileIds, account does not match the bank card) |
| TAPI_FBO_BALANCE_ERROR | Insufficient balance |
| TAPI_FBO_BANK_ID_NOT_EXIST_ERROR | bankId does not exist or is unavailable |
| TAPI_FBO_CHANNEL_CLOSE_ERROR | Withdrawal channel is closed |
| TAPI_FBO_KYC_INFO_ERROR | KYC information validation failed |
| TAPI_FBO_NOT_SUPPORT_COIN_ERROR | Coin is not supported |
| TAPI_FBO_IN_BLACK_LIST_ERROR | User is on the blacklist |
| TAPI_FBO_QUESTIONNAIRE_NOT_EXIST_ERROR | Large-amount withdrawal questionnaire not completed |
| TAPI_FBO_SYSTEM_ERROR | Downstream system error |

---

### 6. List Payout Banks

Query the payout banks bound to the account, filtered by `transferType`.

**Permission required:** `Enable reading`

```
GET /api/v1/fiat/banks/list
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| transferType | string | Yes | Transfer type, currently only `wire` is supported |

**Response Example:**

```json
{
  "result": true,
  "data": {
    "banks": [
      {
        "bankId": "8c66e62274375facda91ecffef83321c",
        "transferType": "wire",
        "bankAccountUs": { "accountNumber": "111111", "routingNumber": "123456789" },
        "bankAddress": { "bankName": "11", "city": "", "country": "US", "district": "", "line1": "", "line2": "" },
        "billingDetails": {
          "name": "Jamesaweg Tayloraega", "line1": "11", "line2": "", "city": "11",
          "district": "AL", "country": "US", "postalCode": "12321411", "addressUrl": ""
        }
      }
    ],
    "uploadFiles": [
      { "fileId": "bank-cert/webot.us/xxx.png", "uploadUrl": "https://..." }
    ],
    "phone": "+14235735382"
  },
  "timestamp": 1751000000000
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| banks | object[] | Bank list. Each element has the same structure as the `bank` object in [Create Withdrawal](#5-create-withdrawal) (same payout bank structure); here `bankId` is populated |
| uploadFiles | object[] | Proof files: `{ fileId, uploadUrl }` |
| phone | string | Contact phone number |

---

### 7. Query Withdrawal Records

Query withdrawal records, filtered by time window, returning all matching records. Pass `orderId` to fetch a single order precisely (poll its status after creating a withdrawal).

**Permission required:** `Enable reading`

```
GET /api/v1/fiat/common/getWithdraws
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| channel | string | Yes | Withdrawal channel, currently only `fvbank` is supported |
| transferType | string | Yes | Transfer type: `wire` / `ach` |
| startTime | int64 | No | Start time (millisecond timestamp) |
| endTime | int64 | No | End time (millisecond timestamp) |
| orderId | string | No | Withdrawal order ID; if passed, only that order is returned |

**Response Example:**

```json
{
  "result": true,
  "data": {
    "records": [
      {
        "orderId": "order-001",
        "amount": "100",
        "realAmount": "99",
        "fee": "1",
        "status": "success",
        "createTime": 1700000000000,
        "remitTime": 1700000200000
      }
    ]
  },
  "timestamp": 1751000000000
}
```

**Response Fields** (`records[]` element):

| Field | Type | Description |
|-------|------|-------------|
| orderId | string | Order ID |
| amount | string | Requested amount |
| realAmount | string | Actual amount received |
| fee | string | Fee |
| status | string | Status: `success` (succeeded, terminal) / `fail` (failed, terminal) / `pending` (processing, keep polling) |
| createTime | int64 | Creation time (millisecond timestamp) |
| remitTime | int64 | Remittance time (millisecond timestamp) |
