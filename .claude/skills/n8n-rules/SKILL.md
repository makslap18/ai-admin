---
name: n8n-rules
description: Правила редактирования n8n workflows — updateNode, webhookId, errorWorkflow, retry
allowed-tools: Bash, Read, Grep, Glob
---

# Правила редактирования n8n workflows

## updateNode через API
При обновлении ноды (`updateNode`) — передавать ВСЕ параметры ноды, не только изменённые. API заменяет `parameters` целиком — пропущенные поля будут удалены.

## webhookId
НИКОГДА не менять `webhookId` существующих webhook-нод. Это сломает все интеграции (Telegram, Авито, виджет).

## errorWorkflow
`errorWorkflow = "3muBfH7DpVipSOpN"` (06_error) — подключать ко ВСЕМ новым workflow.

## HTTP-ноды
На всех HTTP Request нодах:
- `retryOnFail: true`
- `maxTries: 3`

## Каналы
Telegram, Web, Avito. Роутинг по полю `channel` в 02 brain.
При добавлении нового канала — добавить ветку в Switch Channel.

## Порядок обновления WF
1. `n8n_get_workflow` — прочитать текущее состояние
2. Внести изменения
3. Деактивировать WF
4. Активировать WF
5. `docker compose restart n8n` (webhook баг)
