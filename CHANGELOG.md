# Changelog

## [2.1.0] — 2026-05-20 (macOS)

### Changed
- macOS: `claude-ds` переведён на прямое подключение к `api.deepseek.com/anthropic` (native Anthropic endpoint DeepSeek)
- CLAUDE.md / README.md / PROJECT_OVERVIEW.md переписаны под состояние без fcc-server

### Removed
- fcc-server, `~/.fcc/`, клон `free-claude-code`, uv tool `free-claude-code`, LaunchAgent `com.fcc-server.zai`

### Why
fcc-server конвертирует Anthropic→OpenAI и ломает `tools[]` / MCP на `claude-cli/2.1.145+`. Z.AI уже был переведён на прямое подключение; теперь и DeepSeek. После этого fcc-server не нужен ни одному провайдеру.

## [2.0.0] — 2026-05-20

### Changed
- Windows: переход с fcc-server (прокси) на прямое подключение к `api.z.ai/api/anthropic`
- Структура: разделены ветки `windows` и `macos` для разных подходов
- `main` теперь содержит только обзор и ссылки на платформенные ветки

### Removed (Windows)
- fcc-server больше не нужен на Windows — убран из Windows-инструкции
- uv и free-claude-code не требуются для Windows-подхода
- `.env` и Scheduled Task для fcc-server

### Why
claude CLI 2.1.145+ шлёт `output_config` + `tools[]` + `structured-outputs-2025-12-15 beta`.
fcc-server конвертирует Anthropic→OpenAI и GLM-5.1 через конвертер возвращает мусор → `Provider API request failed`.
Native Anthropic endpoint Z.AI (`api.z.ai/api/anthropic`) принимает запросы напрямую и работает корректно.

## [1.0.0] — 2026-05-20

### Added
- Базовая инструкция для macOS: fcc-server + launchd автозапуск
- Windows 11: fcc-server через Task Scheduler (Scheduled Task At Logon)
- Обёртка `claude-zai` (.ps1 + .cmd для Windows, bash для macOS)
- Описание thinking-флагов (на Windows отключены из-за `thinking_delta` в SSE)
