# Proposal 5: Port the best of upstream nos2x

## Context

Upstream [fiatjaf/nos2x](https://github.com/fiatjaf/nos2x) (v2.x,
Chromium-only, MV3, plain JS) has shipped several features we lack. We
reviewed it and decided to **stay on our codebase** (TypeScript,
multi-profile, PIN encryption at rest, time-boxed grants) — upstream's own
README points Firefox users here, and its MV3 state handling is naive (see
proposal 3). This proposal cherry-picks their proven ideas, in priority
order.

## Ports

### 5a. Kind-scoped accept/reject conditions

The biggest one — the concrete implementation of proposal 2c.

- **Data model**: adopt upstream's
  `policies[host][accept][type] = { conditions: { kinds: { 7: true } }, created_at }`.
  Policies can be accept _or_ reject; a reverse policy with identical
  conditions cancels the existing one. Conditions merge on repeat grants.
- **Merge with our time-boxed grants**: conditions and expiry are
  orthogonal — a policy becomes `{ conditions, expires_at? }`, so
  "allow kind 7 for this site for 1 hour" is expressible.
- **Prompt UI**: "authorize kind N forever" / "reject kind N forever"
  buttons alongside the existing duration choices.
- **Migration**: existing level-based `activePermissions` entries map onto
  accept-true policies with empty conditions on first run.

### 5b. `peekPublicKey`

`window.nostr.peekPublicKey()`: returns the pubkey if `getPublicKey` was
previously authorized for this host, `''` otherwise — never prompts. Lets
clients probe on page load without spawning popup storms. Cheap: provider
method + a permission check in the background.

### 5c. Notification audit log

`optional_permissions: ["notifications"]` + a toggle in Options; fire a
basic browser notification on every auto-allowed/denied action (kind,
content summary, host). A cheap answer to proposal 2's audit question —
policy grants become visible instead of silent.

### 5d. Result-in-prompt

Perform the crypto operation _before_ asking, and show the decryption
result in the prompt so users see the actual plaintext they're approving.
**Caveat**: crypto happens regardless of consent; only disclosure is
gated. Limit to decrypt operations and land alongside proposal 2a
(human-readable prompts), not before.

### 5e. Centered prompt windows

Position prompt popups centered on the last focused window (upstream's
`getPosition`). Trivial UX win; verify behavior on Safari.

### 5f. `ncryptsec` (NIP-49) import

Accept `ncryptsec1…` paste in the key input (decrypt-on-import via
nostr-tools' `nip49`, no new dependencies). QR-code scanning is **deferred**
— it would add `qrcode-parser`/`react-qr-code`, against our
no-new-dependencies policy.

### 5g. Open the options page on first install

`browser.runtime.onInstalled` with `reason === 'install'` →
`browser.runtime.openOptionsPage()`. Onboarding win, one line.

### 5h. (Deferred) `nip60.signSecret`

Lives only on upstream's unmerged `nip60` branch:
`window.nostr.nip60.signSecret(secret)` — signs Cashu NUT-10 proof secrets
(NIP-60 wallets) with the nsec (~40 lines, `@noble/*` comes transitively via
nostr-tools). Small and self-contained, but niche and unmerged upstream —
port only if Cashu wallet clients adopt the convention.

## Conscious divergences (keep ours)

- **`getRelays` + relay management UI** — upstream removed it; NIP-07 marks
  it optional but clients use it. Keep.
- **Prompt queue over single mutex** — our persisted `PromptManager` queue
  survives MV3 worker restarts (proposal 3c); upstream's in-memory mutex
  doesn't.
- **Multi-profile, PIN → passkey path, time-boxed grants, TypeScript** —
  the fork's identity; not negotiable.

## Interactions with other proposals

- **Proposal 2**: 5a is 2c's data model; 5d belongs with 2a; 5c answers
  2's audit open question.
- **Proposal 3**: 5c's notification permission goes into the per-target
  manifest template; 5e needs Safari verification.
