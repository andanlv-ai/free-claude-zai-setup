# Free-Claude + Z.AI — Полное описание проекта

Документ описывает полную настройку среды, в которой `claude` (официальный CLI Anthropic) работает рядом с `claude-zai` (тот же CLI, но проксированный через [free-claude-code](https://github.com/Alishahryar1/free-claude-code) на модели Z.AI / GLM).

Дата: 2026-05-20
ОС: macOS (Darwin 25.4.0)
Пользователь: andrej

---

## 1. Архитектура

```
┌────────────────┐         OAuth (Anthropic)         ┌──────────────────────┐
│  claude        │ ────────────────────────────────▶ │  api.anthropic.com    │
└────────────────┘                                   └──────────────────────┘

┌────────────────┐    HTTP    ┌──────────────────┐   HTTPS   ┌──────────────┐
│  claude-zai    │ ─────────▶ │  fcc-server      │ ────────▶ │  api.z.ai    │
│  (обёртка)     │  :8082     │  (free-claude)   │           │  (GLM)       │
└────────────────┘            └──────────────────┘           └──────────────┘
                                       ▲
                                       │ launchd (RunAtLoad + KeepAlive)
                                       │
                              com.fcc-server.zai.plist
```

`claude-zai` снимает OAuth-переменные окружения и подменяет `ANTHROPIC_BASE_URL`
на локальный прокси, который транслирует Anthropic-протокол в Z.AI.

---

## 2. Команды и модели

| Команда      | Backend              | Модель                              | Авторизация              |
|--------------|----------------------|-------------------------------------|--------------------------|
| `claude`     | Anthropic API        | Claude Opus/Sonnet/Haiku            | OAuth (macOS Keychain)   |
| `claude-zai` | Z.AI через fcc-server| `glm-5.1` (Opus/Sonnet), `glm-4.5-air` (Haiku/default) | `ANTHROPIC_API_KEY=freecc` (прокси) |

`claude` в shell — это alias: `claude --allow-dangerously-skip-permissions`.

### Маппинг моделей (фиксированный в `.env`)

| Tier Claude  | Z.AI модель      | Thinking |
|--------------|------------------|----------|
| `MODEL_OPUS`   | `zai/glm-5.1`      | on       |
| `MODEL_SONNET` | `zai/glm-5.1`      | on       |
| `MODEL_HAIKU`  | `zai/glm-4.5-air`  | —        |
| `MODEL` (fallback) | `zai/glm-4.5-air` | — |

---

## 3. Файлы и расположения

| Путь                                                          | Назначение                                |
|---------------------------------------------------------------|-------------------------------------------|
| `/Users/andrej/projects/freeClaude/`                          | Этот репозиторий с инструкцией            |
| `/Users/andrej/projects/freeClaude/README.md`                 | Краткая инструкция установки              |
| `/Users/andrej/projects/freeClaude/prompt-zai.md`             | Исходный prompt для повторной настройки   |
| `/Users/andrej/projects/freeClaude/PROJECT_OVERVIEW.md`       | Этот документ                             |
| `/Users/andrej/projects/free-claude-code/`                    | Клон upstream-репозитория free-claude-code|
| `/Users/andrej/projects/free-claude-code/.env`                | Конфиг провайдеров и моделей              |
| `/Users/andrej/.local/bin/claude-zai`                         | Обёртка-launcher                          |
| `/Users/andrej/.local/bin/fcc-server`                         | Бинарь uv tool (free-claude-code server)  |
| `/Users/andrej/Library/LaunchAgents/com.fcc-server.zai.plist` | LaunchAgent для автозапуска               |
| `/tmp/fcc-server.log`                                         | Логи сервера (stdout + stderr)            |
| `/Users/andrej/.claude/settings.json`                         | Глобальные настройки Claude Code          |

---

## 4. Содержимое ключевых файлов

### 4.1. `~/.local/bin/claude-zai`

```bash
#!/usr/bin/env bash
set -e
curl -sf http://127.0.0.1:8082/health >/dev/null || {
  echo "fcc-server не запущен. Запусти: FCC_OPEN_BROWSER=false FCC_ENV_FILE=~/projects/free-claude-code/.env fcc-server --port 8082 &"
  exit 1
}
unset ANTHROPIC_API_KEY ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL
export ANTHROPIC_API_KEY=freecc ANTHROPIC_BASE_URL=http://127.0.0.1:8082
exec claude --bare "$@"
```

Ключевые моменты:
- `unset` чистит OAuth-токены, которые Claude CLI иначе подсунет в заголовок.
- `--bare` отключает интеграцию с Claude.ai (нужна для прокси).
- Health-check выполняется до старта, чтобы получить понятную ошибку.

### 4.2. `~/Library/LaunchAgents/com.fcc-server.zai.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key><string>com.fcc-server.zai</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/andrej/.local/bin/fcc-server</string>
        <string>--port</string>
        <string>8082</string>
    </array>
    <key>EnvironmentVariables</key>
    <dict>
        <key>FCC_ENV_FILE</key><string>/Users/andrej/projects/free-claude-code/.env</string>
        <key>FCC_OPEN_BROWSER</key><string>false</string>
    </dict>
    <key>RunAtLoad</key><true/>
    <key>KeepAlive</key><true/>
    <key>StandardOutPath</key><string>/tmp/fcc-server.log</string>
    <key>StandardErrorPath</key><string>/tmp/fcc-server.log</string>
</dict>
</plist>
```

Статус: загружен (`launchctl list | grep fcc-server` → активен, PID присвоен).

### 4.3. `~/projects/free-claude-code/.env` (ключевые поля, секреты скрыты)

```env
# Z.AI — единственный активный провайдер
ZAI_API_KEY="***REDACTED***"     # 5ae6...4c3d (pay-per-token, GLM Coding Plan не нужен)

# Маппинг моделей: provider/model
MODEL_OPUS=zai/glm-5.1
MODEL_SONNET=zai/glm-5.1
MODEL_HAIKU=zai/glm-4.5-air
MODEL="zai/glm-4.5-air"

# Thinking / reasoning
ENABLE_OPUS_THINKING=true
ENABLE_SONNET_THINKING=true
ENABLE_HAIKU_THINKING=
ENABLE_MODEL_THINKING=true

# Локальный API-токен для входа в fcc-server
ANTHROPIC_AUTH_TOKEN="freecc"

# Открывать /admin в браузере при старте — выключено (для launchd)
FCC_OPEN_BROWSER=true   # переопределяется plist'ом → false

# Прочее
MESSAGING_PLATFORM="none"
VOICE_NOTE_ENABLED=false
WHISPER_DEVICE="cpu"
WHISPER_MODEL="openai/whisper-large-v3"

# HTTP timeouts
HTTP_READ_TIMEOUT=300
HTTP_WRITE_TIMEOUT=60
HTTP_CONNECT_TIMEOUT=60

# Rate-limit и concurrency прокси
PROVIDER_RATE_LIMIT=1
PROVIDER_RATE_WINDOW=3
PROVIDER_MAX_CONCURRENCY=5

# Логи — без сырого payload
LOG_RAW_API_PAYLOADS=false
LOG_RAW_SSE_EVENTS=false
LOG_API_ERROR_TRACEBACKS=false
```

Все остальные провайдеры в `.env` присутствуют как заглушки с пустыми ключами:
NVIDIA NIM, OpenRouter, DeepSeek, Kimi, Wafer, OpenCode, LM Studio, Llama.cpp, Ollama.
Это нормально — `fcc-server` использует только тот провайдер, чья модель указана
в `MODEL_*`.

---

## 5. Жизненный цикл

### Старт системы
1. `launchd` поднимает `fcc-server` (RunAtLoad).
2. Сервер читает `FCC_ENV_FILE` → загружает `.env`.
3. Слушает `127.0.0.1:8082`. Health → `{"status":"healthy"}`.

### Запуск `claude-zai`
1. Обёртка проверяет `/health`.
2. Чистит OAuth-переменные окружения.
3. Запускает `claude --bare` с `ANTHROPIC_BASE_URL=http://127.0.0.1:8082`.
4. Claude CLI шлёт Anthropic-формат запросы в прокси.
5. Прокси переводит их в Z.AI Anthropic-compatible endpoint (`api.z.ai/api/anthropic`).

### Падение сервера
`KeepAlive=true` — `launchd` рестартует автоматически.

---

## 6. Управление

```bash
# Статус
launchctl list | grep fcc-server
curl -s http://127.0.0.1:8082/health

# Остановить
launchctl unload ~/Library/LaunchAgents/com.fcc-server.zai.plist

# Запустить
launchctl load ~/Library/LaunchAgents/com.fcc-server.zai.plist

# Логи в реальном времени
tail -f /tmp/fcc-server.log

# Admin UI (если FCC_OPEN_BROWSER=true)
open http://127.0.0.1:8082/admin
```

---

## 7. Подводные камни (зафиксированы при настройке)

1. **`fcc-claude` не использовать.** Эта обёртка пытается читать OAuth-токен из
   macOS Keychain и перебивает `ANTHROPIC_API_KEY=freecc` → 401. Только
   `claude-zai` с явным `unset`.
2. **`--env-file` у `fcc-server` нет.** Путь к `.env` передаётся через
   переменную окружения `FCC_ENV_FILE`.
3. **Подписка GLM Coding Plan не обязательна.** Обычного pay-per-token ключа Z.AI
   достаточно.
4. **`--bare`** — обязательный флаг при запуске через прокси, иначе CLI
   попытается синхронизироваться с Claude.ai.
5. **`FCC_OPEN_BROWSER=false`** в plist — иначе при каждом старте launchd будет
   пытаться открыть браузер.

---

## 8. Текущее состояние (проверено)

| Проверка                          | Результат          |
|-----------------------------------|--------------------|
| `launchctl list \| grep fcc-server` | PID 34725          |
| `curl http://127.0.0.1:8082/health` | `{"status":"healthy"}` |
| `which claude-zai`                  | `/Users/andrej/.local/bin/claude-zai` |
| `which fcc-server`                  | `/Users/andrej/.local/bin/fcc-server` |
| Git remote                          | `github.com/andanlv-ai/free-claude-zai-setup.git` |

---

## 9. Ссылки

- Upstream: <https://github.com/Alishahryar1/free-claude-code>
- Z.AI docs: <https://docs.z.ai/devpack/tool/claude>
- Z.AI Anthropic endpoint: `https://api.z.ai/api/anthropic`
- Этот репозиторий: <https://github.com/andanlv-ai/free-claude-zai-setup>
