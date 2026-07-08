# Webot US Open API

Official API documentation for the Webot US cryptocurrency exchange.

## Documentation

- [webot-openapi.md](./webot-openapi.md) — Full API reference covering public market data, private trading, and account balances endpoints.
- [webot-fiat-openapi.md](./webot-fiat-openapi.md) — Fiat Open API reference covering fiat deposit/withdrawal and stablecoin conversion (`/api/v1/fiat/*`).
- [webot-wallet-openapi.md](./webot-wallet-openapi.md) — Wallet Open API reference covering crypto asset deposit/withdrawal (`/api/v1/asset/*`).

## Overview

| | |
|---|---|
| Base URL | `https://api.webot.com` |
| Protocol | HTTPS |
| Data Format | JSON |
| Authentication | HMAC SHA256 signature |

### Public Endpoints (no authentication required)

| Endpoint | Description |
|----------|-------------|
| `GET /api/v1/common/symbols` | Trading pair information |
| `GET /api/v1/market/trades` | Recent trades |
| `GET /api/v1/market/depth` | Order book depth |
| `GET /api/v1/market/tickers` | 24-hour ticker statistics |
| `GET /api/v1/market/klines` | Kline / candlestick data |

### Private Endpoints (authentication required)

| Endpoint | Description |
|----------|-------------|
| `GET /api/v1/account/balances` | Account balances |
| `POST /api/v1/trade/order` | Place new order |
| `GET /api/v1/trade/order` | Get order details |
| `DELETE /api/v1/trade/order` | Cancel order |
| `POST /api/v1/trade/massOrder` | Place multiple orders |
| `GET /api/v1/trade/orderByClientOrderId` | Get order by client order ID |
| `GET /api/v1/trade/openOrders` | Get open orders |
| `GET /api/v1/trade/allOrders` | Get all orders |
| `DELETE /api/v1/trade/allOrders` | Cancel all orders |
| `GET /api/v1/trade/fills` | Get trade fills |
| `GET /api/v1/trade/fillsByOrderId` | Get fills by order ID |

### Fiat Endpoints

See [webot-fiat-openapi.md](./webot-fiat-openapi.md) for full details.

| Endpoint | Description |
|----------|-------------|
| `POST /api/v1/fiat/fvb/withdraw/create` | Create fiat withdrawal |
| `GET /api/v1/fiat/banks/list` | List payout banks |
| `GET /api/v1/fiat/common/getWithdraws` | Query withdrawal records |
| `GET /api/v1/fiat/deposit/getVirtualAccount` | Get deposit virtual account |
| `GET /api/v1/fiat/common/getDeposits` | Query deposit records |
| `POST /api/v1/fiat/convert/create` | Create stablecoin conversion |
| `GET /api/v1/fiat/convert/record` | Query conversion result |

### Wallet Endpoints

See [webot-wallet-openapi.md](./webot-wallet-openapi.md) for full details.

| Endpoint | Description |
|----------|-------------|
| `GET /api/v1/asset/currencies` | Query currency list (public, no auth) |
| `GET /api/v1/asset/address` | Query deposit address |
| `POST /api/v1/asset/withdraw` | Request crypto withdrawal |
| `GET /api/v1/asset/withdraw` | Query single withdrawal record |
| `GET /api/v1/asset/records` | Query deposit/withdrawal history |

## Quick Start

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

### 2. Configure credentials

Apply for an API Key at: https://www.webot.com/us/en/my-account/api

> **Note:** You need to contact the official team to enable API whitelist access before you can create an API Key.

```bash
cp .env.example .env
```

Edit `.env` and fill in your API Key and Secret:

```
API_KEY=your_api_key_here
API_SECRET=your_api_secret_here
```

### 3. Run the test script

```bash
python3 test_openapi.py
```

The script tests all public endpoints (no auth) and private endpoints (HMAC SHA256 signed), and prints a summary at the end.
