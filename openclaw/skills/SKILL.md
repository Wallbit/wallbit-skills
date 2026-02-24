---
name: wallbit-skill
version: 1.0.0
description: Integration with the Wallbit public API for balances, transactions, trades, assets and wallets. Use when working with Wallbit API, trading endpoints, account balances, stock portfolio, investment operations, or when the user mentions Wallbit.
metadata:
  {
    "openclaw":
      {
        "emoji": "📈",
        "category": "finance",
        "primaryEnv": "WALLBIT_API_KEY",
        "requires": { "env": ["WALLBIT_API_KEY"] },
      },
  }
---

```
WALLBIT API QUICK REFERENCE v1.0.0
Base:   https://api.wallbit.io
Auth:   X-API-Key: <WALLBIT_API_KEY>
Docs:   This file is canonical (skills guide + params)

Key endpoints:
  GET  /api/public/v1/balance/checking       -> checking account balance
  GET  /api/public/v1/balance/stocks          -> investment portfolio
  GET  /api/public/v1/transactions            -> transaction history
  POST /api/public/v1/trades                  -> buy/sell (BUY/SELL)
  GET  /api/public/v1/account-details          -> bank account details
  GET  /api/public/v1/wallets                 -> crypto wallet addresses
  GET  /api/public/v1/assets                  -> list assets
  GET  /api/public/v1/assets/{symbol}         -> single asset info
  POST /api/public/v1/operations/internal     -> deposit/withdraw to investment

Rules: requests are JSON for POST | response data in "data" | camelCase for code
Errors: HTTP status + JSON message/errors
```

# Wallbit Public API - Agent Skills Guide

This skill is **doc-only**. Agents may use Wallbit MCP tools when available, or call the REST API directly.

## Available MCP Tools

When the Wallbit MCP server is configured, use these tools instead of building requests by hand:

| MCP Tool                           | Endpoint                              | Description              |
| ---------------------------------- | ------------------------------------- | ------------------------ |
| `mcp_wallbit_get_checking_balance` | GET `/api/public/v1/balance/checking` | Checking account balance |
| `mcp_wallbit_get_stocks_balance`   | GET `/api/public/v1/balance/stocks`   | Investment portfolio     |
| `mcp_wallbit_list_transactions`    | GET `/api/public/v1/transactions`     | History with pagination  |
| `mcp_wallbit_create_trade`         | POST `/api/public/v1/trades`          | Execute buy or sell      |
| `mcp_wallbit_get_asset`            | GET `/api/public/v1/assets/{symbol}`  | Single asset info        |

For account-details, wallets, listing assets with filters, and operations/internal, use the REST API directly (see examples below).

## Base URL and authentication

- **Base URL**: `https://api.wallbit.io`
- **Header**: `X-API-Key: <WALLBIT_API_KEY>`
- **Content-Type**: `application/json` (POST)
- **Accept**: `application/json`

```bash
export WALLBIT_API_URL="https://api.wallbit.io"
export WALLBIT_API_KEY="your_api_key"
```

## Tool -> Endpoint Map

| Tool / Action        | Method | Path                                 | Params/Body                                                                                                                               |
| -------------------- | ------ | ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| get_checking_balance | GET    | `/api/public/v1/balance/checking`    | —                                                                                                                                         |
| get_stocks_balance   | GET    | `/api/public/v1/balance/stocks`      | —                                                                                                                                         |
| list_transactions    | GET    | `/api/public/v1/transactions`        | Query: `page`, `limit`, `status`, `type`, `currency`, `from_date`, `to_date`, `from_amount`, `to_amount`                                  |
| create_trade         | POST   | `/api/public/v1/trades`              | Body: `symbol`, `direction` (BUY/SELL), `currency`, `order_type`, `amount` or `shares`, opt. `limit_price`, `stop_price`, `time_in_force` |
| get_account_details  | GET    | `/api/public/v1/account-details`     | Query: `country`, `currency`                                                                                                              |
| get_wallets          | GET    | `/api/public/v1/wallets`             | Query: `currency`, `network`                                                                                                              |
| list_assets          | GET    | `/api/public/v1/assets`              | Query: `category`, `search`, `page`, `limit`                                                                                              |
| get_asset            | GET    | `/api/public/v1/assets/{symbol}`     | Path: `symbol`                                                                                                                            |
| internal_transfer    | POST   | `/api/public/v1/operations/internal` | Body: `currency`, `from` (DEFAULT/INVESTMENT), `to`, `amount`                                                                             |

Body field names: see [api-reference.md](api-reference.md) (API uses snake_case, e.g. `limit_price`, `time_in_force`). In PHP/Laravel code use camelCase and docstrings.

## Quick Start

### 1) Balance checking

```bash
curl -sS "$WALLBIT_API_URL/api/public/v1/balance/checking" \
  -H "X-API-Key: $WALLBIT_API_KEY" \
  -H "Accept: application/json"
```

### 2) Stock portfolio

```bash
curl -sS "$WALLBIT_API_URL/api/public/v1/balance/stocks" \
  -H "X-API-Key: $WALLBIT_API_KEY" \
  -H "Accept: application/json"
```

### 3) Market buy

```bash
curl -sS -X POST "$WALLBIT_API_URL/api/public/v1/trades" \
  -H "X-API-Key: $WALLBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "symbol": "AAPL",
    "direction": "BUY",
    "currency": "USD",
    "order_type": "MARKET",
    "amount": 100
  }'
```

## Examples by category

### Transactions (history)

```bash
curl -sS "$WALLBIT_API_URL/api/public/v1/transactions?page=1&limit=10&status=COMPLETED" \
  -H "X-API-Key: $WALLBIT_API_KEY" \
  -H "Accept: application/json"
```

### Market sell

```bash
curl -sS -X POST "$WALLBIT_API_URL/api/public/v1/trades" \
  -H "X-API-Key: $WALLBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "symbol": "TSLA",
    "direction": "SELL",
    "currency": "USD",
    "order_type": "MARKET",
    "shares": 2.5
  }'
```

### Limit order

```bash
curl -sS -X POST "$WALLBIT_API_URL/api/public/v1/trades" \
  -H "X-API-Key: $WALLBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "symbol": "AAPL",
    "direction": "BUY",
    "currency": "USD",
    "order_type": "LIMIT",
    "amount": 200,
    "limit_price": 170,
    "time_in_force": "GTC"
  }'
```

### Account details (bank account)

```bash
curl -sS "$WALLBIT_API_URL/api/public/v1/account-details?country=US&currency=USD" \
  -H "X-API-Key: $WALLBIT_API_KEY" \
  -H "Accept: application/json"
```

### Crypto wallets

```bash
curl -sS "$WALLBIT_API_URL/api/public/v1/wallets?currency=USDC&network=ethereum" \
  -H "X-API-Key: $WALLBIT_API_KEY" \
  -H "Accept: application/json"
```

### List assets (with filters)

```bash
curl -sS "$WALLBIT_API_URL/api/public/v1/assets?category=TECHNOLOGY&search=apple&page=1&limit=20" \
  -H "X-API-Key: $WALLBIT_API_KEY" \
  -H "Accept: application/json"
```

### Single asset info

```bash
curl -sS "$WALLBIT_API_URL/api/public/v1/assets/AAPL" \
  -H "X-API-Key: $WALLBIT_API_KEY" \
  -H "Accept: application/json"
```

### Deposit to investment (DEFAULT → INVESTMENT)

```bash
curl -sS -X POST "$WALLBIT_API_URL/api/public/v1/operations/internal" \
  -H "X-API-Key: $WALLBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "currency": "USD",
    "from": "DEFAULT",
    "to": "INVESTMENT",
    "amount": 500
  }'
```

### Withdrawal from investment (INVESTMENT → DEFAULT)

```bash
curl -sS -X POST "$WALLBIT_API_URL/api/public/v1/operations/internal" \
  -H "X-API-Key: $WALLBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "currency": "USD",
    "from": "INVESTMENT",
    "to": "DEFAULT",
    "amount": 200
  }'
```

## Error Handling

| Code | Description                       | Action                                                        |
| ---- | --------------------------------- | ------------------------------------------------------------- |
| 401  | Invalid or missing API Key        | Check X-API-Key header                                        |
| 403  | Insufficient permissions          | Check API Key permissions                                     |
| 412  | Incomplete KYC or blocked account | Complete verification in app                                  |
| 422  | Validation error                  | Check body/query against [api-reference.md](api-reference.md) |
| 429  | Rate limit exceeded               | Wait `retry_after` seconds                                    |

Error responses include `message` and sometimes `errors` (validation) or `your_permissions` (403).

## Rate Limiting

Response headers:

- `X-RateLimit-Limit`: requests per minute allowed
- `X-RateLimit-Remaining`: remaining requests
- `X-RateLimit-Reset`: Unix timestamp of reset
- `Retry-After`: seconds until retry (on 429)

## Quick reference

### Supported currencies

- Transactions: USD, EUR, ARS, MXN, USDC, USDT, BOB, COP, PEN, DOP
- Account details: USD, EUR (countries: US, EU)
- Wallets: USDT, USDC (networks: ethereum, arbitrum, solana, polygon, tron)

### Asset categories

MOST_POPULAR, ETF, DIVIDENDS, TECHNOLOGY, HEALTH, CONSUMER_GOODS, ENERGY_AND_WATER, FINANCE, REAL_ESTATE, TREASURY_BILLS, VIDEOGAMES, ARGENTINA_ADR

### Order types (trades)

- `MARKET`: market price (requires `amount` or `shares`)
- `LIMIT`: requires `limit_price` and `time_in_force` (DAY, GTC)
- `STOP`: requires `stop_price`
- `STOP_LIMIT`: requires `stop_price` and `limit_price`

## Code generation guidelines

1. **PHP/Laravel**: camelCase for functions, PHPDoc docstrings on all functions
2. Validate parameters against OpenAPI / api-reference types
3. Always handle 401, 403, 412, 422, 429
4. Do not hardcode API keys; use environment variables

## Additional Resources

- Detailed endpoint documentation: [api-reference.md](api-reference.md)
- Full PHP, JavaScript, Python examples: [examples.md](examples.md)
