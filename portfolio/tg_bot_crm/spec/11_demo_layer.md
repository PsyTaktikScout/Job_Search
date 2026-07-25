# 11 Demo Layer — Прозрачность

## 1. Overview

Demo Layer показывает внутренние шаги бота через дописывание в одно редактируемое сообщение. Если после сообщения пользователя нет сообщений от бота — создаётся новое (без звука), иначе — редактируется существующее с дописыванием.

## 2. Behavior

```python
# src/demo/progress.py

from aiogram.types import Message
from aiogram import Bot
import asyncio

class DemoProgress:
    def __init__(self, bot: Bot):
        self.bot = bot
        self.debug_mode = True  # настраивается из config.yaml
    
    async def progress(
        self,
        user_id: int,
        message: str,
        message_id: int | None = None,
    ) -> int | None:
        """
        Показать прогресс: дописать в существующее сообщение или создать новое.
        
        Args:
            user_id: Telegram user ID
            message: текст прогресса (одна строка)
            message_id: ID существующего сообщения, или None если нет
        
        Returns:
            message_id (int) для последующего апдейта
        """
        if not self.debug_mode:
            return message_id
        
        try:
            if message_id:
                # Редактировать существующее сообщение: append
                chat = await self.bot.get_chat(user_id)
                msg = await self.bot.edit_message_text(
                    chat_id=user_id,
                    message_id=message_id,
                    text=message,
                )
                return msg.message_id
            else:
                # Отправить новое сообщение (без звука)
                msg = await self.bot.send_message(
                    chat_id=user_id,
                    text=message,
                    disable_notification=True,
                )
                await asyncio.sleep(0.1)  # минимальная пауза для Telegram rate limit
                return msg.message_id
        except Exception as e:
            log(f"DemoProgress error: {e}")
            return message_id
    
    async def append(
        self,
        user_id: int,
        new_line: str,
        message_id: int | None,  # ID демо-сообщения из сессии
        session: dict,
    ) -> int | None:
        """
        Дописать строку к существующему демо-сообщению. Не затирает предыдущее.
        Если демо-сообщения нет — создаёт новое.
        """
        if not self.debug_mode:
            return message_id
        
        try:
            if message_id:
                chat = await self.bot.get_chat(user_id)
                msg = await self.bot.edit_message_text(
                    chat_id=user_id,
                    message_id=message_id,
                    text=msg.text + "\n" new_line,
                )
                return msg.message_id
            else:
                msg = await self.bot.send_message(
                    chat_id=user_id,
                    text=new_line,
                    disable_notification=True,
                )
                session_manager.update_session(user_id, demo_message_id=msg.message_id)
                return msg.message_id
        except Exception:
            log(f"DemoProgress append error for user {user_id}")
            return message_id
```

## Sequence of Updates

Каждый шаг в DIALOG pipeline и ACTION добавляет строку в демо-сообщение:

```python
# Где-то в state_machine.py
demo = DemoProgress(bot)

# Step 1: После Router
demo_id = await demo.append(user_id, "🤖 Анализирую запрос...", demo_id, session)

# Step 2: После классификации
demo_id = await demo.append(user_id, f"🤖 Язык: {lang}, тон: {tone}", demo_id, session)
demo_id = await demo.append(user_id, f"🤖 Блок: {block_name} ({block_id})", demo_id, session)

# Step 3: Script generation
demo_id = await demo.append(user_id, "🤖 Готовлю скрипт диалога...", demo_id, session)
demo_id = await demo.append(user_id, f"🤖 Скрипт ✅", demo_id, session)

# Step 4: Response
demo_id = await demo.append(user_id, "🤖 Готовлю ответ...", demo_id, session)

# Step 5: Guardrails
demo_id = await demo.append(user_id, f"🤖 Проверяю ответ: соответствие политикам ✅, юридические нормы ...", demo_id, session)

# Step 6: Collect
demo_id = await demo.append(user_id, f"🤖 Собираю данные: {collected_str}", demo_id, session)

# Step 7: Action
demo_id = await demo.append(user_id, "🤖 Выполняю действие: создаю лид в CRM...", demo_id, session)
demo_id = await demo.append(user_id, f"🤖 ✅ Готово! Лид #{lead_id}, таблица: {crm_url}", demo_id, session)

# Сохранить demo_message_id в сессии
session_manager.set_demo_message_id(user_id, demo_id)
```

## 5. Demo Message ID в SQLite

В `sessions` таблице:
```sql
...
demo_message_id INTEGER,
...
```

При каждом создании/обновлении демо-сообщения, `demo_message_id` сохраняется.

## 6. Debug / Non-debug mode

```yaml
# config.yaml
debug_mode: true   # true = показывать прогресс, false = без демо
```

Если `debug_mode: false`, `DemoProgress.progress()` и `append()` просто возвращают `message_id` без отправки сообщений.

## 7. Demo Message on Start

При старте диалога (state: `WELCOME` → `ROUTER`):

```python
demo_id = await demo.message(
    user_id,
    "🤖 Бот-оператор готов к работе.",
    None,
    session
)
session_manager.set_demo_message_id(user_id, demo_id)
```

## 8. File Map

```
src/demo/
└── progress.py    # DemoProgress class
```