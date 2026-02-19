# Wallbit API Reference

Detailed documentation for all Wallbit public API endpoints.

---

## Balance Endpoints

### GET /api/public/v1/balance/checking

Gets the available balances in the user's checking account.

**Parameters**: None

**Response 200**:
```json
{
  "data": [
    {
      "currency": "USD",
      "balance": 1000.50
    }
  ]
}
```

**Notes**: Only returns currencies with a positive balance.

---

### GET /api/public/v1/balance/stocks

Gets the user's investment portfolio (stocks).

**Parameters**: None

**Response 200**:
```json
{
  "data": [
    {
      "symbol": "AAPL",
      "shares": 10.5
    },
    {
      "symbol": "TSLA",
      "shares": 5.25
    },
    {
      "symbol": "USD",
      "shares": 500.00
    }
  ]
}
```

**Notes**: `USD` in the response represents the USD balance available in the investment account.

---

## Transactions Endpoints

### GET /api/public/v1/transactions

Lists the transaction history with pagination and filters.

**Query Parameters**:

| Parameter   | Type    | Required | Description |
|-------------|---------|----------|-------------|
| page        | integer | No       | Page number (default: 1) |
| limit       | integer | No       | Results per page: 10, 20, 50 (default: 10) |
| status      | string  | No       | Filter by status (e.g. COMPLETED) |
| type        | string  | No       | Filter by type (e.g. TRADE) |
| currency    | string  | No       | Currency: USD, EUR, ARS, MXN, USDC, USDT, BOB, COP, PEN, DOP |
| from_date   | date    | No       | Start date (format: Y-m-d) |
| to_date     | date    | No       | End date (format: Y-m-d) |
| from_amount | number  | No       | Minimum amount |
| to_amount   | number  | No       | Maximum amount |

**Response 200**:
```json
{
  "data": {
    "data": [
      {
        "uuid": "550e8400-e29b-41d4-a716-446655440000",
        "type_id": 1,
        "source_currency": {
          "code": "USD",
          "alias": "USD"
        },
        "dest_currency": {
          "code": "USD",
          "alias": "USD"
        },
        "source_amount": 100.00,
        "dest_amount": 100.00,
        "status": "COMPLETED",
        "created_at": "2024-01-15T10:30:00.000000Z",
        "comment": null
      }
    ],
    "pages": 5,
    "current_page": 1,
    "count": 50
  }
}
```

---

## Trades Endpoints

### POST /api/public/v1/trades

Executes a buy (BUY) or sell (SELL) operation on assets.

**Request Body**:

| Field         | Type   | Required    | Description |
|---------------|--------|-------------|-------------|
| symbol        | string | Yes         | Asset symbol (e.g. AAPL, TSLA) |
| direction     | string | Yes         | BUY or SELL |
| currency      | string | Yes         | Currency (USD only supported) |
| order_type    | string | Yes         | MARKET, LIMIT, STOP, STOP_LIMIT |
| amount        | number | Conditional | Amount in USD (required if shares not specified) |
| shares        | number | Conditional | Number of shares (required if amount not specified) |
| limit_price   | number | Conditional | Limit price (required for LIMIT and STOP_LIMIT) |
| stop_price    | number | Conditional | Stop price (required for STOP and STOP_LIMIT) |
| time_in_force | string | Conditional | DAY or GTC (required for LIMIT) |

**Market Buy by amount example**:
```json
{
  "symbol": "AAPL",
  "direction": "BUY",
  "currency": "USD",
  "order_type": "MARKET",
  "amount": 100.00
}
```

**Limit Sell by shares example**:
```json
{
  "symbol": "TSLA",
  "direction": "SELL",
  "currency": "USD",
  "order_type": "LIMIT",
  "shares": 2.0,
  "limit_price": 250.00,
  "time_in_force": "GTC"
}
```

**Response 200**:
```json
{
  "data": {
    "symbol": "AAPL",
    "direction": "BUY",
    "amount": 100.00,
    "shares": 0.5847953,
    "status": "REQUESTED",
    "order_type": "MARKET",
    "created_at": "2024-01-15T10:30:00.000000Z",
    "updated_at": "2024-01-15T10:30:00.000000Z"
  }
}
```

**Specific errors**:
- 400: Insufficient funds
- 412: Incomplete investment KYC

---

## Account Details Endpoints

### GET /api/public/v1/account-details

Gets the user's bank account details.

**Query Parameters**:

| Parameter | Type   | Default | Description |
|-----------|--------|---------|-------------|
| country   | string | US      | Country: US, EU |
| currency  | string | USD     | Currency: USD, EUR |

**Response 200 (US Account - ACH)**:
```json
{
  "data": {
    "bank_name": "Community Federal Savings Bank",
    "currency": "USD",
    "account_type": "CHECKING",
    "account_number": "9876543210",
    "routing_number": "026073150",
    "swift_code": "CTFNUS33",
    "holder_name": "Juan Pérez",
    "address": {
      "street_line_1": "123 Main St",
      "city": "New York",
      "state": "NY",
      "postal_code": "10001",
      "country": "US"
    }
  }
}
```

**Response 200 (EU Account - SEPA)**:
```json
{
  "data": {
    "bank_name": "Deutsche Bank",
    "currency": "EUR",
    "account_type": "CHECKING",
    "iban": "DE89370400440532013000",
    "bic": "COBADEFFXXX",
    "holder_name": "Juan Pérez",
    "address": {
      "street_line_1": "Hauptstraße 1",
      "city": "Berlin",
      "postal_code": "10115",
      "country": "DE"
    }
  }
}
```

**Specific errors**:
- 422: Unverified account or unsupported country

---

## Wallets Endpoints

### GET /api/public/v1/wallets

Gets the crypto wallet addresses for receiving deposits.

**Query Parameters**:

| Parameter | Type   | Description |
|-----------|--------|-------------|
| currency  | string | Filter by currency: USDT, USDC |
| network   | string | Filter by network: ethereum, arbitrum, solana, polygon, tron |

**Response 200**:
```json
{
  "data": [
    {
      "address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
      "network": "ethereum",
      "currency_code": "USDT"
    },
    {
      "address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
      "network": "ethereum",
      "currency_code": "USDC"
    }
  ]
}
```

---

## Assets Endpoints

### GET /api/public/v1/assets

Lists the assets available for trading with pagination and filters.

**Query Parameters**:

| Parameter | Type    | Default | Description |
|-----------|---------|---------|-------------|
| category  | string  | -       | Asset category (see list below) |
| search    | string  | -       | Search by symbol, name or keywords (max 100 chars) |
| page      | integer | 1       | Page number |
| limit     | integer | 10      | Assets per page (max: 50) |

**Available categories**:
- MOST_POPULAR
- ETF
- DIVIDENDS
- TECHNOLOGY
- HEALTH
- CONSUMER_GOODS
- ENERGY_AND_WATER
- FINANCE
- REAL_ESTATE
- TREASURY_BILLS
- VIDEOGAMES
- ARGENTINA_ADR

**Response 200**:
```json
{
  "data": [
    {
      "symbol": "AAPL",
      "name": "Apple Inc.",
      "price": 175.50,
      "asset_type": "Stock",
      "exchange": "NASDAQ",
      "sector": "Technology",
      "market_cap_m": "2750000",
      "description": "Apple Inc. designs, manufactures...",
      "logo_url": "https://static.atomicvest.com/AAPL.svg"
    }
  ],
  "pages": 15,
  "current_page": 1,
  "count": 150
}
```

---

### GET /api/public/v1/assets/{symbol}

Gets detailed information about a specific asset.

**Path Parameters**:

| Parameter | Type   | Required | Description |
|-----------|--------|----------|-------------|
| symbol    | string | Yes      | Asset symbol (e.g. AAPL) |

**Response 200**:
```json
{
  "data": {
    "symbol": "AAPL",
    "name": "Apple Inc.",
    "price": 175.50,
    "asset_type": "Stock",
    "exchange": "NASDAQ",
    "sector": "Technology",
    "market_cap_m": "2750000",
    "description": "Apple Inc. designs, manufactures, and markets smartphones...",
    "description_es": "Apple Inc. diseña, fabrica y comercializa teléfonos inteligentes...",
    "country": "United States",
    "ceo": "Tim Cook",
    "employees": "164000",
    "logo_url": "https://static.atomicvest.com/AAPL.svg",
    "dividend": {
      "amount": 0.24,
      "yield": 0.52,
      "ex_date": "2024-02-09",
      "payment_date": "2024-02-15"
    }
  }
}
```

**Specific errors**:
- 404: Asset not found

---

## Operations Endpoints

### POST /api/public/v1/operations/internal

Moves funds between the checking account (DEFAULT) and the investment account (INVESTMENT).

**Request Body**:

| Field    | Type   | Required | Description |
|----------|--------|----------|-------------|
| currency | string | Yes      | Currency code (e.g. USD) |
| from     | string | Yes      | Source account: DEFAULT or INVESTMENT |
| to       | string | Yes      | Destination account: DEFAULT or INVESTMENT |
| amount   | number | Yes      | Amount to transfer (min: 1, max: 999999) |

**Deposit to investment example**:
```json
{
  "currency": "USD",
  "from": "DEFAULT",
  "to": "INVESTMENT",
  "amount": 100.00
}
```

**Withdrawal from investment example**:
```json
{
  "currency": "USD",
  "from": "INVESTMENT",
  "to": "DEFAULT",
  "amount": 50.00
}
```

**Response 200**: Returns a Transaction object.

**Specific errors**:
- 412: Blocked account, incomplete KYC, or account in migration
- 422: Insufficient funds or validation error

---

## Common Error Responses

### 401 Unauthorized
```json
{
  "message": "API Key required. Please provide X-API-Key header."
}
```
```json
{
  "message": "Invalid or expired API Key."
}
```

### 403 Forbidden
```json
{
  "message": "Insufficient permissions. Required permission: read",
  "your_permissions": ["trade"]
}
```

### 422 Validation Error
```json
{
  "message": "The given data was invalid.",
  "errors": {
    "symbol": ["The symbol field is required."]
  }
}
```

### 429 Too Many Requests
```json
{
  "message": "Too many requests. Please try again later.",
  "retry_after": 42
}
```
