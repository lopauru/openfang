# OpenFang - Patched Fork (lopauru/openfang)

Fork de [RightNow-AI/openfang](https://github.com/RightNow-AI/openfang) con fixes que upstream no mergea.

## Parches aplicados

### 1. Fix rustls CryptoProvider (Discord/TLS panic)
- **Archivo:** `crates/openfang-cli/src/main.rs`, `crates/openfang-cli/Cargo.toml`
- **Issue:** RightNow-AI/openfang#973
- **Problema:** Discord (y cualquier canal wss://) paniquea con "Could not automatically determine the process-level CryptoProvider"
- **Fix:** `rustls::crypto::ring::default_provider().install_default()` al inicio de `main()`

### 2. Soporte de attachments en Discord
- **Archivo:** `crates/openfang-channels/src/discord.rs`
- **Problema:** Mensajes con imagenes/archivos sin texto se descartaban silenciosamente
- **Fix:** Parsea el array `attachments` del payload de Discord y genera `ChannelContent::Image`, `File`, o `Voice`

### 3. Imagenes en driver claude-code
- **Archivo:** `crates/openfang-runtime/src/drivers/claude_code.rs`
- **Problema:** `build_prompt()` solo extraia texto, las imagenes se perdian
- **Fix:** Guarda imagenes base64 como archivos temporales en `/tmp/` e instruye al CLI que las lea con Read tool

## Build

```bash
# Requisitos: Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Clonar y compilar
git clone https://github.com/lopauru/openfang.git
cd openfang
git checkout patches/cod-fixes
cargo build --release --bin openfang

# Instalar
cp target/release/openfang ~/.openfang/bin/openfang
chmod +x ~/.openfang/bin/openfang
```

## Actualizar desde upstream

```bash
git remote add upstream https://github.com/RightNow-AI/openfang.git
git fetch upstream
git checkout patches/cod-fixes
git rebase upstream/main
# Resolver conflictos si los hay
cargo build --release --bin openfang
cp target/release/openfang ~/.openfang/bin/openfang
```

## launchd (macOS)

El plist esta en `~/Library/LaunchAgents/ai.openfang.daemon.plist`.

```bash
# Parar
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.openfang.daemon.plist

# Arrancar
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.openfang.daemon.plist
```
