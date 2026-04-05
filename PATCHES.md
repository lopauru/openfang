# OpenFang - Patched Fork (lopauru/openfang)

Fork de [RightNow-AI/openfang](https://github.com/RightNow-AI/openfang) con fixes y features que upstream no mergea.

**Version actual:** `v0.5.6-cod1` (basado en upstream v0.5.6)

---

## Parches aplicados

### 1. Fix rustls CryptoProvider (Discord/TLS panic)
- **Archivo:** `crates/openfang-cli/src/main.rs`, `crates/openfang-cli/Cargo.toml`
- **Issue:** RightNow-AI/openfang#973
- **Problema:** Discord (y cualquier canal wss://) paniquea con "Could not automatically determine the process-level CryptoProvider"
- **Fix:** `rustls::crypto::ring::default_provider().install_default()` al inicio de `main()`

### 2. Recibir attachments en Discord
- **Archivo:** `crates/openfang-channels/src/discord.rs`
- **Problema:** Mensajes con imagenes/archivos sin texto se descartaban silenciosamente (`content_text.is_empty()` retornaba `None`)
- **Fix:** Parsea el array `attachments` del payload de Discord y genera `ChannelContent::Image`, `File`, o `Voice` segun el `content_type`

### 3. Pasar imagenes al driver claude-code
- **Archivo:** `crates/openfang-runtime/src/drivers/claude_code.rs`
- **Problema:** `build_prompt()` solo llamaba `text_content()`, las imagenes se perdian silenciosamente
- **Fix:** Extrae `ContentBlock::Image` de los mensajes, decodifica base64, guarda como archivo temporal en `/tmp/openfang-img-*.{ext}` e instruye al CLI que las lea con el Read tool

### 4. Enviar archivos/imagenes a Discord
- **Archivo:** `crates/openfang-channels/src/discord.rs`
- **Problema:** El metodo `send()` solo soportaba `ChannelContent::Text`, todo lo demas mandaba "(Unsupported content type)"
- **Fix:** Implementa `api_send_file()` y `api_send_file_data()` que descargan/suben archivos via multipart form upload de Discord. Soporta `Image`, `File`, y `FileData`

### 5. Config `exclude_channels` para Discord
- **Archivos:** `crates/openfang-types/src/config.rs`, `crates/openfang-channels/src/discord.rs`, `crates/openfang-api/src/channel_bridge.rs`
- **Problema:** `group_policy` era todo-o-nada: o respondia a todo en todos los canales, o requeria @ en todos
- **Fix:** Nuevo campo `exclude_channels` en `DiscordConfig`. Los canales listados requieren @mention aunque `group_policy = "all"`. Usa metadata `was_mentioned` del parser para filtrar
- **Config ejemplo:**
  ```toml
  [channels.discord]
  exclude_channels = [
      "1473069902972719309",  # #gambler
      "1476349630894702643",  # #ops-alerts
  ]
  ```

### 6. Sesiones per-channel (procesamiento paralelo)
- **Archivos:** `crates/openfang-channels/src/bridge.rs`, `crates/openfang-api/src/channel_bridge.rs`, `crates/openfang-kernel/src/kernel.rs`
- **Problema:** Todos los mensajes de todos los canales iban a una sola sesion global del agente, serializada con un lock. Si COD estaba respondiendo en un canal, los DMs y otros canales esperaban en cola
- **Fix:**
  - Nuevo metodo `send_message_scoped(agent_id, message, channel_scope)` en el trait `ChannelBridgeHandle` y en el kernel
  - El bridge construye un `channel_scope` unico por canal: `discord:channel_id`, `discord-dm:user_id`, `whatsapp:phone`, `telegram-dm:user_id`
  - El kernel busca una sesion con `label = channel_scope` (via `find_session_by_label`). Si no existe, la crea con `create_session_with_label`
  - Lock per-session en vez de per-agente: canales distintos corren en paralelo
  - API REST y webchat siguen usando la sesion global (backward compatible)

### 7. Discord reconnect hardening
- **Archivo:** `crates/openfang-channels/src/discord.rs`
- **Problema:** Despues de varios cierres de conexion por el server (normal en Discord por load balancing), el gateway loop se daba por vencido y el bot quedaba offline permanentemente
- **Fix:**
  - El loop **nunca sale** a menos que sea shutdown explicito
  - Si la conexion duro >30 segundos, resetea backoff a 1s (cierre normal)
  - Solo crece backoff exponencial en fallos rapidos sucesivos
  - En desconexion inesperada, limpia session_id para hacer IDENTIFY fresco en vez de RESUME fallido

---

## Build

```bash
# Requisitos: Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Clonar y compilar
git clone https://github.com/lopauru/openfang.git
cd openfang
git checkout main
cargo build --release --bin openfang

# Instalar
cp target/release/openfang ~/.openfang/bin/openfang
chmod +x ~/.openfang/bin/openfang

# Reiniciar el daemon
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.openfang.daemon.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.openfang.daemon.plist
```

## Actualizar desde upstream

```bash
cd openfang
git remote add upstream https://github.com/RightNow-AI/openfang.git
git fetch upstream
git checkout main
git rebase upstream/main
# Resolver conflictos si los hay
cargo build --release --bin openfang
cp target/release/openfang ~/.openfang/bin/openfang
chmod +x ~/.openfang/bin/openfang
```

## launchd (macOS)

El servicio se registra como `ai.openfang.daemon` en launchd. Arranca automaticamente al prender la Mac y se reinicia si se cae (`KeepAlive`).

Plist: `~/Library/LaunchAgents/ai.openfang.daemon.plist`

```bash
# Parar
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.openfang.daemon.plist

# Arrancar
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.openfang.daemon.plist

# Reiniciar rapido (kill + auto-restart por KeepAlive)
launchctl kickstart -k gui/$(id -u)/ai.openfang.daemon

# Ver logs
tail -f ~/.openfang/logs/daemon.err.log
```

## Config relevante

```toml
# ~/.openfang/config.toml

api_listen = "127.0.0.1:4200"

[channels.discord]
bot_token_env = "DISCORD_BOT_TOKEN"
ignore_bots = false                    # ver mensajes de otros bots
exclude_channels = [                   # requieren @ para responder
    "1473069902972719309",  # #gambler
    "1476349630894702643",  # #ops-alerts
]

[channels.discord.overrides]
group_policy = "all"                   # responde sin @ (excepto exclude_channels)
dm_policy = "allowed_only"

[channels.whatsapp]
gateway_url = "http://127.0.0.1:3009"
default_agent = "UUID-del-agente"      # IMPORTANTE: usar UUID, no nombre

[default_model]
provider = "claude-code"
model = "claude-code/opus"
```

## Troubleshooting

| Problema | Solucion |
|----------|----------|
| Bot offline en Discord | `launchctl kickstart -k gui/$(id -u)/ai.openfang.daemon` |
| WhatsApp no responde | Verificar `default_agent` sea UUID, no nombre |
| "permissions acceptance" error | Correr una vez: `claude --dangerously-skip-permissions -p "test"` |
| DMs no contestan | Con per-channel sessions esto no deberia pasar. Verificar logs |
| Imagenes no llegan | Verificar que el binario sea el parcheado: `shasum ~/.openfang/bin/openfang` |
