---
name: widget-rules
description: Правила разработки веб-виджета — XSS, CSP, rate limit, сессии
allowed-tools: Bash, Read, Grep, Glob
---

# Правила разработки виджета

## XSS-защита (обязательно)
- `esc()` для экранирования HTML
- `textContent` вместо `innerHTML`
- НИКОГДА не использовать `eval()` или `innerHTML` с пользовательским вводом

## CSP заголовки
- `frame-ancestors` — динамический из `allowed_origins`

## Rate limiting
- 10 msg/min по `session_id` (уровень workflow)
- 20 req/min по IP (Traefik)

## Сессии
- `localStorage` с ключами `asw_{slug}_*`
- НЕ sessionStorage — теряется при обновлении страницы

## Тестирование
Всегда проверять изменения виджета через curl или реальный браузер. Не предполагать, что работает.
