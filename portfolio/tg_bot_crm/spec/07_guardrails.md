# 07 Guardrails — Проверка ответов

## 1. Overview

Перед отправкой каждого сообщения пользователю, Guardrails проверяет его на соответствие эталонным документам компании. Если не соответствует — редактирует и перепроверяет. Если лимит попыток исчерпан — отправляет на human_review.

**Не заглушка.** Использует реальные документы из `data/`.

## 2. Документы для проверки

Источники информации о компании (эталонные данные):

```
data/
├── company_info.json       # Главный документ (FAQ, ценности, контакты)
├── products.yaml           # Каталог продуктов с характеристиками
├── prices.yaml             # Цены и скидки
├── policies.md             # Правила компании, legal notes
└── faq.md                  # Часто задаваемые вопросы
```

Любой из файлов может быть **пустым** — проверка пропускает пустые источники.

## 3. Guardrails Check

### 3.1 Prompt

```python
GUARDRAILS_SYSTEM_PROMPT = """
Ты — валайдер данных. Проверяешь ответ оператора на соответствие официальным документам компании.

Документы:
{company_docs}

Продукты и цены:
{products_and_prices}

Политики и правила:
{policies}

ПРОВЕРЬ:
1. Вся-ли  ФАКТИЧЕСКАЯ информация в ответе оператора соответствует явно указанному в документах?
   - Цены, скидки, характеристики, сроки доставки, контакты
   - Если не соответствует документам — это ОШИБКА
2. Не содержит ли ответ обещаний или гарантий которых нет в документах?
   - Даже если звучит естественно — это ошибка
3. Следует ли ответ юридическим нормам или этике из документов?
   - Не обещает ли возврат который не прописан в политике
   - Не гарантирует ли сроки которых нет в документации
4. Не содержит ли ответ вредных для бизнеса утверждений?

ЕСЛИ НАЙДЕНА ОШИБКА:
- Укажи что именно не соответствует документам
- Предложи ИСПРАВЛЕННЫЙ вариант ответа (полный текст)

ЕСЛИ ЕСТЬ ИНФОРМАЦИЯ КОТОРУЮ НЕ ПОДТВЕРДИТЬ:
- Укажи что нельзя подтвердить
- Предложи вариант где открыто указано что бот уточнит у руководителя
- Например: если цена не указана в документах "Точную сумму нужно уточнить, но ориентировочно..."

Формат ответа — строгий JSON:
{
  "valid": true/false,
  "issues": ["issue text 1", "issue text 2"],
  "corrected_response": "текст правильного ответа (если valid=false)"
}
"""

GUARDRAILS_USER_PROMPT = """
Ответ оператора:
{response_to_check}

Проверь ответ и верни JSON результат."""
```

### 3.2 Retry loop

```python
# src/guardrails/checker.py

GUARDRAILS_MAX_RETRIES = 3  # Переопределяется из config.yaml

async def guard_check(
    client: LLMClient,
    response: str,
    company_docs: dict,
    products: dict,
    policies: str,
    session: dict,
) -> tuple[bool, str]:
    """
    Правильные ответ на ошибки, при ошибки — редактирует до N попыток.
    Returns:
    - (True, final_answer_str) если пройден
    Выкидывает исключение если лимит исчерпан → human_review
    """
    
    config = load_config()
    max_report = config.get("guardrails", {}).get("max_retries", GUARDRAILS_MAX_RETRIES)
    
    current_response = response
    
    for attempt in range(max_retries):
        result = await client.query(
            group="guardrails",
            system_prompt=GUARDRAILS_SYSTEM_PROMPT.format(
                company_docs=json.dumps(company_docs, ensure_ascii=False),
                products_and_prices=json.dumps(products, ensure_ascii=False),
                policies=json.dumps(policies, ensure_ascii=False),
            ),
            user_message=GUARDRAILS_USER_PROMPT.format(
                response_to_check=current_response,
            ),
            request_type="json",
        )
        
        if result["valid"]:
            log_guardrails_pass(attempt + 1, current_response)
            return True, current_response
        
        # Не пройден — использовать исправленный вариант
        current_response = result.get("corrected_response", current_response)
        log_guardrails_fail(attempt + 1, result["issues"])
    
    # Лимит исчерпан
    await send_to_human_review(
        user_id=session["user_id"],
        dialogue_id=session.get("dialogue_id"),
        query="Guardrails не смог исправить ответ после {max_retries} попыток",
        context=json.dumps(session.get("history", [])[-5:], ensure_ascii=False),
        generated_response=current_response,
        issues=result.get("issues", []),
    )
    
    raise HumanReviewNeeded(f"human_review после {max_retries} попыток")
```

### 3.3 Integration in DIALOG pipeline

Guardrails всегда вызывается после Response Generator и перед отправкой сообщения пользователю.

## 4. Config

```yaml
# config.yaml
guardrails:
  max_retries: 5
  human_review_on_limit: true
  data_paths:
    company_info: "data/company_info.json"
    products: "data/products.yaml"
    prices: "data/prices.yaml"
    policies: "data/policies.md"

llm_groups:
  guardrails:
    - priority: 1
      provider: "openai"
      model: "gpt-4o-mini"
```

## 5. Demo Mode

При `debug_mode: True` (`config.yaml`), guardrails отправляет в demo сообщение каждую итерацию:

```
⏳ Проверка ответа (попытка 1/5)...
⚠️  Ответ не прошёл: цена не соответствует прайсу
⏳ Редактирую ответ (попытка 2/5)...
✅ Ответ прошёл проверку!
```

## 6. File Map

```
src/guardrails/
└── checker.py          # guardrails_check() + поддержка retry loop
```