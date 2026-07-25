# 02 LLM Client — Failover & API Integration

## 1. Overview

Единый модуль для всех LLM-запросов в проекте. OpenAI-совместимое API. Поддерживает автоматический failover между моделями внутри группы и exponential backoff при недоступности всех моделей.

## 2. Configuration

### `config.yaml` — LLM groups

```yaml
llm_groups:
  router:
    - priority: 1
      provider: "openai"
      model: "gpt-4o-mini"
    - priority: 2
      provider: "deepseek"
      model: "deepseek-chat"
  dialogue:
    - priority: 1
      provider: "openai"
      model: "gpt-4o"
    - priority: 2
      provider: "anthropic"
      model: "claude-3.5-sonnet"
  guardrails:
    - priority: 1
      provider: "openai"
      model: "gpt-4o-mini"
```

### `.env` — API keys (one on this file per provider)

```
OPENAI_API_KEY=sk-...
DEEPSEEK_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-...
```

### Процесс из ключ в `.env`

- Каждый провайдер использует ключ с именем `{PROVIDER}_API_KEY` (upper case)
- Базовый URL для провайдера либо стандартный (знает клиент), либо явно указан в `config.yaml` в поле `base_url`
- Формат имени ключа: `{PROVIDER}_API_KEY` транслированы в верхний регистр
- **Пример**:

```.env
OPENAI_API_KEY=sk-abc123
DEEPSEEK_API_KEY=sk-xyz789
ANTHROPIC_API_KEY=sk-ant-...
```

## 3. Failover Logic

### Принцип

- Запрос C диалоговой группе (например, `group="router"`)
- Модели в группе упорядочены по `priority`
- Попытка запроса к первой модели → ошибка → вторая → ...
- Если **все модели в группе** вернули ошибку:
  - пауза 1s
  - повторный проход по всем моделям
  - если опять все ошибки → пауза 2s → 4s → 8s → 16s → 32s → 64s → 128s → 256s
  - после 256s паузы → сброс к 1s и всё заново
  - Всего 10 циклов (10 × N_models попыток)
  - Если после 10 циклов всё ещё ошибка → исключение `LLMFailoverExhausted`

### Exponential backoff

```
Цикл 1: 0s (first attempt)
Цикл 2: пауза 1s
Цикл 3: пауза 2s
Цикл 4: пауза 4s
Цикл 5: пауза 8s
Цикл 6: пауза 16s
Цикл 7: пауза 32s
Цикл 8: пауза 64s
Цикл 9: пауза 128s
Цикл 10: пауза 256s → либо удача, либо выброс исключения `LLMClientFailoverExhausted`
```

## 4. API

### `LLMClient`

```python
class LLMClient:
    def __init__(self, config_path: str = "config.yaml", env_path: str = ".env"):
        """Загружает config.yaml и .env при создании"""

    async def query(
        self,
        group: str,              # "router" | "dialogue" | "guardrails"
        system_prompt: str,
        user_message: str,
        request_type: str = "json",  # "json" | "json_tool_call" | "text"
        tools: list[dict] | None = None,  # for function calling
        max_tokens: int = 4096,
        temperature: float = 0.3
    ) -> dict | str:
        """
        Отправляет запрос к LLM-группе группой group.
        Returns:
          - parsed JSON dict (request_type="json")
          - raw string (request_type="text")
        Throws LLMClientFailoverExhausted after 10 cycles of all models in group
        """
```

### Метод `query_with_tools`

```python
async def query_with_tools(
    self,
    group: str,
    system_prompt: str,
    user_message: str,
    tools: list[dict],
) -> dict:
    """
    Отправляет запрос с tools. Если модель решит вызвать tool,
    отправляет ответ модели обратно с tool_result и продолжает цикл.
    
    Returns final JSON response from model after tool calls.
    """
```

## 5. Providers Configuration

В `config.yaml`:

```yaml
providers:
  openai:
    base_url: "https://api.openai.com/v1"
  deepseek:
    base_url: "https://api.deepseek.com"
  anthropic:
    base_url: "https://api.anthropic.com"
```

Если `base_url` не указан → используется стандартный URL для провайдера.

## 6. JSON Mode

Все запросы используют `response_format: { "type": "json_object" }` (или `response_format: {"type": "json_schema"}` где возможно).

**Формат запроса (OpenAI API):**
```python
response = await client.chat.completions.create(
    model=model,
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_message},
    ],
    response_format={"type": "json_object"},
    temperature=temperature,
    max_tokens=max_tokens,
)
```

**Парсинг ответа:**
- Если `request_type="json"` → `json.loads(response.choices[0].message.content)`
- Если `request_type="text"` → `response.choices[0].message.content`
- Если `request_type="json_tool_call"` → парсить structure

## 7. Error Handling

```python
class LLMClientFailoverExhausted(Exception):
    """Все 10 циклов failed"""
    pass
```

**Логирование:** Каждая ошибка логируется:
- timestamp
- provider + model
- group
- error type / status code
- текущий цикл / попытка

## 8. File Map

```
src/llm_client/
├── client.py          # LLMClient class: failover + api calls
├── groups.py          # For parsing llm_groups config
├── config.py          # Load providers, base URLs, keys
└── tools/
    ├── registry.py    # tool registration
    ├── base.py        # Tool abstract class
    ├── create_lead.py
    ├── create_payment.py
    ├── create_ticket.py
    └── request_human.py
```

## 9. Dependencies

- `openai` — asynced OpenAI client
- `python-dotenv` — loading .env
- `PyYAML` — loading config.yaml
- `json` — stdlib