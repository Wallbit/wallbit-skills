# Wallbit API - Code Examples

Complete code examples for integrating the Wallbit API in different languages.

---

## PHP/Laravel Examples

### Base Client

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
     * Creates an HTTP client configured for the Wallbit API.
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
     * Handles the API response and throws exceptions on error.
     *
     * @param \Illuminate\Http\Client\Response $response
     * @return array
     * @throws Exception
     */
    protected function handleResponse($response): array
    {
        if ($response->status() === 429) {
            $retryAfter = $response->json('retry_after', 60);
            throw new Exception("Rate limit exceeded. Retry in {$retryAfter} seconds.");
        }

        if ($response->status() === 401) {
            throw new Exception("Invalid or missing API Key.");
        }

        if ($response->status() === 403) {
            $permissions = $response->json('your_permissions', []);
            throw new Exception("Insufficient permissions. You have: " . implode(', ', $permissions));
        }

        if ($response->status() === 412) {
            throw new Exception($response->json('error', 'Precondition not met (KYC or blocked account).'));
        }

        if ($response->status() === 422) {
            $errors = $response->json('errors', []);
            throw new Exception("Validation error: " . json_encode($errors));
        }

        if (!$response->successful()) {
            throw new Exception($response->json('message', 'Unknown error'));
        }

        return $response->json('data') ?? $response->json();
    }
}
```

### Balance Methods

```php
/**
 * Gets the checking account balance.
 *
 * @return array List of balances per currency
 * @throws Exception
 */
public function getCheckingBalance(): array
{
    $response = $this->client()->get('/api/public/v1/balance/checking');
    return $this->handleResponse($response);
}

/**
 * Gets the user's stock portfolio.
 *
 * @return array List of positions (symbol, shares)
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
 * Lists the user's transactions with optional filters.
 *
 * @param array $filters Optional filters (page, limit, status, type, currency, from_date, to_date, from_amount, to_amount)
 * @return array Paginated transaction data
 * @throws Exception
 */
public function listTransactions(array $filters = []): array
{
    $response = $this->client()->get('/api/public/v1/transactions', $filters);
    return $this->handleResponse($response);
}

// Usage example:
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
 * Executes a market buy order.
 *
 * @param string $symbol Asset symbol (e.g. AAPL)
 * @param float $amount Amount in USD to invest
 * @return array Created trade data
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
 * Executes a market sell order.
 *
 * @param string $symbol Asset symbol
 * @param float $shares Number of shares to sell
 * @return array Created trade data
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
 * Creates a limit buy order.
 *
 * @param string $symbol Asset symbol
 * @param float $amount Amount in USD
 * @param float $limitPrice Limit price
 * @param string $timeInForce DAY or GTC
 * @return array Created trade data
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
 * Lists available assets with filters.
 *
 * @param string|null $category Asset category
 * @param string|null $search Search term
 * @param int $page Page number
 * @param int $limit Assets per page (max 50)
 * @return array Paginated asset data
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
 * Gets detailed information about an asset.
 *
 * @param string $symbol Asset symbol
 * @return array Asset data
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
 * Deposits funds from checking to the investment account.
 *
 * @param float $amount Amount to deposit
 * @param string $currency Currency (default: USD)
 * @return array Transaction data
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
 * Withdraws funds from the investment account to checking.
 *
 * @param float $amount Amount to withdraw
 * @param string $currency Currency (default: USD)
 * @return array Transaction data
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

### Base Client

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
      throw new Error(`Rate limit exceeded. Retry in ${json.retry_after} seconds.`);
    }

    if (response.status === 401) {
      throw new Error('Invalid or missing API Key.');
    }

    if (response.status === 403) {
      throw new Error(`Insufficient permissions. You have: ${json.your_permissions?.join(', ')}`);
    }

    if (response.status === 422) {
      throw new Error(`Validation error: ${JSON.stringify(json.errors)}`);
    }

    if (!response.ok) {
      throw new Error(json.message || json.error || 'Unknown error');
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

// Usage
const wallbit = new WallbitClient(process.env.WALLBIT_API_KEY);

// Examples
async function examples() {
  // Get balance
  const balance = await wallbit.getCheckingBalance();
  console.log('Balance:', balance);

  // Buy $100 of Apple
  const trade = await wallbit.marketBuy('AAPL', 100);
  console.log('Trade:', trade);

  // Search technology assets
  const assets = await wallbit.listAssets({ category: 'TECHNOLOGY', limit: 20 });
  console.log('Assets:', assets);
}
```

---

## Python Examples

### Base Client

```python
import requests
from typing import Optional, Dict, Any, List


class WallbitClient:
    """Client for the Wallbit public API."""
    
    def __init__(self, api_key: str, base_url: str = "https://api.wallbit.io"):
        self.base_url = base_url
        self.headers = {
            "X-API-Key": api_key,
            "Accept": "application/json",
            "Content-Type": "application/json"
        }
    
    def _request(self, method: str, endpoint: str, params: Dict = None, json: Dict = None) -> Any:
        """Executes an API request with error handling."""
        url = f"{self.base_url}{endpoint}"
        response = requests.request(method, url, headers=self.headers, params=params, json=json)
        
        if response.status_code == 429:
            retry_after = response.json().get("retry_after", 60)
            raise Exception(f"Rate limit exceeded. Retry in {retry_after} seconds.")
        
        if response.status_code == 401:
            raise Exception("Invalid or missing API Key.")
        
        if response.status_code == 403:
            permissions = response.json().get("your_permissions", [])
            raise Exception(f"Insufficient permissions. You have: {', '.join(permissions)}")
        
        if response.status_code == 412:
            error = response.json().get("error", "Precondition not met.")
            raise Exception(error)
        
        if response.status_code == 422:
            errors = response.json().get("errors", {})
            raise Exception(f"Validation error: {errors}")
        
        response.raise_for_status()
        data = response.json()
        return data.get("data", data)
    
    # Balance
    def get_checking_balance(self) -> List[Dict]:
        """Gets the checking account balance."""
        return self._request("GET", "/api/public/v1/balance/checking")
    
    def get_stocks_balance(self) -> List[Dict]:
        """Gets the stock portfolio."""
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
        """Lists transactions with optional filters."""
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
        """Executes a market buy order."""
        return self._request("POST", "/api/public/v1/trades", json={
            "symbol": symbol,
            "direction": "BUY",
            "currency": "USD",
            "order_type": "MARKET",
            "amount": amount
        })
    
    def market_sell(self, symbol: str, shares: float) -> Dict:
        """Executes a market sell order."""
        return self._request("POST", "/api/public/v1/trades", json={
            "symbol": symbol,
            "direction": "SELL",
            "currency": "USD",
            "order_type": "MARKET",
            "shares": shares
        })
    
    def limit_buy(self, symbol: str, amount: float, limit_price: float, time_in_force: str = "GTC") -> Dict:
        """Creates a limit buy order."""
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
        """Lists available assets."""
        params = {"page": page, "limit": limit}
        if category: params["category"] = category
        if search: params["search"] = search
        return self._request("GET", "/api/public/v1/assets", params=params)
    
    def get_asset(self, symbol: str) -> Dict:
        """Gets information about a specific asset."""
        return self._request("GET", f"/api/public/v1/assets/{symbol}")
    
    # Wallets
    def get_wallets(self, currency: Optional[str] = None, network: Optional[str] = None) -> List[Dict]:
        """Gets crypto wallet addresses."""
        params = {}
        if currency: params["currency"] = currency
        if network: params["network"] = network
        return self._request("GET", "/api/public/v1/wallets", params=params)
    
    # Account Details
    def get_account_details(self, country: str = "US", currency: str = "USD") -> Dict:
        """Gets bank account details."""
        return self._request("GET", "/api/public/v1/account-details", params={
            "country": country,
            "currency": currency
        })
    
    # Operations
    def deposit_to_investment(self, amount: float, currency: str = "USD") -> Dict:
        """Deposits funds to the investment account."""
        return self._request("POST", "/api/public/v1/operations/internal", json={
            "currency": currency,
            "from": "DEFAULT",
            "to": "INVESTMENT",
            "amount": amount
        })
    
    def withdraw_from_investment(self, amount: float, currency: str = "USD") -> Dict:
        """Withdraws funds from the investment account."""
        return self._request("POST", "/api/public/v1/operations/internal", json={
            "currency": currency,
            "from": "INVESTMENT",
            "to": "DEFAULT",
            "amount": amount
        })


# Usage
if __name__ == "__main__":
    import os
    
    client = WallbitClient(os.environ["WALLBIT_API_KEY"])
    
    # Get balance
    balance = client.get_checking_balance()
    print(f"Balance: {balance}")
    
    # Buy $100 of Apple
    trade = client.market_buy("AAPL", 100)
    print(f"Trade: {trade}")
    
    # Search technology assets
    assets = client.list_assets(category="TECHNOLOGY", limit=20)
    print(f"Assets: {assets}")
```
