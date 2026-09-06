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
**Status:** Proposed
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
**Status:** Proposed
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
**Status:** Proposed
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

Follow-ons (not in the first wave): Linux sandbox not globally off, fail CI if codesign fails, pin/hash Python and llama.cpp downloads, drop `@ts-nocheck`, Electron fuses. Electron is pinned to 39.8.10 (`chore/bump-electron`). PR typecheck CI is on `develop`.
