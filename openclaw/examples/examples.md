# OpenClaw - Code Examples

Code examples for integrating with OpenClaw: Gateway HTTP API, programmatic configuration, webhooks and CLI.

---

## JavaScript/Node.js Examples

### HTTP Client (Chat Completions)

The Gateway can expose `POST /v1/chat/completions` (OpenAI-compatible). Enable with `gateway.http.endpoints.chatCompletions.enabled: true`.

```javascript
/**
 * Client for the OpenClaw Gateway Chat Completions endpoint.
 * Requires gateway.http.endpoints.chatCompletions.enabled: true in openclaw.json.
 */
class OpenClawGatewayClient {
  constructor(options = {}) {
    this.baseUrl = options.baseUrl || process.env.OPENCLAW_GATEWAY_URL || 'http://127.0.0.1:18789';
    this.token = options.token || process.env.OPENCLAW_GATEWAY_TOKEN;
    this.agentId = options.agentId || 'main';
  }

  /**
   * Sends a message to the agent and returns the response (non-streaming).
   *
   * @param {string} message - User message
   * @param {object} options - { user?: string (stable session), agentId?: string }
   * @returns {Promise<object>} Response with choices[0].message
   */
  async chat(message, options = {}) {
    const url = `${this.baseUrl}/v1/chat/completions`;
    const agentId = options.agentId || this.agentId;
    const body = {
      model: 'openclaw',
      messages: [{ role: 'user', content: message }],
    };
    if (options.user) body.user = options.user;

    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.token}`,
        'Content-Type': 'application/json',
        'x-openclaw-agent-id': agentId,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429) {
      const retryAfter = response.headers.get('Retry-After') || 60;
      throw new Error(`Rate limit. Retry in ${retryAfter} seconds.`);
    }
    if (response.status === 401) {
      throw new Error('Invalid or missing Gateway token.');
    }
    if (!response.ok) {
      const err = await response.json().catch(() => ({}));
      throw new Error(err.error?.message || response.statusText);
    }

    return response.json();
  }

  /**
   * Sends a message and receives the response as a stream (SSE).
   *
   * @param {string} message - User message
   * @param {object} options - { agentId?, user?, onChunk?(text: string) }
   * @returns {Promise<string>} Full response text
   */
  async chatStream(message, options = {}) {
    const url = `${this.baseUrl}/v1/chat/completions`;
    const agentId = options.agentId || this.agentId;
    const body = {
      model: 'openclaw',
      stream: true,
      messages: [{ role: 'user', content: message }],
    };
    if (options.user) body.user = options.user;

    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.token}`,
        'Content-Type': 'application/json',
        'x-openclaw-agent-id': agentId,
      },
      body: JSON.stringify(body),
    });

    if (!response.ok) {
      const err = await response.json().catch(() => ({}));
      throw new Error(err.error?.message || response.statusText);
    }

    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let fullText = '';
    for (;;) {
      const { value, done } = await reader.read();
      if (done) break;
      const chunk = decoder.decode(value, { stream: true });
      const lines = chunk.split('\n').filter((l) => l.startsWith('data: '));
      for (const line of lines) {
        const data = line.slice(6);
        if (data === '[DONE]') continue;
        try {
          const parsed = JSON.parse(data);
          const content = parsed.choices?.[0]?.delta?.content;
          if (content) {
            fullText += content;
            options.onChunk?.(content);
          }
        } catch (_) {}
      }
    }
    return fullText;
  }
}

// Usage
// const client = new OpenClawGatewayClient();
// const res = await client.chat('Hello, summarize in one line what OpenClaw is.');
// console.log(res.choices[0].message.content);
```

### Channel setup script (CLI)

```javascript
/**
 * Adds a Telegram or Discord channel using the OpenClaw CLI.
 * Run with: node add-channel.js telegram $BOT_TOKEN
 */
const { execSync } = require('child_process');

/**
 * Adds a channel via openclaw channels add (non-interactive).
 *
 * @param {string} channel - whatsapp | telegram | discord | slack | googlechat
 * @param {string} token - Bot token
 * @param {string} account - Account ID (default: default)
 * @param {string} name - Display name
 */
function addChannel(channel, token, account = 'default', name = '') {
  const args = [
    'channels', 'add',
    '--channel', channel,
    '--account', account,
    '--token', token,
  ];
  if (name) args.push('--name', name);
  execSync(`openclaw ${args.join(' ')}`, {
    stdio: 'inherit',
    env: { ...process.env },
  });
}

const [channel, token] = process.argv.slice(2);
if (!channel || !token) {
  console.error('Usage: node add-channel.js <telegram|discord|...> <BOT_TOKEN>');
  process.exit(1);
}
addChannel(channel, token);
```

### Programmatic openclaw.json configuration

```javascript
const fs = require('fs');
const path = require('path');

const configPath = path.join(process.env.HOME || process.env.USERPROFILE, '.openclaw', 'openclaw.json');

/**
 * Reads the current OpenClaw config (JSON5-compatible, read as JSON).
 *
 * @returns {object} Current config
 */
function getConfig() {
  const raw = fs.readFileSync(configPath, 'utf8');
  return JSON.parse(raw);
}

/**
 * Writes the OpenClaw config. Prefer config.patch via RPC in production.
 *
 * @param {object} config - Complete configuration object
 */
function writeConfig(config) {
  fs.writeFileSync(configPath, JSON.stringify(config, null, 2), 'utf8');
}

/**
 * Enables the HTTP Chat Completions endpoint.
 */
function enableChatCompletionsEndpoint() {
  const config = getConfig();
  config.gateway = config.gateway || {};
  config.gateway.http = config.gateway.http || {};
  config.gateway.http.endpoints = config.gateway.http.endpoints || {};
  config.gateway.http.endpoints.chatCompletions = { enabled: true };
  writeConfig(config);
}
```

### Webhook handler (Express)

Example endpoint that receives notifications and forwards to the agent or logs events. Adjust the URL and signature according to how OpenClaw (or an external service) calls the webhook.

```javascript
const express = require('express');
const app = express();

app.use(express.json());

/**
 * Example webhook: receives a payload and optionally sends a message to the Gateway.
 * In production, validate the origin (signature header, IP, etc.).
 *
 * @param {object} req - Request with body { message?, event? }
 * @param {object} res - Response
 */
app.post('/webhook/openclaw', async (req, res) => {
  try {
    const { message, event } = req.body || {};
    const token = process.env.OPENCLAW_GATEWAY_TOKEN;
    const baseUrl = process.env.OPENCLAW_GATEWAY_URL || 'http://127.0.0.1:18789';

    if (event) {
      // Log event (e.g. system event via RPC or log)
      console.log('Event received:', event);
    }

    if (message && token && baseUrl) {
      const response = await fetch(`${baseUrl}/v1/chat/completions`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json',
          'x-openclaw-agent-id': 'main',
        },
        body: JSON.stringify({
          model: 'openclaw',
          messages: [{ role: 'user', content: message }],
        }),
      });
      if (!response.ok) {
        console.error('Gateway error:', response.status, await response.text());
      }
    }

    res.status(200).json({ ok: true });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: err.message });
  }
});

app.listen(process.env.PORT || 3000);
```

---

## PHP/Laravel Examples

### Service for the Gateway HTTP API

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Http;
use Exception;

/**
 * Client for the OpenClaw Gateway Chat Completions endpoint.
 * Requires gateway.http.endpoints.chatCompletions.enabled: true.
 */
class OpenClawGatewayService
{
    private string $baseUrl;
    private string $token;
    private string $agentId;

    public function __construct()
    {
        $this->baseUrl = config('services.openclaw.gateway_url', 'http://127.0.0.1:18789');
        $this->token = config('services.openclaw.gateway_token');
        $this->agentId = config('services.openclaw.agent_id', 'main');
    }

    /**
     * HTTP client with authentication and agent headers.
     *
     * @return \Illuminate\Http\Client\PendingRequest
     */
    protected function client(): \Illuminate\Http\Client\PendingRequest
    {
        return Http::withHeaders([
            'Authorization' => 'Bearer ' . $this->token,
            'Content-Type' => 'application/json',
            'x-openclaw-agent-id' => $this->agentId,
        ])->baseUrl($this->baseUrl)->timeout(120);
    }

    /**
     * Sends a message to the agent and returns the response (non-streaming).
     *
     * @param string $message User message
     * @param string|null $user User id for stable session (optional)
     * @return array Response with choices[0].message
     * @throws Exception
     */
    public function chat(string $message, ?string $user = null): array
    {
        $body = [
            'model' => 'openclaw',
            'messages' => [['role' => 'user', 'content' => $message]],
        ];
        if ($user !== null) {
            $body['user'] = $user;
        }

        $response = $this->client()->post('/v1/chat/completions', $body);

        if ($response->status() === 429) {
            $retryAfter = $response->header('Retry-After') ?? 60;
            throw new Exception("Rate limit. Retry in {$retryAfter} seconds.");
        }
        if ($response->status() === 401) {
            throw new Exception('Invalid or missing Gateway token.');
        }
        if (!$response->successful()) {
            $err = $response->json('error.message') ?? $response->body();
            throw new Exception($err ?: 'Gateway error');
        }

        return $response->json();
    }

    /**
     * Returns the text content of the first agent response.
     *
     * @param string $message User message
     * @param string|null $user Optional user id for stable session
     * @return string Response content
     * @throws Exception
     */
    public function chatContent(string $message, ?string $user = null): string
    {
        $data = $this->chat($message, $user);
        return $data['choices'][0]['message']['content'] ?? '';
    }
}
```

### Laravel Configuration

In `config/services.php`:

```php
'openclaw' => [
    'gateway_url' => env('OPENCLAW_GATEWAY_URL', 'http://127.0.0.1:18789'),
    'gateway_token' => env('OPENCLAW_GATEWAY_TOKEN'),
    'agent_id' => env('OPENCLAW_AGENT_ID', 'main'),
],
```

Environment variables (`.env`):

```bash
OPENCLAW_GATEWAY_URL=http://127.0.0.1:18789
OPENCLAW_GATEWAY_TOKEN=your_gateway_token
OPENCLAW_AGENT_ID=main
```

### Webhook controller

```php
<?php

namespace App\Http\Controllers;

use App\Services\OpenClawGatewayService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class OpenClawWebhookController extends Controller
{
    /**
     * Receives a webhook (e.g. notification or message) and optionally queries the agent.
     * In production, validate the request signature/origin.
     *
     * @param Request $request
     * @return JsonResponse
     */
    public function __invoke(Request $request): JsonResponse
    {
        try {
            $message = $request->input('message');
            $event = $request->input('event');
            $user = $request->input('user');

            if ($event) {
                Log::info('OpenClaw event received', ['event' => $event]);
            }

            if (empty($message)) {
                return response()->json(['ok' => true]);
            }

            $service = app(OpenClawGatewayService::class);
            $content = $service->chatContent($message, $user);
            return response()->json(['ok' => true, 'reply' => $content]);
        } catch (\Throwable $e) {
            Log::error('OpenClaw webhook error', ['error' => $e->getMessage()]);
            return response()->json(['error' => $e->getMessage()], 500);
        }
    }
}
```

Route (in `routes/web.php` or `api.php`):

```php
use App\Http\Controllers\OpenClawWebhookController;

Route::post('/webhook/openclaw', OpenClawWebhookController::class);
```

### Artisan command for config

Example command that reads or writes the OpenClaw config (file on the server where the Gateway runs). Run on the same host as the Gateway or via deployment.

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;

class OpenClawConfigCommand extends Command
{
    protected $signature = 'openclaw:config 
                            {action : get|enable-chat-completions} 
                            {key? : For get, path in dot notation}';

    /**
     * Gets a value from the OpenClaw config or enables Chat Completions.
     * Requires OPENCLAW_CONFIG_PATH to point to ~/.openclaw/openclaw.json on the Gateway host.
     *
     * @return int
     */
    public function handle(): int
    {
        $configPath = env('OPENCLAW_CONFIG_PATH', $this->userHome() . '/.openclaw/openclaw.json');
        if (!is_readable($configPath)) {
            $this->error("Config not found: {$configPath}");
            return self::FAILURE;
        }

        $action = $this->argument('action');
        $config = json_decode(file_get_contents($configPath), true);
        if (json_last_error() !== JSON_ERROR_NONE) {
            $this->error('Invalid config (JSON)');
            return self::FAILURE;
        }

        if ($action === 'get') {
            $key = $this->argument('key');
            if (!$key) {
                $this->error('Provide key for get (e.g. gateway.port)');
                return self::FAILURE;
            }
            $value = data_get($config, $key);
            $this->line($value === null ? 'null' : json_encode($value));
            return self::SUCCESS;
        }

        if ($action === 'enable-chat-completions') {
            data_set($config, 'gateway.http.endpoints.chatCompletions.enabled', true);
            if (!is_writable($configPath)) {
                $this->error('No write permission on config');
                return self::FAILURE;
            }
            file_put_contents($configPath, json_encode($config, JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES));
            $this->info('Chat Completions endpoint enabled. Restart the Gateway.');
            return self::SUCCESS;
        }

        $this->error("Unknown action: {$action}");
        return self::FAILURE;
    }

    private function userHome(): string
    {
        return getenv('HOME') ?: (getenv('USERPROFILE') ?: '/tmp');
    }
}
```

---

## Integration Summary

| Language    | Typical use                                           |
| ----------- | ----------------------------------------------------- |
| JavaScript  | CLI scripts, webhooks, Node services, streaming       |
| PHP/Laravel | Webhooks, Gateway service, Artisan commands           |

Conventions:

- **Naming**: camelCase for functions and variables; docstrings on all functions.
- **Secrets**: Do not hardcode `OPENCLAW_GATEWAY_TOKEN`; use environment variables or `config()`.
- **Errors**: Check 401, 429 (and `Retry-After`) and error body in HTTP responses.
- **Endpoint**: Enable Chat Completions in `openclaw.json` when using the HTTP API.

Official documentation: https://docs.openclaw.ai/
