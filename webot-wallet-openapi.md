# Webot US Wallet Open API Documentation

HTTP API reference for crypto asset deposit/withdrawal (`/api/v1/asset/*`): currency queries, deposit addresses, withdrawal requests, and deposit/withdrawal record queries.

> This is part of the Webot US Open API. **General information and authentication are shared** — see the main reference [webot-openapi.md](./webot-openapi.md).

## General Information

Base URL (`https://api.webot.com`), HTTPS, JSON format, camelCase fields, the success/error response envelope, and data-type conventions are shared across all Webot US APIs. See the [General Information](./webot-openapi.md#general-information) section of the main reference.

### Endpoint Summary

| Method | Path | Description | Permission |
|--------|------|-------------|------------|
| GET | `/api/v1/asset/currencies` | Query currency list (public) | None (public) |
| GET | `/api/v1/asset/address` | Query deposit address | Enable reading |
| POST | `/api/v1/asset/withdraw` | Request crypto withdrawal | Enable trading |
| GET | `/api/v1/asset/withdraw` | Query single withdrawal record | Enable reading |
| GET | `/api/v1/asset/records` | Query deposit/withdrawal history | Enable reading |

> **Note:** For account balances, see `GET /api/v1/account/balances` in [webot-openapi.md](./webot-openapi.md#6-get-account-balances).

---

## Authentication

All Webot US APIs share the same authentication (API Key + HMAC SHA256 signature, `PIONEX-KEY` / `PIONEX-SIGNATURE` headers, and the `timestamp` query parameter). See the [Authentication](./webot-openapi.md#authentication) section of the main reference for the full signature construction steps. Public endpoints (marked below) do not require signing.

---

## Error Codes

Common error codes returned by the asset endpoints. See also the [Error Codes](./webot-openapi.md#error-codes) section of the main reference for shared authentication errors.

| Error Code | Description |
|------------|-------------|
| BOT_INVALID_ARGUMENT | Invalid argument |
| FROZEN_FORBIDDEN | Account deposit (`fz_deposit`) or withdrawal (`fz_withdraw`) function is frozen |
| KYC_FORBIDDEN | KYC verification is required |
| FORBIDDEN | Operation blocked after high-risk behavior; `data` contains `restrict_expired_on` (restriction expiry timestamp) and `restrict_ttl` (remaining restriction seconds) |
| SIGN_FORBIDDEN | Security signature verification failed |
| WITHDRAW_WITHELIST_ADDRESS_CLOSED | Address whitelist is not enabled (required for OpenAPI withdrawals) |
| WITHDRAW_ADDRESS_NOT_WHITELISTED | Withdrawal address is not in the whitelist |
| WITHDRAW_ADDRESS_NOT_IN_ALLOWED_LIST | Withdrawal address is not in the configured allowed list |

---

## Assets (Crypto Deposit/Withdrawal)

### 1. Query Currency List

Query the currencies supported by the platform and their chain information.

**Permission required:** None (public endpoint, no authentication)

```
GET /api/v1/asset/currencies
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| currency | string | No | Currency name; returns all currencies if omitted, max length 60 |

**Response Example:**

```json
{
  "result": true,
  "data": {
    "currencies": [
      {
        "currency": "BTC",
        "displayName": "BTC",
        "fullName": "Bitcoin",
        "chainList": [
          {
            "chain": "BTC",
            "txType": "...",
            "depositEnable": true,
            "withdrawEnable": true,
            "depositMin": "0.0001",
            "withdrawMin": "0.001",
            "withdrawMax": "100",
            "withdrawFee": "0.0005",
            "hasTag": false,
            "confirm": 2,
            "withdrawPrecision": 8,
            "contractAddress": "",
            "walletType": "...",
            "preConfirm": 1
          }
        ]
      }
    ]
  },
  "timestamp": 1679900000000
}
```

**Response Fields** (`currencies[]` element):

| Field | Type | Description |
|-------|------|-------------|
| currency | string | Currency identifier |
| displayName | string | Currency display name |
| fullName | string | Currency full name |
| chainList | array | Supported chain list, see below |

`chainList[]` element:

| Field | Type | Description |
|-------|------|-------------|
| chain | string | Chain name |
| txType | string | Transaction type |
| depositEnable | boolean | Whether deposits are supported |
| withdrawEnable | boolean | Whether withdrawals are supported |
| depositMin | string | Minimum deposit amount |
| withdrawMin | string | Minimum withdrawal amount |
| withdrawMax | string | Maximum withdrawal amount |
| withdrawFee | string | Withdrawal fee |
| hasTag | boolean | Whether a Tag/Memo is required |
| confirm | integer | Deposit confirmations |
| withdrawPrecision | integer | Withdrawal precision (decimal places) |
| contractAddress | string | Contract address |
| walletType | string | Wallet type |
| preConfirm | integer | Pre-confirmations |

---

### 2. Query Deposit Address

Query the deposit address for a specified currency and chain.

**Permission required:** `Enable reading`

```
GET /api/v1/asset/address
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| currency | string | Yes | Currency name, max length 60 |
| chain | string | Yes | Chain name, max length 60 |

**Response Example:**

```json
{
  "result": true,
  "data": {
    "address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
    "tag": ""
  },
  "timestamp": 1679900000000
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| address | string | Deposit address |
| tag | string | Address Tag/Memo (required by some chains, e.g. XRP, EOS) |

> **Note:** Possible errors: `FROZEN_FORBIDDEN`, `KYC_FORBIDDEN`. See [Error Codes](#error-codes).

---

### 3. Request Withdrawal

Submit a withdrawal request. OpenAPI requires the user to enable the address whitelist, and the withdrawal address must be in the whitelist.

**Permission required:** `Enable trading`

```
POST /api/v1/asset/withdraw
```

**Request Body:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| currency | string | Yes | Currency name, max length 60 |
| chain | string | Yes | Chain name, max length 60 |
| clientId | string | Yes | Client-defined ID (idempotency key), max length 64 |
| address | string | Yes | Withdrawal destination address, max length 300 |
| tag | string | No | Address Tag/Memo, max length 100 |
| amount | string | Yes | Withdrawal amount, max length 60 |
| note | string | No | Note, max length 100 |

**Request Example:**

```json
{
  "currency": "USDT",
  "chain": "TRC20",
  "clientId": "my-withdraw-001",
  "address": "TXxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "amount": "100.5",
  "note": "test withdraw"
}
```

**Response Example:**

```json
{
  "result": true,
  "data": {
    "id": "123456789",
    "clientId": "my-withdraw-001"
  },
  "timestamp": 1679900000000
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| id | string | System-generated withdrawal order ID |
| clientId | string | Client-defined ID |

> **Note:** Possible errors: `FROZEN_FORBIDDEN`, `KYC_FORBIDDEN`, `FORBIDDEN`, `SIGN_FORBIDDEN`, `WITHDRAW_WITHELIST_ADDRESS_CLOSED`, `WITHDRAW_ADDRESS_NOT_WHITELISTED`, `WITHDRAW_ADDRESS_NOT_IN_ALLOWED_LIST`. See [Error Codes](#error-codes).

---

### 4. Query Single Withdrawal Record

Query a single withdrawal record by system order ID or client-defined ID.

**Permission required:** `Enable reading`

```
GET /api/v1/asset/withdraw
```

**Request Parameters** (at least one of `id` and `client_id` must be provided):

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | No* | System order ID, max length 64 |
| client_id | string | No* | Client-defined ID, max length 64 |

**Response Example:**

```json
{
  "result": true,
  "data": {
    "id": "123456789",
    "clientId": "my-withdraw-001",
    "currency": "USDT",
    "chain": "TRC20",
    "type": "withdraw",
    "internal": false,
    "address": "TXxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "tag": "",
    "addressFrom": "",
    "tagFrom": "",
    "amount": "100.5",
    "fee": "1",
    "status": "completed",
    "hash": "abc123def456...",
    "confirmations": 20,
    "note": "test withdraw",
    "approvalReason": "",
    "createTime": 1679900000000,
    "updateTime": 1679900100000
  },
  "timestamp": 1679900000000
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| id | string | System order ID |
| clientId | string | Client-defined ID |
| currency | string | Currency |
| chain | string | Chain name |
| type | string | Transaction type (`deposit` / `withdraw`) |
| internal | boolean | Whether it is an internal transfer |
| address | string | Destination address |
| tag | string | Destination address Tag/Memo |
| addressFrom | string | Source address |
| tagFrom | string | Source address Tag/Memo |
| amount | string | Amount |
| fee | string | Fee |
| status | string | Status |
| hash | string | On-chain transaction hash |
| confirmations | integer | Confirmations |
| note | string | Note |
| approvalReason | string | Approval reason |
| createTime | integer | Creation time (millisecond timestamp) |
| updateTime | integer | Update time (millisecond timestamp) |

---

### 5. Query Deposit/Withdrawal History

Query the history list of deposits and withdrawals.

**Permission required:** `Enable reading`

```
GET /api/v1/asset/records
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| currency | string | No | Currency name; queries all if omitted, max length 60 |
| type | string | No | Transaction type (`deposit` / `withdraw`); queries all if omitted, max length 30 |
| start_time | integer | No | Start time (millisecond timestamp) |
| end_time | integer | No | End time (millisecond timestamp) |

**Response Example:**

```json
{
  "result": true,
  "data": {
    "records": [
      {
        "id": "123456789",
        "clientId": "my-withdraw-001",
        "currency": "USDT",
        "chain": "TRC20",
        "type": "withdraw",
        "internal": false,
        "address": "TXxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "tag": "",
        "addressFrom": "",
        "tagFrom": "",
        "amount": "100.5",
        "fee": "1",
        "status": "completed",
        "hash": "abc123def456...",
        "confirmations": 20,
        "note": "",
        "approvalReason": "",
        "createTime": 1679900000000,
        "updateTime": 1679900100000
      }
    ]
  },
  "timestamp": 1679900000000
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| records | object[] | Deposit/withdrawal records; each element has the same fields as `data` in [Query Single Withdrawal Record](#4-query-single-withdrawal-record) |
