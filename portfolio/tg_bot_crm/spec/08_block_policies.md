# 08 Block Policies & Action Executor

## 1. Overview

Каждый бизнес-блок (1–6) — это отдельный класс, наследующий `BlockAction` в `src/executor/blocks/`. Класс определяет:

- `required_fields` — набор обязательных полей для сбора до выполнения действий
- `allowed_actions` — какие целевые действия разрешены в этом блоке
- `execute_action()` — выполнение целевого действия
- `tool_schemas` — какие tools доступны LLM для этого блока

## 2. Base Class

```python
# src/executor/blocks/base.py

from dataclasses import dataclass, field
from typing import Any, Awaitable, Callable

@dataclass
class BlockAction:
    block_id: int
    block_name: str
    required_fields: list[dict]       # [{name: "full_name", type: "string", desc: "Имя клиента"}, ...]
    allowed_actions: list[dict]       # [{type: "create_lead", tool: "create_lead"}, ...]
    
    async def execute_action(self, action_type: str, session: dict) -> dict:
        """Выполнить целевое действие. Выводит dict с результатом."""
        raise NotImplementedError
    
    async def on_complete(self, session: dict) -> str:
        """Сообщение после ВСЕХ действий (ACTION_DONE)."""
        ...

class Task:
    """1 задача в task_list"""
    title: str                          # Название задачи на языке пользователя
    action_type: str                    # "create_lead", "create_payment", "create_ticket", "none"
    dependencies: list[str]             # поля которые нужно собрать перед выполнением
    outcome: str                        # ожидаемый результат (для демо-сообщения)

```

## 3. Блок 1 — Продажи и новые возможности

```python
# src/executor/blocks/block_1_sales.py

from .base import BlockAction, Task

class Block1Sales(BlockAction):
    block_id = 1
    block_name = "Продажи и новые возможности"

    required_fields = [
        {"name": "client_name", "type": "string", "desc": "Имя клиента"},
        {"name": "phone", "type": "phone", "desc": "Номер телефона"},
        {"name": "email", "type": "email", "desc": "Email для отправки КП"},
        {"name": "product_interest", "type": "string", "desc": "Какой продукт/услуга интересует"},
    ]

    async def execute(self, action_type: str, session: dict, llm_client: LLMClient | None = None) -> dict:
        collected = session.get("collected", {})
        user_id = session["user_id"]
        
        if action_type == "create_lead":
            lead = await crm.create_lead(
                user_id=user_id,
                name=collected.get("client_name"),
                phone=collected.get("phone"),
                email=collected.get("email"),
                block_source=1,
            )
            return {"status": "ok", "lead_id": lead["lead_id"],
                    "crm_link": session.get("crm_sheet_url")}
        
        if action_type == "create_payment":
            payment = await payments.create_stripe_link(
                amount=collected.get("amount"),
                currency="usd",
                description="CP: " + collected.get("product_interest", "Order"),
            )
            return {"status": "ok", "payment_url": payment["url"]}
        
        return {"status": "error", "message": f"Unknown action: {action_type}"}

    def get_tasks(self, context: dict) -> list[Task]:
        """Какие задачи нужно решить в этом диалоге."""
        return [
            Task("Создать лид в CRM", "create_lead", ["client_name", "phone", "email"], "Лид создан"),
            Task("Отправить коммерческое предложение", "create_payment", ["amount"], "Ссылка на оплату отправлена"),
        ]

    async def on_complete(self, session: dict) -> str:
        lead_id = session.get("last_lead_id")
        crm_url = session.get("crm_sheet_url", "")
        return f"Готово! Лид #{lead_id} создан. Таблица CRM: {crm_url}"
```

## 4. Blocks Registry

```python
# src/executor/blocks/registry.py

from .base import BlockAction
from .block_1_sales import Block1Sales
from .block_2_support import Block2Support
from .block_3_info import Block3Info
from .block_4_promo import Block4Promo
from .block_5_trash import Block5Trash
from .block_6_retention import Block6Retention

BLOCKS: dict[int, BlockAction] = {
    1: Block1Sales(),
    2: Block2Support(),
    3: Block3Info(),
    4: Block4Promo(),
    5: Block5Trash(),
    6: Block6Retention(),
}

def get_block(block_id: int) -> BlockAction | None:
    return BLOCKS.get(block_id)

def get_all_blocks() -> dict[int, BlockAction]:
    return BLOCKS
```

## 5. Block 2: Support Example

```python
# src/executor/blocks/block_2_support.py

from .block import BlockAction

class Block2Support(BlockAction):
    block_id = 2
    block_name = "Поддержка существующих клиентов"

    required_fields = [
        {"name": "client_name", "type": "string", "desc": "Имя клиента"},
        {"name": "phone", "type": "phone", "desc": "Телефон для связи"},
        {"name": "product_model", "type": "string", "desc": "Модель продукта"},
        {"name": "issue_description", "type": "string", "desc": "Описание проблемы"},
    ]

    async def execute(self, action_type: str, session: dict, **kwargs) -> dict:
        collected = session.get("collected", {})
        
        if action_type == "create_ticket":
            ticket = await crm.create_ticket(
                user_id=session["user_id"],
                issue_type=collected.get("issue_type", "product"),
                description=collected["issue_description"],
            )
            return {
                "status": "ok",
                "ticket_num": ticket["ticket_number"],
                "crm_url": session.get("crm_sheet_url"),
                "eta": "2 hours",  # из company_info
            }
        
        return {"status": "error", "message": f"Unknown action: {action_type}"}
    
    async def on_complete(self, session: dict) -> str:
        return f"Заявка #{session['last_ticket_num']} создана. Специалист свяжется с вами в течение 2 часов"
```

## 6. Block 5: Non Profile / Trash

```python
class Block5Trash(BlockAction):
    block_id = 5
    block_name = "Непрофильные сообщения"

    required_fields = []  # Ничего не нужно собирать
    allowed_actions = []  # Никаких действий

    async def execute(self, action_type: str, session: dict) -> dict:
        return {"status": "closed", "message": "Непрофильное сообщение"}

    async def on_complete(self, session: dict) -> str:
        return "Похоже, вы ошиблись номером. Всего доброго!"
```

## 7. Action Executor

```python
# src/executor/actions.py

from src.executor.blocks.registry import get_block

async def execute_actions(session: dict, ctx) -> dict:
    """Выполняет все actions из task_list последовательно."""
    block = get_block(session["block_id"])
    if not block:
        return {"status": "error", "message": f"Block {session['block_id']} not found"}
    
    tasks = block.get_tasks(ctx)
    results = []
    
    for task in tasks:
        await demo.progress(f"⏳ {task.title}...")
        result = await block.execute(task.action_type, session, llm_client=ctx.llm)
        results.append({"task": task.title, "result": result})
        await demo.progress(f"✅ {task.title}: {task.outcome}")
    
    return {"status": "ok", "results": results}
```

## 8. Adding New Block

1. Создать `src/executor/blocks/block_X_custom.py`
2. Наследовать `BlockAction`, определить required_fields, execute, on_complete
3. Зарегистрировать в `registry.py`: `BLOCKS[X] = BlockXCustom()`

## 9. File Map

```
src/executor/blocks/
├── __init__.py
├── base.py           # Базовый класс BlockAction
├── block_1_sales.py  # Продажи и новые возможности
├── block_2_support.py # Поддержка (заготовка)
├── block_3_info.py   # Информационные запросы (заготовка)
├── block_4_promo.py  # Маркетинговые акции (заготовка)
├── block_5_trash.py  # Непрофильные сообщения
├── block_6_retention.py # Ультиматум/удержание (заготовка)
└── registry.py       # BLOCKS[block_id] → класс
```