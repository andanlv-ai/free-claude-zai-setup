Установи free-claude-code (https://github.com/Alishahryar1/free-claude-code)
и настрой работу через Z.AI (GLM) параллельно с обычным claude.

1. Клонируй репо и поставь: uv tool install --force .
2. Скопируй .env.example → .env, в нём:
   - ZAI_API_KEY="<спроси у меня>"
   - MODEL="zai/glm-4.5-air"
   - MODEL_HAIKU=zai/glm-4.5-air
   - MODEL_SONNET=zai/glm-5.1
   - MODEL_OPUS=zai/glm-5.1
   - ENABLE_OPUS_THINKING=true
   - ENABLE_SONNET_THINKING=true
   - MESSAGING_PLATFORM="none"
   - WHISPER_DEVICE="cpu"
3. Запусти fcc-server в фоне (порт 8082).
4. НЕ используй fcc-claude — OAuth токен из macOS keychain перебивает
   токен прокси и возвращает 401. Создай ~/.local/bin/claude-zai:

   #!/usr/bin/env bash
   set -e
   curl -sf http://127.0.0.1:8082/health >/dev/null || { echo "fcc-server не запущен"; exit 1; }
   unset ANTHROPIC_API_KEY ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL
   export ANTHROPIC_API_KEY=freecc ANTHROPIC_BASE_URL=http://127.0.0.1:8082
   exec claude --bare "$@"

   chmod +x и протестируй: claude-zai -p "тест".

Итог: claude — обычный OAuth, claude-zai — GLM через прокси.
Модели Z.AI: glm-5.1 (вместо Opus/Sonnet, с thinking), glm-4.5-air (вместо Haiku).
Base URL z.ai при прямом обращении: https://api.z.ai/api/anthropic
Подписка GLM Coding Plan не обязательна — pay-per-token ключ работает.
