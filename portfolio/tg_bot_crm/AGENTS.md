# AGENTS.md — Telegram CRM Bot

## О проекте

**tg_bot_crm** — AI-оператор в Telegram для автоматической обработки входящих сообщений клиентов.
Бот классифицирует запросы по 6 бизнес-блокам, ведёт диалог по сценарию,
собирает данные и выполняет целевые действия (CRM, Stripe-оплату, тикеты).

## Принцип работы

**Спецификация → Реализация → Верификация**

1. Изучи spec-файл нужного модуля
2. Реализуй код строго по спецификации
3. Проверь соответствие кода спецификации

## Структура

```
tg_bot_crm/
├── AGENTS.md              # ← этот файл
├── spec/                   # Спецификации (читать перед реализацией)
│   ├── 01_architecture.md  # Общая архитектура, слои, FSM
│   ├── ...                 # (добавляются по мере написания)
├── src/
│   ├── main.py
│   ├── llm_client/         # OpenAI-совместимый failover-клиент
│   ├── classifier/         # Router (language + tone + block_id)
│   ├── context/            # Context builder для LLM
│   ├── dialogue/           # FSM, session manager, script generator, response generator
│   ├── executor/           # CRM (Google Sheets), Stripe, human_review, blocks
│   ├── guardrails/         # Проверка ответов против эталонных документов
│   ├── demo/               # Прозрачность шагов (append to message)
│   ├── config_watcher/     # Live-reload config.yaml
│   └── integrations/       # aiogram helpers, клавиатуры
├── human_review/
│   ├── pending/            # Вопросы, ожидающие ответа руководителя
│   ├── replied/            # Ответы от руководителя
│   └── processed/          # Обработанные ответы
└── data/
    └── company_info.json   # База знаний компании + эталонные документы
```

## Ключевые правила

1. **LLM — строгий JSON Mode** на все запросы. Никакого free-text от LLM.
2. **LLM failover** — exponential backoff: 1s → 2s → 4s → ... → 256s, 10 циклов.
3. **Tone** — не фиксированный enum, свободное описание на английском.
4. **Бот отвечает на языке пользователя** — определяется Router-ом.
5. **Response не цитата из скрипта** — скрипт только как контекст.
6. **Guardrails** — проверка реальных документов, retry до лимита, потом human_review.
7. **Config-driven** — `config.yaml` (LLM groups, retry limits, intervals), `.env` (API keys).
8. **SQLite + aiosqlite** — таблицы sessions, leads, tiсkets.
9. **Google Sheets** — отдельная таблица на user_id, права по ссылке.
10. **Stripe** — реальный тестовый режим.
11. **Блоки — отдельные модули** — наследники `BlockAction` в `executor/blocks/`.
12. **Config watcher** — live-reload без перезагрузки.
13. **Demo message** — дописывается в одно сообщение (без звука), не затирает предыдущее.
14. **Файлы ≤ 500 строк** — при превышении разбивать.
15. **Человек вручную** перемещает файлы из pending/ в replied/.

## Workflow типовой задачи

Когда получаешь задачу «реализуй модуль X»:
1. Прочитай соответствующий spec-файл и `01_architecture.md`
2. Напиши код
3. Проверь, что покрыты все состояния из спецификации
4. Если модуль имеет зависимости (LLM client, DB) — проверяй интерфейсы совместимы