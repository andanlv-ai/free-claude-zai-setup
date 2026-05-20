# Free-Claude + Z.AI + DeepSeek (macOS) — Обзор

Краткая выжимка. Подробная установка — в [README.md](README.md).

Дата: 2026-05-20
ОС: macOS arm64 (Darwin 25.4.0)

## Архитектура

```
claude     ──OAuth──▶  api.anthropic.com
claude-zai ──HTTPS─▶  api.z.ai/api/anthropic         (native Anthropic endpoint)
claude-ds  ──HTTPS─▶  api.deepseek.com/anthropic     (native Anthropic endpoint)
```

Все три — один и тот же `claude` бинарь. Различаются только обёртки, которые подменяют `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` / маппинг моделей. Прокси-серверов **нет**.

## Команды и модели

| Команда | Endpoint | Модели | Авторизация |
|---------|----------|--------|-------------|
| `claude` | `api.anthropic.com` | Claude 4.x | OAuth Keychain |
| `claude-zai` | `api.z.ai/api/anthropic` | `glm-5.1` (Opus/Sonnet), `glm-4.5-air` (Haiku) | `ANTHROPIC_AUTH_TOKEN` = Z.AI key |
| `claude-ds` | `api.deepseek.com/anthropic` | `deepseek-v4-flash` (все тиры) | `ANTHROPIC_AUTH_TOKEN` = DeepSeek key |

## Файлы

| Путь | Назначение |
|------|------------|
| `~/.local/bin/claude-zai` | Обёртка Z.AI |
| `~/.local/bin/claude-ds`  | Обёртка DeepSeek |
| `~/.claude/settings.json` | Глобальные настройки Claude Code (общие для всех трёх) |

LaunchAgent, `~/.fcc/`, `uv tool free-claude-code`, клон `free-claude-code` — больше не используются и удалены.

## Версии (зафиксировано 2026-05-20)

| Компонент | Версия |
|-----------|--------|
| macOS | 26.4.1 (build 25E253), arm64 |
| Node.js / npm | 25.7.0 / 11.14.1 |
| `@anthropic-ai/claude-code` | 2.1.144+ |

## Подводные камни

1. **`fcc-claude` не использовать** — пытается читать OAuth-токен из Keychain и перебивает `ANTHROPIC_AUTH_TOKEN` → 401. Только `claude --bare` через обёртку.
2. **`unset ANTHROPIC_API_KEY ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL`** в начале обёртки — иначе OAuth из Keychain перебивает.
3. **`API_TIMEOUT_MS=3000000`** обязателен для Z.AI (GLM-5.1 думает 15–20 с); для DeepSeek тоже не помешает.
4. **`--bare`** обязателен — без него CLI идёт в OAuth-flow.
5. **fcc-server не нужен.** На `claude-cli/2.1.145+` он ломает запросы с `tools[]` / `output_config` (конвертер Anthropic→OpenAI теряет fidelity → `Provider API request failed` / `Invalid request sent to provider`). Native Anthropic endpoint провайдеров принимает запросы напрямую.

## Ссылки

- Z.AI Anthropic endpoint: <https://api.z.ai/api/anthropic> ([docs](https://docs.z.ai/devpack/tool/claude))
- DeepSeek Anthropic endpoint: <https://api.deepseek.com/anthropic> ([docs](https://api-docs.deepseek.com/guides/anthropic_api))
- Этот репозиторий: <https://github.com/andanlv-ai/free-claude-zai-setup>
