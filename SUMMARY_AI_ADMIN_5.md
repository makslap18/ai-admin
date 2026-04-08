# AI-Администратор для автосервисов — Техническое саммари

> Обновлено: 2026-04-08

## Общая идея

Мультитенантная SaaS-платформа AI-ботов для автосервисов и детейлинг-студий. Каждый автосервис получает своего ИИ-бота, который работает через **3 канала** (Telegram, веб-виджет, Авито) и умеет:

- Отвечать клиентам через LLM (Gemini Flash via OpenRouter) с контекстом конкретного сервиса (RAG по прайсу, услугам, FAQ)
- Записывать на приём (выбор даты/времени, подтверждение, отмена)
- Эскалировать диалог к живому оператору (владельцу) с передачей контекста
- Поддерживать режим "оператор-мост" — владелец отвечает клиенту через Telegram бота
- Напоминать о записях (за 24ч и 2ч)
- Уведомлять владельца о новых записях/отменах
- Контролировать лимиты сообщений по тарифу
- Соблюдать 152-ФЗ: реальные ПДн не передаются в LLM, используются алиасы

### Общая схема

```
                    ┌──────────────────────────────────────────────────────────┐
                    │                    02 brain (AI Agent)                   │
                    │  LLM + RAG + Booking Tool + Escalation Tool             │
                    │  Telegram / Web / Avito → LLM → ответ по каналу        │
                    └──────┬──────────┬────────────────┬───────────────────────┘
                           │          │                │
               ┌───────────┘          │                └──────────────┐
               ▼                      ▼                              ▼
       04_booking               15_escalation                 05_owner_notifications
   (Book/Cancel/List/Slots)   (Notify owner, handoff)         (Notify owner)
               │                      │
               ▼                      ▼
         03 call back           12_operator_bridge
     (Inline keyboard:          (Operator ↔ Client
      PD, booking,               message relay)
      escalation)

Входные точки:
  Telegram  → 01 recieve        → [Op check] → 02 brain / 12_operator_bridge
  Web       → 01 recieve web    → [Op check] → 02 brain / 12_operator_bridge
  Web poll  → 13_web_poll       → новые сообщения для виджета
  Avito     → 11_avito_receive  → 02 brain
  Лендинг   → 14_lead_form     → Telegram alert

Виджет:
  09_widget_loader (JS) → 10_widget_frame (HTML iframe) → 01 recieve web
                                                        ← 13_web_poll (поллинг)

Фоновые:
  07_appointment_reminders — каждые 30 мин (напоминания + cleanup)
  08_table_embeddings_sync — каждые 3 часа (RAG sync)
  06_error — ловит ошибки всех WF
```

---

## Инфраструктура

- **Сервер n8n (Россия)**: sese (141.105.70.225), SSH root, ключ `~/.ssh/id_ed25519`
- **VPN-сервер (зарубежный)**: 46.149.72.77, SSH root (порт 22), 3X-UI панель: `46.149.72.77:2053/5ae0a54929028fdb5e88c280f8381e8b/`
- **n8n**: self-hosted, домен `n8n57362.hostkey.in`
- **БД**: PostgreSQL с pgvector, на том же сервере
- **LLM**: Gemini 3 Flash Preview через OpenRouter
- **Embeddings**: OpenAI-совместимый API через proxyapi
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
| auto_services | Автосервисы (6 шт) | id, name, bot_id, telegram_bot_token, owner_telegram_id, plan, custom_prompt, escalation_enabled, escalation_threshold |
| clients | Клиенты (real + alias данные) | service_id, real_name/phone/email/telegram_id, alias_*, pd_consent_given |
| appointments | Записи на приём | service_id, client_id, date/time, status (pending→pending_consent→booked→completed/cancelled) |
| vehicles | Автомобили в наличии/под заказ | service_id, name, year, price, in_stock_price, description |
| work_schedule / special_days | Расписание | service_id, day_of_week, open/close_time |
| service_prices / parts_prices | Прайс-листы | service_id, name, price, category |
| faq | FAQ | service_id, question, answer |
| embeddings | RAG-данные с vector(1536) | service_id, content, embedding |
| n8n_chat_histories8 | Чат-память (+ поле `source` для фильтрации) | session_id (= alias_telegram), message (jsonb с type, content, source) |
| pd_consent_log | Аудит согласий ПДн (152-ФЗ) | client_id, action, channel |
| service_credentials | DB credentials per service | service_id, pg_user, pg_password |
| widget_config | Конфигурация виджета | service_id (PK), primary_color, greeting, collect_name/phone/email, allowed_origins |
| escalation_log | Журнал эскалаций к оператору | service_id, client_id, channel, level (notify/handoff), category, reason, owner_response |
| operator_sessions | Сессии живого оператора | service_id, client_id, client_chat_id, operator_chat_id, channel, status, is_current |
| message_usage | Подсчёт сообщений по тарифу | service_id, month (varchar "2026-04"), count |
| message_buffer | Буфер батчинга сообщений | service_id, chat_id, message_text, created_at |
| rate_limits | Rate limiting по IP | ip_address (inet), endpoint, window_start, request_count |
| avito_items_cache | Кэш объявлений Авито | service_id, avito_item_id |
| avito_webhook_log | Лог Авито webhook'ов | slug, raw_body (jsonb), created_at |
| rag_clients | Клиенты RAG-системы (Google Drive) | client_id (PK), table_name, gdrive_folder_id |
| rag_processed_files | Обработанные файлы для RAG | client_id, file_id, file_name |

**EXCLUDE constraint** на appointments: запрещает пересечение слотов для статусов booked, pending, pending_consent.

### Мультитенантность

Каждый автосервис получает schema `service_N` с view на public таблицы (WHERE service_id = N, WITH LOCAL CHECK OPTION). Отдельный DB user `svc_N`. Триггер `trg_auto_service_schema` автоматически создаёт schema + user + views при INSERT в auto_services.

### Автосервисы (актуальный список)

| id | name | bot_id | plan | escalation | owner_telegram_id |
|----|------|--------|------|------------|-------------------|
| 1 | test | bot2 | premium | включена | 306522564 |
| 2 | АвтоМастер Центр | bot1 | premium | включена | 306522564 |
| 3 | СТО Гараж Плюс | bot_garazh_02 | premium | выключена | — |
| 4 | ТехноСервис Авто | bot_techno_03 | premium | выключена | — |
| 5 | Ganin Garage test | bot_ganin_05 | start | выключена | — |
| 8 | АвтоКультура | bot_kultura_06 | premium | выключена | 306522564 |

Schemas: `service_1` .. `service_5`, `service_8` (id=6,7 отсутствуют).

### Новые колонки auto_services (добавлены после 2026-03-26)

| Колонка | Тип | Описание |
|---------|-----|----------|
| `plan` | varchar(20), default 'business' | Тарифный план (start/business/premium) |
| `custom_prompt` | text, default '' | Кастомный системный промпт для LLM |
| `escalation_enabled` | boolean, default false | Включена ли эскалация к оператору |
| `escalation_threshold` | integer, default 30000 | Порог срабатывания эскалации |

### Credentials в n8n

| ID | Имя | Назначение |
|----|-----|-----------|
| IYVS1xv3JEXM9Cve | NocoDB | PostgreSQL |
| 2y0lv5fuNyJBVIq7 | proxyapi | OpenAI Embeddings |
| PgdwDcBo4eMKNRMw | maks | OpenRouter (Gemini Flash) |

---

## Воркфлоу — Полный список

| ID | Название | Назначение | Нод | Статус |
|----|----------|-----------|-----|--------|
| VpBY64I8MPl83xr7 | 01 recieve | Telegram Router + Voice + Batching + Op Bridge + Limit | 34 | Активен |
| tlvrVB9CPTpp1bRr | 01 recieve web | Web Chat Router + Op Bridge + Limit | 25 | Активен |
| A2IeiFBc0I0uKgPJ | 02 brain | AI Agent (Telegram + Web + Avito) + Escalation Tool | 19 | Активен |
| mrIFyE6lwdEAIuaW | 03 call back | Callback Handler (PD, Booking, Escalation) | 34 | Активен |
| u5Xui5bQEJw3lwAS | 04_booking | Booking Service (book/cancel/list/slots) + Plan Check | 32 | Активен |
| QKLGNaGcDpkYTKJm | 05_owner_notifications | Owner Notify | 3 | Активен |
| 3muBfH7DpVipSOpN | 06_error | Error Handler | 3 | Активен |
| qUIDyP7ZNxU5iPSq | 07_appointment_reminders | Reminders + Cleanup + Buffer Cleanup | 8 | Активен |
| nZvVhdHgHkHSyXsh | 08_table_embeddings_sync | RAG Sync (каждые 3 часа) | 13 | Активен |
| agbjjq3sM4ovTxow | 09_widget_loader | Widget JS Loader | 4 | Активен |
| UvFL82lc98eSpRiT | 10_widget_frame | Widget Chat Frame (HTML/CSS/JS) | 4 | Активен |
| 7xRN29kgedmm9s4H | 11_avito_receive | Avito Messenger Integration | 18 | Активен |
| OiD4Qw3qz5l1jbVG | 12_operator_bridge | Оператор-мост (relay сообщений owner↔client) | 36 | Активен |
| Rp34jGsWdC68ZqMs | 13_web_poll | Поллинг новых сообщений для виджета | 6 | Активен |
| 4ohPRBwYploC2Ugm | 14_lead_form | Landing Page Lead Form → TG Alert | 3 | Активен |
| pkJdtB4sxTZVMHvS | 15_escalation | Эскалация диалога к владельцу | 11 | Активен |
| pq3tdawZFkZAnrYJ | 07_onboarding_bot | Onboarding (нет auth) | 7 | ВЫКЛ |
| Z424iXlqh2UNEzkz | QA Evaluation — AI Bot Tests | Automated QA для 02 brain | 7 | ВЫКЛ |

**Не входящие в проект**: `TG BOT transkrib` (MCjOULArlVfEoAFa, ВЫКЛ), `DM Bitrix Task AI` (jqXCO3GCoF0y0XZ2, ВЫКЛ), `_test_webhook_check` (sSj2ZM8EmufTYbXi, ВЫКЛ).

**Экспериментальные / неактивные**: `06_daily_stats_aggregator`, `08_pd_consent_flow`, `09_health_check`, `01_telegram_webhook_router`, `02_message_processor`, `11_document_ingestion`, `12_rag_db_creator`, `13_rag_document_ingestion`, `MCP Connection Test`.

**Архивные**: `_migration_avito`, `DM Voice Archive Transcriber`, `My workflow`, `My workflow 2`, `CSV Import: Parts Prices`, `Ultimate Agentic RAG AI Agent Template` + старые версии.

---

## 01 recieve — Telegram Router (VpBY64I8MPl83xr7, 34 ноды)

Точка входа для Telegram webhook'ов. Webhook: `POST /webhook/telegram-router/:slug`.

**Поток**:
1. Webhook → Get Service (v_auto_services по slug) → Has Subscription? (нет → No Subscription Response → Send No Sub Message)
2. Parse Input → **Check Op Sessions** → **Determine Op Mode** → **IF Op Mode** (оператор активен → **Call Op Bridge** → стоп)
3. Если оператор НЕ активен: **Check Message Limit** → IF Limit Reached → (лимит → Limit Response → Send Limit Msg / ОК → Switch Type)
4. Switch Type: message / voice / callback_query / contact

**message**: Send Typing ∥ Save to Buffer → Wait for More → Check If Latest → (если не последнее — стоп) → Collect Messages → Prepare Combined → Upsert Client → Intercept PD → Save PD → Anonymize & Prepare → Call 02

**voice**: Get Voice File → Download Voice → Transcribe (Whisper) → Set Voice Text → далее в общий батчинг-поток (Save to Buffer)

**callback_query**: → Call '03 call back' (Execute Workflow)

**contact**: Save Phone → Phone Saved (подтверждение в Telegram)

**Новое vs предыдущей версии**: +4 ноды — маршрутизация через оператор-мост (Check Op Sessions, Determine Op Mode, IF Op Mode, Call Op Bridge). Если для данного клиента есть активная сессия оператора, сообщение идёт не в AI, а напрямую владельцу через 12_operator_bridge.

---

## 01 recieve web — Web Chat Router (tlvrVB9CPTpp1bRr, 25 нод)

Точка входа для веб-виджета. Webhook: `POST /webhook/web-chat/:slug`.

**Поток**:
1. Webhook → Get Service → Validate Request → Is Valid? (нет → Respond Error)
2. **Check Message Limit** → IF Limit Reached (да → Respond Limit)
3. Is First Message? →
   - TRUE: Upsert Client → Log PD Consent → Anonymize First → **Check Op Session**
   - FALSE: Lookup Client → Intercept PD → Save PD → Anonymize & Prepare → **Check Op Session**
4. **Merge Op Data** → **Op Active?**
   - Оператор активен → **Forward to Operator** → **Send to Owner TG** → **Save Web Client Hist** → **Respond Op Active**
   - Оператор НЕ активен → Call 02 → Respond OK

**Новое vs предыдущей версии**: +7 нод — полная поддержка оператор-моста для веб-канала. Когда оператор активен, сообщение клиента пересылается владельцу в Telegram, сохраняется в истории, и клиент получает подтверждение.

**Валидация контактов** (из WIDGET_UPDATE_STATUS): контактные данные теперь необязательны. Валидация имени/телефона срабатывает только если данные реально переданы (`hasContactData`).

---

## 02 brain — AI Agent (A2IeiFBc0I0uKgPJ, 19 нод)

ИИ-агент с анонимизированным контекстом. Поддерживает **3 канала**: Telegram, Web, Avito.

- **LLM**: Gemini 3 Flash Preview через OpenRouter (lmChatOpenRouter)
- **Память**: Postgres Chat Memory (ключ = alias_telegram; для web: `web_` + sessionId)
- **RAG**: PGVector Store + Embeddings OpenAI, фильтр по service_id
- **Tools**: Booking Tool → вызывает 04_booking, **Escalation Tool** → вызывает 15_escalation

**Поток**: Execute Workflow Trigger → Edit Fields → AI Agent → Merge Channel → Switch Channel → ответ по каналу

**Маршрутизация ответа (Switch Channel)**:
- `web` → Return Web Response (Code)
- `avito` → Send Avito Response (HTTP Request)
- default (telegram) → Send Telegram Response (HTTP Request)

**Обработка ошибок (Switch Channel Error)**: аналогичный switch на 3 канала + Send Dev Alert параллельно.

**Новое vs предыдущей версии**: +1 нода — Escalation Tool. LLM может вызвать эскалацию, когда считает нужным (клиент просит оператора, сложный вопрос, срочная ситуация).

---

## 03 call back — Callback Handler (mrIFyE6lwdEAIuaW, 34 ноды)

Обрабатывает callback_query от inline-клавиатур. Вызывается из 01 recieve через Execute Workflow.

1. Execute Workflow Trigger → Answer Callback (убирает "часики") → Remove Keyboard → Switch Callback: **6 веток** (было 4)

**pd_accept**: Save PD Consent → Log PD Consent → Find Pending Appointment → Has Pending Booking?
- TRUE → Activate Appointment (status→pending) → Send Booking Keyboard (confirm/cancel)
- FALSE → Send Consent Only ("Согласие принято")

**pd_decline**: Cancel Pending Consents → Send PD Decline

**booking_confirm_{id}**: Confirm Booking → Update Booking Status (pending→booked) → Notify Booking Confirmed → Send Confirm to User ∥ Has Owner? → Notify Owner (→ 05_owner_notifications)

**booking_cancel_{id}**: Cancel Booking CB → Cancel Appointment (status→cancelled) → Send Cancelled

**esc_accept / esc_takeover** (НОВОЕ): Parse Escalation CB → Update Escalation Log → Send Esc Confirm → IF Took Over → Get Esc Client → Create Esc Session → Get Esc Chat History → Build Esc Bridge Msg → Send Esc History → Esc Client TG? → Notify Esc Client → Save Esc Connect

**esc_reject** (НОВОЕ): Parse Escalation CB → Update Escalation Log → Send Esc Confirm

**Новое vs предыдущей версии**: +12 нод — полный блок обработки эскалационных callback'ов. Владелец получает кнопки "Принять"/"Взять на себя"/"Отклонить". При "Взять на себя" создаётся operator_session, клиенту отправляется уведомление, владелец получает историю чата.

---

## 04_booking — Booking Service (u5Xui5bQEJw3lwAS, 32 ноды)

Сервис записи. Вызывается как tool из 02 brain. bot_token и chat_id НЕ через LLM — из БД по service_id + client_id.

**Поток**: Workflow Trigger → Parse Input → Resolve Context → Merge Context → **IF Plan Allows Booking** → Switch Action

**IF Plan Allows Booking**: проверка тарифа сервиса. План `start` → No Booking Response.

**book**: Get Booking Data (CTE: расписание + записи + прайс) → Process Booking (валидация слота) → Slot Available?
- TRUE → Insert Appointment (status=pending) → Has PD Consent?
  - TRUE + web → Auto Confirm Web (status→booked) → Get Client For Owner → Notify Owner Book → Web Book Response
  - TRUE + telegram → Send Confirm Keyboard → Book Response
  - FALSE → Cleanup Old Consents → Update to Pending Consent → Send PD Keyboard Booking → PD Consent Response
- FALSE → Slot Conflict Response

**cancel**: Cancel Appointment → IF Cancel OK → Notify Owner Cancel → Cancel Response

**list**: List Appointments → List Response

**slots**: Get Slots Data → Build Slots Response — возвращает LLM доступные слоты

---

## 12_operator_bridge — Оператор-мост (OiD4Qw3qz5l1jbVG, 36 нод) — НОВЫЙ

Реализует прямое общение владельца с клиентом через Telegram-бота. Владелец пишет в чат бота → сообщение доставляется клиенту. Клиент отвечает → сообщение доставляется владельцу.

**Режимы (Switch Mode)**:
1. **Сообщение от клиента** → проверка web-канала → пересылка владельцу
2. **Сообщение от владельца** → Get Client for Bridge → Format Bridge Msg → Send to Client
3. **Команда от владельца** → Parse Command → Switch Command:
   - `/find {имя/телефон}` → Find Client → Client Found? → Send Client List
   - `/close` → Close Session → Had Session? → Build End Msgs → End Notify Client + End Notify Owner
   - `/list` → List Sessions → Send Client List
   - `/switch {id}` → Do Switch → Switch Response
   - `/help` → Send Help

**Ключевые SQL-операции**:
- `Save Op History` — сохраняет сообщение оператора с `source: 'operator'`
- `Save Connect to History` — `source: 'system'`, текст "[Оператор подключился к диалогу]"
- `Save End to History` — `source: 'system'`, текст "[Оператор отключился. AI-бот снова отвечает.]"

**Поддержка web-канала**: проверки `Is Web Client?`, `Connect Client Web?`, `End Client Web?` — для корректного уведомления web-клиентов.

---

## 13_web_poll — Поллинг для виджета (Rp34jGsWdC68ZqMs, 6 нод) — НОВЫЙ

Endpoint для long-polling — виджет периодически запрашивает новые сообщения.

**Поток**: Poll Webhook → Get Service → Check Session → Get New Messages → Build Response → Respond

**SQL-фильтрация сообщений** (из Get New Messages):
```sql
WHERE session_id = 'web_' || $1 AND id > $2
  AND message->>'type' = 'ai'
  AND message->>'source' IS DISTINCT FROM 'system'
  AND (message->'tool_calls' IS NULL OR message->'tool_calls' = '[]'::jsonb)
  AND message->>'content' NOT LIKE 'Calling %'
```

Фильтры гарантируют, что клиент видит только ответы бота/оператора. Системные сообщения ("[Оператор подключился]"), tool call сообщения ("Calling Escalation_Tool...") — скрыты.

---

## 15_escalation — Эскалация к владельцу (pkJdtB4sxTZVMHvS, 11 нод) — НОВЫЙ

Вызывается как tool из 02 brain, когда LLM решает, что нужен живой оператор.

**Поток**: Workflow Trigger → Parse Input → Get Service Info → **Escalation Enabled?**
- ДА → Get Real Client → Log Escalation → Has Owner? → Build Notification → Send to Owner (TG с inline-кнопками) → Return Response
- НЕТ → Log Only → Return Response

**Категории эскалации**: `human_request` (клиент попросил), `urgent` (срочная ситуация).

**Inline-кнопки владельцу**: "Принять" (esc_accept), "Взять на себя" (esc_takeover), "Отклонить" (esc_reject) → обрабатываются в 03 call back.

**Данные**: работает с реальными ПДн клиента (не алиасами) — для уведомления владельца нужны настоящие имя/телефон.

---

## 11_avito_receive — Avito Integration (7xRN29kgedmm9s4H, 18 нод)

Интеграция с Авито Мессенджером. Webhook: `POST /webhook/671365b4-59db-4c48-a7d4-17f7b026e74b/avito/:slug`.

**Поток**: Avito Webhook → Parse Webhook → (Respond OK ∥ Log Webhook) → Filter Message → Get Service → Has Service? → Has Subscription? → Is Own Message? → Get Avito Token (OAuth) → Check Message Limit → IF Limit Reached → (лимит → Send Avito Limit Msg / ОК → Get Item Details → Cache Item → Upsert Client → Prepare Context → Call 02 brain)

**Respond OK** отправляет `{"ok":true}` немедленно после парсинга (до обработки).

**Parse Webhook** — поддерживает 3 формата payload (v1/v2/v3).

**Upsert Client** — `ON CONFLICT (service_id, real_avito_id) WHERE real_avito_id IS NOT NULL` (partial index).

**Контекст для 02 brain**: `channel='avito'`, `alias_telegram='avito_user_{author_id}'`, `avito_item_context`, `avito_chat_id`, `avito_token`.

### Ограничения канала Avito

1. Нет батчинга (не нужен — в Авито пишут по одному)
2. Нет голосовых (API не передаёт)
3. Нет inline-клавиатур — подтверждение записи: auto-confirm (как web)
4. ПДн: нет кнопки согласия, запись без согласия невозможна
5. Токен запрашивается каждый раз (можно оптимизировать кэшированием)
6. **Блокер**: у bot1 (АвтоМастер Центр) нет платной подписки API мессенджера (HTTP 402)

---

## 14_lead_form — Landing Lead Form (4ohPRBwYploC2Ugm, 3 ноды)

Форма захвата лидов с лендинга "Записали".

**Поток**: Webhook (POST /webhook/lead-form) → Send to Telegram (алерт в chat_id `306522564` через `assistmax_bot`) → Respond OK

- **Endpoint**: `POST https://n8n57362.hostkey.in/webhook/4ohPRBwYploC2Ugm/webhook/lead-form`
- **CORS**: `allowedOrigins: https://makslap18.github.io`
- errorWorkflow подключён (06_error), retryOnFail: true, maxTries: 3

---

## Вспомогательные воркфлоу

**05_owner_notifications** (QKLGNaGcDpkYTKJm, 3 ноды): Workflow Trigger → Build Notification → Send to Owner. Уведомляет владельца о новой записи с РЕАЛЬНЫМИ данными клиента.

**06_error** (3muBfH7DpVipSOpN, 3 ноды): Error Trigger → Format Error → Send Error Alert в Telegram. Dev chat_id: `306522564`.

**07_onboarding_bot** (pq3tdawZFkZAnrYJ, 7 нод, ВЫКЛ): POST-эндпоинт создания сервисов. Деактивирован — нет аутентификации.

**07_appointment_reminders** (qUIDyP7ZNxU5iPSq, 8 нод, каждые 30 мин): Schedule → 3 параллельные ветки:
1. Get Reminders Due → Build Messages → Send Reminder → Mark Reminded
2. Expire Pending Consents — отмена pending_consent записей старше 15 мин
3. Cleanup Buffer — очистка батчинг-буфера

**08_table_embeddings_sync** (nZvVhdHgHkHSyXsh, 13 нод, каждые 3 часа): Schedule Trigger → Get Last Sync Time → Read Changed Rows → Format Content → Delete Orphans → Has Changes? → Prepare Batch → OpenAI Embeddings → Build Upserts → Upsert Embeddings → Save Sync Time. LIMIT 200 на запрос.

---

## Веб-виджет

Встраиваемый чат-виджет для сайтов клиентов. Клиент видит кнопку чата → открывает окно с ИИ-консультантом (тот же 02 brain). Поддерживает режим оператора через поллинг (13_web_poll).

Встраивание одной строкой:
```html
<script src="https://n8n57362.hostkey.in/webhook/de26595b-cb57-4110-8eaf-04bcd303011a/wgt-loader/{slug}" defer></script>
```

### Архитектура (5 воркфлоу)

```
Сайт клиента
  └─ <script src=".../wgt-loader/{slug}">        ← WF 09 (GET → JS)
       └─ JS создаёт кнопку чата + iframe
            └─ iframe src=".../wgt-frame/{slug}"  ← WF 10 (GET → HTML)
                 ├─ Пользователь пишет сообщение
                 │    └─ fetch POST .../wgt-chat/{slug}  ← WF 01 recieve web
                 │         └─ [Op check] → 02 brain / 12_operator_bridge → JSON response
                 └─ Поллинг новых сообщений
                      └─ fetch GET .../web-poll/{slug}   ← WF 13_web_poll
                           └─ Новые сообщения (бот + оператор)
```

### Сессия и postMessage bridge

Iframe не может писать в `localStorage` напрямую (cross-origin блокировка Chrome). Реализован postMessage bridge:

```
[Iframe (n8n57362.hostkey.in)]              [Parent page (сайт клиента)]
         |                                              |
         |--- {type:'asw-init', prefix:'asw_bot1_'} -->|  (при загрузке)
         |<-- {type:'asw-init-data', data:{...}}  -----|  (ответ с данными из localStorage)
         |                                              |
         |--- {type:'asw-set', key, value}        ---->|  (при каждом изменении)
         |                                              |  localStorage.setItem(key, value)
```

**Ключи в localStorage родителя**: `asw_{slug}_sid` (Session ID), `asw_{slug}_msgs` (JSON сообщений), `asw_{slug}_contact`, `asw_{slug}_name/phone/email`, `asw_{slug}_opActive`, `asw_{slug}_lastPollId`, `asw_{slug}_cardDismissed`, `asw_{slug}_firstSent`.

**ВАЖНО: origin = "null"**: iframe с n8n webhook отправляет postMessage с `origin: "null"` (строка). Проверка origin не работает — проверяется структура сообщения (ключ начинается с `asw_`). targetOrigin = `'*'`.

### Приветственное сообщение и форма контактов

- Чат показывается сразу при открытии (нет блокирующего overlay)
- Приветственное сообщение из `widget_config.greeting`
- Форма контактов — inline-карточка внутри чата (заголовок "Представьтесь в чате", подзаголовок "Необязательно")
- Карточка исчезает при: "Пропустить", заполнении, или отправке первого сообщения

### WF 09 — widget_loader

Отдаёт JS. Читает `widget_config` + `v_auto_services` по slug. Проверяет подписку и план (`start` → пустой комментарий). Генерирует floating-кнопку, iframe-контейнер, lazy-load, адаптив, postMessage bridge listener.

### WF 10 — widget_frame

Отдаёт HTML-страницу чата (внутри iframe). CSS-переменные из конфига. In-memory `_cache` + postMessage bridge (sGet/sSet/initFromParent). Батчинг сообщений (15 сек таймер). Typing indicator, timestamps, аватарки. Поллинг через 13_web_poll для получения ответов оператора.

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
**Response**: `{"response": "Ответ ИИ"}` или `{"operator": true}` (если оператор активен)

### Конфигурация виджета (БД)

Таблица `widget_config` (PostgreSQL), PK = `service_id`.

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
| Валидация | UUID sessionId, message <= 4000, форматы phone/email/name |
| Фильтрация poll | source != 'system', tool_calls excluded, "Calling %" excluded |

---

## Система эскалации и оператор-мост

Две связанные подсистемы, добавленные после 2026-03-26:

### Эскалация (15_escalation + 03 call back)

1. LLM вызывает Escalation Tool → 15_escalation
2. Проверяется `escalation_enabled` для сервиса
3. Запись в `escalation_log` (level=handoff, category=human_request/urgent)
4. Владелец получает Telegram-сообщение с контекстом + inline-кнопки
5. Владелец нажимает кнопку → 03 call back обрабатывает:
   - "Принять" → отмечает в логе, бот продолжает
   - "Взять на себя" → создаёт `operator_session`, отправляет историю чата владельцу, уведомляет клиента
   - "Отклонить" → отмечает в логе

### Оператор-мост (12_operator_bridge)

После "Взять на себя":
1. Все сообщения клиента (Telegram/Web) перехватываются в 01 recieve / 01 recieve web
2. Перенаправляются владельцу через 12_operator_bridge
3. Владелец отвечает → сообщение доставляется клиенту
4. Владелец пишет `/close` → сессия закрывается, AI-бот возобновляет работу
5. Системные сообщения ("[Оператор подключился]", "[Оператор отключился]") сохраняются с `source: 'system'` и скрыты от клиента в поллинге

**Команды владельца**: `/find`, `/close`, `/list`, `/switch`, `/help`

---

## Лендинг "Записали"

- **Production**: https://makslap18.github.io/zapisali-landing/
- **Репозиторий**: https://github.com/makslap18/zapisali-landing
- **Хостинг**: GitHub Pages (статический HTML)
- **Структура**: один файл `index.html` + `favicon.svg`, CSS и JS инлайновые
- **PostMessage bridge**: inline `<script>` рядом с iframe (id="asw-frame") для сохранения сессии виджета

### Секции страницы

1. Навигация — логотип "Записали", якорные ссылки, CTA
2. Hero — заголовок, статистика (24/7, <2 мин ответ, 14 дней бесплатно)
3. Проблема/Решение — боли клиента → что делает бот
4. Демо — iframe с живым виджетом (bot1, АвтоМастер Центр)
5. Возможности — 6 карточек
6. Как это работает — 3 шага
7. Тарифы — Start (3990 руб), Business (6990 руб), Premium (11990 руб)
8. Калькулятор экономии
9. FAQ — аккордеон
10. CTA + Форма заявки (→ 14_lead_form)
11. Footer

---

## Архитектурные паттерны

| Паттерн | Описание |
|---------|----------|
| Service lookup | `v_auto_services` view по `bot_id` (slug) |
| Client anonymization | Intercept PD → Save PD → Anonymize (алиасы для LLM) |
| Chat memory key | `alias_telegram` для TG, `'web_' + sessionId` для Web, `'avito_user_' + author_id` для Avito |
| Channel routing (02 brain) | Switch Channel: web → Code return / avito → HTTP / default → Telegram HTTP |
| Channel routing (04_booking) | IF Channel Web: web → auto-confirm / telegram → keyboard |
| Message batching | Save to Buffer → Wait → Check If Latest → Collect → Combine |
| Message limit | Check Message Limit (PG, message_usage) → IF Limit Reached → block/allow |
| Plan check (booking) | IF Plan Allows Booking → Switch Action / No Booking Response |
| Owner notifications | На confirm: Notify Owner через 05 WF. На cancel: прямо в 04 |
| Operator bridge routing | Check Op Sessions → IF Op Mode → Call Op Bridge / продолжить в AI |
| Escalation flow | LLM tool → 15_escalation → TG кнопки → 03 callback → create session → 12_bridge |
| Message source tagging | operator/system/ai — для фильтрации в поллинге |
| PostMessage bridge | iframe ↔ parent localStorage через postMessage (cross-origin workaround) |
| WF inter-call | Execute Workflow node (passthrough mode) |

---

## Аудит безопасности (2026-03-06)

### Исправлено

| # | Уровень | Проблема | Решение |
|---|---------|----------|---------|
| 1 | КРИТ | SQL-инъекции в WF01, WF03 (9 нод) | Все на parameterized queries |
| 2 | КРИТ | `/webhook/onboarding` без auth | WF деактивирован |
| 3 | КРИТ | WF03 `Has Pending Booking?` — strict type error | Оператор string isNotEmpty |
| 4 | СРЕД | Dead end при subscription=false | Ноды "Сервис недоступен" |
| 5 | СРЕД | Пустой pd_consent_log (152-ФЗ) | Log PD Consent в WF03 |
| 6 | СРЕД | Ошибки валидации в WF04 | Исправлены типы и операторы |
| 7 | СРЕД | ~15 HTTP нод без retry | retryOnFail: true, maxTries: 3 |
| 8 | СРЕД | Нет errorWorkflow на виджетах | errorWorkflow = 06_error |
| 9 | СРЕД | Нет индексов БД | 5 индексов добавлены |

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
| АвтоКультура (id=8) | — | svc_8 |

Admin: `makslap18@gmail.com` (super роль)

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

## Открытые вопросы

1. **07_onboarding_bot** — требует Header Auth для активации.
2. **Дублирование номера 07** — и onboarding, и reminders.
3. **NocoDB аккаунты владельцев** — базы созданы, аккаунты не приглашены.
4. **АвтоКультура (id=8)** — NocoDB база не создана.
5. **Ganin Garage (id=5)** — NocoDB база не создана, план `start`.
6. **Авито** — у bot1 нет платной подписки API мессенджера (HTTP 402).
7. **14_lead_form** — нет валидации входных данных и rate-limiting.
8. **Экспериментальные WF** — `06_daily_stats_aggregator`, `08_pd_consent_flow`, `09_health_check` и другие неактивные — очистить или довести до production.

---

## Известные баги виджета

**1. Вебхуки 404 (баг n8n #21614)**: После обновления WF через API/MCP: деактивация → активация → `docker compose restart n8n`.

**2. Короткие URL не работают**: n8n 2.x требует полный URL с webhookId.

**3. Draft vs Published**: MCP/API обновляет draft. Публикация: деактивация → активация → рестарт.
