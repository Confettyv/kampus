# Контракт API

Полная схема запросов и ответов — [openapi.yaml](openapi.yaml). Открыть удобно:
`npx @redocly/cli preview-docs docs/openapi.yaml` или вставить файл в editor.swagger.io.
Типы для фронтенда: `npx openapi-typescript docs/openapi.yaml -o frontend/src/shared/api/schema.d.ts`.

## Общие правила

| Правило | Как |
|---|---|
| Адрес | Все пути от `/api`, тот же домен, что и фронтенд |
| Сессия | Cookie `kampus_session` (`httpOnly`, `Secure`, `SameSite=Lax`) после входа или регистрации |
| Деньги | Целые копейки: `15050` = 150,50 ₽. Дробных сумм в API нет |
| Даты | `2026-10-09` и `2026-10-09T14:30:00Z` |
| Списки | `?limit=20&cursor=…` → `{ items, next }`, `next: null` — конец |
| Чужое | Чужой или несуществующий объект — `404`, не `403` |
| Ошибка | `{ "error": { "code": "validation_error", "message": "…", "fields": { "amount": "…" } } }` |

Коды ошибок: `400 validation_error`, `401 unauthorized`, `404 not_found`, `409 conflict` /
`confirm_required`, `429 too_many_requests` (больше 5 входов за 15 минут, заголовок `Retry-After`).

## Эндпоинты

«Желательное» — делаем, если к 6-й неделе успеваем.

### Аккаунт

| Метод и путь | Что делает |
|---|---|
| `POST /auth/register` | Регистрация (почта, пароль от 8 символов, имя), сразу вход |
| `POST /auth/login` | Вход |
| `POST /auth/logout` | Выход, сессия гасится на сервере |
| `GET /me` · `PATCH /me` | Профиль: имя, корпус |
| `DELETE /me` | Удаление аккаунта и данных по паролю; в группах — «Удалённый участник» |

### Бюджет

| Метод и путь | Что делает |
|---|---|
| `GET /budget` · `PUT /budget` | Стипендия и дата следующей (`404` — ещё не настроен) |
| `GET /budget/today` | Лимит на сегодня, потрачено, осталось, дней до стипендии |
| `GET /budget/operations` | Операции с фильтром по датам и категории |
| `POST /budget/operations` | Трата или доход; в ответе — пересчитанный лимит |
| `DELETE /budget/operations/{id}` | Удалить операцию |
| `GET /budget/categories` | Категории |
| `GET /budget/summary` | Итоги периода по категориям — желательное |

Пример `GET /budget/today`:

```json
{ "date": "2026-10-09", "days_left": 16, "left_morning": 480000, "limit": 30000,
  "spent_today": 12000, "left_today": 18000, "period_left": 468000 }
```

### Сплит

| Метод и путь | Что делает |
|---|---|
| `GET /groups` · `POST /groups` | Мои группы с моим балансом; создать группу |
| `GET /groups/{id}` | Группа и участники |
| `DELETE /groups/{id}/members/me` | Выйти (только при нулевом балансе, иначе `409`) |
| `POST /groups/{id}/invites` | Ссылка-приглашение на 7 дней |
| `GET /invites/{token}` · `POST /invites/{token}/accept` | Куда зовут; вступить |
| `GET /groups/{id}/expenses` · `POST …` | Расходы; добавить расход — делится поровну до копейки |
| `DELETE /groups/{id}/expenses/{expenseId}` | Удалить свой расход |
| `GET /groups/{id}/balances` | Балансы: плюс — ему должны |
| `GET /groups/{id}/settle-plan` | Минимум переводов, чтобы рассчитаться — желательное |
| `POST /groups/{id}/settlements` | Отметить возврат долга — желательное |

Пример: 100,00 ₽ на троих — доли `3334, 3333, 3333`, сумма долей всегда равна сумме расхода.

### Защита

| Метод и путь | Что делает |
|---|---|
| `POST /transfers/check` | Проверка: кому, сколько, назначение, текст сообщения, ссылка → риск и признаки |
| `POST /transfers` | Смоделировать перевод; при риске `high` без `confirmed: true` — `409 confirm_required` |
| `GET /transfers` | Мои переводы |
| `GET /schemes` | Справочник схем с признаками — желательное |

Пример ответа проверки:

```json
{ "id": "chk_81", "risk": "high", "signs": [
  { "code": "fake_dean", "title": "«Деканат» пишет в мессенджере",
    "hint": "Деканат не просит денег в Telegram. Позвоните по номеру с сайта вуза." },
  { "code": "urgency", "title": "Торопят", "hint": "Срочность — главный приём мошенников." } ] }
```

Поток на экране: форма → `check` → риск `low` — сразу `POST /transfers`; `medium`/`high` — шторка
с признаками, кнопки «Отменить» и «Всё равно перевести» (`confirmed: true`).

### Скидки

| Метод и путь | Что делает |
|---|---|
| `GET /deals?category=&campus=` | Каталог с фильтром, доступен без входа |
| `GET /deals/categories` | Категории |
| `PUT` · `DELETE /deals/{id}/favorite`, `GET /deals?favorite=true` | Избранное — желательное |
