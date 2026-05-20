# free-claude-zai-setup — Windows 11

Инструкция по запуску двух `claude` в терминале параллельно на Windows 11: один ходит к Anthropic, второй к [Z.AI](https://z.ai) (GLM) — **без прокси-сервера**.

> Для macOS см. ветку [`macos`](../../tree/macos).

## Что получается

| Команда | Провайдер | Модели | Авторизация |
|---------|-----------|--------|-------------|
| `claude` | Anthropic | Claude 4.x | OAuth Pro (без изменений) |
| `claude-zai` | Z.AI | GLM-5.1 / GLM-4.5-air | API-ключ Z.AI |

Оба запускаются из PowerShell, cmd.exe и VS Code. Хуки, `.claude/settings.json` — общие. Бинарник один (`claude.exe`) — `claude-zai` это обёртка с другими переменными окружения.

## Как это работает

`claude-zai` — PowerShell-скрипт, который перед запуском `claude --bare`:

1. Очищает OAuth-токены Anthropic (`ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_MODEL`)
2. Устанавливает `ANTHROPIC_AUTH_TOKEN` = Z.AI API-ключ
3. Устанавливает `ANTHROPIC_BASE_URL` = `https://api.z.ai/api/anthropic`
4. Устанавливает маппинг моделей на GLM

Z.AI поднимает нативный Anthropic API — Claude Code работает без адаптаций: те же инструменты, хуки, structured outputs, streaming.

## Требования

- Windows 11 (PowerShell 5.1+)
- `claude` CLI установлен через npm: `npm install -g @anthropic-ai/claude-code`
- API-ключ Z.AI — получить на [z.ai](https://z.ai)

## Установка

### 1. Создать обёртку claude-zai.ps1

Создать файл `%USERPROFILE%\.local\bin\claude-zai.ps1`:

```powershell
#!/usr/bin/env pwsh
# claude-zai: direct connection to Z.AI native Anthropic endpoint
$ErrorActionPreference = 'Stop'

# Clear any OAuth/legacy creds that could override our token
Remove-Item Env:ANTHROPIC_API_KEY    -ErrorAction SilentlyContinue
Remove-Item Env:ANTHROPIC_MODEL      -ErrorAction SilentlyContinue
Remove-Item Env:ANTHROPIC_AUTH_TOKEN -ErrorAction SilentlyContinue

$env:ANTHROPIC_AUTH_TOKEN           = 'ВАШ_КЛЮЧ_ZAI'
$env:ANTHROPIC_BASE_URL             = 'https://api.z.ai/api/anthropic'
$env:ANTHROPIC_DEFAULT_OPUS_MODEL   = 'glm-5.1'
$env:ANTHROPIC_DEFAULT_SONNET_MODEL = 'glm-5.1'
$env:ANTHROPIC_DEFAULT_HAIKU_MODEL  = 'glm-4.5-air'
$env:API_TIMEOUT_MS                 = '3000000'

& claude --bare @args
exit $LASTEXITCODE
```

### 2. Создать cmd-шим

Создать файл `%USERPROFILE%\.local\bin\claude-zai.cmd`:

```bat
@echo off
powershell -NoProfile -ExecutionPolicy Bypass -File "%~dp0claude-zai.ps1" %*
```

Шим нужен чтобы `claude-zai` работал из cmd.exe и терминала VS Code.

### 3. Добавить ~/.local/bin в PATH (если нет)

```powershell
$path = [System.Environment]::GetEnvironmentVariable('PATH', 'User')
if ($path -notlike '*\.local\bin*') {
    [System.Environment]::SetEnvironmentVariable(
        'PATH', "$path;$env:USERPROFILE\.local\bin", 'User'
    )
}
```

Перезапустить терминал после изменения PATH.

### 4. Проверить

```powershell
claude -p "ответь одним словом: anthropic"
claude-zai -p "какая ты модель? ответь кратко"
```

## Модели Z.AI

| Модель | Тир Claude | Описание |
|--------|-----------|----------|
| `glm-5.1` | Opus / Sonnet | Топ-модель, поддерживает reasoning. Ответ 15–20 с. |
| `glm-4.5-air` | Haiku / базовая | Лёгкая и быстрая |

## Важные замечания

- **`API_TIMEOUT_MS=3000000` обязателен** — GLM-5.1 думает 15–20 с, дефолтный таймаут claude Code обрывает соединение.
- **`ANTHROPIC_AUTH_TOKEN`, не `API_KEY`** — Z.AI требует именно `AUTH_TOKEN` для совместимости с Claude Code.
- **`--bare` обязателен** — без него Claude Code открывает OAuth-экран вместо использования env vars.
- **Не использовать `fcc-claude`** — OAuth из Windows Credential Manager перебивает токен, возвращает 401.
- **fcc-server не нужен** — Claude Code 2.1.145+ шлёт `output_config` + `tools[]` + `structured-outputs-2025-12-15 beta`. Через OpenAI-конвертер fcc-server это ломается (`Provider API request failed`). Native Anthropic endpoint Z.AI принимает напрямую.

## Отладка

```powershell
# Полный лог запросов — видно куда идут запросы и что отвечает Z.AI
$env:ANTHROPIC_LOG = 'debug'; claude-zai -p "тест"

# Убедиться что endpoint — Z.AI, а не Anthropic
$env:ANTHROPIC_LOG = 'debug'; claude-zai -p "тест" 2>&1 | Select-String "baseUrl|api.z.ai"
```
