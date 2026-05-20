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

## 9. Версии используемого ПО

Снимок зафиксирован 2026-05-20 на рабочей машине. Для воспроизведения один-в-один
другому ИИ/инженеру достаточно поднять эти же версии.

### 9.1. Операционная система и железо

| Компонент       | Версия                                       |
|-----------------|----------------------------------------------|
| macOS           | 26.4.1 (build 25E253)                        |
| Kernel (Darwin) | 25.4.0 (xnu-12377.101.15~1 / RELEASE_ARM64_T8132) |
| Архитектура     | `arm64` (Apple Silicon)                      |
| Хост            | `Air-Andrej.fritz.box`                       |

### 9.2. Системные shells и утилиты

| Утилита   | Версия                                                          |
|-----------|-----------------------------------------------------------------|
| zsh       | 5.9 (arm64-apple-darwin25.0)                                    |
| bash      | 3.2.57(1)-release (arm64-apple-darwin25) — системный            |
| curl      | 8.7.1 (libcurl/8.7.1, SecureTransport, LibreSSL/3.3.6, nghttp2/1.68.0) |
| git       | 2.50.1 (Apple Git-155)                                          |
| Homebrew  | 5.1.12                                                          |
| Node.js   | v25.7.0                                                         |
| npm       | 11.14.1                                                         |

### 9.3. Python / uv

| Компонент              | Версия                                              |
|------------------------|-----------------------------------------------------|
| uv                     | 0.11.15 (3cffe97c2 2026-05-18, aarch64-apple-darwin)|
| Python (системный)     | 3.9.6                                               |
| Python (для fcc-server)| 3.14.0 (зафиксирован в `.python-version`)           |
| pyproject `requires-python` | `>=3.14`                                       |

### 9.4. Claude Code CLI

| Компонент                  | Версия                                                |
|----------------------------|-------------------------------------------------------|
| `@anthropic-ai/claude-code` | 2.1.144 (установлен глобально через npm)             |
| `claude --version`         | `2.1.144 (Claude Code)`                               |

### 9.5. free-claude-code

| Компонент             | Значение                                              |
|-----------------------|-------------------------------------------------------|
| Версия пакета         | `2.0.0` (из `pyproject.toml`)                         |
| Git commit            | `54a9dc4e341e0591d48cabb83970bcde9e1ceff8`            |
| Дата коммита          | 2026-05-18 13:51:56 -0700                             |
| Сообщение             | `Update README voice section`                         |
| Источник              | <https://github.com/Alishahryar1/free-claude-code>    |
| Установлен через      | `uv tool install --force .`                           |
| Префикс инсталляции   | `~/.local/share/uv/tools/free-claude-code/`           |
| Бинари в `~/.local/bin/` | `fcc-server`, `fcc-claude`, `fcc-init`, `free-claude-code` |

### 9.6. Python-зависимости free-claude-code (фактически установленные)

Pinned через `uv.lock` upstream-репозитория, версии в работающем окружении:

| Пакет                   | Версия       |
|-------------------------|--------------|
| fastapi                 | 0.136.1      |
| fastapi-cloud-cli       | 0.17.1       |
| starlette               | 1.0.0        |
| uvicorn                 | 0.47.0       |
| httpx                   | 0.28.1       |
| httpcore                | 1.0.9        |
| h11                     | 0.16.0       |
| aiohttp                 | 3.13.5       |
| websockets              | 16.0         |
| requests                | 2.34.2       |
| openai                  | 2.37.0       |
| pydantic                | 2.13.4       |
| pydantic-core           | 2.46.4       |
| pydantic-settings       | 2.14.1       |
| pydantic-extra-types    | 2.11.1       |
| python-dotenv           | 1.2.2        |
| python-telegram-bot     | 22.7         |
| discord.py              | 2.7.1        |
| tiktoken                | 0.13.0       |
| loguru                  | 0.7.3        |
| rich                    | 15.0.0       |
| rich-toolkit            | 0.19.9       |
| certifi                 | 2026.4.22    |
| dnspython               | 2.8.0        |
| email-validator         | 2.3.0        |
| multidict               | 6.7.1        |
| yarl                    | 1.23.0       |
| propcache               | 0.5.2        |
| python-multipart        | 0.0.29       |
| socksio                 | 1.0.0        |
| shellingham             | 1.5.4        |
| sentry-sdk              | 2.60.0       |
| markdown-it-py          | ≥4.2.0 (объявлено в pyproject) |

Опциональные группы (НЕ устанавливались, т.к. `VOICE_NOTE_ENABLED=false` и
`MESSAGING_PLATFORM=none`):
- `voice` — `grpcio>=1.80.0`, `grpcio-tools>=1.80.0`, `nvidia-riva-client>=2.25.1`
- `voice_local` — `torch>=2.12.0` + Hugging Face transformers Whisper

### 9.7. Конфигурация запуска

| Параметр                | Значение                                        |
|-------------------------|-------------------------------------------------|
| Порт fcc-server         | 8082                                            |
| Bind address            | `0.0.0.0:8082` (доступен по `127.0.0.1:8082`)   |
| Внутренний API-токен    | `freecc` (`ANTHROPIC_AUTH_TOKEN`)               |
| Z.AI endpoint           | `https://api.z.ai/api/anthropic`                |
| Менеджер процесса       | macOS `launchd` (LaunchAgent в `~/Library/LaunchAgents/`) |
| Label LaunchAgent       | `com.fcc-server.zai`                            |

### 9.8. Z.AI модели (на момент настройки)

| Модель Z.AI     | Назначение                                  | Reasoning |
|-----------------|---------------------------------------------|-----------|
| `glm-5.1`       | Замена Opus/Sonnet                          | да (включено флагом) |
| `glm-4.5-air`   | Замена Haiku и `MODEL` (fallback)           | нет       |

---

## 10. Воспроизведение с нуля (чек-лист для другого ИИ)

```bash
# 0. Предпосылки — должно совпадать с разделом 9.1–9.4
#    macOS arm64, zsh, git, npm, uv 0.11.15+, Python 3.14 (через uv),
#    Claude Code CLI 2.1.144 (npm i -g @anthropic-ai/claude-code).

# 1. Клонировать free-claude-code на конкретный commit
git clone https://github.com/Alishahryar1/free-claude-code.git \
  ~/projects/free-claude-code
cd ~/projects/free-claude-code
git checkout 54a9dc4e341e0591d48cabb83970bcde9e1ceff8

# 2. Установить как uv tool (Python 3.14 подтянется uv'ом по .python-version)
uv tool install --force .

# 3. Создать .env (см. раздел 4.3 — модели, ключ, токен)
cp .env.example .env
# отредактировать ZAI_API_KEY, MODEL_*, ANTHROPIC_AUTH_TOKEN=freecc,
# MESSAGING_PLATFORM=none, VOICE_NOTE_ENABLED=false

# 4. Создать обёртку claude-zai (раздел 4.1) — chmod +x

# 5. Создать LaunchAgent (раздел 4.2) и загрузить:
launchctl load ~/Library/LaunchAgents/com.fcc-server.zai.plist

# 6. Проверки
launchctl list | grep fcc-server          # должен быть PID
curl -s http://127.0.0.1:8082/health      # {"status":"healthy"}
claude-zai -p "тест"                       # ответ от GLM
```

Если версии Python/пакетов в окружении совпадают с разделами 9.3 и 9.6, поведение
прокси будет идентичным.

---

## 11. Ссылки

- Upstream: <https://github.com/Alishahryar1/free-claude-code>
- Z.AI docs: <https://docs.z.ai/devpack/tool/claude>
- Z.AI Anthropic endpoint: `https://api.z.ai/api/anthropic`
- Этот репозиторий: <https://github.com/andanlv-ai/free-claude-zai-setup>
