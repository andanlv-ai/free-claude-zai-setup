# free-claude-zai-setup — macOS

Инструкция по запуску трёх `claude` параллельно на macOS: Anthropic + [Z.AI](https://z.ai) (GLM) + [DeepSeek](https://deepseek.com) — **все три без прокси-сервера**.

> Для Windows см. ветку [`windows`](../../tree/windows).

## Что получается

| Команда | Провайдер | Модели | Авторизация |
|---------|-----------|--------|-------------|
| `claude` | Anthropic | Claude 4.x | OAuth Pro (без изменений) |
| `claude-zai` | Z.AI | GLM-5.1 / GLM-4.5-air | API-ключ Z.AI |
| `claude-ds` | DeepSeek | deepseek-v4-pro / deepseek-v4-flash | API-ключ DeepSeek |

Все запускаются из терминала и VS Code. Хуки, `.claude/settings.json` — общие. Бинарник один (`claude`) — обёртки только меняют переменные окружения.

## Как это работает

`claude-zai` — bash-скрипт, который перед запуском `claude --bare`:

1. Очищает OAuth-токены Anthropic (`ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_MODEL`)
2. Устанавливает `ANTHROPIC_AUTH_TOKEN` = Z.AI API-ключ
3. Устанавливает `ANTHROPIC_BASE_URL` = `https://api.z.ai/api/anthropic`
4. Устанавливает маппинг моделей на GLM

Z.AI поднимает нативный Anthropic API — Claude Code работает без адаптаций: те же инструменты, хуки, structured outputs, streaming.

## Требования

- macOS
- `claude` CLI установлен через npm: `npm install -g @anthropic-ai/claude-code`
- API-ключ Z.AI — получить на [z.ai](https://z.ai)

## Установка

### 1. Создать обёртку claude-zai

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/claude-zai << 'EOF'
#!/usr/bin/env bash
set -e
unset ANTHROPIC_API_KEY ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL
export ANTHROPIC_AUTH_TOKEN='ВАШ_КЛЮЧ_ZAI'
export ANTHROPIC_BASE_URL='https://api.z.ai/api/anthropic'
export ANTHROPIC_DEFAULT_OPUS_MODEL='glm-5.1'
export ANTHROPIC_DEFAULT_SONNET_MODEL='glm-5.1'
export ANTHROPIC_DEFAULT_HAIKU_MODEL='glm-4.5-air'
export API_TIMEOUT_MS='3000000'
exec claude --bare "$@"
EOF
chmod +x ~/.local/bin/claude-zai
```

### 1b. Создать обёртку claude-ds (DeepSeek)

```bash
cat > ~/.local/bin/claude-ds << 'EOF'
#!/usr/bin/env bash
set -e
unset ANTHROPIC_API_KEY ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL
export ANTHROPIC_AUTH_TOKEN='ВАШ_КЛЮЧ_DEEPSEEK'
export ANTHROPIC_BASE_URL='https://api.deepseek.com/anthropic'
export ANTHROPIC_DEFAULT_OPUS_MODEL='deepseek-v4-pro'
export ANTHROPIC_DEFAULT_SONNET_MODEL='deepseek-v4-pro'
export ANTHROPIC_DEFAULT_HAIKU_MODEL='deepseek-v4-flash'
export API_TIMEOUT_MS='3000000'
exec claude --bare "$@"
EOF
chmod +x ~/.local/bin/claude-ds
```

DeepSeek поднимает нативный Anthropic endpoint на `https://api.deepseek.com/anthropic` — схема идентична Z.AI.

### 2. Добавить ~/.local/bin в PATH (если нет)

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### 3. Проверить

```bash
claude -p "ответь одним словом: anthropic"
claude-zai -p "какая ты модель? ответь кратко"
claude-ds  -p "какая ты модель? ответь кратко"
```

## Модели Z.AI

| Модель | Тир Claude | Описание |
|--------|-----------|----------|
| `glm-5.1` | Opus / Sonnet | Топ-модель, поддерживает reasoning. Ответ 15–20 с. |
| `glm-4.5-air` | Haiku / базовая | Лёгкая и быстрая |

## Важные замечания

- **`API_TIMEOUT_MS=3000000` обязателен** — GLM-5.1 думает 15–20 с, дефолтный таймаут Claude Code обрывает соединение.
- **`ANTHROPIC_AUTH_TOKEN`, не `API_KEY`** — Z.AI требует именно `AUTH_TOKEN` для совместимости с Claude Code.
- **`--bare` обязателен** — без него Claude Code открывает OAuth-экран вместо использования env vars.
- **fcc-server не нужен** — Claude Code 2.1.145+ шлёт `output_config` + `tools[]` + `structured-outputs-2025-12-15 beta`. Через OpenAI-конвертер fcc-server это ломается (`Provider API request failed`). Native Anthropic endpoint Z.AI принимает напрямую.

## Отладка

```bash
# Убедиться что запросы идут напрямую к Z.AI
ANTHROPIC_LOG=debug claude-zai -p "тест" 2>&1 | grep -E "baseUrl|api\.z\.ai"
```

## Миграция с fcc-server (если использовался ранее)

Если до этого был настроен `fcc-server` через launchd — теперь не нужен ни для Z.AI, ни для DeepSeek. Полная очистка:

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.fcc-server.zai.plist 2>/dev/null || true
rm -f ~/Library/LaunchAgents/com.fcc-server.zai.plist
uv tool uninstall free-claude-code
rm -rf ~/.fcc ~/projects/free-claude-code
```
