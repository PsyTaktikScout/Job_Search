# 05 Router & Classifier

## 1. Overview

Router — первый слой pipeline. Определяет язык сообщения, эмоциональный тон, бизнес-блок (1–6), интент и извлекает данные.

**LLM-группа:** `"router"` (дешёвая модель: gpt-4o-mini)

**Вызывается:** для каждого входящего сообщения пользователя

## 2. Input / Output

### Input

- `user_message: str` — последнее сообщение пользователя
- `history: list[dict]` — последние N сообщений диалога (для контекста)
- `language: str` — язык предыдущего диалога (если есть)

### Output (JSON Mode)

```json
{
  "language": "russian",
  "tone": "frustrated_but_polite",
  "block_id": 2,
  "intent_label": "product_malfunction",
  "extracted_data": {
    "product": "X100",
    "issue": "not_charging"
  },
  "confidence": 0.92,
  "action": "classify"
}
```

Если сообщение приветственное / не по профилю:

```json
{
  "language": "ukrainian",
  "tone": null,
  "block_id": null,
  "intent_label": null,
  "extracted_data": {},
  "confidence": null,
  "action": "welcome"
}
```

Если Блок 5 (спам/ошибка):

```json
{
  "language": "russian",
  "tone": "spam",
  "block_id": 5,
  "intent_label": "wrong_number",
  "extracted_data": {},
  "confidence": 0.99,
  "action": "classify",
  "close_reason": "spam_or_error"
}
```

## 3. Prompt Template

```python
ROUTER_SYSTEM_PROMPT = """
Ты — классификатор входящих сообщений. Определи:

1. **language** — на каком языке написано сообщение. Используй стандартные названия:
   - "russian", "english", "ukrainian", "spanish", etc.

2. **tone** — эмоциональный тон сообщения. Описывай в свободной форме коротко:
   Примеры (НЕ ограничивайся): "neutral", "frustrated", "urgent", "angry", "polite", "confused", "hopeful", "sarcastic", "spam"

3. **block_id** — к какому бизнес-блоку относится сообщение:
   - 1: Продажи / CP (запрос цены, КП, наличие, доставка, демо)
   - 2: Поддержка (проблема с товаром/услугой, вопрос по оплате, продление, перенастройка, жалоба)
   - 3: Информация (график работы, вакансии, партнёрство, статус заказа)
   - 4: Акции / промо (скидка, промокод, розыгрыш, проблема на сайте)
   - 5: Спам / ошибка / не по профилю
   - 6: Угроза ухода (расторжение договора, ультиматум)

4. **intent_label** — конкретный интент внутри блока (человеко-читаемый на английском):
   - Блок 1: "price_request", "availability_check", "custom_calculation", "delivery_info", "demo_request"
   - Блок 2: "product_issue", "payment_issue", "contract_renewal", "configuration_change", "service_complaint"
   - Блок 3: "working_hours", "job_inquiry", "partnership", "order_status", "terminology"
   - Блок 4: "promo_code", "raffle", "site_error", "catalogue_request"
   - Блок 5: "wrong_number", "spam", "prank", "security_call"
   - Блок 6: "contract_termination", "ultimatum"
   Если не подходит ни одна категория — придумай свой intent_label.

5. **extracted_data** — извлеки из сообщения:
   - product (название товара/услуги), если упомянуто
   - issue (описание проблемы), если упомянуто
   - name, phone, email — если указаны
   - amount (сумма), если указана
   - other — любые другие значимые детали

6. **confidence** — твоя уверенность в классафикации (0.0 – 1.0):
   - 0.9+ = совершенно уверен
   - 0.7–0.9 = уверен
   - < 0.7 = неуверен (требует уточнения)

7. **block_reason_comment** — краткий комментарий почему выбран именно этот блок (на английском). Используется при необходимости (дебаггинг)

8. Если сообщение — **только приветствие** без запроса (например "Привет", "Добрый день"), верни `action: "welcome"`.

9. Если сообщение явно не относится к профилю компании (например "Кто хочет заработать? открой нашу платформу" — Блок 5, спам), верни `action: "classify"` с `block_id: 5`.

Формат ответа — строгий JSON.
"""

ROUTER_USER_PROMPT = """Предыдущий язык диалога: {language}
История диалога: {history}
Последнее сообщение пользователя: "{message}"

Определи язык, тон, блок, интент и извлечённые данные."""
```

## 4. Classification Logic

```python
# src/classifier/router.py

async def classify(
    client: LLMClient,
    message: str,
    history: list[dict],
    language: str | None = None,
) -> dict:
    """Классифицирует входящее сообщение.
    
    Returns:
        dict с полями: language, tone, block_id, intent, extracted, confidence, action
    """
    prompt = ROUTER_USER_PROMPT.format(
        language=language or "unknown",
        history=json.dumps(history[-10:], ensure_ascii=False),
        message=message,
    )
    
    result = await client.query(
        group="router",
        system_prompt=ROUTER_SYSTEM_PROMPT,
        user_message=prompt,
        request_type="json",
    )
    return result
```

## 12. Post-Classification Logic

```python
# src/classifier/router.py

def handle_classification_result(result: dict, session: dict) -> str:
    """Определяет следующее FSM-состояние на основе классификации."""
    
    if result["action"] == "welcome":
        return State.WELCOME
    
    if result["block_id"] == 5:
        return State.CLOSED
    
    # Проверить confidence порог
    if result["confidence"] < LOW_CONFIDENCE_THRESHOLD:  # 0.7
        return State.ASK_CLARIFICATION  # "Уточните, пожалуйста"
    
    # Проверить, не изменился ли блок
    if session["block_id"] != result["block_id"]:
        # Блок изменился — нужна регенерация script + required_fields
        return "REGENERATE_SCRIPT"
    
    # Проверить, все ли обязательные поля собраны
    required = session.get("required", [])
    collected = session.get("collected", {})
    
    # Merge извлечённые данные в collected
    for key, value in result.get("extracted_data", {}).items():
        if value:
            collected[key] = value
    
    missing = [f for f in required if not collected.get(f)]
    
    if not missing:
        return State.ACTION_EXECUTE
    
    return State.DIALOG  # Continue collecting
```

## 13. File Map

```
src/classifier/
└── router.py              # classify() + handle_result() + prompt templates
```