---
name: db-rules
description: Правила работы с PostgreSQL/pgvector — схемы, индексы, constraints, мультитенантность
allowed-tools: Bash, Read, Grep, Glob
---

# Правила работы с БД

## Мультитенантность
- Каждый сервис = отдельная schema `service_N` с views на public-таблицы.
- Новые таблицы: создавать в `public` с колонкой `service_id`, затем views в schema каждого сервиса.
- ВСЕГДА фильтровать по `service_id`. Запрос без фильтра = баг.

## SQL-безопасность
- ТОЛЬКО параметризованные запросы (`$1`, `$2`).
- НИКАКОЙ string interpolation. Это было исправлено в аудите — не регрессировать.

## Индексы
Обязательные индексы на часто запрашиваемые поля:
- `service_id`, `session_id`, `status`, `date`

## Constraints
- **EXCLUDE constraint на appointments** — НЕ ТРОГАТЬ. Предотвращает пересечение слотов записи.
- `ON CONFLICT` — использовать partial indexes где уместно.
  - Пример (avito): `WHERE real_avito_id IS NOT NULL`

## Подключение
```bash
sudo -u postgres psql -d n8n
```
