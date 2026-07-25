# 09 Google Sheets CRM Integration

## 1. Overview

Google Sheets используется как CRM:
- Отдельная таблица на каждого пользователя бота (чтобы не смешивать данные при демонстрации)
- Подключение через сервисный аккаунт Google
- Права редактирования всем у кого есть ссылка
- Ссылка на таблицу отправляется в demo-сообщение после создания

## 2. Prerequisites

- Google Cloud проект с включенным Google Sheets API
- Сервисный аккаунт с ключом в JSON формате
- Email сервисного аккаунта с доступом к Google Sheets

### `.env`

```
GOOGLE_SERVICE_ACCOUNT_KEY=path/to/service_account.json
```

### `config.yaml`

```yaml
crm:
  type: "google_sheets"
  service_account_key: ${GOOGLE_SERVICE_ACCOUNT_KEY}
  template_sheet_id: "1AbCd..."  # шаблонная таблица (копируется при создании)
```

## 3. Sheet Structure

Каждая таблица пользователя содержит два листа:

### Лист 1: `Clients`
| user_id | client_name | phone | email | product_interest | source | status | created_at |
|---|---|---|---|---|---|---|---|
| 12345 | Иван Петров | +7900... | ivan@... | X200 | telegram_bot | new | 2026-07-25 |

### Лист 2: `Tickets`
| user_id | ticket_num | issue_type | description | status | resolution | created_at | resolved_at |
|---|---|---|---|---|---|---|---|
| 12345 | #42 | product | Не заряжается | open | | 2026-07-25 14:30 | |

## 4. Google Sheets Client (`src/executor/crm.py`)

```python
import gspread
from google.oauth2.service_account import Credentials

class GoogleSheetsCRM:
    def __init__(self, service_account_path: str, template_sheet_id: str = None):
        self.creds = ServiceAccountCredentials.from_service_account_file(service_account_path)
        self.client = gspread.authorize(self.creds)
        self.template_id = template_sheet_id
        
    async def get_or_create_sheet(self, user_id: int) -> str:
        """
        Найти существующую таблицу для user_id или создать новую.
        Returns: sheet URL
        """
        # Проверить в SQLite: есть ли уже crm_sheet_url для этого user_id
        session = await SessionManager.get_session(user_id)
        if session and session.get("crm_sheet_url"):
            return session["crm_sheet_url"]
        
        # Создать новую таблицу
        sheet = await self._create_sheet(user_id, f"CRM_User_{user_id}")
        
        # Сохранить URL в SQLite
        await SessionManager.update_session(user_id, crm_sheet_url=sheet.url)
        return sheet.url
    
    async def _create_sheet(self, user_id: int, title: str) -> gspread.Spreadsheet:
        """Создать новую таблицу и настроить права."""
        sheet = await self.client.create(title)
        
        # Права: любой, у кого есть ссылка — editor
        sheet.share(None, perm_type="anyone", role="writer")
        
        # Создать листы с заголовками
        clients_ws = sheet.worksheet("Sheet1")
        clients_ws.update_title("Clients")
        clients_ws.append_row(["user_id", "client_name", "phone", "email", "product_interest", "source", "status", "created_at"])
        
        tickets_ws = sheet.add_worksheet("Tickets", rows=100, cols=10)
        tickets_ws.append_row(["user_id", "ticket_num", "issue_type", "description", "status", "created_at", "resolved_at"])
        
        # Удалить стандартные листовки
        for ws in sheet.worksheets():
            if ws.title.startswith("Лист") and ws.title != "Sheet1":
                sheet.del_worksheet(ws)
        
        return sheet
    
    async def append_client(self, user_id: int, **fields) -> dict:
        """Добавить строку в лист Clients."""
        sheet_url = await self.get_or_create_sheet(user_id)
        sheet = self.client.open_by_url(sheet_url)
        ws = sheet.worksheet("Clients")
        
        row = [
            fields.get("user_id", user_id),
            fields.get("client_name", ""),
            fields.get("phone", ""),
            fields.get("email", ""),
            fields.get("product_interest", ""),
            fields.get("source", "telegram_bot"),
            fields.get("status", "new"),
            datetime.now().isoformat()
        ]
        ws.append_row(row)
        return {"status": "ok", "lead_id": user_id}
    
    async def add_ticket(self, user_id: int, issue_type: str, description: str) -> dict:
        """Добавить строку в лист Tickets."""
        sheet_url = await self.get_or_create_sheet(user_id)
        sheet = self.client.open_by_url(sheet_url)
        ws = sheet.worksheet("Tickets")
        
        # Посчитать следующий ticket_num
        rows = ws.get_all_values()
        ticket_num = len(rows)
        
        row = [user_id, f"#{ticket_num}", issue_type, description, "open", datetime.now().isoformat(), ""]
        ws.append_row(row)
        
        return {"status": "ok", "ticket_number": f"#{ticket_num}"}
```

## 5. Service Account Key

Файл `service_account.json` (скачанный из Google Cloud Console) **никогда не коммитится в репозиторий**. Путь к нему в `.env`:

```
GOOGLE_SERVICE_ACCOUNT_PATH=~/.google/service_account.json
```

## 6. Dependencies

```
# requirements.txt
gspread==6.1.2
google-auth==2.28.0
oauth2client==4.1.3
```

## 7. Demo

При создании таблицы (для state_history / или впервые):

```
🤖 CRM для диалога с > Мистер А: https://docs.google.com/spreadsheets/d/{id}
```

## 8. File Map

```
src/executor/
├── crm.py           # Google Sheets client
├── payments.py      # Stripe payment link creation
├── actions.py       # Action Executor: последовательное выполнение
└── blocks/          # Block definitions
```