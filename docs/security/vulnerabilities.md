# Vulnerability Review

Security review of the nos2x-fox codebase, covering three questions:

1. Is the software free of malware?
2. Does it use good key-storage practices?
3. Are there issues that put users at risk?

## 1. Malware assessment — no signs ✅

- **No network exfiltration**: the only `fetch()` in the codebase reads the local
  `manifest.json` (`options.tsx`). No beacons, WebSockets, or remote calls anywhere.
- **Minimal permissions**: the manifest requests only `storage`. No `tabs`, `cookies`,
  `webRequest`, etc.
- **No obfuscation**: ~4.5k lines of readable, commented TypeScript; dependencies are
  all mainstream (`nostr-tools`, React, `webextension-polyfill`).
- **No remote code loading**: everything is bundled; CSP restricts scripts to `'self'`.

## 2. Key-storage practices — good, with caveats ⚠️

Good:

- PIN encryption is **PBKDF2-SHA256 (100k iterations) + AES-GCM-256** with random
  salt/IV — correct use of WebCrypto, no home-rolled crypto.
- When PIN is enabled, plaintext keys are scrubbed from storage; all profile keys are
  encrypted; the code refuses to store plaintext keys while PIN is on.
- Private key material is converted to `Uint8Array` and zeroed after use.

Caveats (inherent, not bugs):

- **Without a PIN, the nsec sits in plaintext** in `browser.storage.local`. Anything
  with read access to the Firefox profile gets it. Enable the PIN.
- **PIN entropy**: a 4-digit PIN can be brute-forced in minutes by an attacker who
  steals the browser profile (100k PBKDF2 iterations × 10,000 PINs). Use a long,
  passphrase-style PIN.
- "Memory clearing" of strings is cosmetic (JS strings are immutable) — the code
  comments admit this. Don't over-trust it.

## 3. Findings

### 🔴 HIGH: Cached PIN leaks to any web page

`background.ts` handles a `getCachedPin` message by **returning the plaintext PIN**,
and the `runtime.onMessage` listener does not check the sender. The content script
(`content-script.ts`) forwards **any** `window.postMessage` from a web page into
`browser.runtime.sendMessage`, preserving attacker-controlled `type`. While the PIN
is cached (after any unlock, for the configured duration), any website can do:

```js
window.postMessage({ id: 'x', ext: 'nos2x-fox', type: 'getCachedPin', params: {} }, '*');
```

…and receive the PIN back through the normal response path. The PIN alone does not
decrypt the key (the ciphertext is in extension storage), but users reuse PINs, and
PIN + profile access = key. The same unauthenticated surface lets pages spuriously
reject an open PIN prompt (DoS).

**Fix:** only honor `getCachedPin` (and all `setupPin` / `verifyPin` / `disablePin` /
`openPinPrompt` / `encryptPrivateKey` messages) from the extension's own pages —
check that `sender.url` starts with `browser.runtime.getURL('')` — and never return
the PIN value; return a boolean status instead.

### 🟡 MEDIUM: Permission-prompt harassment

Rejected hosts aren't remembered (`REJECT` stores nothing), so a site can re-request
signing in a loop, spawning endless approval popups. Inherited from upstream nos2x.
A fix would be a session-level "remember rejection."

### 🟡 LOW: Unknown message types trigger a misleading prompt

`PERMISSIONS_REQUIRED[type]` is `undefined` for unknown types → `level >= undefined`
is false → a permission prompt opens, and `getAllowedCapabilities(undefined)` lists
**all** capabilities (since `perm > undefined` is always false). The operation then
fails harmlessly, but the prompt misrepresents the request. Validate `type` before
prompting.

### 🟢 Minor / informational

- `getSharedSecret` in `background.ts` never assigns `previousSk`, so the
  shared-secret cache is cleared on every call — performance bug, not security.
- CSP includes `'unsafe-eval'` — probably unnecessary; removing it would harden
  against XSS. No injectable sinks were found; all page-controlled data renders
  through React-escaped JSX.
- `signEvent` permits **any** event kind once authorized — that's the NIP-07 model,
  but an authorized site can sign DMs, contact lists, metadata, etc. The prompt does
  show the full event JSON for inspection. Prefer "Authorize just this" or short
  expiries over "forever."
- Doc nit: documentation said the PIN cache default is 10 minutes; `storage.ts`
  defaults to 10 seconds.

## Verdict

No malware, sound crypto, clean architecture. The one finding that matters is the
`getCachedPin` leak: it doesn't directly hand out the key, but it hands out the PIN
to every website while cached, and it reveals that the internal message surface isn't
sender-validated. Treat the extension as safe to use **after** patching that (add a
sender-origin check at the top of `onMessage`), with a strong PIN enabled.
