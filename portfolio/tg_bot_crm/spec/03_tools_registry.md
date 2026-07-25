# 03 Tools Registry — Function Calling

## 1. Overview

Централизованный реестр инструментов (tools) для OpenAI function calling. Каждый tool — это JSON Schema + функция execute(). Tools регистрируются в реестре и могут быть включены в запрос к LLM для автоматического вызова.

## 2. Base Class

```python
# src/llm_client/tools/base.py

from dataclasses import dataclass, field
from typing import Any, Callable, Awaitable

@dataclass
class Tool:
    name: str                              # unique tool name
    description: str                       # human-readable description (for LLM)
    parameters: dict                       # JSON Schema for parameters
    groups: list[str]                      # which LLM groups can use ("dialogue", "router")
    handler: Callable[[dict[str, Any]], Awaitable[dict[str, Any]]]

    def to_openai_format(self) -> dict:
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters,
            }
        }
```

## 3. Registry

```python
# src/llm_client/tools/registry.py

from .base import Tool

_tools: dict[str, Tool] = {}

def register(tool: Tool) -> None:
    _tools[tool.name] = tool

def get_all(group: str = None) -> list[Tool]:
    if group:
        return [t for t in _tools.values() if group in t.allowed_groups]
    return list(_tools.values())

def get(name: str) -> Tool | None:
    return _tools.get(name)

def to_openai_format(group: str = None) -> list[dict]:
    return [t.to_openai_format() for t in get_all(group)]
```

## 4. Tool Definitions

### 4.1 `create_lead` — Создать лид

```python
# src/llm_client/tools/create_lead.py

from .base import Tool

async def create_lead_handler(params: dict) -> dict:
    from src.executor.crm import create_lead_in_sheets
    return await create_lead_in_sheets(
        user_id=params["user_id"],
        name=params.get("name"),
        phone=params.get("phone"),
        email=params.get("email"),
        source="telegram_bot",
        notes=params.get("notes"),
    )

create_lead_tool = Tool(
    name="create_lead",
    description="Создать нового клиента в CRM (Google Sheets). Возвращает ссылку на таблицу.",
    parameters={
        "type": "object",
        "properties": {
            "name": {"type": "string", "description": "Имя клиента"},
            "phone": {"type": "string", "description": "Телефон клиента"},
            "email": {"type": "string", "description": "Email клиента"},
            "notes": {"type": "string", "description": "Примечания / контекст"}
        },
        "required": ["name", "phone", "email"]
    },
    allowed_groups=["dialogue"],
    handler=create_lead_handler
)
```

### 4.2 `create_payment`

```python
# src/llm_client/tools/create_payment.py

from .base import Tool

async def create_payment_handler(params: dict) -> dict:
    """Генерирует Stripe Payment Link в тестовом режиме"""
    from src.executor.payments import create_stripe_payment_link
    return await create_stripe_payment_link(
        amount=params["amount"],
        currency=params.get("currency", "usd"),
        description=params.get("description", ""),
        quantity=int(params.get("quantity", 1))
    )

create_payment_tool = Tool(
    name="create_payment",
    description="Сгенерировать ссылку на оплату через Stripe (тестовый режим)",
    parameters={
        "type": "object",
        "properties": {
            "amount": {"type": "number", "description": "Сумма в минималъных единицах валюты (центах / копейках)"},
            "currency": {"type": "string", "description": "Валюта: usd, eur, etc."},
            "description": {"type": "string", "description": "Описание платежа"},
            "quantity": {"type": "integer", "description": "Количество товаров/услуг"}
        },
        "required": ["amount", "description"]
    },
    allowed_groups=["dialogue"],
    handler=create_payment_handler
)
```

### 4.3 `create_ticket`

```python
# src/llm_client/tools/create_ticket.py

from .base import Tool

async def create_ticket_handler(params: dict) -> dict:
    from src.executor.crm import create_ticket_in_sheets
    return await create_ticket_in_sheets(
        user_id=params["user_id"],
        issue_type=params["issue_type"],
        description=params.get("description", ""),
    )

create_ticket_tool = Tool(
    name="create_ticket",
    description="Создать тикет поддержки для существующего клиента",
    parameters={
        "type": "object",
        "properties": {
            "issue_type": {"type": "string", "description": "Тип проблемы: 'product', 'payment', 'other'"},
            "description": {"type": "string", "description": "Описание проблемы"},
        },
        "required": ["issue_type", "description"]
    },
    allowed_groups=["dialogue"],
    handler=create_ticket_handler
)
```

### 4.4 `request_human`

```python
# src/llm_client/tools/request_human.py

from .base import Tool

async def request_human_handler(params: dict) -> dict:
    from src.executor.human_review import create_review_file
    return await create_review_file(
        user_id=params["user_id"],
        query=params["query"],
        context=params.get("context", ""),
        response=params.get("response", ""),
    )

request_human_tool = Tool(
    name="request_human",
    description="Запросить помощь руководителя. Используется когда информация не соответствует политикам компании.",
    parameters={
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "Вопрос к руководителю"},
            "context": {"type": "string", "description": "Контекст диалога"},
            "response": {"type": "string", "description": "Проблемный ответ который не прошёл проверку"}
        },
        "required": ["query", "context"]
    },
    allowed_groups=["dialogue"],
    handler = request_human_handler
)
```

## 5. Tool Initialization

При старте приложения, все tool'ы регистрируются в `registry`:

```python
# src/llm_client/tools/__init__.py

from .registry import register
from .create_lead import create_lead_tool
from .create_payment import create_payment_tool
from .create_ticket import create_ticket_tool
from .request_human import request_human_tool

def init_tools():
    register(create_lead_tool)
    register(create_payment_tool)
    register(create_ticket_tool)
    register(request_human_tool)
```

## 6. Tool Used in Dialogue

Когда `Dialogue Manager` вызывает Response Generator, он передаёт tools:

```python
tools = tools_registry.get_all(group="dialogue")

response = await llm_client.query_with_tools(
    group="dialogue",
    system_prompt=system_prompt,
    user_message=context,
    tools=tools,
)
```

OpenAI модель может ответить либо текстом (ответ клиенту), либо tool_call. После tool_call результат возвращается в модель для финального ответа.

## 7. Tool Execution Flow

```
Response Generator
    │
    ▼
LLM запрос (system + user + tools)
    │
    ├── model returns: choice = "text"   → отправляем пользователю
    └── model returns: choice = "function_call" → execute tool → результат call_point
        ↓ обратно в LLM с tool_call_response role="tool"
        LLM возвращает финальный ответ → отправляем пользователю
```

## 8. Adding New Tool

1. Создать файл в `tools/` с классом Tool
2. Импортировать и зарегистрировать в `__init__.py`
3. (Опционально) реализовать handler в соотв. executor-модуле

## 9. Error Handling

- Если handler выбрасывает исключение → возвращается error dict в tool_result
- LLM видит ошибку и может перефразировать ответ
- Ошибки логируются