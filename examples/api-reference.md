# Wallbit API Reference

Documentación detallada de todos los endpoints de la API pública de Wallbit.

---

## Balance Endpoints

### GET /api/public/v1/balance/checking

Obtiene los balances disponibles en la cuenta checking del usuario.

**Parámetros**: Ninguno

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

**Notas**: Solo retorna monedas con balance positivo.

---

### GET /api/public/v1/balance/stocks

Obtiene el portfolio de inversión (stocks) del usuario.

**Parámetros**: Ninguno

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

**Notas**: `USD` en la respuesta representa el balance en USD disponible en la cuenta de inversión.

---

## Transactions Endpoints

### GET /api/public/v1/transactions

Lista el historial de transacciones con paginación y filtros.

**Parámetros Query**:

| Parámetro | Tipo | Requerido | Descripción |
|-----------|------|-----------|-------------|
| page | integer | No | Número de página (default: 1) |
| limit | integer | No | Resultados por página: 10, 20, 50 (default: 10) |
| status | string | No | Filtrar por estado (ej: COMPLETED) |
| type | string | No | Filtrar por tipo (ej: TRADE) |
| currency | string | No | Moneda: USD, EUR, ARS, MXN, USDC, USDT, BOB, COP, PEN, DOP |
| from_date | date | No | Fecha inicial (formato: Y-m-d) |
| to_date | date | No | Fecha final (formato: Y-m-d) |
| from_amount | number | No | Monto mínimo |
| to_amount | number | No | Monto máximo |

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

Ejecuta una operación de compra (BUY) o venta (SELL) de activos.

**Request Body**:

| Campo | Tipo | Requerido | Descripción |
|-------|------|-----------|-------------|
| symbol | string | Sí | Símbolo del asset (ej: AAPL, TSLA) |
| direction | string | Sí | BUY o SELL |
| currency | string | Sí | Moneda (solo USD soportado) |
| order_type | string | Sí | MARKET, LIMIT, STOP, STOP_LIMIT |
| amount | number | Condicional | Monto en USD (requerido si no se especifica shares) |
| shares | number | Condicional | Cantidad de acciones (requerido si no se especifica amount) |
| limit_price | number | Condicional | Precio límite (requerido para LIMIT y STOP_LIMIT) |
| stop_price | number | Condicional | Precio stop (requerido para STOP y STOP_LIMIT) |
| time_in_force | string | Condicional | DAY o GTC (requerido para LIMIT) |

**Ejemplo Market Buy por monto**:
```json
{
  "symbol": "AAPL",
  "direction": "BUY",
  "currency": "USD",
  "order_type": "MARKET",
  "amount": 100.00
}
```

**Ejemplo Limit Sell por shares**:
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

**Errores específicos**:
- 400: Fondos insuficientes
- 412: KYC de inversión incompleto

---

## Account Details Endpoints

### GET /api/public/v1/account-details

Obtiene los detalles de la cuenta bancaria del usuario.

**Parámetros Query**:

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| country | string | US | País: US, EU |
| currency | string | USD | Moneda: USD, EUR |

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

**Errores específicos**:
- 422: Cuenta no verificada o país no soportado

---

## Wallets Endpoints

### GET /api/public/v1/wallets

Obtiene las direcciones de wallets crypto para recibir depósitos.

**Parámetros Query**:

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| currency | string | Filtrar por moneda: USDT, USDC |
| network | string | Filtrar por red: ethereum, arbitrum, solana, polygon, tron |

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

Lista los assets disponibles para trading con paginación y filtros.

**Parámetros Query**:

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| category | string | - | Categoría de assets (ver lista abajo) |
| search | string | - | Búsqueda por símbolo, nombre o keywords (max 100 chars) |
| page | integer | 1 | Número de página |
| limit | integer | 10 | Assets por página (max: 50) |

**Categorías disponibles**:
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

Obtiene información detallada de un asset específico.

**Parámetros Path**:

| Parámetro | Tipo | Requerido | Descripción |
|-----------|------|-----------|-------------|
| symbol | string | Sí | Símbolo del asset (ej: AAPL) |

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

**Errores específicos**:
- 404: Asset no encontrado

---

## Operations Endpoints

### POST /api/public/v1/operations/internal

Mueve fondos entre la cuenta checking (DEFAULT) y la cuenta de inversión (INVESTMENT).

**Request Body**:

| Campo | Tipo | Requerido | Descripción |
|-------|------|-----------|-------------|
| currency | string | Sí | Código de moneda (ej: USD) |
| from | string | Sí | Cuenta origen: DEFAULT o INVESTMENT |
| to | string | Sí | Cuenta destino: DEFAULT o INVESTMENT |
| amount | number | Sí | Monto a transferir (min: 1, max: 999999) |

**Ejemplo Depósito a inversión**:
```json
{
  "currency": "USD",
  "from": "DEFAULT",
  "to": "INVESTMENT",
  "amount": 100.00
}
```

**Ejemplo Retiro de inversión**:
```json
{
  "currency": "USD",
  "from": "INVESTMENT",
  "to": "DEFAULT",
  "amount": 50.00
}
```

**Response 200**: Retorna objeto Transaction.

**Errores específicos**:
- 412: Cuenta bloqueada, KYC incompleto, o cuenta en migración
- 422: Fondos insuficientes o error de validación

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
    "symbol": ["El campo symbol es obligatorio."]
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
