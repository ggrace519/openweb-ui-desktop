# Decisions

Architecture and process decisions for this fork. Format is ADR. Status is Proposed until the implementing PR merges into `develop`.

## ADR-0001: develop / main branch model

**Date:** 2026-09-05
**Status:** Accepted
**Phase:** Initialize
**Deciders:** Greg

### Context

The upstream Open WebUI Desktop repo uses `main` only. This fork needs an integration branch so hardening can land as stacked, reviewable PRs without committing straight to `main`, and so each fix stays easy to offer upstream.

### Decision

Two long-lived branches: `develop` (integration) and `main` (blessed). Feature/fix/docs PRs target `develop`. Promotion `develop` → `main` is a separate PR, merge commit or fast-forward, never squash. `develop` is the GitHub default branch (set 2026-09-06).

### Rationale

Matches the standing 519lab branch model. One coherent change per PR keeps upstream cherry-picks small.

### Consequences

CI currently runs only on push to `release` (upstream's packaging branch). PR checks against `develop` are a later hardening item, not this ADR.

---

## ADR-0002: Guest webviews are hostile

**Date:** 2026-09-05
**Status:** Accepted
**Phase:** Hardening
**Deciders:** Greg

### Context

`<webview>` guests load user-supplied Open WebUI URLs, including remotes. The content preload was a generic RPC; the embedder dispatched `window.electronAPI[requestData.type]`, so a guest could call `resetApp` / `setConfig` / spawn APIs. Open WebUI's actual desktop protocol is a small allowlist (`token:update`, `app:info`, `app:data`, `window:isFocused`, plus inbound events `query` / `call` / `theme:update` / `models:refresh` / `connections:terminal` / `connections:openai`).

### Decision

Guests may only send allowlisted message types. The embedder never indexes `electronAPI` with guest strings. Outbound desktop events that carry secrets (`connections:terminal` API key, llama.cpp URL) go only to the local-connection webview. `will-attach-webview` forces `nodeIntegration: false`, `contextIsolation: true`, `sandbox: true`, `webSecurity: true`.

### Rationale

Connecting to a remote server is a headline feature. That origin is untrusted by definition, and local Open WebUI is an XSS surface (markdown, tools, model output).

### Consequences

New desktop↔Open WebUI features need an explicit protocol bump, not a new `electronAPI` method name. Remote connections no longer receive local Open Terminal keys.

---

## ADR-0003: Certificate trust is per-connection, never global

**Date:** 2026-09-05
**Status:** Accepted
**Phase:** Hardening
**Deciders:** Greg

### Context

Issue #108 (self-signed Open WebUI servers) was implemented by trusting every certificate on every Chromium session, including updater and all webviews.

### Decision

Default-verify. `certificate-error` may allow an exception only for origins the user has added as connections, plus localhost. Do not attach `setCertificateVerifyProc(0)` to `defaultSession`. A later PR may add an explicit fingerprint prompt; the first PR only scopes the existing exception.

### Rationale

Self-signed *user servers* do not justify disabling PKI for GitHub, Hugging Face, or auto-update.

### Consequences

Misconfigured remotes with bad certs that are not in the connections list will fail TLS until added. Node `fetch()` (Python / llama.cpp / HF downloads) already used OpenSSL and was not covered by the Chromium hook.

---

## ADR-0004: Child runtimes have a real quit gate

**Date:** 2026-09-05
**Status:** Accepted
**Phase:** Hardening
**Deciders:** Greg

### Context

`before-quit` is `async` without `event.preventDefault()`, so Electron does not wait for llama.cpp / Open Terminal / Open WebUI to stop. `ServiceLock.acquire()` is immediately undone because `start*()` calls `stop*()` which releases the lock.

### Decision

Use `will-quit` + `preventDefault` until process trees are reaped, then `app.exit()`. `stop*({ retainLock: true })` when invoked from start so the lock is held across the spawn window. PID files under userData for crash recovery are a follow-on if the quit gate is not enough.

### Rationale

Orphans hold ports and GPUs; the next launch silently binds the next port and the old process keeps running.

### Consequences

Quit is slightly slower (seconds) while children die. That is preferable to leaked inference servers.

---

## Hardening PR series (against `develop`)

Land in this order. Each is its own branch. Later items may stack if they touch the same file.

| # | Branch | What |
|---|---|---|
| 1 | `fix/webview-guest-ipc-allowlist` | ADR-0002 inbound: allowlist guest `send()`, implement real protocol, `will-attach-webview` |
| 2 | `fix/webview-secret-broadcast` | ADR-0002 outbound: local-only terminal/OpenAI events; stop logging API keys |
| 3 | `fix/tls-verify-default` | ADR-0003 |
| 4 | `fix/open-external-schemes` | `http:`/`https:`/`mailto:` only for `shell.openExternal` |
| 5 | `fix/hf-path-traversal` | Allowlist HF repo/filename; confine to models dir |
| 6 | `fix/service-lock-quit` | ADR-0004 |

Follow-ons from the first wave are landed. Electron is pinned to 39.8.10 (`chore/bump-electron`). PR typecheck CI is on `develop`. Python / llama.cpp downloads are checksum-verified (`fix/artifact-checksums`). Linux `--no-sandbox` is scoped to AppImage/snap/Flatpak and unpackaged runs (`fix/linux-sandbox-scope`). Electron fuses are set in `electron-builder.yml` (`fix/electron-fuses`). macOS/Windows releases fail closed on signing (`fix/release-codesign-required`). Child env / llama extra args are sanitized (`fix/child-env-allowlist`). Main-process `@ts-nocheck` is gone (`chore/drop-ts-nocheck`).

---

## ADR-0009: Child env and llama extra args are not attacker-controlled

**Date:** 2026-09-05
**Status:** Proposed
**Phase:** Hardening
**Deciders:** Greg

### Context

Settings lets the user (or anything that can `setConfig`) inject arbitrary `envVars` into Open WebUI, Open Terminal, and llama-server, and arbitrary `llamaCpp.extraArgs`. `LD_PRELOAD` / `DYLD_INSERT_LIBRARIES` / `NODE_OPTIONS` / `PYTHONHOME` become code execution in those children. `extraArgs` of `--host 0.0.0.0` binds llama-server on every interface after the desktop already chose `127.0.0.1`.

### Decision

Strip a blocklist of loader/runtime keys (case-insensitive) from the inherited environment and from `config.envVars` before `pty.spawn`. Strip `--host`, `--port`, and `--models-dir` from extra args and always append the desktop's values last.

### Rationale

The desktop owns bind address and model path. User extra args are for GPU layers and context size, not for rebinding the server. Env injection is a confused-deputy even when the UI form is "advanced settings".

### Consequences

A user who needs `PYTHONHOME` or `LD_PRELOAD` for a custom build cannot set them through the app. `LD_LIBRARY_PATH` is still allowed for CUDA/ROCm. Extra `--host 0.0.0.0` is silently dropped rather than rejected in the UI.

---

## ADR-0008: Signed macOS and Windows artifacts or no release

**Date:** 2026-09-05
**Status:** Proposed
**Phase:** Hardening
**Deciders:** Greg

### Context

`release.yml` treated codesign as best-effort: missing Apple certs, notarization failure, or Azure Trusted Signing failure still published unsigned installers via `continue-on-error` fallbacks. Auto-update manifests then had to be patched because signed and unsigned hashes differed.

### Decision

macOS jobs require the signing certificate and notarize; Windows jobs disable CSC auto-discovery and require Azure Trusted Signing. Either step failing fails the matrix job (and therefore the GitHub Release). Linux remains unsigned, which is normal for `.deb` / AppImage.

### Rationale

An unsigned `.dmg` / `.exe` is not the product. Publishing it teaches users to bypass Gatekeeper/SmartScreen and makes a supply-chain swap indistinguishable from "the fallback build".

### Consequences

A `release` push without Apple/Azure secrets will fail macOS and Windows (Linux may still package; the release job still requires `package` to succeed). That is the intended gate. Local `electron-builder` without secrets is unchanged.

---

## ADR-0007: Packaged builds flip Electron fuses

**Date:** 2026-09-05
**Status:** Proposed
**Phase:** Hardening
**Deciders:** Greg

### Context

Default Electron fuses allow `ELECTRON_RUN_AS_NODE=1` to turn the app binary into a generic Node, honor `NODE_OPTIONS` / `--inspect`, skip ASAR integrity checks, and grant `file:` extra privileges. A renderer or env-var attacker can use those to escape the packaged app.

### Decision

Set `electronFuses` in `electron-builder.yml`: `runAsNode` false, `enableNodeOptionsEnvironmentVariable` false, `enableNodeCliInspectArguments` false, `enableEmbeddedAsarIntegrityValidation` true, `onlyLoadAppFromAsar` true, `grantFileProtocolExtraPrivileges` false. Keep `asarUnpack` for `node-pty` and `resources/**`.

### Rationale

The app does not use `process.fork` (which needs `ELECTRON_RUN_AS_NODE`). Integrity of `app.asar` plus "only load from asar" is the documented pairing. Unpacked native addons still load from `app.asar.unpacked`.

### Consequences

`npm run dev` is unaffected (fuses apply at pack time). ASAR integrity validation is macOS and Windows only in current Electron; Linux still gets the other fuses. Debugging a packaged build with `--inspect` or `NODE_OPTIONS` will not work — that is the point.

---

## ADR-0006: Linux renderer sandbox stays on for native packages

**Date:** 2026-09-05
**Status:** Proposed
**Phase:** Hardening
**Deciders:** Greg

### Context

Every Linux launch appended `--no-sandbox`, so a renderer compromise on a native `.deb` had the same privileges as the browser process. That flag is required inside AppImage (FUSE), snap, and Flatpak, where Chromium's SUID helper cannot be installed.

### Decision

Apply `--no-sandbox` only when `APPIMAGE`, `SNAP`, or `FLATPAK_ID` is set, when `ELECTRON_DISABLE_SANDBOX=1`, or when the app is unpackaged (`npm run dev` — `chrome-sandbox` is not setuid in `node_modules`). Keep `disable-dev-shm-usage` and `disable-gpu-sandbox` on Linux; those address `/dev/shm` crashes, not renderer isolation.

### Rationale

The renderer sandbox is the main process-isolation boundary for `<webview>` guests. Turning it off for every Linux user to support container formats is broader than the constraint.

### Consequences

A native package whose `chrome-sandbox` helper is missing or not setuid will fail to start until the user sets `ELECTRON_DISABLE_SANDBOX=1` or the package is fixed. That is preferable to silently disabling the sandbox for everyone.

---

## ADR-0005: Downloaded runtimes are hashed before extract

**Date:** 2026-09-05
**Status:** Proposed
**Phase:** Hardening
**Deciders:** Greg

### Context

The app downloads a python-build-standalone tarball and llama.cpp GitHub release archives, then extracts and executes them. Those transfers used HTTPS only — no pinned digest — so a compromised GitHub asset, a truncated cache, or a swapped `python.tar.gz` in userData would still be installed.

### Decision

Pin the CPython `install_only` tarball filenames to the official 20260310 SHA-256 sums and refuse unknown platform/arch pairs. Require GitHub's `digest: sha256:…` on llama.cpp assets (fail closed if missing). Hash while downloading; re-hash cached archives before reuse; delete on mismatch.

Hugging Face GGUF downloads stay on their own path (user-chosen files, no in-repo pin). Model-file LFS SHA-256 is a follow-on.

### Rationale

Integrity of the interpreter and inference binary is a software-supply-chain control, not a nice-to-have. GitHub now publishes asset digests; python-build-standalone publishes SHA256SUMS. Fail closed rather than install unverified bytes.

### Consequences

A llama.cpp release with no `digest` field cannot be installed until GitHub provides one (current `b*` binary releases do). Bumping the Python standalone date requires updating the pin table. `releases/latest` for llama.cpp currently resolves to `v0.4.0`, which has no binary assets — that is a separate bug from checksums.

---

## ADR-0010: Production shell is served on app://, not file://

**Date:** 2026-09-06
**Status:** Proposed
**Phase:** Hardening
**Deciders:** Greg

### Context

Issue #37. Packaged windows `loadFile()` the Svelte shell (`file://`). Electron fuses already set `grantFileProtocolExtraPrivileges: false`. Electron still recommends a custom protocol: `file://` is a unique origin with historically extra privileges; a confined `protocol.handle` can only serve `out/renderer`.

### Decision

Scheme `app`, host `renderer`. Privileges: `standard`, `secure`, `supportFetchAPI`, `stream`. No `bypassCSP`, no `corsEnabled`, no service workers. `protocol.handle` on defaultSession only. Production loadURL `app://renderer/{index,spotlight,voice-input}.html`. Dev still uses `ELECTRON_RENDERER_URL`. Webview guests stay http(s) on `persist:connection-*` and do not receive the handler. Do not register `app` as an OS protocol handler.

### Rationale

Named host `renderer` is allowlisted so `app://index.html` cannot be mis-parsed as a host. Confining the handler to defaultSession means a remote Open WebUI guest cannot read shell files even if it requests `app:`.

### Consequences

Shell `localStorage` origin changes once (`file://` → `app://renderer`); i18n locale cache resets unless already in config. Hero `<video>` depends on `stream: true`. Preview (`npm start`) and `build:unpack` are the proof paths, not `npm run dev`.
