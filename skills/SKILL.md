---
name: wallbit-skills
description: Integration with the Wallbit public API to query balances, transactions, execute trades, fees, exchange rates, assets, wallets, cards, and API key management. Use when working with Wallbit API, trading endpoints, account balances, stock portfolio, investment operations, or when the user mentions Wallbit.
---

# Wallbit Public API

REST API to integrate Wallbit functionality: query balances, view transaction history, execute trades, get fee configuration, exchange rates, manage cards, and more.

**OpenAPI source**: `api-wallbit/docs/public-api/openapi-public-api.json` (v1.0.0)

## Quick Start

**Base URL**: `https://api.wallbit.io` (local: `http://localhost`)

**Authentication**: `X-API-Key` header required on all requests.

```bash
curl -H "X-API-Key: $WALLBIT_API_KEY" \
  -H "Accept: application/json" \
  https://api.wallbit.io/api/public/v1/balance/checking
```

## Endpoints Overview

| Category        | Endpoint                                 | Method | Description                                    |
| --------------- | ---------------------------------------- | ------ | ---------------------------------------------- |
| Balance         | `/api/public/v1/balance/checking`        | GET    | Checking account balance (positive currencies) |
| Balance         | `/api/public/v1/balance/stocks`          | GET    | Investment portfolio (stocks + USD cash)       |
| Transactions    | `/api/public/v1/transactions`            | GET    | Transaction history (pagination + filters)     |
| Trades          | `/api/public/v1/trades`                  | POST   | Execute buy/sell                               |
| Fees            | `/api/public/v1/fees`                    | POST   | Fee config for user tier (`type`: TRADE)       |
| Account Details | `/api/public/v1/account-details`         | GET    | Bank account details (US/EU)                   |
| Wallets         | `/api/public/v1/wallets`                 | GET    | Crypto wallet addresses                        |
| Rates           | `/api/public/v1/rates`                   | GET    | Fiat exchange rate for a currency pair         |
| Assets          | `/api/public/v1/assets`                  | GET    | List available assets                          |
| Assets          | `/api/public/v1/assets/{symbol}`         | GET    | Specific asset info                            |
| Operations      | `/api/public/v1/operations/internal`     | POST   | Deposit/withdraw between DEFAULT ↔ INVESTMENT  |
| Cards           | `/api/public/v1/cards`                   | GET    | List active or suspended cards                 |
| Cards           | `/api/public/v1/cards/{cardUuid}/status` | PATCH  | Block (`SUSPENDED`) or unblock (`ACTIVE`) card |
| API Key         | `/api/public/v1/api-key`                 | DELETE | Revoke the API key used in the request         |

## Authentication

### PHP/Laravel

```php
use Illuminate\Support\Facades\Http;

/**
 * Creates an HTTP client configured for the Wallbit API.
 *
 * @return \Illuminate\Http\Client\PendingRequest
 */
function createWallbitClient()
{
    return Http::withHeaders([
        'X-API-Key' => config('services.wallbit.api_key'),
        'Accept' => 'application/json',
    ])->baseUrl('https://api.wallbit.io');
}
```

### JavaScript

```javascript
const wallbitClient = {
  baseUrl: "https://api.wallbit.io",
  apiKey: process.env.WALLBIT_API_KEY,

  async request(endpoint, options = {}) {
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      ...options,
      headers: {
        "X-API-Key": this.apiKey,
        Accept: "application/json",
        "Content-Type": "application/json",
        ...options.headers,
      },
    });
    return response.json();
  },
};
```

### Python

```python
import requests

class WallbitClient:
    def __init__(self, api_key: str):
        self.base_url = "https://api.wallbit.io"
        self.headers = {
            "X-API-Key": api_key,
            "Accept": "application/json"
        }

    def request(self, method: str, endpoint: str, **kwargs):
        url = f"{self.base_url}{endpoint}"
        response = requests.request(method, url, headers=self.headers, **kwargs)
        response.raise_for_status()
        return response.json()
```

## Error Handling

| Code | Description                       | Action                           |
| ---- | --------------------------------- | -------------------------------- |
| 400  | Insufficient funds (trades)       | Check balance before trading     |
| 401  | Invalid or missing API Key        | Check X-API-Key header           |
| 403  | Insufficient permissions          | Check API Key permissions        |
| 404  | Resource not found                | Verify symbol, rate pair, card   |
| 412  | Incomplete KYC or blocked account | Complete verification in the app |
| 422  | Validation error                  | Review sent parameters           |
| 429  | Rate limit exceeded               | Wait `retry_after` seconds       |
| 503  | Provider unavailable (cards)      | Retry later                      |

### Error handling example (PHP/Laravel)

```php
/**
 * Executes a Wallbit API request with error handling.
 *
 * @param string $method
 * @param string $endpoint
 * @param array $data
 * @return array
 * @throws \Exception
 */
function wallbitRequest(string $method, string $endpoint, array $data = []): array
{
    $client = createWallbitClient();

    $response = match($method) {
        'GET' => $client->get($endpoint, $data),
        'POST' => $client->post($endpoint, $data),
        'PATCH' => $client->patch($endpoint, $data),
        'DELETE' => $client->delete($endpoint, $data),
        default => throw new \Exception("Unsupported method: {$method}")
    };

    if ($response->status() === 429) {
        $retryAfter = $response->json('retry_after', 60);
        throw new \Exception("Rate limit exceeded. Retry in {$retryAfter} seconds.");
    }

    if ($response->status() === 401) {
        throw new \Exception("Invalid or missing API Key.");
    }

    if ($response->status() === 403) {
        $permissions = $response->json('your_permissions', []);
        throw new \Exception("Insufficient permissions. You have: " . implode(', ', $permissions));
    }

    if ($response->status() === 422) {
        $errors = $response->json('errors', []);
        throw new \Exception("Validation error: " . json_encode($errors));
    }

    if (!$response->successful()) {
        throw new \Exception($response->json('message', 'Unknown error'));
    }

    return $response->json('data') ?? $response->json();
}
```

## Rate Limiting

Response headers:

- `X-RateLimit-Limit`: Requests allowed per minute
- `X-RateLimit-Remaining`: Remaining requests
- `X-RateLimit-Reset`: Unix timestamp of reset
- `Retry-After`: Seconds until retry is allowed (only on 429)

## Code Generation Guidelines

When generating code for this API:

1. **PHP/Laravel**: Use camelCase for functions, include PHPDoc docstrings
2. **Validate parameters** before sending according to OpenAPI spec types
3. **Always handle errors** 400, 401, 403, 404, 412, 422, 429 (503 for cards)
4. **Do not hardcode API Keys**, use environment variables
5. **Request body fields** use snake_case in JSON (`limit_price`, `time_in_force`); use camelCase in application code

### Supported currencies

- **Transactions** (`currency` filter): `USD`, `EUR`, `ARS`, `MXN`, `USDC`, `USDT`, `BOB`, `COP`, `PEN`, `DOP`, `BRL`, `PHP`, `CLP`, `GTQ`, `PAB`, `CRC`
- **Account Details**: `USD`, `EUR` (countries: `US`, `EU`)
- **Wallets**: `USDT`, `USDC` (networks: `ethereum`, `arbitrum`, `solana`, `polygon`, `tron`)
- **Rates**: any valid Wallbit currency code for `source_currency` and `dest_currency`

### API Key permissions

Some endpoints require scopes on the API key:

- **`read`**: balances, transactions, account-details, wallets, rates, assets, fees, list cards
- **`trade`**: trades, internal operations, update card status
- **Revoke API key** (`DELETE /api-key`): any valid key may revoke itself (no `read`/`trade` required)

### Asset Categories

`MOST_POPULAR`, `ETF`, `DIVIDENDS`, `TECHNOLOGY`, `HEALTH`, `CONSUMER_GOODS`, `ENERGY_AND_WATER`, `FINANCE`, `REAL_ESTATE`, `TREASURY_BILLS`, `VIDEOGAMES`, `ARGENTINA_ADR`

### Trade Order Types

- `MARKET`: Market order (executes immediately; use `amount` or `shares`, not both)
- `LIMIT`: Limit order (requires `limit_price` and `time_in_force`: `DAY` or `GTC`)
- `STOP`: Stop order (requires `stop_price`)
- `STOP_LIMIT`: Stop-limit order (requires `stop_price` and `limit_price`)

### Fee types (`POST /fees`)

- `TRADE`: Stock trading fees for the user's investment subscription tier (`percentage_fee`, `fixed_fee_usd`); empty `data` array if no row matches

### Card status

- `ACTIVE`: card unblocked
- `SUSPENDED`: card blocked
