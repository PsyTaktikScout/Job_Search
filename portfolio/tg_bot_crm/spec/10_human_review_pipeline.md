# 10 Human Review Pipeline

## 1. Overview

Когда Guardrails не может исправить ответ после N попыток, запрос отправляется на human_review. Руководитель вручную заполняет ответ в файле, и бот отправляет клиенту уточнённый ответ.

## 2. File Structure

```
human_review/
├── pending/          # Вопросы, ожидающие ответа руководителя
├── replied/         # Ответы руководителя, перемещённые вручную в эту папку
└── processed/       # Обработанные ответы (архив)
```

## 3. Question File Format

При создании вопроса в `human_review/pending/` создаётся файл:

**Имя:** `YYYYMMDD_HHMMSS_userID.json`

Пример: `20260725_153000_12345.json`

```json
{
  "user_id": 12345,
  "timestamp": "2026-07-25T15:30:00",
  "query": "Вопрос: Клиент спрашивает про скидку 50%, но в price.yaml максимальная скидка 30%. Уточните политику.",
  "context": {
    "block_id": 1,
    "collected": {"client_name": "Иван", "phone": "+79001234567"},
    "last_message": "У вас на сайте написано 'скидка до 50%', но вы даёте только 30%. Почему?"
  },
  "generated_response": "Извините Иван, действительно у нас бывают акции до 50%, но сейчас действует скидка только 30%. Хотите, я уточню у руководителя о возможности специальных условий для вас?",
  "issues": [
    "Граница скидки не подтверждена в документах",
    "Ответ может быть истолкован как обещание скидки без гарантий"
  ],
  "answer": "",
  "requires_moderation": true
}
```

Ключевое поле `answer` — заполняется руководителем вручную.

## 4. File Worker

```python
# src/executor/human_review.py

import os
import astim
import json
from pathlib import Path

HUMAN_REVIEW_DIR = Path("human_review")
CHECK_INTERVAL = 30  # секунд, переопределяется в config.yaml

class HumanReviewWorker:
    def __init__(self):
        self.counter = 0  # сколько файлов ожидают ответа
        self.closed = False
    
    def increment_counter(self):
        self.counter += 1
    
    def decrement_counter(self):
        self.counter = max(0, self.counter - 1)
    
    def sync_counter(self):
        """Синхронизировать counter с реальным количеством файлов в pending/"""
        pending_files = len(list(HUMAN_REVIEW_DIR.glob("pending/*.json")))
        self.counter = pending_files
    
    async def run(self, bot, session_manager):
        """Воркер проверяет replied/ и обрабатывает ответы."""
        while not self.closed:
            self.sync_counter()
            
            if self.counter > 0:
                await self.check_replied(bot, session_manager)
            
            await asyncio.sleep(CHECK_INTERVAL)
    
    async def check_replied(self, bot, session_manager):
        """Проверить папку replied/ на наличие ответов."""
        replied_files = sorted(HUMAN_REVIEW_DIR.glob("replied/*.json"))
        
        for file in replied_files:
            try:
                review_item = json.load(open(file))
            except Exception as e:
                # Переместить в processed/org/ если файл повреждён
                file.rename(HUMAN_REVIEW_DIR / "processed" / "org" / file.name)
                continue
            
            if not review_item.get("answer"):
                # Ответ не заполнен — файл ещё не готов
                continue
            
            # Обработать ответ
            await self.process_reply(review_item, file, bot, session_manager)
            break  # Обрабатываем по одному за цикл
    
    async def process_review(self, review: dict, file_path: Path, bot, session_manager):
        """Ответить на файле с ответом руководителя."""
        user_id = review["user_id"]
        
        # Получить сессию пользователя
        session = await session_manager.get_session(user_id)
        if not session:
            # Сессия не найдена — перемещаем в processed/
            file_path.rename(HUMAN_REVIEW_DIR / "processed" / file_path.name)
            self.decrement_counter()
            return
        
        # Сгенерировать сообщение с учётом ответа руководителя
        message = await self.generate_response(session, review)
        
        # Отправить Guardrails check
        final_response = await guardrails_check(message, ..., session)
        
        # Отправить пользователю
        await bot.send_message(
            chat_id=user_id,
            text=final_response,
            disable_notification=True,
        )
        
        # Переместить файл в processed/
        file_path.rename(HUMAN_REVIEW_DIR / "processed" / file_path.name)
        self.decrement_counter()
    
    async def generate_review_response(self, session: dict, review_item: dict) -> str:
        """Сгенерировать сообщение на основе ответа руководителя."""
        # Это отдельный LLM запрос с контекстом + ответом руководителя
        ...
```

## 5. Question Creation (from Guardrails)

Когда Guardrails исчерпывает retry_limit:

```python
# src/guardrails/checker.py (continuation)

async def send_to_human_review(
    user_id: int,
    dialogue_id: str,
    query: str,
    context: dict,
    generated_response: str,
    issues: list[str],
):
    """Создать файл в human_review/pending/"""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    filename = HUMAN_REVIEW_DIR / "pending" / f"{timestamp}_{user_id}.json"
    
    item = {
        "user_id": user_id,
        "dialogue_id": dialogue_id,
        "timestamp": datetime.now().isoformat(),
        "query": query,
        "context": context,
        "generated_response": generated_response,
        "issues": issues,
        "answer": "",
    }
    
    with open(filename, "w", encoding="utf-8") as f:
        json.dump(item, f, ensure_ascii=False, indent=2)
    
    worker.increment_counter()
    
    log(f"created_human_review: {filename}")
```

## 6. Руководитель (Manual Process)

1. Зайти в `human_review/pending/`, открыть файл
2. Прочитать поле `query` — что произошло
3. Заполнить поле `answer` — ответ
4. Вручную переместить файл из `pending/` в `replied/`

Бот обнаружит файл в `replied/`, проверит что `answer` заполнено, и отправит ответ клиенту.

## 7. Flow Overview

```
Guardrails (N попыток пройдено +?)
    │
    ├── ✅ ответ пройден → отправить пользователю
    └── ❌ ответ не пройден (попытки исчерпаны)
         │
         ▼
    создать human_review/pending/STAMP_userid.json
         │
         ▼
    Руководитель заполняет answer вручную
         │
         ▼
    Руководитель перемещает в replied/
         │
         ▼
    File worker обнаруживает файл в replied/
    1. Проверяет answer заполнено
    2. Генерирует сообщение с учётом ответа
    3. Guardrails проверяет с учётом ответа руководителя
    4. Отправляет пользователю (без звука)
    5. Перемещает в processed/
```

## 8. Config

```yaml
human_review:
  check_interval: 30        # как часто проверять папку replied/ (сек)
  auto_cleanup: true
  cleanup_days: 7           # авто-очистка файлов в processed/ старше 7 дней (заглушка)
```

## 9. File Map

```
src/executor/
└── human_review.py    # HumanReviewWorker + create_review_file

human_review/
├── pending/           # Ожидает ручного ответа
├── replied/           # Ответ записан, ожидает обработки ботом
└── processed/         # Обработано (архив)
    └── org/           # Файлы не соответствующие ожиданию
```