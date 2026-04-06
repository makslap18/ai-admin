# AI-Администратор для автосервисов — Техническое саммари

> Обновлено: 2026-03-26

## Общее описание системы

Мультитенантная платформа AI-ботов для автосервисов. Каждый автосервис получает своего бота, который:
- Общается с клиентами через ИИ (Gemini Flash via OpenRouter)
- Принимает текстовые и голосовые сообщения (голос транскрибируется через OpenAI Whisper)
- Батчит несколько быстрых сообщений в одно перед отправкой в LLM
- Использует RAG (PGVector) для ответов по прайсу и услугам конкретного сервиса
- Записывает на приём (выбор даты/времени, подтверждение)
- Отменяет записи, напоминает о записи за 24ч и 2ч
- Уведомляет владельца о новых записях и отменах
- Контролирует лимит сообщений по тарифному плану
- Доступен через **3 канала**: Telegram, веб-виджет (iframe), Авито
- Соблюдает 152-ФЗ: реальные ПДн не передаются в LLM, используются алиасы
- LLM может запрашивать свободные слоты через отдельный tool-endpoint
- Имеет лендинг "Записали" (GitHub Pages) с формой заявки, демо-виджетом и тарифами

---

## Инфраструктура

- **Сервер n8n (Россия)**: sese (141.105.70.225), SSH root, ключ `~/.ssh/id_ed25519`
- **VPN-сервер (зарубежный)**: 46.149.72.77, SSH root (порт 22), 3X-UI панель: `46.149.72.77:2053/5ae0a54929028fdb5e88c280f8381e8b/`
- **n8n**: self-hosted, домен `n8n57362.hostkey.in`
- **БД**: PostgreSQL с pgvector, на том же сервере
- **LLM**: Gemini 3 Flash Preview через OpenRouter
- **Embeddings**: OpenAI-совместимый API через proxyapi
- **n8n MCP серверы**: `n8n-mcp-sese` (основной), `n8n-mcp` (secondary, skywowshow.com)
- **Лендинг**: GitHub Pages — https://makslap18.github.io/zapisali-landing/ (репо: `makslap18/zapisali-landing`)

### Прокси-инфраструктура (обход блокировок)

n8n на российском сервере не имеет прямого доступа к OpenAI/OpenRouter/Telegram API. Решение — двусторонний прокси через VPN-сервер.

```
Входящие (Telegram → n8n):
  Telegram → 46.149.72.77:443 (nginx, самоподписанный SSL) → n8n57362.hostkey.in

Исходящие (n8n → API):
  n8n (Docker) → Xray (tun0) → 46.149.72.77:9443 (VLESS+Reality) → api.telegram.org / api.openai.com
```

**Xray-клиент** (исходящий): `/usr/local/bin/xray`, конфиг `/usr/local/etc/xray/config.json`, systemd `xray`. Проксирует: openai.com, openrouter.ai, telegram.org и поддомены. Добавить домен: `routing.rules.domain` в конфиге → `systemctl restart xray`.

**Nginx на VPN-сервере** (входящий от Telegram): `/etc/nginx/sites-available/tg-proxy`, порт 443, самоподписанный SSL. Проксирует на `https://n8n57362.hostkey.in`. SSH с sese на VPN **не работает** — управлять только с локальной машины.

### Telegram Webhook

Webhook каждого бота указывает на зарубежный IP:
```bash
curl -F "url=https://46.149.72.77/webhook/bf526e46-d075-4181-913e-e58ac8646f8a/telegram-router/{bot_id}" \
     -F "certificate=@/tmp/tg-proxy.pem" \
     "https://api.telegram.org/bot{TOKEN}/setWebhook"

# Получить сертификат если /tmp/tg-proxy.pem отсутствует:
echo | openssl s_client -connect 46.149.72.77:443 2>/dev/null | openssl x509 > /tmp/tg-proxy.pem
```

UUID `bf526e46-d075-4181-913e-e58ac8646f8a` — webhookId из ноды Webhook в WF 01 recieve. Одинаковый для всех ботов.

---

## Архитектура БД

### Основные таблицы (public schema)

| Таблица | Назначение | Ключевые поля |
|---------|-----------|---------------|
| auto_services | Автосервисы (5 шт) | id, name, bot_id, telegram_bot_token, owner_telegram_id, payment_date, days_paid |
| clients | Клиенты (real + alias данные) | service_id, real_name/phone/email/telegram_id, alias_*, pd_consent_given |
| appointments | Записи на приём | service_id, client_id, date/time, status (pending→pending_consent→booked→completed/cancelled) |
| vehicles | Автомобили в наличии/под заказ | service_id, name, year, price, in_stock_price, description |
| work_schedule / special_days | Расписание | service_id, day_of_week, open/close_time |
| service_prices / parts_prices | Прайс-листы | service_id, name, price, category |
| faq | FAQ | service_id, question, answer |
| documents / embeddings | RAG-данные с vector(1536) | service_id, content, embedding |
| n8n_chat_histories8 | Чат-память | session_id (= alias_telegram) |
| pd_consent_log | Аудит согласий ПДн (152-ФЗ) | client_id, action, channel |
| service_credentials | DB credentials per service | service_id, pg_user, pg_password |

**EXCLUDE constraint** на appointments: запрещает пересечение слотов для статусов booked, pending, pending_consent.

### Мультитенантность

Каждый автосервис получает schema `service_N` с view на public таблицы (WHERE service_id = N, WITH LOCAL CHECK OPTION). Отдельный DB user `svc_N`. Функция `create_service_schema()` автоматически создаёт всё при добавлении нового сервиса.

### Автосервисы

| id | name | bot_id | owner_telegram_id |
|----|------|--------|-------------------|
| 1 | test | bot2 | — |
| 2 | АвтоМастер Центр | bot1 | — |
| 3 | СТО Гараж Плюс | bot_garazh_02 | — |
| 4 | ТехноСервис Авто | bot_techno_03 | — |
| 5 | Ganin Garage test | bot_ganin_05 | NULL |

### Credentials в n8n

| ID | Имя | Назначение |
|----|-----|-----------|
| IYVS1xv3JEXM9Cve | NocoDB | PostgreSQL |
| 2y0lv5fuNyJBVIq7 | proxyapi | OpenAI Embeddings |
| PgdwDcBo4eMKNRMw | maks | OpenRouter (Gemini Flash) |

---

## Воркфлоу — Обзор

```
                         ┌─────────────────────────────────────────┐
                         │            02 brain (AI Agent)          │
                         │   Telegram / Web / Avito → LLM → Reply │
                         └──────┬──────────┬───────────────────────┘
                                │          │
                    ┌───────────┘          └──────────┐
                    ▼                                  ▼
            04_booking                          05_owner_notifications
        (Book/Cancel/List/Slots)                (Notify owner)
                    │
                    ▼
            03 call back
        (Inline keyboard handler)

Входные точки:
  Telegram  → 01 recieve        → 02 brain
  Web       → 01 recieve web    → 02 brain
  Avito     → 11_avito_receive  → 02 brain
  Лендинг   → 14_lead_form      → Telegram alert

Виджет:
  09_widget_loader (JS) → 10_widget_frame (HTML iframe) → 01 recieve web

Фоновые:
  07_appointment_reminders — каждые 30 мин (напоминания + cleanup)
  08_table_embeddings_sync — каждые 3 часа (RAG sync)
  06_error — ловит ошибки всех WF

Прочее:
  DM Bitrix Task AI — AI-обработка задач Битрикс24
  TG BOT transkrib — отдельный бот-транскрибатор голосовых
  QA Evaluation — тестирование AI-бота (ручной запуск)
```

### Полный список WF

| ID | Название | Назначение | Нод | Статус |
|----|----------|-----------|-----|--------|
| VpBY64I8MPl83xr7 | 01 recieve | Telegram Router + Voice + Batching + Message Limit | 30 | Активен |
| A2IeiFBc0I0uKgPJ | 02 brain | AI Agent (Telegram + Web + Avito) | 18 | Активен |
| mrIFyE6lwdEAIuaW | 03 call back | Callback Handler | 22 | Активен |
| u5Xui5bQEJw3lwAS | 04_booking | Booking Service (book/cancel/list/slots) + Plan Check | 32 | Активен |
| QKLGNaGcDpkYTKJm | 05_owner_notifications | Owner Notify | 3 | Активен |
| 3muBfH7DpVipSOpN | 06_error | Error Handler | 3 | Активен |
| pq3tdawZFkZAnrYJ | 07_onboarding_bot | Onboarding (нет auth) | 7 | ВЫКЛ |
| qUIDyP7ZNxU5iPSq | 07_appointment_reminders | Reminders + Cleanup | 7 | Активен |
| nZvVhdHgHkHSyXsh | 08_table_embeddings_sync | RAG Sync | 13 | Активен |
| agbjjq3sM4ovTxow | 09_widget_loader | Widget JS Loader | 4 | Активен |
| UvFL82lc98eSpRiT | 10_widget_frame | Widget Chat Frame | 4 | Активен |
| tlvrVB9CPTpp1bRr | 01 recieve web | Web Chat Router + Message Limit | 18 | Активен |
| 7xRN29kgedmm9s4H | 11_avito_receive | Avito Messenger Integration | 18 | Активен |
| 4ohPRBwYploC2Ugm | 14_lead_form | Landing Page Lead Form → TG Alert | 3 | Активен |
| jqXCO3GCoF0y0XZ2 | DM Bitrix Task AI | Bitrix24 Task AI (comment → update task) | 13 | Активен |
| MCjOULArlVfEoAFa | TG BOT transkrib | Voice Transcription Bot (standalone) | 5 | Активен |
| Z424iXlqh2UNEzkz | QA Evaluation — AI Bot Tests | Automated QA for 02 brain | 7 | ВЫКЛ |
| sSj2ZM8EmufTYbXi | _test_webhook_check | Test webhook (активен в проде!) | 2 | Активен |

**Архивные / неактивные**: `_migration_avito`, `DM Voice Archive Transcriber`, `My workflow`, `My workflow 2`, `CSV Import: Parts Prices`, `Ultimate Agentic RAG AI Agent Template`, + 8 заархивированных старых версий.

---

## 01 recieve — Telegram Router (VpBY64I8MPl83xr7, 30 нод)

Точка входа для Telegram webhook'ов. Webhook: `POST /webhook/telegram-router/:slug`.

**Поток**:
1. Webhook → Get Service (v_auto_services по slug) → Has Subscription? (нет → No Subscription Response → Send No Sub Message)
2. Parse Input → **Check Message Limit** → IF Limit Reached → (лимит → Limit Response → Send Limit Msg / ОК → Switch Type)
3. Switch Type: message / voice / callback_query / contact

**message**: Send Typing ∥ Save to Buffer → Wait for More → Check If Latest → (если не последнее — стоп) → Collect Messages → Prepare Combined → Upsert Client → Intercept PD → Save PD → Anonymize & Prepare → Call 02

**voice**: Get Voice File → Download Voice → Transcribe (Whisper) → Set Voice Text → далее в общий батчинг-поток (Save to Buffer)

**callback_query**: → Call '03 call back' (Execute Workflow)

**contact**: Save Phone → Phone Saved (подтверждение в Telegram)

**152-ФЗ**: В 02 brain передаются ТОЛЬКО алиасы. Реальные данные — для уведомлений владельцу.

---

## 02 brain — AI Agent (A2IeiFBc0I0uKgPJ, 18 нод)

ИИ-агент с анонимизированным контекстом. Поддерживает **3 канала**: Telegram, Web, Avito.

- **LLM**: Gemini 3 Flash Preview через OpenRouter (lmChatOpenRouter)
- **Память**: Postgres Chat Memory (ключ = alias_telegram; для web: `web_` + sessionId)
- **RAG**: PGVector Store + Embeddings OpenAI, фильтр по service_id
- **Tool**: Booking Tool → вызывает 04_booking

**Поток**: Execute Workflow Trigger → Edit Fields → AI Agent → Merge Channel → Switch Channel → ответ по каналу

**Маршрутизация ответа (Switch Channel)**:
- `web` → Return Web Response (Code)
- `avito` → Send Avito Response (HTTP Request)
- default (telegram) → Send Telegram Response (HTTP Request)

**Обработка ошибок (Switch Channel Error)**: аналогичный switch на 3 канала + Send Dev Alert параллельно.

---

## 03 call back — Callback Handler (mrIFyE6lwdEAIuaW, 22 ноды)

Обрабатывает callback_query от inline-клавиатур. Вызывается из 01 recieve через Execute Workflow.

1. Execute Workflow Trigger → Answer Callback (убирает "часики") → Remove Keyboard → Switch Callback: pd_accept / pd_decline / booking_confirm / booking_cancel

**pd_accept**: Save PD Consent → Log PD Consent → Find Pending Appointment → Has Pending Booking?
- TRUE → Activate Appointment (status→pending) → Send Booking Keyboard (confirm/cancel)
- FALSE → Send Consent Only ("Согласие принято")

**pd_decline**: Cancel Pending Consents → Send PD Decline ("можете задавать вопросы, запись без согласия невозможна")

**booking_confirm_{id}**: Confirm Booking (Code) → Update Booking Status (pending→booked) → Notify Booking Confirmed → Send Confirm to User ∥ Has Owner? → Notify Owner (→ 05_owner_notifications)

**booking_cancel_{id}**: Cancel Booking CB → Cancel Appointment (status→cancelled) → Send Cancelled

---

## 04_booking — Booking Service (u5Xui5bQEJw3lwAS, 32 ноды)

Сервис записи. Вызывается как tool из 02 brain. bot_token и chat_id НЕ через LLM — из БД по service_id + client_id.

**Поток**: Workflow Trigger → Parse Input → Resolve Context → Merge Context → **IF Plan Allows Booking** → Switch Action

**IF Plan Allows Booking**: проверка тарифа сервиса. Нет → No Booking Response.

**book**: Get Booking Data (CTE: расписание + записи + прайс) → Process Booking (валидация слота) → Slot Available?
- TRUE → Insert Appointment (status=pending) → Has PD Consent?
  - TRUE + web → Auto Confirm Web (status→booked) → Get Client For Owner → Notify Owner Book → Web Book Response
  - TRUE + telegram → IF Channel Web → Send Confirm Keyboard → Book Response
  - FALSE → Cleanup Old Consents → Update to Pending Consent → Send PD Keyboard Booking → PD Consent Response
- FALSE → Slot Conflict Response

**cancel**: Cancel Appointment → IF Cancel OK → Notify Owner Cancel → Cancel Response (или сразу Cancel Response если нечего отменять)

**list**: List Appointments → List Response

**slots** (новое): Get Slots Data → Build Slots Response — возвращает LLM доступные слоты

---

## 11_avito_receive — Avito Integration (7xRN29kgedmm9s4H, 18 нод)

Интеграция с Авито Мессенджером. Webhook: `POST /webhook/671365b4-59db-4c48-a7d4-17f7b026e74b/avito/:slug`.

**Поток**: Avito Webhook → Parse Webhook → (Respond OK ∥ Log Webhook) → Filter Message → Get Service → Has Service? → Has Subscription? → Is Own Message? → Get Avito Token (OAuth) → Check Message Limit → IF Limit Reached → (лимит → Send Avito Limit Msg / ОК → Get Item Details → Cache Item → Upsert Client → Prepare Context → Call 02 brain)

**Respond OK** отправляет `{"ok":true}` немедленно после парсинга (до обработки). Авито не ждёт ответа от brain.

**Parse Webhook** — поддерживает 3 формата payload:
- v3: `body.payload.value.content.text` (актуальный)
- v2: `body.payload.content.text`
- v1: `body.payload.text` / `body.text`

**Filter Message** — пропускает только `webhook_type=new_message` + `message_text≠''` + `author_id≠''`.

**Upsert Client** — `ON CONFLICT (service_id, real_avito_id) WHERE real_avito_id IS NOT NULL` (partial index).

**Log Webhook** — записывает raw_body в `avito_webhook_log` для отладки.

**Контекст для 02 brain**: `channel='avito'`, `alias_telegram='avito_user_{author_id}'`, `avito_item_context` (название/цена объявления), `avito_chat_id`, `avito_token`.

**02 brain**: Switch Channel → `avito` → Send Avito Response (POST `api.avito.ru/messenger/v1/accounts/{user_id}/chats/{chat_id}/messages`, body: `{ message: { text }, type: 'text' }`). `onError: continueRegularOutput`.

### Avito API

- **OAuth**: `POST api.avito.ru/token` (client_credentials), токен 24ч
- **Webhook подписка**: `POST api.avito.ru/messenger/v3/webhook` (v1/v2 deprecated)
- **Отправка**: `POST api.avito.ru/messenger/v1/accounts/{user_id}/chats/{chat_id}/messages`
- **Требование**: платная подписка "API мессенджера" в Avito Pro (HTTP 402 без неё)

### БД-таблицы для Авито

- `auto_services`: поля `avito_client_id`, `avito_client_secret`, `avito_user_id`
- `clients`: поля `real_avito_id`, `alias_avito`. Index: `idx_clients_service_avito UNIQUE (service_id, real_avito_id) WHERE real_avito_id IS NOT NULL`
- `avito_items_cache`: кэш объявлений `(service_id, avito_item_id)` UNIQUE
- `avito_webhook_log`: `(slug, raw_body jsonb, created_at)` — лог webhook'ов

### Подключение клиента с Авито

1. У клиента — оплачена подписка "API мессенджера" в Avito Pro
2. `UPDATE auto_services SET avito_client_id=..., avito_client_secret=..., avito_user_id=... WHERE bot_id='{slug}'`
3. Получить токен: `POST api.avito.ru/token`
4. Подписать webhook v3: `POST api.avito.ru/messenger/v3/webhook` → `{"url": "https://n8n57362.hostkey.in/webhook/671365b4-.../avito/{slug}"}`

### Текущий статус (2026-03-31)

- **bot1** (АвтоМастер Центр): credentials заполнены, webhook v3 подписан, код протестирован end-to-end (curl → 02 brain → AI-ответ). **Блокер**: нет платной подписки API мессенджера (HTTP 402).
- Остальные сервисы: avito credentials не заполнены.

### Ограничения канала Avito

1. Нет батчинга (не нужен — в Авито пишут по одному сообщению)
2. Нет голосовых (API не передаёт)
3. Нет inline-клавиатур — подтверждение записи: auto-confirm (как web)
4. ПДн: нет кнопки согласия, `pd_consent_given=false`, запись без согласия невозможна
5. Токен запрашивается каждый раз (можно оптимизировать кэшированием)

---

## 14_lead_form — Landing Lead Form (4ohPRBwYploC2Ugm, 3 ноды)

Форма захвата лидов с лендинга "Записали" (https://makslap18.github.io/zapisali-landing/). Создан 2026-03-26.

**Поток**: Webhook (POST /webhook/lead-form) → Send to Telegram (алерт в chat_id `306522564` через `assistmax_bot`) → Respond OK (`{"success": true, "message": "Заявка принята"}`)

- **Endpoint**: `POST https://n8n57362.hostkey.in/webhook/4ohPRBwYploC2Ugm/webhook/lead-form`
- **Body**: `{ name, phone }`
- **CORS**: `allowedOrigins: https://makslap18.github.io`
- **Формат алерта**: "🔔 Новая заявка с лендинга! Имя: ... Телефон: ... 📅 дата"
- errorWorkflow подключён (06_error), retryOnFail: true, maxTries: 3
- Нет валидации входных данных и rate-limiting (см. открытые вопросы)

---

## DM Bitrix Task AI (jqXCO3GCoF0y0XZ2, 13 нод)

AI-обработка комментариев к задачам в Битрикс24.

**Поток**: Bitrix24 Event (Webhook) → Validate → Get Task → Get Comment → Filter Comment → Build Prompt → Task AI (LLM Agent via OpenRouter) → Parse Response → Has Changes? → Update Task → Add Confirmation

При ошибке агента или отсутствии изменений → Skip (NoOp).

---

## TG BOT transkrib (MCjOULArlVfEoAFa, 5 нод)

Отдельный бот для транскрипции голосовых сообщений. Работает вне основной архитектуры (нативный Telegram Trigger, не через webhook-router).

**Поток**: Telegram Trigger → Get a file → Transcribe a recording (OpenAI) → Send a text message (ответ) / Send a text message1 (ошибка)

**Примечание**: не подключён к 06_error, ошибки не логируются.

---

## 05–10: Вспомогательные и виджет WF

**05_owner_notifications** (QKLGNaGcDpkYTKJm, 3 ноды): Workflow Trigger → Build Notification → Send to Owner. Уведомляет владельца о новой записи с РЕАЛЬНЫМИ данными клиента.

**06_error** (3muBfH7DpVipSOpN, 3 ноды): Error Trigger → Format Error → Send Error Alert в Telegram. Системный бот: `8375612642:AAGUKTCzkZZThrlZPijcrZ0w05JPdrBQwp0`, dev chat_id: `306522564`.

**07_onboarding_bot** (pq3tdawZFkZAnrYJ, 7 нод, ВЫКЛ): POST-эндпоинт создания сервисов. Деактивирован — нет аутентификации. Поток: Onboarding Webhook → Validate Input → Is Valid? → Create Service Center → Build Response → Return Success.

**07_appointment_reminders** (qUIDyP7ZNxU5iPSq, 7 нод, каждые 30 мин): Schedule → 3 параллельные ветки:
1. Get Reminders Due → Build Messages → Send Reminder → Mark Reminded
2. Expire Pending Consents — отмена pending_consent записей старше 15 мин
3. Cleanup Buffer — очистка батчинг-буфера

**08_table_embeddings_sync** (nZvVhdHgHkHSyXsh, 13 нод, каждые 3 часа): Schedule Trigger → Get Last Sync Time → Read Changed Rows → Format Content → Delete Orphans → Has Changes? → Prepare Batch → OpenAI Embeddings → Build Upserts → Upsert Embeddings → Save Sync Time. LIMIT 200 на запрос (защита от OOM).

**09_widget_loader** (agbjjq3sM4ovTxow, 4 ноды): GET Widget Loader → Get Config (PG) → Generate JS → Respond JS. Отдаёт JS-загрузчик виджета.

**10_widget_frame** (UvFL82lc98eSpRiT, 4 ноды): GET Widget Frame → Get Service + Config (PG) → Generate HTML → Respond HTML. Отдаёт HTML/CSS/JS чат-фрейма.

---

## 01 recieve web — Web Chat Router (tlvrVB9CPTpp1bRr, 18 нод)

Точка входа для веб-виджета. Webhook: `POST /webhook/web-chat/:slug`.

**Поток**:
1. Webhook → Get Service → Validate Request → Is Valid? (нет → Respond Error)
2. **Check Message Limit** → IF Limit Reached (да → Respond Limit)
3. Is First Message? →
   - TRUE: Upsert Client → Log PD Consent → Anonymize First → Call 02 → Respond OK
   - FALSE: Lookup Client → Intercept PD → Save PD → Anonymize & Prepare → Call 02 → Respond OK

---

## QA Evaluation — AI Bot Tests (Z424iXlqh2UNEzkz, 7 нод, ВЫКЛ)

Автоматическое тестирование AI-бота. Ручной запуск.

**Поток**: Run Tests (manual) → Get Test Service → Test Cases (Code) → Loop Tests (SplitInBatches) → Call 02 Brain → Evaluate (Code) → обратно в Loop → Report (Code)

---

## Веб-виджет

Встраиваемый чат-виджет для сайтов клиентов. Клиент автосервиса заходит на сайт, видит кнопку чата в углу экрана, кликает — открывается окно с ИИ-консультантом. Тот же AI-мозг (02 brain), что работает в Telegram.

Встраивание одной строкой:
```html
<script src="https://n8n57362.hostkey.in/webhook/de26595b-cb57-4110-8eaf-04bcd303011a/wgt-loader/{slug}" defer></script>
```

### Архитектура (3 воркфлоу)

```
Сайт клиента
  └─ <script src=".../wgt-loader/{slug}">        ← WF 09 (GET → JS)
       └─ JS создаёт кнопку чата + iframe
            └─ iframe src=".../wgt-frame/{slug}"  ← WF 10 (GET → HTML)
                 └─ Пользователь пишет сообщение
                      └─ fetch POST .../wgt-chat/{slug}  ← WF 01 recieve web
                           └─ Call 02 brain → AI-ответ → JSON response
```

**WF 09 — widget_loader**: Отдаёт JS. Читает `widget_config` + `v_auto_services` по slug. Проверяет подписку и план (`start` → пустой комментарий `/* widget: not available */`). Генерирует floating-кнопку, iframe-контейнер, lazy-load, адаптив, анимации. Передаёт конфиг в iframe через `postMessage`.

**WF 10 — widget_frame**: Отдаёт HTML-страницу чата (внутри iframe). CSS-переменные из конфига (цвета, шрифты, радиусы). Форма контактов (имя, телефон, согласие ПДн — настраивается). Батчинг сообщений (15 сек таймер). Typing indicator, timestamps, аватарки. Сессия: `crypto.randomUUID()` в sessionStorage.

**WF 01 recieve web**: Принимает POST, валидирует (UUID sessionId, message ≤ 4000, формат телефона/email), блокирует план `start` (HTTP 403), rate limit 10 msg/min по session_id, лимит по тарифу (HTTP 429), анонимизация ПДн, вызов 02 brain с `channel: 'web'`.

### Webhook URLs (production)

```
Loader: https://n8n57362.hostkey.in/webhook/de26595b-cb57-4110-8eaf-04bcd303011a/wgt-loader/{slug}
Frame:  https://n8n57362.hostkey.in/webhook/8c582cff-dac2-4375-86e4-75f762cebf6c/wgt-frame/{slug}
Chat:   https://n8n57362.hostkey.in/webhook/c748a010-d4b2-446e-ab40-53eaf51a7a05/wgt-chat/{slug}
```

### Chat API

**Request**: `POST .../wgt-chat/{slug}`
```json
{
  "sessionId": "uuid",
  "message": "текст",
  "isFirstMessage": true,
  "name": "Имя",
  "phone": "+79991234567",
  "pdConsent": true
}
```
**Response**: `{"response": "Ответ ИИ"}`

### Конфигурация виджета (БД)

Таблица `widget_config` (PostgreSQL), PK = `service_id`. Владелец редактирует через NocoDB (view `service_N.widget_config`).

| Группа | Поля | Дефолт |
|--------|------|--------|
| Цвета | `primary_color`, `text_color`, `chat_bg_color`, `user_bubble`, `bot_bubble` | `#e8b830`, `#ffffff`, `#ffffff`, `#1a2e4a`, `#f5f6f8` |
| Шрифт/UI | `font_family`, `border_radius` | `Inter, system-ui, sans-serif`, `16` |
| Контент | `title`, `subtitle`, `greeting`, `placeholder`, `logo_url` | название сервиса, "Обычно отвечаем в течение минуты", "Здравствуйте! Чем могу помочь?", "Введите сообщение..." |
| Поведение | `position`, `auto_open` | `bottom-right`, `false` |
| Безопасность | `allowed_origins` | пустое (= все домены) |
| Сбор данных | `collect_name`, `collect_phone`, `collect_email` | `true`, `true`, `false` |

### Безопасность виджета

| Мера | Реализация |
|------|------------|
| XSS-защита | `esc()` для HTML, `sC()` для цветов, `sF()` для шрифтов, `textContent` вместо `innerHTML` |
| CSP | `script-src 'unsafe-inline'; frame-ancestors <dynamic>; connect-src n8n` |
| Origin-валидация | Exact match `allowed_origins` в loader |
| Rate limit (сессия) | 10 msg/min по session_id |
| Rate limit (IP) | Traefik middleware: 20 req/min, burst 5 |
| SQL-инъекции | Параметризованные запросы ($1, $2) |
| ПДн (152-ФЗ) | Чекбокс согласия, анонимизация перед LLM |
| Валидация | UUID sessionId, message ≤ 4000, форматы phone/email/name |

### Известные баги виджета

**1. Вебхуки возвращают 404 (баг n8n #21614)**: API activate не регистрирует вебхуки в Express-роутере. После ЛЮБОГО обновления WF через API/MCP: деактивация → активация → `docker compose restart n8n`.

**2. Короткие URL не работают**: n8n 2.x требует полный URL с webhookId (`/webhook/{webhookId}/wgt-loader/bot1`), не `/webhook/wgt-loader/bot1`.

**3. Draft vs Published**: n8n хранит две версии WF — draft (`workflow_entity.nodes`) и published (`workflow_versions.nodes`). MCP/API обновляет draft. Публикация: деактивация → активация → рестарт.

**4. Кнопка "Начать чат" не работает (ОТКРЫТЫЙ БАГ)**: `batchBuffer.join('\n')` внутри template literal рендерится как реальный перенос строки, ломает `<script>`. Фикс: заменить на `batchBuffer.join(String.fromCharCode(10))` в **published version**. Draft обновлён, published — нет. Инструкция: `WIDGET_BUTTON_FIX_HANDOFF.md`.

---

## Лендинг "Записали"

- **Production**: https://makslap18.github.io/zapisali-landing/
- **Репозиторий**: https://github.com/makslap18/zapisali-landing
- **Хостинг**: GitHub Pages (статический HTML)
- **Структура**: один файл `index.html` + `favicon.svg`, CSS и JS инлайновые (zero dependencies, кроме Google Fonts)

### Секции страницы

1. **Навигация** — логотип "Записали", якорные ссылки, CTA-кнопка
2. **Hero** — заголовок, подзаголовок, статистика (24/7, <2 мин ответ, 14 дней бесплатно)
3. **Проблема/Решение** — боли клиента → что делает бот
4. **Демо** — iframe с живым виджетом (bot1, АвтоМастер Центр)
5. **Возможности** — 6 карточек (ИИ-ответы, запись, напоминания, голос, виджет, аналитика)
6. **Как это работает** — 3 шага (подключение → настройка → работа)
7. **Тарифы** — Start (3990₽), Business (6990₽), Premium (11990₽)
8. **Калькулятор экономии** — слайдеры (звонки, стоимость часа)
9. **FAQ** — аккордеон
10. **CTA + Форма заявки** — имя + телефон → 14_lead_form WF
11. **Footer** — копирайт

### Форма заявки

- **Endpoint**: `POST https://n8n57362.hostkey.in/webhook/4ohPRBwYploC2Ugm/webhook/lead-form`
- **Backend**: WF `14_lead_form` → Telegram alert в chat_id `306522564` через `assistmax_bot`
- **CORS**: `allowedOrigins: https://makslap18.github.io`
- **Формат**: "🔔 Новая заявка с лендинга! Имя: ... Телефон: ... 📅 дата"

### Кнопка Telegram

`https://t.me/maksicrypto?text=Здравствуйте, хочу подключить бота для автосервиса`

### SEO

- Open Graph: og:title, og:description, og:image, og:locale (ru_RU)
- Twitter Card: summary_large_image
- Structured Data: schema.org `SoftwareApplication` с AggregateOffer (3990-11990 RUB)
- Canonical: https://makslap18.github.io/zapisali-landing/

### TODO лендинга

1. Создать `og-image.png` (1200×630px) — файл не создан
2. Завести Яндекс.Метрику — заглушка в `<head>`, закомментирована (нет ID)
3. (Опц.) Google Analytics — аналогично
4. Настроить виджет (баг кнопки "Начать чат" — см. выше)

---

## Аудит безопасности (2026-03-06)

### Исправлено

| # | Уровень | Проблема | Решение |
|---|---------|----------|---------|
| 1 | КРИТ | SQL-инъекции в WF01, WF03 (9 нод) — string interpolation вместо $1/$2 | Все переведены на parameterized queries |
| 2 | КРИТ | `/webhook/onboarding` без auth | WF деактивирован |
| 3 | КРИТ | WF03 `Has Pending Booking?` — strict type error ломал flow | Оператор string isNotEmpty |
| 4 | СРЕД | Dead end при subscription=false в WF01 | Добавлены ноды "Сервис недоступен" |
| 5 | СРЕД | Пустой pd_consent_log (152-ФЗ) | Нода Log PD Consent в WF03 + ретроспективные записи |
| 6 | СРЕД | Ошибки валидации в WF04 (List Response, IF Channel Web) | Исправлены типы и операторы |
| 7 | СРЕД | ~15 HTTP нод без retry | retryOnFail: true, maxTries: 3 на всех |
| 8 | СРЕД | Нет errorWorkflow на виджетах | errorWorkflow = 06_error + onError |
| 9 | СРЕД | Нет индексов БД | 5 индексов: chat_histories, faq, prices, appointments |

### Осознанно оставлено

- Хардкод bot_token в 06_error (доступ только у владельца)
- Plaintext токены в БД (NocoDB-пользователи через svc_N schemas, секреты не видят)
- Устаревшие typeVersions нод (155 warnings, риск breaking changes)

---

## NocoDB — Доступ владельцев

Каждый владелец получает аккаунт в NocoDB с доступом только к своим данным. Тройная изоляция: NocoDB Base → PG user svc_N → Views с WHERE service_id = N.

| Service | NocoDB Base ID | PG User |
|---------|---------------|---------|
| test (id=1) | pihvv12i0coms7b | svc_1 |
| АвтоМастер Центр (id=2) | p4matng0r1rx5th | svc_2 |
| СТО Гараж Плюс (id=3) | pvmn3lx72l03tmf | svc_3 |
| ТехноСервис Авто (id=4) | pku9zpd3y3wmil6 | svc_4 |
| Ganin Garage test (id=5) | — | svc_5 |

Admin: `makslap18@gmail.com` (super роль)

**Доступ владельца**: clients (SELECT only: id, name, phone, consent), appointments/schedule/special_days/faq/prices/parts/vehicles (полный CRUD), my_service (SELECT only).

### Добавление нового владельца

1. INSERT в auto_services → триггер создаёт schema + user + views
2. Пароль: `SELECT pg_password FROM service_credentials WHERE service_id = N`
3. NocoDB: Create Base → External Data Source (host.docker.internal:5432, svc_N, schema service_N) → Meta Sync → отключить internal source
4. Invite владельца: Share Base → email → Editor

**Quirk**: NocoDB хардкодит `WHERE table_schema = 'public'`. Нужен `searchPath` в `nc_sources_v2.config` (БД nocodb_meta) + рестарт.

---

## Процедура онбординга нового автосервиса

```
1. INSERT INTO auto_services → триггер создаёт schema + PG user + views

2. Импорт данных: python import_price.py (XLSX → SQL) → psql -f import.sql

3. setWebhook через прокси:
   curl -F "url=https://46.149.72.77/webhook/bf526e46-.../telegram-router/{bot_id}" \
        -F "certificate=@/tmp/tg-proxy.pem" "https://api.telegram.org/bot{TOKEN}/setWebhook"

4. INSERT INTO work_schedule — 7 строк (расписание)

5. Touch updated_at на prices/vehicles → 08_sync подхватит embeddings

6. (Опц.) NocoDB база для владельца
```

---

## Архитектурные паттерны

| Паттерн | Описание |
|---------|----------|
| Service lookup | `v_auto_services` view по `bot_id` (slug) |
| Client anonymization | Intercept PD → Save PD → Anonymize (алиасы для LLM) |
| Chat memory key | `alias_telegram` для TG, `'web_' + sessionId` для Web |
| Channel routing (02 brain) | Switch Channel: web → Code return / avito → HTTP / default → Telegram HTTP |
| Channel routing (04_booking) | IF Channel Web: web → auto-confirm / telegram → keyboard |
| Message batching | Save to Buffer → Wait → Check If Latest → Collect → Combine |
| Message limit | Check Message Limit (PG) → IF Limit Reached → block/allow |
| Plan check (booking) | IF Plan Allows Booking → Switch Action / No Booking Response |
| Owner notifications | На confirm: Notify Owner через 05 WF. На cancel: Notify Owner Cancel прямо в 04 |
| WF inter-call | Execute Workflow node (passthrough mode) |

---

## Открытые вопросы

1. **07_onboarding_bot** — требует Header Auth для активации.
2. **Дублирование номера 07** — и onboarding, и reminders.
3. **NocoDB аккаунты владельцев** — базы созданы, аккаунты не приглашены.
4. **Ganin Garage (id=5)** — NocoDB база не создана, owner_telegram_id не задан.
5. ~~**11_avito_receive** — Respond OK в конце цепочки~~ — **ИСПРАВЛЕНО 2026-03-31**: Respond OK перенесён в начало (после Parse), webhook v3 подписан, парсер обновлён.
6. **_test_webhook_check** — активен в проде, стоит деактивировать.
7. **TG BOT transkrib / DM Bitrix Task AI** — не подключены к 06_error.
8. **PD-логика дублируется** в 01 recieve, 01 recieve web, 11_avito_receive.
9. **14_lead_form** — нет валидации входных данных и rate-limiting.

---

## Авито — план привлечения первых тестовых клиентов

### Почему Авито — главный канал

1. **Готовая интеграция** — у конкурентов (YCLIENTS, 1С:Автосервис) Авито-интеграции нет
2. **Доказуемая проблема** — скорость ответа на Авито напрямую влияет на рейтинг и показы объявлений. Клиент ждёт ответ 10-15 минут. Мастер на яме — не до телефона. Кто ответил первым, тот получил клиента
3. **База 1000+ автосервисов** — контакты уже есть

### Тактика: "тайный покупатель"

**Шаг 1.** Выбрать 30-50 автосервисов из базы, у которых есть объявления на Авито.

**Шаг 2.** Написать как клиент: *"Здравствуйте, сколько стоит диагностика подвески? Можно записаться на завтра?"*

**Шаг 3.** Зафиксировать время ответа в таблице (автосервис / время ответа / качество ответа / есть ли запись).

**Шаг 4.** Через 2-3 дня тем, кто ответил медленно (1+ час), написать как разработчик:

> Добрый день! Пару дней назад писал вам как клиент — ответ пришёл через [X часов]. На Авито это напрямую снижает ваш рейтинг и показы объявлений.
>
> Я разработчик, сделал AI-бота, который отвечает клиентам на Авито за 30 секунд — отвечает по вашему прайсу, записывает на приём. Ищу 5 сервисов для бесплатного теста на 2 недели.
>
> Вот пример диалога: [скриншот]
>
> Интересно попробовать?

### Почему это лучше холодного спама

- Показываешь **конкретную проблему** (их собственное время ответа), а не абстрактную пользу
- Предлагаешь **бесплатно** — нечего терять
- Пишешь **туда, где они уже сидят** — в Авито-чат

### Что подготовить заранее

1. **2-3 скриншота** диалога бота с тестовым клиентом (запись, ответ по прайсу, голосовое)
2. **Одно число**: "бот отвечает за 30 секунд"

### Источники

- Скорость ответа на Авито влияет на продажи: https://tenchat.ru/media/1475110-skorost-otveta-na-avito-napryamuyu-vliyayet-na-vashi-prodazhi
- Автоответы на Авито — рост конверсии: https://umnico.com/ru/blog/avito-avtootveti/
- Кейс ИИ-консультанта для автосалона (конверсия 32% → 46.8%): https://vc.ru/ai/1723516-kak-vnedrenie-ii-konsultanta-povysilo-effektivnost-avtosalona
