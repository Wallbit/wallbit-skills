# Wallbit Skills

Cursor skill for integration with the Wallbit public API.

## Description

This skill gives Cursor the knowledge needed to work with the Wallbit API, enabling it to generate correct code following best practices for:

- Querying balances (checking and stocks)
- Viewing transaction history
- Executing buy/sell trades
- Getting bank account details
- Managing crypto wallets
- Querying asset information

## Installation

To use this skill in Cursor, copy the `skills` folder to your project or configure the path in Cursor preferences:

```
~/.cursor/skills/wallbit-skill/SKILL.md
```

## Usage

Once installed, Cursor will automatically detect when you are working with the Wallbit API and use this skill to:

1. **Generate code with the correct structure** - Includes docstrings, error handling and camelCase naming
2. **Use the correct endpoints** - Knows all available endpoints and their parameters
3. **Implement authentication** - Correctly configures the `X-API-Key` header
4. **Handle errors** - Implements 401, 403, 422, 429 error handling

## Available Endpoints

| Category     | Endpoint                             | Method | Description                   |
| ------------ | ------------------------------------ | ------ | ----------------------------- |
| Balance      | `/api/public/v1/balance/checking`    | GET    | Checking account balance      |
| Balance      | `/api/public/v1/balance/stocks`      | GET    | Investment portfolio          |
| Transactions | `/api/public/v1/transactions`        | GET    | Transaction history           |
| Trades       | `/api/public/v1/trades`              | POST   | Execute buy/sell              |
| Account      | `/api/public/v1/account-details`     | GET    | Bank account details          |
| Wallets      | `/api/public/v1/wallets`             | GET    | Crypto wallet addresses       |
| Assets       | `/api/public/v1/assets`              | GET    | List available assets         |
| Assets       | `/api/public/v1/assets/{symbol}`     | GET    | Specific asset info           |
| Operations   | `/api/public/v1/operations/internal` | POST   | Investment deposit/withdrawal |

## Supported Languages

The skill includes examples and guides for:

- **PHP/Laravel** - With Illuminate HTTP Client
- **JavaScript** - Native Fetch API
- **Python** - Requests library

## Configuration

### Environment Variables

```bash
# PHP/Laravel
WALLBIT_API_KEY=your_api_key_here

# JavaScript
WALLBIT_API_KEY=your_api_key_here

# Python
WALLBIT_API_KEY=your_api_key_here
```

### Laravel Config

Add in `config/services.php`:

```php
'wallbit' => [
    'api_key' => env('WALLBIT_API_KEY'),
],
```

## Skill Structure

```
skills/
├── SKILL.md           # Main skill definition
├── README.md          # This file
├── api-reference.md   # Detailed endpoint documentation
└── examples.md        # Complete code examples
```

## Code Conventions

This skill follows these conventions:

- **Naming**: camelCase for functions and variables
- **Documentation**: PHPDoc docstrings on all functions
- **Security**: API Keys in environment variables, never hardcoded
- **Error handling**: Always include handling for codes 401, 403, 422, 429

## Integration with Claude

### Claude Desktop (MCP)

To use this skill with Claude Desktop, configure the `claude_desktop_config.json` file:

**macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
**Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "wallbit-api": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-server-fetch"],
      "env": {
        "WALLBIT_API_KEY": "your_api_key_here",
        "WALLBIT_BASE_URL": "https://api.wallbit.io"
      }
    }
  }
}
```

You can then include the content of `SKILL.md` as context in your conversation with Claude or use the Projects feature to add it as base knowledge.

### Claude API (System Prompt)

To integrate with the Claude API, include the skill content in the system prompt:

```python
import anthropic
from pathlib import Path

# Load the skill
skill_content = Path("skills/SKILL.md").read_text()
api_reference = Path("skills/api-reference.md").read_text()

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=4096,
    system=f"""You are an expert assistant for the Wallbit API.

{skill_content}

## API Reference
{api_reference}
""",
    messages=[
        {"role": "user", "content": "How do I check my Wallbit balance?"}
    ]
)

print(message.content[0].text)
```

### Claude Projects

1. Create a new project in Claude
2. In the "Project knowledge" section, upload the files:
   - `SKILL.md`
   - `api-reference.md`
   - `examples.md`
3. Claude will automatically use this knowledge in all project conversations

### Example Prompt with Context

If you don't have access to Projects, you can include the skill directly:

```
Context: I am working with the Wallbit API.
Base URL: https://api.wallbit.io
Auth: X-API-Key header

Available endpoints:
- GET /api/public/v1/balance/checking
- GET /api/public/v1/balance/stocks
- GET /api/public/v1/transactions
- POST /api/public/v1/trades
- GET /api/public/v1/account-details
- GET /api/public/v1/wallets
- GET /api/public/v1/assets
- GET /api/public/v1/assets/{symbol}
- POST /api/public/v1/operations/internal

My question: [your question here]
```

## Additional Resources

- [Official Wallbit Documentation](https://api.wallbit.io/docs)
- [api-reference.md](api-reference.md) - Detailed API reference
- [examples.md](examples.md) - Code examples

## License

Internal Wallbit use.
