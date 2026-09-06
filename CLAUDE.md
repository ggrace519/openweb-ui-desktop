# Open WebUI Desktop

Electron 39 + Svelte 5 shell around Open WebUI. Local install (bundled CPython/`uv`, SHA-256 pinned), optional llama.cpp (GitHub asset digest verified), optional Open Terminal, remote connections as `<webview>` guests.

This is a fork of [open-webui/desktop](https://github.com/open-webui/desktop). Hardening work lands here first so each change can be offered upstream as its own PR.

## Branch model

Two long-lived branches:

- `develop` — integration. Default for PRs. Always promotable.
- `main` — blessed / what is running.

Never commit directly to `main` or `develop`. Branch from `develop` as `<type>/<kebab-summary>` (e.g. `fix/webview-guest-ipc-allowlist`). Feature PRs target `develop` (the GitHub default). Promotion `develop` → `main` is a separate PR and is Greg's call.

## Commands

```bash
npm install
npm run dev
npm run lint
npm run typecheck
npm run build
```

There is no test script yet. `npm run typecheck` runs `tsc` for the main/preload graph (`tsconfig.node.json`) plus `svelte-check`. Main-process modules are included in `typecheck:node` (no `@ts-nocheck`).

## Layout

| Path | Role |
|---|---|
| `src/main/index.ts` | Electron main: windows, IPC, tray, shortcuts, lifecycle |
| `src/main/utils/` | Python/uv, Open WebUI server, llama.cpp, HF models, config |
| `src/preload/` | Per-window bridges (`index`, `content-preload`, `spotlight`, `voice-input`) |
| `src/renderer/` | Svelte 5 UI; `<webview>` guests in `Main/Connections/Content.svelte` |

Guest Open WebUI (local or remote) talks to the desktop through `content-preload.ts` only. It must never reach `window.electronAPI` on the shell renderer. The shell renderer is not given `@electron-toolkit/preload`'s `window.electron` (that object includes `ipcRenderer`). Shell BrowserWindows use `sandbox: true`.

On Linux, Chromium `--no-sandbox` is only for AppImage/snap/Flatpak, `ELECTRON_DISABLE_SANDBOX=1`, and unpackaged dev. Native `.deb` / `.rpm` keep the renderer sandbox.

Packaged builds flip Electron fuses in `electron-builder.yml` (`runAsNode` off, asar integrity, no `NODE_OPTIONS` / `--inspect` / `file:` extra privileges).

The `release` workflow fails closed if macOS codesign/notarization or Windows Azure Trusted Signing fails. It does not publish unsigned macOS/Windows fallbacks.

Child processes get a sanitized env (no `LD_PRELOAD` / `NODE_OPTIONS` / `PYTHONHOME` / …). llama-server `--host` / `--port` / `--models-dir` always come from the desktop, not from extra args.

## Hardening series

See `DECISIONS.md`. Land each fix on its own branch/PR against `develop` so they stay cherry-pickable for upstream.
