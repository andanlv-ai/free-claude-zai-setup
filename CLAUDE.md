# Project: free-claude-zai-setup

Инструкции по запуску `claude` + Z.AI + DeepSeek параллельно. Эта ветка — **macOS**. Для Windows см. ветку `windows`.

## Текущее состояние (macOS, 2026-05-20)

**Все три провайдера через прямые native Anthropic endpoint'ы. fcc-server удалён.**

| Команда | Endpoint | Модели |
|---------|----------|--------|
| `claude` | OAuth Anthropic | Claude 4.x |
| `claude-zai` | `https://api.z.ai/api/anthropic` | OPUS/SONNET=`glm-5.1`, HAIKU=`glm-4.5-air` |
| `claude-ds` | `https://api.deepseek.com/anthropic` | OPUS/SONNET=`deepseek-v4-pro`, HAIKU=`deepseek-v4-flash` |

Обёртки: `~/.local/bin/claude-{zai,ds}`. Каждая делает `unset ANTHROPIC_API_KEY ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL`, ставит `ANTHROPIC_AUTH_TOKEN` + `ANTHROPIC_BASE_URL` + маппинг моделей + `API_TIMEOUT_MS=3000000`, затем `exec claude --bare "$@"`.

## Жёсткие правила

- **Не использовать fcc-server.** Конвертирует Anthropic→OpenAI, ломает `tools[]`/`output_config`/MCP на `claude-cli/2.1.145+`: Z.AI отвечал `Provider API request failed`, DeepSeek — `Invalid request sent to provider`. Native endpoint провайдера принимает запросы напрямую.
- **`fcc-claude` не использовать** — OAuth из Keychain перебивает токен → 401. Только `claude --bare` через обёртку.
- **В обёртке очищать `ANTHROPIC_API_KEY` / `ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_MODEL`** перед установкой нового токена.
- **`ANTHROPIC_AUTH_TOKEN`, не `API_KEY`** — оба провайдера требуют именно `AUTH_TOKEN`.
- **Подписки не нужны** — pay-per-token ключи Z.AI и DeepSeek работают с native Anthropic endpoint.

## Удалено (если кто-то будет восстанавливать — не надо)

- `~/Library/LaunchAgents/com.fcc-server.zai.plist`
- `~/.fcc/` (managed config fcc-server)
- `~/projects/free-claude-code/` (клон upstream)
- `uv tool free-claude-code` (бинари `fcc-server`, `fcc-claude`, `fcc-init`)

## Отладка

- `ANTHROPIC_LOG=debug claude-zai -p "..."` / `claude-ds -p "..."` — лог запросов и реальный endpoint.
- Прямой ping провайдера: `curl https://api.deepseek.com/anthropic/v1/messages -H "x-api-key: $KEY" -H "anthropic-version: 2023-06-01" -d '{"model":"deepseek-v4-flash","max_tokens":30,"messages":[{"role":"user","content":"hi"}]}'`.

## Не редактировать README в кросс-платформенном стиле

Эта ветка — только macOS. Windows-инструкция живёт в ветке `windows`.
