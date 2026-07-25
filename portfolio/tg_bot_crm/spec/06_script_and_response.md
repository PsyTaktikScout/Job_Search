# 06 Script Generator & Response Generator

## 1. Overview

Два компонента DIALOG pipeline:

- **Script Generator** — создаёт `task_list`, `required_fields` и `dialogue_script` (шаблон диалога). Вызывается при входе в блок или смене блока
- **Response Generator** — на основе текущего состояния, скрипта как контекста, и последнего сообщения пользователя генерирует актуальный ответ

## 2. Script Generator (`src/dialogue/script_generator.py`)

### 2.1 Когда вызывается

1. При первом входе в блок после Router
2. При смене блока (block_id изменился)
3. При проверке актуальности скрипта (если скрипт устарел / не соответствует ситуации)

### 2.2 Возвращает

```json
{
  "task_list": [
    "Задача 1: Идентифицировать проблему клиента",
    "Задача 2: Уточнить детали (когда началась, частота, условия)",
    "Задача 3: Предложить решение или направить к специалисту",
    "Задача 4: Создать тикет в поддержку"
  ],
  "required_fields": [
    {"name": "client_name", "type": "string", "description": "Имя клиента"},
    {"name": "phone", "type": "phone", "description": "Номер телефона"},
    {"name": "product_model", "type": "string", "description": "Модель продукта"},
    {"name": "issue_description", "type": "string", "description": "Описание проблемы"},
    {"name": "purchase_date", "type": "date", "description": "Дата покупки"}
  ],
  "dialogue_script": [
    {"role": "operator", "message": "Здравствуйте. Я слышу, у вас проблема с X100. Давайте уточним."},
    {"role": "client", "message": "Да, X100 перестал заряжаться вчера"},
    {"role": "operator", "message": "Понял. Давайте выясним: когда вы точогда заметили?"},
    {"role": "client", "message": "Вчера после обеда, подключил — не реагирует"},
    {"role": "operator", "message": "Спасибо за уточнение. Для регистрации заявки нужны ваше имя и телефон."},
    {"role": "client", "message": "Иван Петров, +79001234567"},
    {"role": "operator", "message": "Спасибо, Иван. Я зарегистрирую заявку. Наш специалист свяжется с вами в течение 2 часов."}
  ]
}
```

### 2.3 Prompt

```python
SCRIPT_SYSTEM_PROMPT = """
Ты — оператор входящей линии компании в месенжере. Тебе нужно составить:

1. **task_list** — список задач, которые нужно решить в этом диалоге. Задачи формулируются на языке пользователя. Если требуется несколько целевых действий — укажи их отдельными пунктами.

2. **required_fields** — список обязательных полей для сбора. Каждое поле: name, type, description.
   Поля зависят от целей в task_list:
   - Для лида: name, phone, email, company
   - Для покупки/оплаты: amount, product, quantity
   - Для поддержки: name, phone, product_model, issue_description
   - Для консультации: name, phone, email

3. **dialogue_script** — пример диалога в виде реплик. Важно чтобы скрипт был реалистичным с:
   - Приветствием
   - Уточнением потребности
   - Активным слушанием (не НЛП)
   - Обработкой возражений (если характерно для диалога)
   - Естественным завершением
   - Рекомендации по струртуре диалога
   Формат: [{"role": "operator/client", "message": "..."}, ...]

Отвечай строго в JSON формате с ключами task_list, required_fields, dialogue_script.
"""

SCRIPT_USER_PROMPT = """Блок: {block_id} ({block_name})
Тип сообщения: {intent}
Язык общения: {language}
Тон: {tone}
Текущие собранные данные: {collected}
Какие поля ещё нужны (из прошлого цикла): {current_required}

Составь task_list, required_fields и dialogue_script для диалога с ответить на входящее сообщение."""
```

### 2.4 Code

```python
# src/dialogue/script_generator.py

async def generate_script(
    client: LLMClient,
    context: dict,  # from context builder
) -> dict:
    """Генерирует task_list, required_fields и dialogue_script."""
    ...

async def check_script_valid(
    client: LLMClient,
    current_script: list,
    context: dict,
) -> bool:
    """Проверяет, актуален ли текущий скрипт."""
    ...
```

## 3. Response Generator (`src/dialogue/response_generator.py`)

### 3.1 Цель

Сгенерировать **конкретный ответ** пользователю, используя скрипт как контекст (не как шаблон). Ответ — не цитата из скрипта.

### 3.2 Prompt

```python
RESPONSE_SYSTEM_PROMPT = """
Ты — оператор диалоговой линии компании на мессенжере. Ты отвечаешь клиенту, который обратился к тебе.

КОНТЕКСТ:
- Информация о компании: {company_info}
- Блок диалога: {block}
- Язык общения: {language}
- Тональность клиента: {tone}
- Собранные данные: {collected}
- Поля которые ещё нужно собрать: {missing}
- Скрипт диалога: {script}

ПРАВИЛА:
1. Не цитируй скрипт. Скрипт — только пример, твой ответ должен звучать БОЛЕЕ естественно.
2. Отвечай на последнее сообщение пользователя, смотря на историю диалога.
3. Если сообщение уже в жанре персональных данных (телефон, email), то проверь валидность (формально) и двигайся дальше без повторного запроса того же.
4. Если клиент задаёт вопрос не по теме — мягко верни к главной цели — дай информацию которая поможет.
5. Если нет недостатков для сбора необходимых данных —  веди к завершению.
5. Если не хватает данных — задай следующий вопрос из нужного списка, но как часть естественного разговора, не просто перечисляй.

Твой ответ должен быть **только текстом** сообщения для клиента. Без JSON, без мета-ответов.
"""

RESPONSE_USER_PROMPT = """
История:
{history}

Последнее сообщение клиента:
"{user_message}"

Твоя ответ:
"""
```

### 3.3 Code

```python
# src/dialogue/response_generator.py

async def generate_response(
    client: LLMClient,
    session: dict,
    company_info: dict,
) -> str:
    """Генерирует текстовый ответ клиенту."""
    context = build_response_context(session, company_info)
    
    resp = await client.query(
        group="dialogue",
        system_prompt=RESPONSE_SYSTEM_PROMPT.format(
            company_info=json.dumps(company_info, ensure_ascii=False),
            block=session.get("block_id", "unknown"),
            language=session.get("language", "unknown"),
            tone=session.get("tone", "neutral"),
            collected=json.dumps(session.get("collected", {}), ensure_ascii=False),
            missing=json.dumps(session.get("missing_fields", []), ensure_ascii=False),
            script=json.dumps(session.get("dialogue_script", []), ensure_ascii=False),
        ),
        user_message=RESPONSE_USER_PROMPT.format(
            history=json.dumps(session.get("history", [])[-5:], ensure_ascii=False),
            message=session.get("last_message", ""),
        ),
        request_type="text",
        temperature=0.7,
    )
    return resp.strip()
```

## 4. Field Validator (`src/dialogue/response_validator.py`)

Проверяет, что полученное от пользователя значение соответствует типу поля:

```python
# src/dialogue/response_validator.py

import re

def validate_email(value: str) -> bool:
    return bool(re.match(r"^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z]{2,}$", value))

def validate_phone(value: str) -> bool:
    cleaned = re.sub(r"[\s\-\(\)]", "", value)
    return len(cleaned) >= 7 and cleaned.startswith(("+", "0", "7", "8", "3"))
```

Применение: после Response Generator, когда пользователь отвечает на поле типа `phone`, проверяем валидность перед сохранением в collected.

## 5. Context Builder Integration

Script Generator и Response Generator не должны сами забирать данные из БД и company_info. Это делает Context Builder (`src/context/builder.py`):

```python
# src/context/builder.py

async def build_context(
    session: dict,
    company_info_path: str = "data/company_info.json",
) -> dict:
    """Собирает полный контекст для Script Generator или Response Generator."""
    with open(company_info_path) as f:
        company_info = json.load(f)
    
    return {
        "block_id": session["block_id"],
        "block_name": BLOCK_NAMES[session["block_id"]],
        "intent": session.get("intent_label"),
        "language": session.get("language", "russian"),
        "tone": session.get("tone", "neutral"),
        "collected": session.get("collected", {}),
        "required": session.get("required", []),
        "script": session.get("script", []),
        "history": session.get("history", [])[-10:],
        "company_info": company_info.get("common", {}),  # Non-block информация
    }
```

## 6. File Map

```
src/dialogue/
├── script_generator.py     # task_list + required_fields + dialogue_script
├── response_generator.py    # compose user message from context
├── response_validator.py    # email/phone validation
├── session_manager.py       # SQLite CRUD
├── state_machine.py         # FSM orchestrator
└── db.py                    # SQLite init

src/context/
└── builder.py               # context assembly for LLM prompts
```