# Widget Update Status - 2026-04-06

## Задачи

### 1. Сохранение сессии при обновлении страницы
**Проблема:** Виджет использовал `sessionStorage`. При обновлении страницы iframe пересоздаётся, и `sessionStorage` сбрасывается. Пользователь теряет всю переписку и session ID.

**Решение (в `temp_wf10_jsCode.js`):**
- `sessionStorage` заменён на in-memory `_cache` + postMessage bridge к родительской странице
- Iframe не может писать в `localStorage` напрямую (cross-origin блокировка Chrome)
- Вместо этого iframe шлёт `postMessage` родительской странице, которая сохраняет данные в свой `localStorage`
- При загрузке iframe запрашивает данные у родителя через `postMessage({type:'asw-init'})` и получает ответ `{type:'asw-init-data', data:{...}}`
- История сообщений, session ID, состояние оператора восстанавливаются через bridge
- Поллинг возобновляется автоматически

**Архитектура postMessage bridge:**
```
[Iframe (n8n57362.hostkey.in)]              [Parent page (makslap18.github.io)]
         |                                              |
         |--- {type:'asw-init', prefix:'asw_bot1_'} -->|  (при загрузке)
         |<-- {type:'asw-init-data', data:{...}}  -----|  (ответ с данными из localStorage)
         |                                              |
         |--- {type:'asw-set', key, value}        ---->|  (при каждом изменении)
         |                                              |  localStorage.setItem(key, value)
```

**Код bridge на стороне iframe (WF 10 Generate HTML):**
```javascript
var _cache = {};
var _parentOk = (window.parent && window.parent !== window);
var KEY_PFX = 'asw_' + slug + '_';

function sGet(k) { return _cache[k] !== undefined ? _cache[k] : null; }
function sSet(k, v) {
  _cache[k] = v;
  if (_parentOk) {
    try { window.parent.postMessage({type:'asw-set', key: KEY_PFX + k, value: v}, '*'); } catch(e){}
  }
}
function initFromParent(cb) {
  if (!_parentOk) { cb(); return; }
  var timeout = setTimeout(function() { window.removeEventListener('message', handler); cb(); }, 500);
  function handler(e) {
    if (e.data && e.data.type === 'asw-init-data') {
      clearTimeout(timeout);
      window.removeEventListener('message', handler);
      var d = e.data.data || {};
      for (var key in d) {
        if (d.hasOwnProperty(key) && key.indexOf(KEY_PFX) === 0) {
          _cache[key.substring(KEY_PFX.length)] = d[key];
        }
      }
      cb();
    }
  }
  window.addEventListener('message', handler);
  window.parent.postMessage({type:'asw-init', prefix: KEY_PFX}, '*');
}
```

**Код bridge на стороне родителя (два варианта):**

1. **Лендинг (zapisali-landing/index.html)** — inline `<script>` рядом с iframe:
```javascript
window.addEventListener('message', function(e) {
  if (!e.data || typeof e.data !== 'object') return;
  if (e.data.type === 'asw-set' && e.data.key && typeof e.data.key === 'string' && e.data.key.indexOf('asw_') === 0) {
    try { localStorage.setItem(e.data.key, e.data.value); } catch(x) {}
    return;
  }
  if (e.data.type === 'asw-init' && e.data.prefix) {
    var pfx = e.data.prefix; var res = {};
    try {
      for (var i = 0; i < localStorage.length; i++) {
        var k = localStorage.key(i);
        if (k.indexOf(pfx) === 0) res[k] = localStorage.getItem(k);
      }
    } catch(x) {}
    var fr = document.getElementById('asw-frame');
    if (fr && fr.contentWindow) {
      fr.contentWindow.postMessage({type:'asw-init-data', data: res}, '*');
    }
    return;
  }
});
```

2. **Клиентские сайты (WF 09 widget_loader)** — встроен в генерируемый JS загрузчика

**ВАЖНО: origin = "null"**
Iframe, загруженный с n8n webhook, отправляет postMessage с `origin: "null"` (строка "null"), а НЕ `"https://n8n57362.hostkey.in"`. Поэтому проверка origin не работает. Вместо origin проверяется структура сообщения (ключ начинается с `asw_`). targetOrigin при ответе тоже `'*'`.

**Затронутые переменные в localStorage родителя:**
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
| `asw_{slug}_firstSent` | Первое сообщение отправлено ('1') |

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

**Текст приветствия:** берётся из `widget_config.greeting` в БД. По умолчанию: "Здравствуйте! Чем могу помочь?". Для АвтоМастер Центр: "Добрый день! Поддержка АвтоМастер Центр, мы на связи. Чем можем вам помочь?"

---

### 3. Фильтрация сообщений — только ответы оператора
**Проблема:** Поллинг (WF 13) забирал ВСЕ сообщения с `type='ai'` из `n8n_chat_histories8`. Сюда попадали:
- Ответы оператора
- "[Оператор подключился к диалогу]"
- "[Оператор отключился. AI-бот снова отвечает.]"
- Tool call сообщения ("Calling Escalation_Tool with input:...")

Системные и tool call сообщения показывались клиенту на сайте как обычные сообщения.

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

`Get New Messages` — добавлены фильтры:
```sql
SELECT id, message->>'content' as content, message->>'type' as type
FROM n8n_chat_histories8
WHERE session_id = 'web_' || $1 AND id > $2
  AND message->>'type' = 'ai'
  AND message->>'source' IS DISTINCT FROM 'system'
  AND (message->'tool_calls' IS NULL OR message->'tool_calls' = '[]'::jsonb)
  AND message->>'content' NOT LIKE 'Calling %'
ORDER BY id ASC LIMIT 20
```

Обратная совместимость: старые сообщения без поля `source` продолжают показываться (`IS DISTINCT FROM` пропускает NULL).

---

### 4. Валидация контактов — сделана необязательной (WF 01)
**Проблема:** `Validate Request` в WF 01 требовал имя/телефон при `isFirstMessage: true`. Если пользователь пропускал форму контактов, отправка сообщения возвращала ошибку "Invalid name".

**Решение (ПРИМЕНЕНО в n8n):**

**WF 01 — `01 recieve web` (нода `Validate Request`):**
```javascript
const hasContactData = !!(body.name && body.name.trim()) || !!(body.phone && body.phone.trim());
if (body.isFirstMessage && hasContactData) {
  // Валидация только если контакты реально переданы
  const name = (body.name || '').trim();
  if (name && (name.length < 2 || name.length > 50)) { return [{ json: { error: true, status: 400, message: 'Invalid name' } }]; }
  const phone = (body.phone || '').replace(/\D/g, '');
  if (phone && phone.length < 5) { return [{ json: { error: true, status: 400, message: 'Invalid phone' } }]; }
  if (!body.pdConsent) { return [{ json: { error: true, status: 400, message: 'PD consent required' } }]; }
}
```

Также добавлен флаг `firstSent` в iframe коде — первое сообщение всегда отправляется с `isFirstMessage: true`, чтобы гарантировать создание/upsert клиента в БД.

---

## Статус применения

| Воркфлоу | Изменение | Статус |
|----------|-----------|--------|
| WF 01 `01 recieve web` | Валидация контактов необязательна + firstSent | ПРИМЕНЕНО |
| WF 09 `09_widget_loader` | postMessage bridge с проверкой структуры (без origin) | ПРИМЕНЕНО |
| WF 10 `10_widget_frame` | postMessage bridge (sGet/sSet/initFromParent) | ПРИМЕНЕНО |
| WF 12 `12_operator_bridge` | 3 SQL-запроса с source метками | ПРИМЕНЕНО |
| WF 13 `13_web_poll` | Фильтр source + tool_calls + "Calling %" | ПРИМЕНЕНО |
| Лендинг `zapisali-landing` | id="asw-frame" + postMessage bridge скрипт | ПРИМЕНЕНО (GitHub Pages) |

---

## Найденные и исправленные баги (хронология)

### Баг 1: "Извините, произошла ошибка" при отправке сообщения
- **Причина:** Новый sessionId не в БД, `isFirstMessage: false` (нет контактов) → `Lookup Client` = 0 items → timeout
- **Фикс:** Добавлен `firstSent` флаг — первое сообщение всегда `isFirstMessage: true`

### Баг 2: "Invalid name" от Validate Request
- **Причина:** `isFirstMessage: true` триггерил валидацию имени, но пользователь пропустил форму → пустое имя
- **Фикс:** Валидация условная — только если `hasContactData` присутствует

### Баг 3: "Calling Escalation_Tool with input:..." видно клиенту
- **Причина:** AI tool call сообщения хранятся с `type='ai'` и попадали в поллинг
- **Фикс:** SQL фильтры в WF 13: `tool_calls IS NULL OR = '[]'` и `content NOT LIKE 'Calling %'`

### Баг 4: Сессия не сохраняется при обновлении страницы
- **Причина 1:** Cross-origin iframe не может писать в localStorage напрямую (Chrome блокирует third-party storage)
- **Фикс 1:** PostMessage bridge — iframe общается с родителем через postMessage, родитель хранит в своём localStorage
- **Причина 2:** На лендинге iframe встроен напрямую (`<iframe src="...">`) без загрузчика WF 09, поэтому postMessage listener отсутствовал
- **Фикс 2:** Добавлены `id="asw-frame"` на iframe и inline `<script>` с bridge listener на лендинг
- **Причина 3:** iframe отправляет postMessage с `origin: "null"` (не URL), а listener фильтровал по `origin !== 'https://n8n57362.hostkey.in'`
- **Фикс 3:** Заменена проверка origin на проверку структуры сообщения (`e.data.key.indexOf('asw_') === 0`) и targetOrigin на `'*'`

---

## Файлы

| Файл | Описание |
|------|----------|
| `temp_wf10_jsCode.js` | jsCode для Generate HTML (WF 10) |
| `temp_wf10_update.json` | JSON payload для REST API обновления |
| `WIDGET_UPDATE_STATUS.md` | Этот документ |
| `zapisali-landing/index.html` | Лендинг с postMessage bridge (строка ~1184) |

## Репозитории

| Репо | URL |
|------|-----|
| Лендинг | https://github.com/makslap18/zapisali-landing |
| GitHub Pages | https://makslap18.github.io/zapisali-landing/ |
