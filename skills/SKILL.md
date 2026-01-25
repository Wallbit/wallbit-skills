---
name: wallbit-skills
description: Integración con la API pública de Wallbit para consultar balances, transacciones, ejecutar trades, obtener datos de assets y wallets. Usar cuando se trabaje con Wallbit API, endpoints de trading, balances de cuenta, portfolio de stocks, operaciones de inversión, o cuando el usuario mencione Wallbit.
---

# Wallbit Public API

API REST para integrar funcionalidad de Wallbit: consultar balances, ver historial de transacciones, ejecutar trades y más.

## Quick Start

**Base URL**: `https://api.wallbit.io`

**Autenticación**: Header `X-API-Key` requerido en todas las requests.

```bash
curl -H "X-API-Key: YOUR_API_KEY" https://api.wallbit.io/api/public/v1/balance/checking
```

## Endpoints Overview

| Categoría    | Endpoint                             | Método | Descripción                   |
| ------------ | ------------------------------------ | ------ | ----------------------------- |
| Balance      | `/api/public/v1/balance/checking`    | GET    | Balance cuenta checking       |
| Balance      | `/api/public/v1/balance/stocks`      | GET    | Portfolio de inversión        |
| Transactions | `/api/public/v1/transactions`        | GET    | Historial de transacciones    |
| Trades       | `/api/public/v1/trades`              | POST   | Ejecutar compra/venta         |
| Account      | `/api/public/v1/account-details`     | GET    | Detalles cuenta bancaria      |
| Wallets      | `/api/public/v1/wallets`             | GET    | Direcciones de wallets crypto |
| Assets       | `/api/public/v1/assets`              | GET    | Listar assets disponibles     |
| Assets       | `/api/public/v1/assets/{symbol}`     | GET    | Info de asset específico      |
| Operations   | `/api/public/v1/operations/internal` | POST   | Depósito/retiro inversión     |

## Authentication

### PHP/Laravel

```php
use Illuminate\Support\Facades\Http;

/**
 * Crea un cliente HTTP configurado para la API de Wallbit.
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

| Código | Descripción                       | Acción                         |
| ------ | --------------------------------- | ------------------------------ |
| 401    | API Key inválida o faltante       | Verificar header X-API-Key     |
| 403    | Permisos insuficientes            | Verificar permisos del API Key |
| 412    | KYC incompleto o cuenta bloqueada | Completar verificación en app  |
| 422    | Error de validación               | Revisar parámetros enviados    |
| 429    | Rate limit excedido               | Esperar `retry_after` segundos |

### Ejemplo manejo de errores (PHP/Laravel)

```php
/**
 * Ejecuta una request a la API de Wallbit con manejo de errores.
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
        default => throw new \Exception("Método no soportado: {$method}")
    };

    if ($response->status() === 429) {
        $retryAfter = $response->json('retry_after', 60);
        throw new \Exception("Rate limit excedido. Reintentar en {$retryAfter} segundos.");
    }

    if ($response->status() === 401) {
        throw new \Exception("API Key inválida o faltante.");
    }

    if ($response->status() === 403) {
        $permissions = $response->json('your_permissions', []);
        throw new \Exception("Permisos insuficientes. Tienes: " . implode(', ', $permissions));
    }

    if ($response->status() === 422) {
        $errors = $response->json('errors', []);
        throw new \Exception("Error de validación: " . json_encode($errors));
    }

    if (!$response->successful()) {
        throw new \Exception($response->json('message', 'Error desconocido'));
    }

    return $response->json('data');
}
```

## Rate Limiting

Headers de respuesta:

- `X-RateLimit-Limit`: Requests permitidas por minuto
- `X-RateLimit-Remaining`: Requests restantes
- `X-RateLimit-Reset`: Timestamp Unix de reset
- `Retry-After`: Segundos hasta poder reintentar (solo en 429)

## Code Generation Guidelines

Al generar código para esta API:

1. **PHP/Laravel**: Usar camelCase para funciones, incluir docstrings PHPDoc
2. **Validar parámetros** antes de enviar según los tipos del OpenAPI spec
3. **Siempre manejar errores** 401, 403, 422, 429
4. **No hardcodear API Keys**, usar variables de entorno

### Monedas soportadas

- Transacciones: `USD`, `EUR`, `ARS`, `MXN`, `USDC`, `USDT`, `BOB`, `COP`, `PEN`, `DOP`
- Account Details: `USD`, `EUR` (países: `US`, `EU`)
- Wallets: `USDT`, `USDC` (redes: `ethereum`, `arbitrum`, `solana`, `polygon`, `tron`)

### Categorías de Assets

`MOST_POPULAR`, `ETF`, `DIVIDENDS`, `TECHNOLOGY`, `HEALTH`, `CONSUMER_GOODS`, `ENERGY_AND_WATER`, `FINANCE`, `REAL_ESTATE`, `TREASURY_BILLS`, `VIDEOGAMES`, `ARGENTINA_ADR`

### Trade Order Types

- `MARKET`: Orden a precio de mercado (ejecuta inmediatamente)
- `LIMIT`: Orden limitada (requiere `limit_price` y `time_in_force`)
- `STOP`: Orden stop (requiere `stop_price`)
- `STOP_LIMIT`: Orden stop-limit (requiere `stop_price` y `limit_price`)

## Additional Resources

- Para documentación detallada de endpoints, ver [api-reference.md](api-reference.md)
- Para ejemplos completos de código, ver [examples.md](examples.md)
