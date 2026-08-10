# Proposal 3: Manifest V3, one codebase → Chrome, Firefox, Safari

## Problem

- The extension is **Manifest V2, Firefox-only**: `background.scripts`
  (persistent page), `browser_action`, array-form
  `web_accessible_resources`, string CSP, and `browser_specific_settings.gecko`
  in `src/manifest.json`.
- Chrome Web Store no longer accepts MV2. Safari web extensions are MV3-oriented.
- The current architecture leans on MV2 assumptions that break under MV3's
  ephemeral service worker:
  - `background.ts` holds pending prompt state in in-memory maps
    (`openPromptMap`, `pinPromptMap`) whose `resolve`/`reject` callbacks
    vanish when the worker is killed.
  - `pinCache.ts` keeps the unlocked PIN in a module-level variable with a
    `setTimeout` expiry — both die with the worker.
  - `secretsCache` (LRU of nip44 conversation keys) is in-memory (acceptable
    loss, just a perf hit).
  - CSP includes `'unsafe-eval'`, which MV3 forbids for extension pages.

## Proposal

One codebase, **per-target builds**: `dist/firefox`, `dist/chrome`,
`dist/safari` from a shared manifest template plus per-target overrides.

### 3a. Manifest templating

`build.js` already has a two-variant pattern (`manifest.json` /
`hosted/manifest.json`). Generalize it:

- A base manifest template with placeholder tokens (version, gecko id,
  update_url).
- Per-target overlay files: Firefox keeps
  `browser_specific_settings.gecko` (id required) and `protocol_handlers`;
  Chrome drops both; Safari drops both.
- Output to `dist/<target>/` with `yarn build --target=<name>`; `yarn build`
  builds all three.

### 3b. MV3 conversion

- `background.service_worker` instead of `background.scripts`. Firefox
  supports MV3 service workers since FF 121; Safari since 16.4.
- `browser_action` → `action`; `web_accessible_resources` → object form with
  `matches`; move `<all_urls>` from `permissions` to `host_permissions`;
  CSP → `content_security_policy.extension_pages`, **drop `'unsafe-eval'`**
  (verify the esbuild/React bundle never evals — expected fine, must be
  confirmed in a spike).
- `webextension-polyfill` is bundled by esbuild, so it works in the service
  worker on all three targets unchanged.

### 3c. State survives worker restarts

The real work. Everything the background "remembers" must be rehydratable:

- **PIN cache** → `browser.storage.session` (per-profile, cleared on browser
  exit — the MV3-safe equivalent of today's in-memory cache). Expiry moves
  from `setTimeout` to stored timestamps checked lazily (already the fallback
  path in `pinCache.ts`) plus `browser.alarms` for proactive clearing.
- **Pending prompts** → `PromptManager` already persists queue entries to
  storage, but resolution is callback-based. Redesign: prompts are resolved
  by writing the decision to storage keyed by prompt id, and the background
  awaits via storage-change listeners instead of closures. On worker restart
  with pending prompts, re-render from the persisted queue rather than
  silently rejecting.
- **Passkey interaction** (proposal 1) makes this easier, not harder:
  WebAuthn assertions happen in an extension _page_, not the worker, so the
  worker can stay stateless about key material.

### 3d. Feature parity gaps (per-target flags)

| Feature                           | Firefox    | Chrome      | Safari   |
| --------------------------------- | ---------- | ----------- | -------- |
| `protocol_handlers` (`ext+nostr`) | ✅         | ❌ (no API) | ❌       |
| MV3 service worker                | ✅ FF 121+ | ✅          | ✅ 16.4+ |
| `options_ui.browser_style`        | ✅         | ignored     | ignored  |

`protocol_handlers` stays a Firefox-only feature behind a build flag;
`nostr-handler.html` ships only in the Firefox target.

**Firefox Android: dropped.** Mozilla does not support MV3 on Android, which
would force a permanent MV2 legacy target and dual-lifecycle code — with no
way for the maintainer to test it. The `browser.tabs` popup fallbacks have
been removed from `background.ts` / `common.ts` (recoverable from git
history). The mobile story is NIP-46 remote signing instead (proposal 4):
phone apps connect to the desktop extension as a bunker. Revisit if Mozilla
ships MV3 on Android and a tester is available.

### 3e. Distribution

- **Chrome**: zip `dist/chrome`, submit to Chrome Web Store.
- **Firefox**: `web-ext build` from `dist/firefox`, AMO as today. Note AMO's
  data-collection-permission declaration requirements.
- **Safari**: `xcrun safari-web-extension-converter dist/safari` → Xcode
  wrapper → App Store. This is the least automatable step; expect a manual
  release runbook. Needs a spike for Safari-specific quirks (prompt popup
  sizing, `browser.windows` behavior, WebAuthn in extension pages).

## Phasing

1. Manifest templating + per-target output dirs (still MV2, Firefox-only
   shipping target).
2. State refactor (3c) and CSP cleanup (3b) — behavior-preserving, still MV2.
3. MV3 manifests for all targets; Firefox switches to MV3; Chrome submission.
4. Safari converter spike + release runbook.

Steps 1–2 are independently shippable and de-risk the MV3 switch.

## Note on upstream nos2x

Upstream (fiatjaf/nos2x v2.x) is already MV3 — but naively: pending-prompt
state is a single in-memory `openPrompt` closure plus a mutex in the service
worker, so a worker restart with an open prompt silently hangs the request.
Its manifest (`action`, service worker, object-form
`web_accessible_resources`) is a useful reference, but section 3c is still
required — don't copy the lifecycle shortcuts.

## Open questions

- Does any dependency path actually need `'unsafe-eval'` today?
- CI: add `web-ext lint` per target and a headless Chrome smoke test?

## Interactions with other proposals

- **Proposal 1 (passkeys)**: WebAuthn RP-ID/origin behavior differs per
  browser — must be spiked per target. `storage.session` design is shared.
- **Proposal 4 (remote signer)**: relay WebSockets vs. service-worker
  lifecycle is the key constraint; the lifecycle work here (3c) is a
  prerequisite.
