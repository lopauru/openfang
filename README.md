<p align="center">
  <img src="public/assets/openfang-logo.png" width="160" alt="OpenFang Logo" />
</p>

<h1 align="center">OpenFang - FOMO Edition</h1>
<h3 align="center">Because upstream moves slow and we have agents to ship</h3>

<p align="center">
  Fork mantenido por <a href="https://github.com/lopauru">lopauru</a> con los fixes que el repo oficial no mergea.<br/>
  <strong>Si funciona, no lo toques. Si no funciona, parchealo vos.</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/version-0.5.6--cod1-green?style=flat-square" alt="v0.5.6-cod1" />
  <img src="https://img.shields.io/badge/upstream-RightNow--AI%2Fopenfang-blue?style=flat-square" alt="Upstream" />
  <img src="https://img.shields.io/badge/patches-7-orange?style=flat-square" alt="7 patches" />
  <img src="https://img.shields.io/badge/vibes-production-brightgreen?style=flat-square" alt="Production vibes" />
</p>

---

## Que tiene este fork que no tiene el original?

| # | Patch | Problema | Status |
|---|-------|----------|--------|
| 1 | **Fix rustls/TLS** | Discord paniquea al conectar | Resuelto |
| 2 | **Recibir imagenes Discord** | Fotos y archivos se descartaban silenciosamente | Resuelto |
| 3 | **Imagenes al CLI** | Claude Code no veia las imagenes que le mandaban | Resuelto |
| 4 | **Enviar archivos Discord** | Bot no podia mandar imagenes/archivos de vuelta | Resuelto |
| 5 | **`exclude_channels`** | group_policy era todo-o-nada, sin filtro por canal | Resuelto |
| 6 | **Sesiones per-channel** | DMs esperaban en cola mientras respondia en un canal | Resuelto |
| 7 | **Reconnect infinito** | Discord se desconectaba y el bot no volvia | Resuelto |

Documentacion detallada de cada patch en [PATCHES.md](PATCHES.md).

## Quick Start

```bash
# Clonar y compilar
git clone https://github.com/lopauru/openfang.git
cd openfang
cargo build --release --bin openfang

# Instalar
cp target/release/openfang ~/.openfang/bin/openfang
chmod +x ~/.openfang/bin/openfang

# Arrancar
openfang start
```

## Actualizar desde upstream

```bash
git remote add upstream https://github.com/RightNow-AI/openfang.git
git fetch upstream
git rebase upstream/main
cargo build --release --bin openfang
cp target/release/openfang ~/.openfang/bin/openfang
```

## macOS (launchd)

```bash
# Reiniciar
launchctl kickstart -k gui/$(id -u)/ai.openfang.daemon

# Logs
tail -f ~/.openfang/logs/daemon.err.log
```

---

<p align="center">
  <sub>Upstream: <a href="https://github.com/RightNow-AI/openfang">RightNow-AI/openfang</a> | Hecho con mate y Claude a las 3am</sub>
</p>
