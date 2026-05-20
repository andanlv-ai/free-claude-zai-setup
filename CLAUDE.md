# Project: free-claude-zai-setup

Инструкции по установке free-claude-code + Z.AI + DeepSeek параллельно с обычным claude CLI. README покрывает macOS и Windows 11.

## Команды (macOS, актуально)

| Команда | Провайдер |
|---------|-----------|
| `claude` | Anthropic OAuth |
| `claude-zai` | Z.AI (GLM-5.1), прямое подключение |
| `claude-ds` | DeepSeek через fcc-server (:8082) |

## DeepSeek / fcc-server (macOS, 2026-05-20)

**Managed config:** `~/.fcc/.env` — именно здесь, проектный `.env` игнорируется.

**Запуск fcc-server** (после перезагрузки или если упал):
```bash
cd /Users/andrej/projects/free-claude-code
nohup fcc-server > /tmp/fcc-server-ds.log 2>&1 &
# проверка:
curl http://127.0.0.1:8082/health
```

**Ключевые настройки в `~/.fcc/.env`:**
- `DEEPSEEK_API_KEY=sk-f9c532187ff04c89adf19810c8e67835`
- `MODEL=deepseek/deepseek-v4-flash`, `MODEL_OPUS/SONNET=deepseek/deepseek-v4-pro`
- `ENABLE_MODEL_THINKING=false` (и все THINKING=false) — DeepSeek не поддерживает Anthropic thinking, иначе 500 на `?beta=true`
- `MESSAGING_PLATFORM=none`

**Обёртка** `~/.local/bin/claude-ds`: сбрасывает OAuth токены, `ANTHROPIC_API_KEY=freecc`, `ANTHROPIC_BASE_URL=http://127.0.0.1:8082`, запускает `claude --bare`.

## Текущее состояние (Windows, 2026-05-20)

**Подключение прямое к Z.AI, без прокси.** fcc-server отключён — он конвертирует Anthropic→OpenAI и теряет structured-outputs/tool-use fidelity на `claude-cli/2.1.145`+, что вызывает `Provider API request failed` на запросах с `output_config`/`tools` (особенно с не-латинским prompt).

- Обёртка: `C:\Users\user\.local\bin\claude-zai.ps1` + `claude-zai.cmd`
- Endpoint: `https://api.z.ai/api/anthropic` (native Anthropic API от Z.AI)
- Auth: `ANTHROPIC_AUTH_TOKEN` (не `API_KEY`!) — ключ pay-per-token
- Модели: OPUS/SONNET=`glm-5.1`, HAIKU=`glm-4.5-air`
- `API_TIMEOUT_MS=3000000` обязательно — GLM-5.1 «думает» 15-20с
- Claude CLI: `C:\Users\user\AppData\Roaming\npm\claude.ps1`, 2.1.145, запуск через `claude --bare`

## Жёсткие правила

- **Не использовать fcc-server для Z.AI** — даёт `Provider API request failed` на запросах с `output_config`/`tools[]`/`thinking` (`claude-cli/2.1.145`+). Прямой Anthropic-эндпоинт Z.AI работает корректно.
- **`fcc-claude` не использовать** — OAuth из системного credential store (Keychain/Credential Manager) перебивает токен и возвращает 401. Только `claude --bare` через обёртку.
- **В обёртке очищать `ANTHROPIC_API_KEY` и `ANTHROPIC_MODEL`** перед установкой `AUTH_TOKEN` — иначе OAuth из Credential Manager перебивает.
- **Подписка GLM Coding Plan не нужна** — pay-per-token ключ работает с native Anthropic endpoint.

## Устаревшее (не использовать)

- `fcc-server` на порту 8082 — отключён. Процесс убит, scheduled task `fcc-server-zai` остался (можно `schtasks /change /tn fcc-server-zai /disable` из admin).
- `.env` в `C:\Users\user\projects\free-claude-code\.env` — больше не читается. Все 4 thinking-флага в нём = false, но это уже не имеет значения.
- `LOG_RAW_API_PAYLOADS` / `LOG_RAW_SSE_EVENTS` — флаги fcc-server, не нужны.

## Модели Z.AI (актуальный маппинг)

- `glm-4.5-air` → MODEL (default) + MODEL_HAIKU
- `glm-5.1` → MODEL_SONNET + MODEL_OPUS

## Отладка

- `$env:ANTHROPIC_LOG='debug'; claude-zai -p "..."` — покажет полный лог запросов/ответов claude CLI.
- Admin endpoint `/admin/api/config` отдаёт текущую конфигурацию провайдеров и моделей с источником каждого поля (`explicit_env_file` / `template`).
- При `LOG_RAW_API_PAYLOADS=true` + `LOG_RAW_SSE_EVENTS=true` в `.env` логи стримов попадают в stdout сервера.

## Не редактировать README

README содержит ОБА раздела — macOS оригинальный + Windows 11 (добавлен 2026-05-20). При обновлении инструкции — править соответствующий раздел, второй не трогать.
