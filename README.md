# Wallbit Skills

Skill de Cursor para integración con la API pública de Wallbit.

## Descripción

Este skill proporciona a Cursor el conocimiento necesario para trabajar con la API de Wallbit, permitiendo generar código correcto y siguiendo las mejores prácticas para:

- Consultar balances (checking y stocks)
- Ver historial de transacciones
- Ejecutar trades de compra/venta
- Obtener detalles de cuenta bancaria
- Gestionar wallets crypto
- Consultar información de assets

## Instalación

Para usar este skill en Cursor, copia la carpeta `skills` a tu proyecto o configura la ruta en las preferencias de Cursor:

```
~/.cursor/skills/wallbit-skill/SKILL.md
```

## Uso

Una vez instalado, Cursor detectará automáticamente cuándo trabajas con la API de Wallbit y utilizará este skill para:

1. **Generar código con la estructura correcta** - Incluye docstrings, manejo de errores y nomenclatura camelCase
2. **Usar los endpoints correctos** - Conoce todos los endpoints disponibles y sus parámetros
3. **Implementar autenticación** - Configura correctamente el header `X-API-Key`
4. **Manejar errores** - Implementa manejo de errores 401, 403, 422, 429

## Endpoints Disponibles

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

## Lenguajes Soportados

El skill incluye ejemplos y guías para:

- **PHP/Laravel** - Con Illuminate HTTP Client
- **JavaScript** - Fetch API nativo
- **Python** - Requests library

## Configuración

### Variables de Entorno

```bash
# PHP/Laravel
WALLBIT_API_KEY=your_api_key_here

# JavaScript
WALLBIT_API_KEY=your_api_key_here

# Python
WALLBIT_API_KEY=your_api_key_here
```

### Laravel Config

Agrega en `config/services.php`:

```php
'wallbit' => [
    'api_key' => env('WALLBIT_API_KEY'),
],
```

## Estructura del Skill

```
skills/
├── SKILL.md           # Definición principal del skill
├── README.md          # Este archivo
├── api-reference.md   # Documentación detallada de endpoints
└── examples.md        # Ejemplos completos de código
```

## Convenciones de Código

Este skill sigue las siguientes convenciones:

- **Nomenclatura**: camelCase para funciones y variables
- **Documentación**: Docstrings PHPDoc en todas las funciones
- **Seguridad**: API Keys en variables de entorno, nunca hardcodeadas
- **Manejo de errores**: Siempre incluir manejo de códigos 401, 403, 422, 429

## Integración con Claude

### Claude Desktop (MCP)

Para usar este skill con Claude Desktop, configura el archivo `claude_desktop_config.json`:

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

Luego, puedes incluir el contenido de `SKILL.md` como contexto en tu conversación con Claude o usar la funcionalidad de Projects para agregarlo como conocimiento base.

### Claude API (System Prompt)

Para integrar con la API de Claude, incluye el contenido del skill en el system prompt:

```python
import anthropic
from pathlib import Path

# Cargar el skill
skill_content = Path("skills/SKILL.md").read_text()
api_reference = Path("skills/api-reference.md").read_text()

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=4096,
    system=f"""Eres un asistente experto en la API de Wallbit.

{skill_content}

## Referencia de API
{api_reference}
""",
    messages=[
        {"role": "user", "content": "¿Cómo consulto mi balance en Wallbit?"}
    ]
)

print(message.content[0].text)
```

### Claude Projects

1. Crea un nuevo proyecto en Claude
2. En la sección "Project knowledge", sube los archivos:
   - `SKILL.md`
   - `api-reference.md`
   - `examples.md`
3. Claude usará automáticamente este conocimiento en todas las conversaciones del proyecto

### Ejemplo de Prompt con Contexto

Si no tienes acceso a Projects, puedes incluir el skill directamente:

```
Contexto: Estoy trabajando con la API de Wallbit.
Base URL: https://api.wallbit.io
Auth: Header X-API-Key

Endpoints disponibles:
- GET /api/public/v1/balance/checking
- GET /api/public/v1/balance/stocks
- GET /api/public/v1/transactions
- POST /api/public/v1/trades
- GET /api/public/v1/account-details
- GET /api/public/v1/wallets
- GET /api/public/v1/assets
- GET /api/public/v1/assets/{symbol}
- POST /api/public/v1/operations/internal

Mi pregunta: [tu pregunta aquí]
```

## Recursos Adicionales

- [Documentación Oficial de Wallbit](https://api.wallbit.io/docs)
- [api-reference.md](api-reference.md) - Referencia detallada de la API
- [examples.md](examples.md) - Ejemplos de código

## Licencia

Uso interno de Wallbit.
