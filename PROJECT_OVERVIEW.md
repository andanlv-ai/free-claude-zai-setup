# Free-Claude + Z.AI — Полное описание проекта

Документ описывает настройку среды, в которой `claude` (официальный CLI Anthropic) работает рядом с `claude-zai` (тот же CLI, но с прямым подключением к [Z.AI](https://z.ai) / GLM без прокси-сервера).

Дата: 2026-05-20
ОС: macOS (Darwin 25.4.0)
Пользователь: andrej

---

## 1. Архитектура

```
┌────────────────┐         OAuth (Anthropic)         ┌──────────────────────┐
│  claude        │ ────────────────────────────────▶ │  api.anthropic.com    │
└────────────────┘                                   └──────────────────────┘

┌────────────────┐              HTTPS                ┌──────────────────────┐
│  claude-zai    │ ────────────────────────────────▶ │  api.z.ai/api/       │
│  (обёртка)     │   ANTHROPIC_AUTH_TOKEN=zai_key    │  anthropic  (GLM)    │
└────────────────┘                                   └──────────────────────┘
```

`claude-zai` снимает OAuth-переменные окружения и устанавливает `ANTHROPIC_BASE_URL`
на нативный Anthropic-совместимый эндпоинт Z.AI — без локального прокси.

---

## 2. Команды и модели

| Команда      | Backend              | Модель                              | Авторизация              |
|--------------|----------------------|-------------------------------------|--------------------------|
| `claude`     | Anthropic API        | Claude Opus/Sonnet/Haiku            | OAuth (macOS Keychain)   |
| `claude-zai` | Z.AI (native Anthropic endpoint) | `glm-5.1` (Opus/Sonnet), `glm-4.5-air` (Haiku) | `ANTHROPIC_AUTH_TOKEN` = Z.AI API-ключ |

`claude` в shell — это alias: `claude --allow-dangerously-skip-permissions`.

### Маппинг моделей (env vars в скрипте claude-zai)

| Env var                          | Z.AI модель      |
|----------------------------------|------------------|
| `ANTHROPIC_DEFAULT_OPUS_MODEL`   | `glm-5.1`        |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | `glm-5.1`        |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL`  | `glm-4.5-air`    |

---

## 3. Файлы и расположения

| Путь                                                          | Назначение                                |
|---------------------------------------------------------------|-------------------------------------------|
| `/Users/andrej/projects/freeClaude/`                          | Этот репозиторий с инструкцией            |
| `/Users/andrej/projects/freeClaude/README.md`                 | Краткая инструкция установки              |
| `/Users/andrej/projects/freeClaude/prompt-zai.md`             | Исходный prompt для повторной настройки   |
| `/Users/andrej/projects/freeClaude/PROJECT_OVERVIEW.md`       | Этот документ                             |
| `/Users/andrej/.local/bin/claude-zai`                         | Обёртка-launcher (прямой коннект к Z.AI)  |
| `/Users/andrej/.claude/settings.json`                         | Глобальные настройки Claude Code          |

---

## 4. Содержимое ключевых файлов

### 4.1. `~/.local/bin/claude-zai`

```bash
#!/usr/bin/env bash
set -e
unset ANTHROPIC_API_KEY ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL
export ANTHROPIC_AUTH_TOKEN='ZAI_API_KEY_HERE'
export ANTHROPIC_BASE_URL='https://api.z.ai/api/anthropic'
export ANTHROPIC_DEFAULT_OPUS_MODEL='glm-5.1'
export ANTHROPIC_DEFAULT_SONNET_MODEL='glm-5.1'
export ANTHROPIC_DEFAULT_HAIKU_MODEL='glm-4.5-air'
export API_TIMEOUT_MS='3000000'
exec claude --bare "$@"
```

Ключевые моменты:
- `unset` чистит OAuth-токены, которые Claude CLI иначе подсунет в заголовок.
- `ANTHROPIC_AUTH_TOKEN` = реальный Z.AI API-ключ (не прокси-токен).
- `--bare` отключает интеграцию с Claude.ai.
- `API_TIMEOUT_MS=3000000` — GLM-5.1 думает 15–20 с, без этого Claude Code обрывает запрос.

---

## 5. Жизненный цикл

### Запуск `claude-zai`
1. Скрипт чистит OAuth-переменные окружения.
2. Устанавливает `ANTHROPIC_AUTH_TOKEN` и `ANTHROPIC_BASE_URL`.
3. Запускает `claude --bare`.
4. Claude CLI шлёт Anthropic-формат запросы напрямую на `api.z.ai/api/anthropic`.
5. Z.AI отвечает в нативном Anthropic-формате — никакой конвертации не нужно.

---

## 6. Управление

```bash
# Проверить, куда идут запросы
ANTHROPIC_LOG=debug claude-zai -p "тест" 2>&1 | grep -E "baseUrl|api\.z\.ai"

# Smoke-test
claude-zai -p "какая ты модель? ответь одним словом"
```

---

## 7. Подводные камни (зафиксированы при настройке)

1. **`ANTHROPIC_AUTH_TOKEN`, не `ANTHROPIC_API_KEY`** — Z.AI требует именно `AUTH_TOKEN` для совместимости с Claude Code.
2. **`--bare` обязателен** — без него Claude Code открывает OAuth-экран вместо использования env vars.
3. **`API_TIMEOUT_MS=3000000` обязателен** — GLM-5.1 думает 15–20 с, дефолтный таймаут обрывает соединение.
4. **fcc-server не нужен** — Claude Code 2.1.145+ шлёт `output_config` + `tools[]` + `structured-outputs-2025-12-15 beta`. Через OpenAI-конвертер fcc-server это ломается (`Provider API request failed`). Native Anthropic endpoint Z.AI принимает напрямую.
5. **Подписка GLM Coding Plan не обязательна.** Обычного pay-per-token ключа Z.AI достаточно.

---

## 8. Текущее состояние (проверено 2026-05-20)

| Проверка                          | Результат          |
|-----------------------------------|--------------------|
| `which claude-zai`                  | `/Users/andrej/.local/bin/claude-zai` |
| `ANTHROPIC_BASE_URL` в скрипте      | `https://api.z.ai/api/anthropic` |
| fcc-server                          | остановлен (не нужен) |
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

### 9.3. Claude Code CLI

| Компонент                  | Версия                                                |
|----------------------------|-------------------------------------------------------|
| `@anthropic-ai/claude-code` | 2.1.144 (установлен глобально через npm)             |
| `claude --version`         | `2.1.144 (Claude Code)`                               |

### 9.4. Z.AI модели (на момент настройки)

| Модель Z.AI     | Назначение                                  | Reasoning |
|-----------------|---------------------------------------------|-----------|
| `glm-5.1`       | Замена Opus/Sonnet                          | да (включено флагом) |
| `glm-4.5-air`   | Замена Haiku и `MODEL` (fallback)           | нет       |

---

## 10. Воспроизведение с нуля (чек-лист)

```bash
# 0. Предпосылки
#    macOS arm64, npm, Claude Code CLI (npm i -g @anthropic-ai/claude-code),
#    API-ключ Z.AI.

# 1. Создать обёртку claude-zai (раздел 4.1), chmod +x
mkdir -p ~/.local/bin
# вставить содержимое из раздела 4.1, заменить ZAI_API_KEY_HERE на реальный ключ
chmod +x ~/.local/bin/claude-zai

# 2. Убедиться что ~/.local/bin в PATH
echo $PATH | grep -q ".local/bin" || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc

# 3. Проверки
claude-zai -p "тест"  # ответ от GLM
ANTHROPIC_LOG=debug claude-zai -p "тест" 2>&1 | grep "api\.z\.ai"  # прямой коннект
```

---

## 11. Ссылки

- Upstream: <https://github.com/Alishahryar1/free-claude-code>
- Z.AI docs: <https://docs.z.ai/devpack/tool/claude>
- Z.AI Anthropic endpoint: `https://api.z.ai/api/anthropic`
- Этот репозиторий: <https://github.com/andanlv-ai/free-claude-zai-setup>
