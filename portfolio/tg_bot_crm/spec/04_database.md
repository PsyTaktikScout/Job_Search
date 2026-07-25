# 04 Database — SQLite Schema & Session Manager

## 1. Overview

SQLite через `aiosqlite` для хранения сессий, лидов и тикетов. Один файл `data/bot.db`.

## 2. Tables

### 2.1 `sessions` — Состояние диалогов

```sql
CREATE TABLE IF NOT EXISTS sessions (
    session_id   INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id      INTEGER NOT NULL UNIQUE,       -- Telegram user_id
    state        TEXT    NOT NULL DEFAULT 'WELCOME',  -- WELCOME | DIALOG | ACTION_EXECUTE | ACTION_DONE | CLOSED
    block_id     INTEGER,                       -- 1-6 или NULL
    language     TEXT,                           -- "russian", "english", etc.
    tone         TEXT,                           -- свободное описание тона
    intent_label TEXT,                           -- конкретный интент
    collected    TEXT    DEFAULT '{}',           -- JSON: собранные данные {name, phone, email, ...}
    required     TEXT    DEFAULT '[]',           -- JSON: поля которые ещё нужно собрать
    task_list    TEXT    DEFAULT '[]',           -- JSON: список задач для диалога
    script       TEXT,                           -- текущий скрипт 
    h                  TEXT,                       -- JSON: история сообщений [{role, content, timestamp}]
    demo_message_id INTEGER,                     -- Telegram message_id для demo прогресса
    crm_sheet_url TEXT,                         -- URL Google Sheets таблицы
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Пояснения:**
- `collected` — накопленные данные от пользователя. Меняются на каждом шаге цикла DIALOG
- `required` — список полей которые ещё нужно собрать. Устанавливается Script Generator-ом
- `task_list` — список задач + target actions. Устанавливается Script Generator-ом
- `script` — текущий скрипт-шаблон диалога
- `history` — последние N сообщений (ограничение из конфига). Каждое сообщение: `{role, content, timestamp}`
- `demo_message_id` — ID сообщения в Telegram которое бот редактирует для показа прогресса

### 2.2 `leads` — CRM: контакты и лиды

```sql
CREATE TABLE IF NOT EXISTS leads (
    lead_id      INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id      INTEGER NOT NULL,              -- Telegram user_id
    name         TEXT,
    phone        TEXT,
    email        TEXT,
    company      TEXT,
    source       TEXT DEFAULT 'telegram_bot',
    block_source INTEGER,                       -- из какого блока (1-6)
    status       TEXT DEFAULT 'new',            -- new | contacted | qualified | closed
    crm_sheet_url TEXT,                        -- URL Google таблицы этого пользователя
    notes        TEXT,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2.3 `tickets` — Обращения в поддержку

```sql
CREATE TABLE IF NOT EXISTS tickets (
    ticket_id    INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id      INTEGER NOT NULL,
    lead_id      INTEGER,                       -- FK на leads
    issue_type   TEXT,                          -- product | payment | complaint | other
    description  TEXT,
    status       TEXT DEFAULT 'open',           -- open | in_progress | resolved | closed
    resolution   TEXT,                          -- описание решения
    ticket_number TEXT,                         -- номер для клиента (#XYZ)
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    resolved_at  TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_tickets_user ON tickets(user_id);
CREATE INDEX IF NOT EXISTS idx_tickets_status ON tickets(status);
```

### 2.4 `action_log` — История выполненных действий

```sql
CREATE TABLE IF NOT EXISTS action_log (
    log_id       INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id      INTEGER NOT NULL,
    action_type  TEXT NOT NULL,                 -- create_lead | create_ticket | create_payment | close
    action_data  TEXT,                          -- JSON: параметры действия
    result       TEXT,                          -- JSON: результат действия
    success      BOOLEAN DEFAULT 1,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 3. Session Manager API

```python
# src/dialogue/session_manager.py

class SessionManager:
    def __init__(self, db_path: str = "data/bot.db"):
        ...

    async def init_db(self):
        """Создать таблицы если нет (вызывается при старте)"""
    
    # --- CRUD сессий ---
    
    async def get_session(self, user_id: int) -> dict | None:
        """Загрузить сессию. Десериализует JSON поля."""
    
    async def create_session(self, user_id: int) -> dict:
        """Создать новую сессию со статусом WELCOME."""
    
    async def update_session(self, user_id: int, **fields) -> None:
        """
        Частичное обновление сессии: update_session(user_id, state='DIALOG', block_id=2)
        JSON поля (collected, required, history, task_list) требуют предаврительного
        преобразования (dict → json.dumps перед записью)
        """
    
    async def set_state(self, user_id: int, state: str) -> None:
        """Изменить FSM состояние."""
    
    async def add_to_collected(self, user_id: int, field: str, value: Any) -> None:
        """Добавить поле в collected JSON. Не перезаписывать весь collected."""
    
    async def add_to_history(self, user_id: int, role: str, content: str) -> None:
        """Добавить сообщение в историю. Роль: user / assistant / system."""
        # Автоматически обрезает до N последних сообщений (значение из config.yaml)
    
    # --- CRM ---
    
    async def create_lead(self, user_id: int, name: str, phone: str = None, email: str = None, **kwargs) -> int:
        """Создать лид. Возвращает lead_id."""
    
    async def get_lead(self, user_id: int) -> dict | None:
        """Найти лид по user_id."""
    
    async def create_ticket(self, user_id: int, issue_type: str, description: str) -> int:
        """Создать ticket. Возвращает ticket_id."""
    
    # --- History ---
    
    async def get_history(self, user_id: int, limit: int = 20) -> list[dict]:
        """Получить последние N сообщений."""
```

## 4. JSON Field Serialization

Все `TEXT` поля которые хранят JSON должны сериализоваться/десериализоваться:

```python
import json

async def get_session(self, user_id: int) -> dict | None:
    async with aiosqlite.connect(self.db_path) as db:
        row = await db.execute("SELECT * FROM sessions WHERE user_id = ?", (user_id,))
        row = await row.fetchone()
        if not row:
            return None
        return {
            "collected": json.loads(row["collected"] or "{}"),
            "required": json.loads(row["required"] or "[]"),
            "task_list": json.loads(row["task_list"] or "[]"),
            "history": json.loads(row["history"] or "[]"),
            # ... other fields as-is
        }

async def update_session(self, user_id: int, **kwargs) -> None:
    # Автоматически сериализовать JSON поля
    for key in ("collected", "required", "task_list", "history", "script"):
        if key in kwargs:
            kwargs[key] = json.dumps(kwargs[key], ensure_ascii=False)
    # ... execute UPDATE
```

## 5. Database Initialization

При старте приложения:

```python
# src/dialogue/db.py

async def init_db(db_path: str = "data/bot.db"):
    async with aiosqlite.connect(db_path) as db:
        db.row_factory = aiosqlite.Row
        await db.execute("CREATE TABLE IF NOT EXISTS sessions (...)")
        await db.execute("CREATE TABLE IF NOT EXISTS leads (...)")
        await db.execute("CREATE TABLE IF NOT EXISTS tickets (...)")
        await db.execute("CREATE TABLE IF NOT EXISTS action_log (...)")
        await db.commit()
```

## 6. Config-Driven Settings

```yaml
# config.yaml
database:
  path: "data/bot.db"
  history_limit: 20           # сколько сообщений хранить в history
  session_cleanup_days: 30    # удалять сессии старше N дней (заглушка)
```

## 7. File Map

```
src/dialogue/
├── db.py                  # init_db(), асинхронные helpers
├── session_manager.py     # все CRUD + JSON serialization
├── state_machine.py       # FSM orchestrator (вызывает session_manager)
├── script_generator.py    # task_list + required_fields + script
├── response_generator.py  # compose user message from context
└── response_validator.py  # email/phone validation
```