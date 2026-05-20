# free-claude-zai-setup

Инструкция по запуску двух `claude` в терминале параллельно: один ходит к Anthropic, второй к [Z.AI](https://z.ai) (GLM) — без прокси-сервера.

## Что получается

| Команда | Провайдер | Модели | Авторизация |
|---------|-----------|--------|-------------|
| `claude` | Anthropic | Claude 4.x | OAuth Pro (без изменений) |
| `claude-zai` | Z.AI | GLM-5.1 / GLM-4.5-air | API-ключ Z.AI |

Оба запускаются из терминала и IDE. Хуки, `.claude/settings.json` — общие, бинарник один.

## Инструкции по платформам

| Платформа | Ветка | Подход |
|-----------|-------|--------|
| **Windows 11** | [`windows`](../../tree/windows) | Прямое подключение к Z.AI — без прокси, без fcc-server |
| **macOS** | [`macos`](../../tree/macos) | Прямое подключение к Z.AI — без прокси, без fcc-server |

## Как это работает

`claude-zai` — скрипт-обёртка, который перед запуском `claude --bare`:

1. Очищает OAuth-токены Anthropic
2. Устанавливает `ANTHROPIC_AUTH_TOKEN` = Z.AI API-ключ
3. Устанавливает `ANTHROPIC_BASE_URL` = `https://api.z.ai/api/anthropic`
4. Устанавливает маппинг моделей на GLM

Z.AI поднимает нативный Anthropic API — Claude Code работает без адаптаций: те же инструменты, хуки, structured outputs, streaming.

### Быстрый старт (macOS / Linux)

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

## Модели Z.AI

| Модель | Тир Claude | Описание |
|--------|-----------|----------|
| `glm-5.1` | Opus / Sonnet | Топ-модель, поддерживает reasoning. Ответ 15–20 с. |
| `glm-4.5-air` | Haiku / базовая | Лёгкая и быстрая |

## Важные замечания

- **`API_TIMEOUT_MS=3000000` обязателен** — GLM-5.1 думает 15–20 с, дефолтный таймаут Claude Code обрывает соединение.
- **`ANTHROPIC_AUTH_TOKEN`, не `API_KEY`** — Z.AI требует именно `AUTH_TOKEN`.
- **`--bare` обязателен** — без него Claude Code открывает OAuth-экран.
- **fcc-server не нужен** — Claude Code 2.1.145+ шлёт `output_config` + `structured-outputs-2025-12-15 beta`. Через OpenAI-конвертер fcc-server это ломается. Native Anthropic endpoint Z.AI принимает напрямую.

## Changelog

See [CHANGELOG.md](CHANGELOG.md).
