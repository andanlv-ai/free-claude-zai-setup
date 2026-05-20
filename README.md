# free-claude-zai-setup

Инструкция по запуску двух `claude` в терминале параллельно: один ходит к Anthropic, второй к [Z.AI](https://z.ai) (GLM).

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
| **macOS** | [`macos`](../../tree/macos) | fcc-server (локальный прокси Anthropic→OpenAI) |

> Windows-инструкция рекомендуется как более простая: не требует uv, free-claude-code или прокси-сервера.

## Модели Z.AI

| Модель | Тир Claude | Описание |
|--------|-----------|----------|
| `glm-5.1` | Opus / Sonnet | Топ-модель, поддерживает reasoning. Ответ 15–20 с. |
| `glm-4.5-air` | Haiku / базовая | Лёгкая и быстрая |

## Changelog

See [CHANGELOG.md](CHANGELOG.md).
