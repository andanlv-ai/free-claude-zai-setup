# free-claude-zai-setup — macOS

Инструкция по установке [free-claude-code](https://github.com/Alishahryar1/free-claude-code) с провайдером [Z.AI](https://z.ai) (GLM) параллельно с обычным `claude` на macOS.

> Для Windows см. ветку [`windows`](../../tree/windows) — там более простой подход без прокси-сервера.

## Что получается

| Команда | Провайдер | Модели | Авторизация |
|---------|-----------|--------|-------------|
| `claude` | Anthropic | Claude 4.x | OAuth Pro (без изменений) |
| `claude-zai` | Z.AI через fcc-server | GLM-5.1 / GLM-4.5-air | Z.AI API-ключ |

## Требования

- macOS
- [uv](https://docs.astral.sh/uv/) (устанавливается автоматически если нет)
- API-ключ Z.AI — получить на [z.ai](https://z.ai)
- Установленный `claude` CLI

## Установка

### 1. Установить uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. Склонировать и установить free-claude-code

```bash
git clone https://github.com/Alishahryar1/free-claude-code.git ~/projects/free-claude-code
cd ~/projects/free-claude-code
uv tool install --force .
```

### 3. Настроить .env

```bash
cp .env.example ~/projects/free-claude-code/.env
```

Отредактировать `~/projects/free-claude-code/.env`:

```env
ZAI_API_KEY="ваш_ключ_z.ai"
MODEL="zai/glm-4.5-air"
MODEL_HAIKU=zai/glm-4.5-air
MODEL_SONNET=zai/glm-5.1
MODEL_OPUS=zai/glm-5.1
ENABLE_OPUS_THINKING=true
ENABLE_SONNET_THINKING=true
MESSAGING_PLATFORM="none"
WHISPER_DEVICE="cpu"
```

### 4. Создать обёртку claude-zai

```bash
cat > ~/.local/bin/claude-zai << 'EOF'
#!/usr/bin/env bash
set -e
curl -sf http://127.0.0.1:8082/health >/dev/null || {
    echo "fcc-server не запущен. Запусти:"
    echo "  FCC_OPEN_BROWSER=false FCC_ENV_FILE=~/projects/free-claude-code/.env fcc-server --port 8082 &"
    exit 1
}
unset ANTHROPIC_API_KEY ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL
export ANTHROPIC_API_KEY=freecc ANTHROPIC_BASE_URL=http://127.0.0.1:8082
exec claude --bare "$@"
EOF
chmod +x ~/.local/bin/claude-zai
```

### 5. Автозапуск fcc-server через launchd

```bash
cat > ~/Library/LaunchAgents/com.fcc-server.zai.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.fcc-server.zai</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/.local/bin/fcc-server</string>
        <string>--port</string>
        <string>8082</string>
    </array>
    <key>EnvironmentVariables</key>
    <dict>
        <key>FCC_ENV_FILE</key>
        <string>/Users/YOUR_USERNAME/projects/free-claude-code/.env</string>
        <key>FCC_OPEN_BROWSER</key>
        <string>false</string>
    </dict>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/fcc-server.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/fcc-server.log</string>
</dict>
</plist>
EOF

sed -i '' "s/YOUR_USERNAME/$USER/g" ~/Library/LaunchAgents/com.fcc-server.zai.plist
launchctl load ~/Library/LaunchAgents/com.fcc-server.zai.plist
```

### 6. Проверить

```bash
curl -s http://127.0.0.1:8082/health   # {"status":"healthy"}
claude-zai -p "тест"
```

## Управление сервером

```bash
# Остановить
launchctl unload ~/Library/LaunchAgents/com.fcc-server.zai.plist

# Запустить
launchctl load ~/Library/LaunchAgents/com.fcc-server.zai.plist

# Логи
tail -f /tmp/fcc-server.log

# Проверить статус
curl -s -H "x-api-key: freecc" http://127.0.0.1:8082/
```

## Модели Z.AI

| Модель | Тир Claude | Описание |
|--------|-----------|----------|
| `glm-5.1` | Opus / Sonnet | Топ-модель, поддерживает reasoning |
| `glm-4.5-air` | Haiku / базовая | Лёгкая и быстрая |

## Важные замечания

- **Не использовать `fcc-claude`** — OAuth токен из macOS Keychain перебивает токен прокси и возвращает 401.
- `fcc-server` ищет `.env` через переменную `FCC_ENV_FILE` (флага `--env-file` нет).
- Подписка GLM Coding Plan не обязательна — pay-per-token ключ работает.
- **Thinking включён** (`ENABLE_OPUS_THINKING=true` и т.д.) — на macOS работает корректно в отличие от Windows.
