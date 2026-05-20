# Changelog

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
