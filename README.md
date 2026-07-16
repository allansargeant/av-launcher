# av-launcher

> **AI-assisted project.** This codebase was created with [Claude](https://claude.com/claude-code)
> (Anthropic), directed and reviewed by a human author. The Rust backend
> compiles and the panel UI has been exercised via its mock backend; the
> full tray app has **not yet** been run end-to-end against a live server on
> the target machine.

A Bitfocus Companion–style **tray launcher** for the local web-server apps in
this fleet (srt-router, flock, RFutils, …). It gives any of them a
small splash panel to pick a **network interface** and **port**, **Start/Stop**
the server, **Launch GUI** in the browser, and live in the **system tray** —
without each app having to build its own launcher.

Built with [Tauri v2](https://tauri.app) (Rust + a tiny HTML/CSS/JS panel), so
the whole thing is a ~35 MB native app, not a bundled browser.

## Shipped launchers

Each fleet app now carries its own self-contained copy of this shell (with its
config baked in and, for the Rust apps, its server binary bundled), built and
released from that repo:

| app | launcher | download |
| --- | --- | --- |
| srt-router | [`launcher/`](https://github.com/allansargeant/srt-router/tree/main/launcher) | [release](https://github.com/allansargeant/srt-router/releases/tag/launcher-v0.1.0) |
| flock | [`launcher/`](https://github.com/allansargeant/flock/tree/master/launcher) | [release](https://github.com/allansargeant/flock/releases/tag/launcher-v0.1.0) |
| RFutils | [`launcher/`](https://github.com/allansargeant/RFutils/tree/main/launcher) | [release](https://github.com/allansargeant/RFutils/releases/tag/launcher-v0.1.0) |

This repo remains the canonical template/shell. A shipped `.app` finds its
baked-in config + server binary via bundled resources (see
[`scripts/screenshot.sh`](scripts/screenshot.sh) for how the README panel images
are rendered).

## How it works

The launcher itself is **app-agnostic**. It supervises a child process and
knows nothing about any particular server. Each app it can launch is described
by a single [`src-tauri/launcher.toml`](src-tauri/launcher.toml), which says how
to start the binary and — the important part — **how to inject the chosen
`host:port`**. Three modes cover the whole fleet:

| mode | what it does | fleet example |
| --- | --- | --- |
| `configfile` | patches a dotted key in the app's own TOML, then passes the rendered copy as the config arg | **srt-router** (nested `web.bind`, `--config`), **flock** (top-level `bind`, positional arg) |
| `env` | sets environment variables | **RFutils** (`RFUTILS_SERVER_PORT` / `RFUTILS_HOST`) |
| `args` | `{host}`/`{port}` placeholders already in `[app].args` | a plain `--host --port` server |

Ready-made configs for each app live in [`launchers/`](launchers/). To point the
launcher at one you only pick a config and swap the icon — no Rust changes.
(A future version can ship one bundled app per fleet member, each with its own
baked-in `launcher.toml` + icon.)

See [docs/adding-an-app.md](docs/adding-an-app.md) for the full schema and a
new-app checklist.

### What the panel does

- **GUI Interface** dropdown — every bindable IPv4 interface, plus an
  "All interfaces (0.0.0.0)" entry. Choosing a specific interface binds to that
  IP and shows it in the URL; "All" binds `0.0.0.0` and shows your primary LAN IP.
- **Port** — persisted per app.
- **Start / Stop** — spawns/kills the server child process and supervises it
  (detects if it exits on its own).
- **Launch GUI** — opens the resolved URL in your default browser.
- **Hide** — hides to the tray; **Quit** — stops the server and exits.
- Interface/port are locked while the server is running.

Settings persist to the OS app-config dir
(`~/Library/Application Support/com.allansargeant.av-launcher/` on macOS), along
with the rendered config the launcher hands to the server.

## Running (dev)

```bash
cd ~/Projects/av-launcher
npm install                 # first time only (fetches @tauri-apps/cli)
npm run tauri dev           # launches the panel + tray
```

`launcher.toml` is read from the working directory (`src-tauri/` under
`tauri dev`), or from `$AV_LAUNCHER_CONFIG`, or next to the built executable.

The bundled `launcher.toml` targets srt-router and expects its debug binary at
`~/Projects/srt-router/target/debug/srtrouter` — build it once with
`cargo build` in that repo (or edit `command`/`cwd` to taste).

To target another fleet app, point at its config in [`launchers/`](launchers/):

```bash
AV_LAUNCHER_CONFIG=launchers/flock.toml   npm run tauri dev   # flock
AV_LAUNCHER_CONFIG=launchers/rfutils.toml npm run tauri dev   # RFutils
```

## Building a distributable app

```bash
npm run tauri build         # produces a .app / .dmg (macOS), etc.
```

## Tests

```bash
cd src-tauri && cargo test  # covers host:port injection for all three modes,
                            # incl. flock (top-level bind) vs srt-router (web.bind)
```

## Adopting it for another app

1. Copy the closest config from [`launchers/`](launchers/) (or the active
   [`src-tauri/launcher.toml`](src-tauri/launcher.toml)).
2. Set `[app].name`, `command`, `args`, `default_port`, `cwd`, and the
   `[inject]` block. Full guide: [docs/adding-an-app.md](docs/adding-an-app.md).
3. Replace `src-tauri/icons/` with the app's icon
   (`npm run tauri icon path/to/icon.png` regenerates all sizes).

## Layout

```
src/                 panel UI (index.html, styles.css, main.js)
launchers/           ready-made per-app configs (srt-router, flock, rfutils)
docs/adding-an-app.md   schema + injection modes + new-app checklist
src-tauri/
  launcher.toml      the ACTIVE per-app config (this build → srt-router)
  src/config.rs      config parsing, interface enumeration, host:port injection (+ tests)
  src/lib.rs         Tauri commands, process supervision, system tray
```
