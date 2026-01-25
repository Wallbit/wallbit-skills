# Wallbit API - Code Examples

Ejemplos completos de código para integrar la API de Wallbit en diferentes lenguajes.

---

## PHP/Laravel Examples

### Cliente Base

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Http;
use Illuminate\Http\Client\PendingRequest;
use Exception;

class WallbitService
{
    private string $baseUrl;
    private string $apiKey;

    public function __construct()
    {
        $this->baseUrl = config('services.wallbit.base_url', 'https://api.wallbit.io');
        $this->apiKey = config('services.wallbit.api_key');
    }

    /**
     * Crea un cliente HTTP configurado para la API de Wallbit.
     *
     * @return PendingRequest
     */
    protected function client(): PendingRequest
    {
        return Http::withHeaders([
            'X-API-Key' => $this->apiKey,
            'Accept' => 'application/json',
        ])->baseUrl($this->baseUrl);
    }

    /**
     * Maneja la respuesta de la API y lanza excepciones en caso de error.
     *
     * @param \Illuminate\Http\Client\Response $response
     * @return array
     * @throws Exception
     */
    protected function handleResponse($response): array
    {
        if ($response->status() === 429) {
            $retryAfter = $response->json('retry_after', 60);
            throw new Exception("Rate limit excedido. Reintentar en {$retryAfter} segundos.");
        }

        if ($response->status() === 401) {
            throw new Exception("API Key inválida o faltante.");
        }

        if ($response->status() === 403) {
            $permissions = $response->json('your_permissions', []);
            throw new Exception("Permisos insuficientes. Tienes: " . implode(', ', $permissions));
        }

        if ($response->status() === 412) {
            throw new Exception($response->json('error', 'Precondición no cumplida (KYC o cuenta bloqueada).'));
        }

        if ($response->status() === 422) {
            $errors = $response->json('errors', []);
            throw new Exception("Error de validación: " . json_encode($errors));
        }

        if (!$response->successful()) {
            throw new Exception($response->json('message', 'Error desconocido'));
        }

        return $response->json('data') ?? $response->json();
    }
}
```

### Balance Methods

```php
/**
 * Obtiene el balance de la cuenta checking.
 *
 * @return array Lista de balances por moneda
 * @throws Exception
 */
public function getCheckingBalance(): array
{
    $response = $this->client()->get('/api/public/v1/balance/checking');
    return $this->handleResponse($response);
}

/**
 * Obtiene el portfolio de stocks del usuario.
 *
 * @return array Lista de posiciones (symbol, shares)
 * @throws Exception
 */
public function getStocksBalance(): array
{
    $response = $this->client()->get('/api/public/v1/balance/stocks');
    return $this->handleResponse($response);
}
```

### Transactions Methods

```php
/**
 * Lista las transacciones del usuario con filtros opcionales.
 *
 * @param array $filters Filtros opcionales (page, limit, status, type, currency, from_date, to_date, from_amount, to_amount)
 * @return array Datos paginados de transacciones
 * @throws Exception
 */
public function listTransactions(array $filters = []): array
{
    $response = $this->client()->get('/api/public/v1/transactions', $filters);
    return $this->handleResponse($response);
}

// Ejemplo de uso:
// $transactions = $wallbit->listTransactions([
//     'status' => 'COMPLETED',
//     'currency' => 'USD',
//     'from_date' => '2024-01-01',
//     'to_date' => '2024-12-31',
//     'limit' => 20
// ]);
```

### Trade Methods

```php
/**
 * Ejecuta una orden de compra a precio de mercado.
 *
 * @param string $symbol Símbolo del asset (ej: AAPL)
 * @param float $amount Monto en USD a invertir
 * @return array Datos del trade creado
 * @throws Exception
 */
public function marketBuy(string $symbol, float $amount): array
{
    $response = $this->client()->post('/api/public/v1/trades', [
        'symbol' => $symbol,
        'direction' => 'BUY',
        'currency' => 'USD',
        'order_type' => 'MARKET',
        'amount' => $amount,
    ]);
    return $this->handleResponse($response);
}

/**
 * Ejecuta una orden de venta a precio de mercado.
 *
 * @param string $symbol Símbolo del asset
 * @param float $shares Cantidad de acciones a vender
 * @return array Datos del trade creado
 * @throws Exception
 */
public function marketSell(string $symbol, float $shares): array
{
    $response = $this->client()->post('/api/public/v1/trades', [
        'symbol' => $symbol,
        'direction' => 'SELL',
        'currency' => 'USD',
        'order_type' => 'MARKET',
        'shares' => $shares,
    ]);
    return $this->handleResponse($response);
}

/**
 * Crea una orden límite de compra.
 *
 * @param string $symbol Símbolo del asset
 * @param float $amount Monto en USD
 * @param float $limitPrice Precio límite
 * @param string $timeInForce DAY o GTC
 * @return array Datos del trade creado
 * @throws Exception
 */
public function limitBuy(string $symbol, float $amount, float $limitPrice, string $timeInForce = 'GTC'): array
{
    $response = $this->client()->post('/api/public/v1/trades', [
        'symbol' => $symbol,
        'direction' => 'BUY',
        'currency' => 'USD',
        'order_type' => 'LIMIT',
        'amount' => $amount,
        'limit_price' => $limitPrice,
        'time_in_force' => $timeInForce,
    ]);
    return $this->handleResponse($response);
}
```

### Assets Methods

```php
/**
 * Lista assets disponibles con filtros.
 *
 * @param string|null $category Categoría de assets
 * @param string|null $search Término de búsqueda
 * @param int $page Número de página
 * @param int $limit Assets por página (max 50)
 * @return array Datos paginados de assets
 * @throws Exception
 */
public function listAssets(?string $category = null, ?string $search = null, int $page = 1, int $limit = 10): array
{
    $params = array_filter([
        'category' => $category,
        'search' => $search,
        'page' => $page,
        'limit' => $limit,
    ]);
    
    $response = $this->client()->get('/api/public/v1/assets', $params);
    return $this->handleResponse($response);
}

/**
 * Obtiene información detallada de un asset.
 *
 * @param string $symbol Símbolo del asset
 * @return array Datos del asset
 * @throws Exception
 */
public function getAsset(string $symbol): array
{
    $response = $this->client()->get("/api/public/v1/assets/{$symbol}");
    return $this->handleResponse($response);
}
```

### Operations Methods

```php
/**
 * Deposita fondos desde checking a la cuenta de inversión.
 *
 * @param float $amount Monto a depositar
 * @param string $currency Moneda (default: USD)
 * @return array Datos de la transacción
 * @throws Exception
 */
public function depositToInvestment(float $amount, string $currency = 'USD'): array
{
    $response = $this->client()->post('/api/public/v1/operations/internal', [
        'currency' => $currency,
        'from' => 'DEFAULT',
        'to' => 'INVESTMENT',
        'amount' => $amount,
    ]);
    return $this->handleResponse($response);
}

/**
 * Retira fondos desde la cuenta de inversión a checking.
 *
 * @param float $amount Monto a retirar
 * @param string $currency Moneda (default: USD)
 * @return array Datos de la transacción
 * @throws Exception
 */
public function withdrawFromInvestment(float $amount, string $currency = 'USD'): array
{
    $response = $this->client()->post('/api/public/v1/operations/internal', [
        'currency' => $currency,
        'from' => 'INVESTMENT',
        'to' => 'DEFAULT',
        'amount' => $amount,
    ]);
    return $this->handleResponse($response);
}
```

---

## JavaScript/Node.js Examples

### Cliente Base

```javascript
class WallbitClient {
  constructor(apiKey) {
    this.baseUrl = 'https://api.wallbit.io';
    this.apiKey = apiKey;
  }

  async request(method, endpoint, data = null) {
    const options = {
      method,
      headers: {
        'X-API-Key': this.apiKey,
        'Accept': 'application/json',
        'Content-Type': 'application/json',
      },
    };

    if (data && method !== 'GET') {
      options.body = JSON.stringify(data);
    }

    let url = `${this.baseUrl}${endpoint}`;
    if (data && method === 'GET') {
      const params = new URLSearchParams(data);
      url += `?${params.toString()}`;
    }

    const response = await fetch(url, options);
    const json = await response.json();

    if (response.status === 429) {
      throw new Error(`Rate limit excedido. Reintentar en ${json.retry_after} segundos.`);
    }

    if (response.status === 401) {
      throw new Error('API Key inválida o faltante.');
    }

    if (response.status === 403) {
      throw new Error(`Permisos insuficientes. Tienes: ${json.your_permissions?.join(', ')}`);
    }

    if (response.status === 422) {
      throw new Error(`Error de validación: ${JSON.stringify(json.errors)}`);
    }

    if (!response.ok) {
      throw new Error(json.message || json.error || 'Error desconocido');
    }

    return json.data ?? json;
  }

  // Balance
  async getCheckingBalance() {
    return this.request('GET', '/api/public/v1/balance/checking');
  }

  async getStocksBalance() {
    return this.request('GET', '/api/public/v1/balance/stocks');
  }

  // Transactions
  async listTransactions(filters = {}) {
    return this.request('GET', '/api/public/v1/transactions', filters);
  }

  // Trades
  async marketBuy(symbol, amount) {
    return this.request('POST', '/api/public/v1/trades', {
      symbol,
      direction: 'BUY',
      currency: 'USD',
      order_type: 'MARKET',
      amount,
    });
  }

  async marketSell(symbol, shares) {
    return this.request('POST', '/api/public/v1/trades', {
      symbol,
      direction: 'SELL',
      currency: 'USD',
      order_type: 'MARKET',
      shares,
    });
  }

  async limitBuy(symbol, amount, limitPrice, timeInForce = 'GTC') {
    return this.request('POST', '/api/public/v1/trades', {
      symbol,
      direction: 'BUY',
      currency: 'USD',
      order_type: 'LIMIT',
      amount,
      limit_price: limitPrice,
      time_in_force: timeInForce,
    });
  }

  // Assets
  async listAssets({ category, search, page = 1, limit = 10 } = {}) {
    const params = { page, limit };
    if (category) params.category = category;
    if (search) params.search = search;
    return this.request('GET', '/api/public/v1/assets', params);
  }

  async getAsset(symbol) {
    return this.request('GET', `/api/public/v1/assets/${symbol}`);
  }

  // Wallets
  async getWallets({ currency, network } = {}) {
    const params = {};
    if (currency) params.currency = currency;
    if (network) params.network = network;
    return this.request('GET', '/api/public/v1/wallets', params);
  }

  // Account Details
  async getAccountDetails({ country = 'US', currency = 'USD' } = {}) {
    return this.request('GET', '/api/public/v1/account-details', { country, currency });
  }

  // Operations
  async depositToInvestment(amount, currency = 'USD') {
    return this.request('POST', '/api/public/v1/operations/internal', {
      currency,
      from: 'DEFAULT',
      to: 'INVESTMENT',
      amount,
    });
  }

  async withdrawFromInvestment(amount, currency = 'USD') {
    return this.request('POST', '/api/public/v1/operations/internal', {
      currency,
      from: 'INVESTMENT',
      to: 'DEFAULT',
      amount,
    });
  }
}

// Uso
const wallbit = new WallbitClient(process.env.WALLBIT_API_KEY);

// Ejemplos
async function examples() {
  // Obtener balance
  const balance = await wallbit.getCheckingBalance();
  console.log('Balance:', balance);

  // Comprar $100 de Apple
  const trade = await wallbit.marketBuy('AAPL', 100);
  console.log('Trade:', trade);

  // Buscar assets de tecnología
  const assets = await wallbit.listAssets({ category: 'TECHNOLOGY', limit: 20 });
  console.log('Assets:', assets);
}
```

---

## Python Examples

### Cliente Base

```python
import requests
from typing import Optional, Dict, Any, List


class WallbitClient:
    """Cliente para la API pública de Wallbit."""
    
    def __init__(self, api_key: str, base_url: str = "https://api.wallbit.io"):
        self.base_url = base_url
        self.headers = {
            "X-API-Key": api_key,
            "Accept": "application/json",
            "Content-Type": "application/json"
        }
    
    def _request(self, method: str, endpoint: str, params: Dict = None, json: Dict = None) -> Any:
        """Ejecuta una request a la API con manejo de errores."""
        url = f"{self.base_url}{endpoint}"
        response = requests.request(method, url, headers=self.headers, params=params, json=json)
        
        if response.status_code == 429:
            retry_after = response.json().get("retry_after", 60)
            raise Exception(f"Rate limit excedido. Reintentar en {retry_after} segundos.")
        
        if response.status_code == 401:
            raise Exception("API Key inválida o faltante.")
        
        if response.status_code == 403:
            permissions = response.json().get("your_permissions", [])
            raise Exception(f"Permisos insuficientes. Tienes: {', '.join(permissions)}")
        
        if response.status_code == 412:
            error = response.json().get("error", "Precondición no cumplida.")
            raise Exception(error)
        
        if response.status_code == 422:
            errors = response.json().get("errors", {})
            raise Exception(f"Error de validación: {errors}")
        
        response.raise_for_status()
        data = response.json()
        return data.get("data", data)
    
    # Balance
    def get_checking_balance(self) -> List[Dict]:
        """Obtiene el balance de la cuenta checking."""
        return self._request("GET", "/api/public/v1/balance/checking")
    
    def get_stocks_balance(self) -> List[Dict]:
        """Obtiene el portfolio de stocks."""
        return self._request("GET", "/api/public/v1/balance/stocks")
    
    # Transactions
    def list_transactions(
        self,
        page: int = 1,
        limit: int = 10,
        status: Optional[str] = None,
        type: Optional[str] = None,
        currency: Optional[str] = None,
        from_date: Optional[str] = None,
        to_date: Optional[str] = None,
        from_amount: Optional[float] = None,
        to_amount: Optional[float] = None
    ) -> Dict:
        """Lista transacciones con filtros opcionales."""
        params = {"page": page, "limit": limit}
        if status: params["status"] = status
        if type: params["type"] = type
        if currency: params["currency"] = currency
        if from_date: params["from_date"] = from_date
        if to_date: params["to_date"] = to_date
        if from_amount: params["from_amount"] = from_amount
        if to_amount: params["to_amount"] = to_amount
        return self._request("GET", "/api/public/v1/transactions", params=params)
    
    # Trades
    def market_buy(self, symbol: str, amount: float) -> Dict:
        """Ejecuta una compra a precio de mercado."""
        return self._request("POST", "/api/public/v1/trades", json={
            "symbol": symbol,
            "direction": "BUY",
            "currency": "USD",
            "order_type": "MARKET",
            "amount": amount
        })
    
    def market_sell(self, symbol: str, shares: float) -> Dict:
        """Ejecuta una venta a precio de mercado."""
        return self._request("POST", "/api/public/v1/trades", json={
            "symbol": symbol,
            "direction": "SELL",
            "currency": "USD",
            "order_type": "MARKET",
            "shares": shares
        })
    
    def limit_buy(self, symbol: str, amount: float, limit_price: float, time_in_force: str = "GTC") -> Dict:
        """Crea una orden límite de compra."""
        return self._request("POST", "/api/public/v1/trades", json={
            "symbol": symbol,
            "direction": "BUY",
            "currency": "USD",
            "order_type": "LIMIT",
            "amount": amount,
            "limit_price": limit_price,
            "time_in_force": time_in_force
        })
    
    # Assets
    def list_assets(
        self,
        category: Optional[str] = None,
        search: Optional[str] = None,
        page: int = 1,
        limit: int = 10
    ) -> Dict:
        """Lista assets disponibles."""
        params = {"page": page, "limit": limit}
        if category: params["category"] = category
        if search: params["search"] = search
        return self._request("GET", "/api/public/v1/assets", params=params)
    
    def get_asset(self, symbol: str) -> Dict:
        """Obtiene información de un asset específico."""
        return self._request("GET", f"/api/public/v1/assets/{symbol}")
    
    # Wallets
    def get_wallets(self, currency: Optional[str] = None, network: Optional[str] = None) -> List[Dict]:
        """Obtiene direcciones de wallets crypto."""
        params = {}
        if currency: params["currency"] = currency
        if network: params["network"] = network
        return self._request("GET", "/api/public/v1/wallets", params=params)
    
    # Account Details
    def get_account_details(self, country: str = "US", currency: str = "USD") -> Dict:
        """Obtiene detalles de cuenta bancaria."""
        return self._request("GET", "/api/public/v1/account-details", params={
            "country": country,
            "currency": currency
        })
    
    # Operations
    def deposit_to_investment(self, amount: float, currency: str = "USD") -> Dict:
        """Deposita fondos a la cuenta de inversión."""
        return self._request("POST", "/api/public/v1/operations/internal", json={
            "currency": currency,
            "from": "DEFAULT",
            "to": "INVESTMENT",
            "amount": amount
        })
    
    def withdraw_from_investment(self, amount: float, currency: str = "USD") -> Dict:
        """Retira fondos de la cuenta de inversión."""
        return self._request("POST", "/api/public/v1/operations/internal", json={
            "currency": currency,
            "from": "INVESTMENT",
            "to": "DEFAULT",
            "amount": amount
        })


# Uso
if __name__ == "__main__":
    import os
    
    client = WallbitClient(os.environ["WALLBIT_API_KEY"])
    
    # Obtener balance
    balance = client.get_checking_balance()
    print(f"Balance: {balance}")
    
    # Comprar $100 de Apple
    trade = client.market_buy("AAPL", 100)
    print(f"Trade: {trade}")
    
    # Buscar assets de tecnología
    assets = client.list_assets(category="TECHNOLOGY", limit=20)
    print(f"Assets: {assets}")
```
