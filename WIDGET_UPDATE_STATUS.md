# Widget Update Status - 2026-04-06

## Задачи

### 1. Сохранение сессии при обновлении страницы
**Проблема:** Виджет использовал `sessionStorage`. При обновлении страницы iframe пересоздаётся, и `sessionStorage` сбрасывается. Пользователь теряет всю переписку и session ID.

**Решение (в `temp_wf10_jsCode.js`):**
- `sessionStorage` заменён на `localStorage` с ключами, привязанными к slug (`asw_${slug}_sid`, `asw_${slug}_msgs`, и т.д.)
- История сообщений сохраняется в `localStorage` (массив JSON, макс 100 сообщений)
- При загрузке iframe восстанавливаются: session ID, сообщения, состояние оператора, lastPollId
- Поллинг возобновляется автоматически

**Затронутые переменные в localStorage:**
| Ключ | Назначение |
|------|-----------|
| `asw_{slug}_sid` | Session ID (UUID) |
| `asw_{slug}_msgs` | JSON-массив сообщений [{text, sender, time}] |
| `asw_{slug}_contact` | Контакт заполнен ('1') |
| `asw_{slug}_name` | Имя пользователя |
| `asw_{slug}_phone` | Телефон |
| `asw_{slug}_email` | Email |
| `asw_{slug}_opActive` | Оператор активен ('1'/'0') |
| `asw_{slug}_lastPollId` | Последний ID опрошенного сообщения |
| `asw_{slug}_cardDismissed` | Карточка контактов закрыта ('1') |

---

### 2. Приветственное сообщение + добровольная форма контактов
**Проблема:** При открытии чата показывалась блокирующая форма (`contact-overlay`). Без заполнения формы чат недоступен. Нет приветственного сообщения.

**Решение (в `temp_wf10_jsCode.js`):**
- Убран блокирующий `contact-overlay`
- Чат (messages + input area) показывается сразу при открытии
- При первом открытии автоматически отображается приветственное сообщение из конфига (`greeting` поле в widget_config)
- Форма контактов показывается как inline-карточка внутри чата:
  - Заголовок "Представьтесь в чате"
  - Подзаголовок "Необязательно"
  - Поля имя/телефон/email (в зависимости от конфига)
  - Кнопка "Отправить" + кнопка "Пропустить"
- Карточка исчезает при: нажатии "Пропустить", заполнении и отправке, или отправке первого сообщения

**Текст приветствия:** берётся из `widget_config.greeting` в БД. По умолчанию: "Здравствуйте! Чем могу помочь?". Для АвтоМастер Центр нужно установить в NocoDB: "Добрый день! Поддержка АвтоМастер Центр, мы на связи. Чем можем вам помочь?"

---

### 3. Фильтрация сообщений — только ответы оператора
**Проблема:** Поллинг (WF 13) забирал ВСЕ сообщения с `type='ai'` из `n8n_chat_histories8`. Сюда попадали:
- Ответы оператора ✅
- "[Оператор подключился к диалогу]" ❌
- "[Оператор отключился. AI-бот снова отвечает.]" ❌

Системные сообщения показывались клиенту на сайте как обычные сообщения.

**Решение (ПРИМЕНЕНО в n8n):**

**WF 12 — `12_operator_bridge` (3 ноды изменены):**

1. `Save Op History` — добавлено `'source', 'operator'` в `jsonb_build_object`:
```sql
INSERT INTO n8n_chat_histories8 (session_id, message)
VALUES ($1, jsonb_build_object('type', 'ai', 'content', $2,
  'source', 'operator', ...));
```

2. `Save Connect to History` — добавлено `"source": "system"`:
```sql
INSERT INTO n8n_chat_histories8 (session_id, message)
VALUES ($1, '{"type": "ai", "content": "[Оператор подключился к диалогу]",
  "source": "system", ...}'::jsonb)
```

3. `Save End to History` — добавлено `"source": "system"`:
```sql
INSERT INTO n8n_chat_histories8 (session_id, message)
VALUES ($1, '{"type": "ai", "content": "[Оператор отключился. AI-бот снова отвечает.]",
  "source": "system", ...}'::jsonb)
```

**WF 13 — `13_web_poll` (1 нода изменена):**

`Get New Messages` — добавлен фильтр:
```sql
SELECT id, message->>'content' as content, message->>'type' as type
FROM n8n_chat_histories8
WHERE session_id = 'web_' || $1 AND id > $2
  AND message->>'type' = 'ai'
  AND message->>'source' IS DISTINCT FROM 'system'  -- NEW
ORDER BY id ASC LIMIT 20
```

Обратная совместимость: старые сообщения без поля `source` продолжают показываться (`IS DISTINCT FROM` пропускает NULL).

---

## Статус применения

| Воркфлоу | Изменение | Статус |
|----------|-----------|--------|
| WF 12 `12_operator_bridge` | 3 SQL-запроса с source метками | ✅ Применено (draft) |
| WF 13 `13_web_poll` | Фильтр source в poll запросе | ✅ Применено (draft) |
| WF 10 `10_widget_frame` | Новый jsCode (localStorage, greeting, inline form) | ❌ НЕ применено |

---

## Что нужно сделать

### Шаг 1: Применить jsCode к WF 10
Файл с готовым кодом: `C:\Users\GGL\Desktop\KOt ru max\temp_wf10_jsCode.js`

Нужно обновить ноду `Generate HTML` (id: `wf-generate`) в WF `UvFL82lc98eSpRiT`.

Через MCP:
```
n8n_update_partial_workflow
  id: UvFL82lc98eSpRiT
  operations: [{
    type: updateNode,
    nodeId: wf-generate,
    updates: { "parameters.jsCode": <содержимое temp_wf10_jsCode.js> }
  }]
```

### Шаг 2: Опубликовать все 3 воркфлоу
Согласно правилу из WIDGET_SUMMARY (баг n8n #21614):
```bash
# Деактивировать + активировать каждый WF
for WF_ID in UvFL82lc98eSpRiT OiD4Qw3qz5l1jbVG Rp34jGsWdC68ZqMs; do
  curl -X POST "https://n8n57362.hostkey.in/api/v1/workflows/$WF_ID/deactivate" -H 'X-N8N-API-KEY: ...'
  curl -X POST "https://n8n57362.hostkey.in/api/v1/workflows/$WF_ID/activate" -H 'X-N8N-API-KEY: ...'
done

# Рестарт n8n (ОБЯЗАТЕЛЬНО)
ssh sese "cd /root/n8n-compose-file && docker compose restart n8n"

# Подождать ~15 сек и проверить вебхуки
ssh sese "sudo docker cp n8n-compose-file-n8n-1:/root/.n8n/database.sqlite /tmp/n8n.sqlite && sqlite3 /tmp/n8n.sqlite 'SELECT webhookPath, method FROM webhook_entity;'"
```

### Шаг 3: Установить greeting для АвтоМастер Центр
В NocoDB → `widget_config` → строка service_id=2:
```
greeting = "Добрый день! Поддержка АвтоМастер Центр, мы на связи. Чем можем вам помочь?"
```

---

## Файлы

| Файл | Описание |
|------|----------|
| `temp_wf10_jsCode.js` | Готовый jsCode для Generate HTML (22KB) |
| `temp_wf10_update.json` | JSON payload для REST API обновления |
| `WIDGET_UPDATE_STATUS.md` | Этот документ |
