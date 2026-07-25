# 01 Architecture — Telegram CRM Bot

## 1. Overview

AI-оператор для Telegram, который классифицирует входящие сообщения по 6 бизнес-блокам, выстраивает диалог по сценарию, собирает необходимую информацию и выполняет целевое действие (создание лида в CRM, Stripe-ссылка на оплату и т.д.).

**Прозрачность:** бот показывает внутренние шаги через дописывание в одно редактируемое сообщение (Intent → Script → Guardrails → Data → Action).

## 2. Component Layers

Flow данных слева направо:

```
Telegram User
    ↓
main.py (entry point: aiogram polling)
    ↓
Router (language, tone, block_id, extracted_data, confidence)
    ↓
Session Manager (load/create session from SQLite)
    ↓
Context Builder (company info + history + policies + collected data)
    ↓
Script Generator (plan of tasks, required fields, script)
    ↓
Response Generator (compose actual message → NOT from script template)
    ↓
Guardrails (verify response against company docs, facts, legal)
    ↓                └── retry loop (up to config.limit) ─────────┐
    │                     Если лимит превышен → human_review/     │
    ↓                     Если пройдено → send message             │
Demo Layer (append to progress message, or send new if none)
    ↓
Telegram User replies
    ↓
Router again (re-classify updated context)
    ↓
    ├── block changed       → regenerate task list + required fields
    └── block stays same    → validate existing data, collect missing
    ↓
Loop (generate script → generate response → guardrails → send → reply)
    ↓
All required fields collected → save to SQLite
    ↓
Action Executor (CRM, Stripe, etc.)
    ↓
Telegram User (+ demo updates + CRM link)
```

## 3. Detailed Component Descriptions

### 3.1 Router (`src/classifier/router.py`)

**Назначение:** Первым дешёвым LLM-запросом определяет:
- `language` — язык сообщения (натуральное описание, не enum)
- `tone` — эмоциональный тон (натуральное описание, не фиксированный перечень)
- `block_id` — бизнес-блок 1–6
- `intent_label` — конкретный интент внутри блока
- `extracted_data` — извлечённые из сообщения данные (product, name, etc.)
- `confidence` — уверенность (если action == "welcome", confidence может отсутствовать)

**Формат ответа (JSON Mode):**
```json
{
  "language": "russian",
  "tone": "frustrated_but_contolled", 
  "block_id": 2,
  "intent_label": "product_malfunction",
  "extracted_data": {"product": "X100", "issue": "not_charging"},
  "confidence": 0.92,
  "action": "classify"
}
```

**Правила:**
- `tone` — свободное описание на английском, не ограничивается перечнем. Название tone используется как подсказка для стиля ответа
- `language` — натуральное название языка. Бот всегда отвечает на языке последнего сообщения пользователя
- Если сообщение приветственное/пустое/не относится к профилю компании — `action: "welcome"`
- Если Блок 5 (спам/ошибка) — вежливый ответ без CRM, диалог закрыт
- `confidence < 0.7` → переспросить

### 3.2 Session Manager (`src/dialogue/session_manager.py`)

Backend: **SQLite** через `aiosqlite`.

Хранит:
- Текущее FSM-состояние
- `block_id`, `tone`, `language`
- `collected_data` (JSON: собранные поля)
- `required_fields` (JSON: какие поля ещё нужно собрать)
- `task_list` (JSON: список задач для диалога)
- Историю сообщений (последние N для контекста)

Схема БД в `spec/08_database.md`.

### 3.3 Context Builder (`src/context/builder.py`)

Собирает контекст для LLM-запроса из:
- `data/company_info.json` — информация о компании, продукты, цены
- Истории диалога (последние N сообщений из SQLite)
- Политик текущего блока
- Собранных данных пользователя
- Языка диалога

### 3.4 Script Generator (`src/dialogue/script_generator.py`)

**Когда вызывается:**
1. При первом входе в блок (после Router)
2. При смене блока
3. При проверке актуальности: если скрипт устарел / не соответствует ситуации → генерирует новый

**Что генерирует:**
1. `task_list` — список задач для диалога на языке пользователя (естественная форма)
2. `required_fields` — обязательные поля для сбора
3. `dialogue_script` — скрипт разговора

**Важно:** 
`dialogue_script` — скрипт разговора задается промптом "Нужно дать пример диалога. Важно сделать его реалистичным, с реакцией оператора, с элементами активного слушания, уточнения, завершения. Пример диалога (оператор - клиент). Можно в виде реплик. Также учтем, что скрипты должны быть для "входящей линии компании" (в месенжере). Добавим рекомендации по структуре: приветствие, идентификация, выяснение потребности, обработка возражений (если нужно), завершение. Можно кратко."
Ответ пользователю — это **не цитата** из скрипта. Скрипт используется как дополнительный контекст для генерации актуального сообщения.

### 3.5 Response Generator (`src/dialogue/response_generator.py`)

Генерирует ответ пользователю с учётом:
- Последнего сообщения пользователя
- Скрипта (как контекста, не шаблона)
- Истории диалога
- Информации о компании
- Список полей которые осталось заполнить

### 3.6 Guardrails (`src/guardrails/checker.py`)

**Не заглушка.** Проверяет каждый сгенерированный ответ перед отправкой.

**Процесс проверки:**
1. Сопоставить информацию в ответе с эталонными документами:
   - Правила компании
   - Характеристики и описание продуктов/услуг
   - Установленные цены и скидки
   - Юридические нормы
2. Если информация соответствует → **пройдено**
3. Если информация противоречит → **редактировать** ответ до соответствия
4. Если информации для подтверждения нет → **переписать** без этой информации, или акцентировать что бот уточнит у руководителя

**Цикл проверки:**
- Ответ проверяется и переписывается до тех пор, пока не пройдёт проверку
- Если **количество попыток превышает лимит** (`response.guards.retry_limit` в config.yaml):
  - Создать файл в `human_review/pending/`
  - В файле: запрос к LLM, последний сгенерированный ответ, контекст разговора, графа для ответа руководителя
  - Пометить как требующий модерации
  - Увеличить счётчик ожидающих вопросов

**Эталонные документы:**
- Могут быть пустыми
- Хранятся в `data/` (company_info.json и дополнительные yaml/json файлы)
- Проверка — сопоставление LLM (группа `guardrails` из конфига)

### 3.7 Human Review Pipeline (`src/executor/human_review.py`)

**Создание вопроса:**
1. Генерируется файл `human_review/pending/YYYYMMDD_HHMMSS_userID.json` со структурой:
   ```json
   {
     "user_id": 12345,
     "dialogue_id": "...",
     "query": "Вопрос, требующий уточнения от руководителя",
     "context": "Контекст разговора",
     "generated_response": "Ответ, который не прошёл проверку",
     "answer": "",
     "requires_moderation": true
   }
   ```
2. Счётчик ожидающих вопросов увеличивается

**Файловый воркер:**
- Воркер отслеживает количество в `human_review/pending/` и сравнивает с голосам счётчика
- Если счётчик > 0: каждые N секунд (значение из конфига) проверяет `human_review/replied/`
- **Руководитель:** вручную заполняет поле `answer` в файле и перемещает из `pending/` в `replied/`
- Когда бот находит файл в `replied/`:
  1. Генерирует сообщение с опорой на ответ руководителя + контекст диалога
  2. Сообщение проходит проверку Guardrails с учётом ответа
  3. Сообщение отправляется в соответствующий диалог
  4. Файл перемещается в `human_review/processed/`
  5. Счётчик уменьшается
- Если находит в `replied/` файлы не соответствующие ожиданию (например проблема с идентификатором диалога), перемещается в `processed/org/`
- Обработка — по одному файлу за цикл

**Структура папобшей:**
```
human_review/
├── pending/           # Файлы, ожидающие ответа потомка
├── replied/           # Файлы, которые потомк переместил с ответом
└── processed/         # Файлы после обработки
```

### 3.8 Demo Layer (`src/demo/progress.py`)

**Поведение:**
- Если после сообщения пользователя нет ни одного сообщения бота → **отправить новое сообщение** (without notification, "без звука")
- Если уже есть сообщение бота → **редактировать его** (append, не затирать предыдущий текст)

**Прогресс-сообщение (пример):**
```
🤖 Анализирую запрос...
🤖 Язык: русский, тон: расстроенный
🤖 Блок: Поддержка (2)
🤖 Готовлю скрипт диалога...
🤖 Скрипт: 1. Идентификация проблемы 2. Выяснение деталей 3. Решение
🤖 Готовлю ответ...
🤖 Проверяю ответ: соответствие политикам ✅, юридические нормы ⏳... ✅
🤖 Собираю данные: Имя (получено), Email (ожидаю...)
🤖 Выполняю действие: создаю заявку в CRM...
🤖 ✅ Готово! Вот ваша заявка (#42), ссылка на таблицу: https://docs.google.com/...
```

### 3.9 Action Executor (`src/executor/`)

Выполняет целевое действие когда все обязательные поля собраны.

**Модульная система блоков:** Каждый блок и его целевое действие — отдельный подмодуль в `src/executor/blocks/`.

```
src/executor/blocks/
├── base.py              # Abstract base: required_fields, execute(), tools
├── block_1_sales.py     # Продажи и новые возможности
├── block_2_support.py   # Поддержка существующих клиентов
├── block_3_info.py      # Информационные запросы
├── block_4_promo.py     # Маркетинговые акции
├── block_5_trash.py     # Непрофильные сообщения
├── block_6_retention.py # Ультиматум/удержание
└── registry.py          # Реестр блоков: id → класс
```

**Целевые действия по блокам:**
- **Блок 1:** Лид в Google Sheets + Stripe-payment link (тестовый режим)
- **Блок 2:** Тикет в Google Сsheets + номер заявки
- **Блок 4:** Корзина + Stripe ссылка на оплату

**Легкое добавление/удаление блоков:**
- Новый блок: создать файл в `executor/blocks/`, унаследовать `BlockAction`, зарегистрировать в `registry.py`
- Удаление: удалить файл и запись в реестре
- Новое целевое действие: добавить функцию в `BlockAction.execute_action()`
- Новые обязательные поля: добавить в `BlockAction.required_fieldss`

### 3.10 Config Watcher (`src/config_watcher.py`)

Наблюдает за изменениями `config.yaml` и перезагружает параметры **без остановки приложения**.

Механизм:
- `watchfiles` или `inotify` на `config.yaml`
- При изменении — перечитать файл, обновить глобальные настройки
- Настройки, которые обновляются: лимиты повторой проверки, интервал проверки human_review, список полей по умолчанию, пути к данным

## 4. LLM Client (`src/llm_client/`)

Единый модуль для всех LLM-запросов. OpenAI-совместимое API.

**Failover-логика:**
- Модели сгрупппорованы (`group="router"`, `group="dialogue"`, `group="guardrails"`, etc.)
- В группе модели с приоритетами
- Запрос → первая модель → ошибка → вторая → ...
- Если все недоступны → пауза 1s → повтор → 2s → 4s → 8s → 16s → 32s → 64s → 128s → 256s → reset (10 циклов)
- API-ключи из `.env`, базовые URL и названия провайдеров из `config.yaml`

**Конфиг LLM (пример):**
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

## 5. CRM: Google Sheets (`src/executor/crm.py`)

- Отдельная таблица на каждого пользователя бота
- Сервисный аккаунт Google
- Права редактирования: всем у кого есть ссылка
- Ссылка отправляется в demo-сообщение после создания

## 6. Database: SQLite (`src/dialogue/db.py`)

Таблицы: `sessions`, `leads`, `tickets`. Async через `aiosqlite`.
Детальная схема в `spec/08_database.md`.

## 7. Config & Secrets

- `config.yaml` — LLM groups, retry limits, block settings, watching intervals, data paths
- `.env` — API keys (Telegram, OpenAI, Google Sheets service account, Stripe)
- `data/company_info.json` — FAQ, цены, контакты, политики
- `data/` — доп. эталонные документы для guardrails

## 8. FSM States

```
WELCOME → ROUTER → SCRIPT → COLLECT ←─┐
                        ↑_____↓         │
                        loop (cycle) ────┘
                        ↓
                     ACTION → CLOSED
```

### Состояния

| State | Описание |
|---|---|
| `WELCOME` | Приветственное сообщение, представление бота |
| `ROUTER` | Вызов Router для классификации языка, tone, block_id |
| `SCRIPT` | Проверка/генерация скрипта, списка задач, обязательных полей |
| `COLLECT` | Цикл: запросить поле → получить ответ → валидировать → сохранить |
| `ACTION` | Все поля собраны → выполнить целевое действие |
| `CLOSED` | Диалог завершён |

### COLLECT loop (детально):

1. Проверить/сгенерировать скрипт (если блок изменился — перегенерировать задачи и поля)
2. Сгенерировать ответ пользователю (с учётом скрипта как контекста)
3. Guardrails: проверить ответ на соответствие документам
   - Если не прошёл → отредактировать / переписать → проверка снова
   - Если лимит исчерпан → создать файл в `human_review/pending/`
4. Отправить ответ пользователю
5. User replies
6. Router: переклассифицировать сообщение (проверить язык, тон, не изменился ли блок)
   - Блок изменился → сгенерировать новые задачи и обязательные поля
   - Блок не изменился → проверить валидность имеющихся данных →
     - Если все обязательные поля собраны → ACTION
     - Если нет → добавить недостатки → вернуться к шагу 1

## 10. File Map

```
src/
├── main.py                       # aiogram polling, entry point
├── config.py                     # load config.yaml + .env, config watcher
├── llm_client/
│   ├── client.py                 # failover LLM client
│   └── groups.py                 # group config parsing
├── classifier/
│   └── router.py                  # language + tone + block_id + intent
├── context/
│   └── builder.py                  # context assembly for LLM prompts
├── dialogue/
│   ├── state_machine.py          # FSM orchestrator
│   ├── session_manager.py         # SQLite session CRUD
│   ├── script_generator.py        # task_list + required_fields + script
│   ├── response_generator.py      # compose user message from context
│   ├── response_validator.py      # email/phone validation
│   └── db.py                        # SQLite init + helpers
├── executor/
│   ├── crm.py                       # Google Sheets integration
│   ├── payments.py                  # Stripe test mode
│   ├── human_review.py              # human_review pipeline
│   ├── actions.py                    # per-block action dispatcher
│   └── blocks/                      # block definitions
│       ├── base.py
│       ├── block_1_sales.py
│       ├── block_2_support.py
│       ├── block_3_info.py
│       ├── block_4_promo.py
│       ├── block_5_trash.py
│       ├── block_6_retention.py
│       └── registry.py
├── guardrails/
│   └── checker.py                  # response verification against docs
├── demo/
│   └── progress.py                  # demo message append/creation
├── config_watcher/
│   ├── __init__.py
│   └── watcher.py                   # watch config.yaml for changes
...

## 11. Key Design Decisions

| Decision | Rationale |
|---|---|
| Router — отдельный дешёвый запрос | gpt-4o-mini для классификации, gpt-4o для генерации скрипта и ответа |
| JSON Mode для всех LLM-ответов | Детерминированный парсинг, без галлюциаций в структуре |
| Tone — свободный текст, не enum | LLM сама определяет подходящее описание эмоции клиента |
| Бот отвечает на языке пользователя | Определение языка в Router, генерация ответов на том же языке |
| SQLite (aiosqlite) | Не внешних зависимостей, async, типовая схема |
| Google Sheets CRM | Отдельная таблица на пользователя, права по ссылке, visually |
| Stripe тестовый режим | Не заглушка, реальный платёжный шлюз с тестовыми параметрами |
| Блоки — отдельные модули | Легко добавлять/удалять блоки и действия |
| Guardrails с retry до лимита | Безоставительный сценарий: составительское перекрывание протокола |
| human_review pipeline | Делегатные вопросы товарищу через файлы |
| Config watcher | live-reload параметров без перезагрузки |
| Одно demo-сообщение (дописывание) | Чистый UI в Telegram, не-спам уведомлений |

## 12. Not Covered (Future)

- Векторное хранилище (ChromaDB/FAISS) для RAG-поиска
- Redis for session cache
- Полноценные Guardrails с несколькими LLM-проходами
- Production-ready Stripe
- Админ-панель `/stats`
- Concurrent user > 100