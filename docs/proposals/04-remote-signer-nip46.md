# Proposal 4: Remote signer mode (NIP-46)

## Problem / direction

Today keystr is a **local** signer: the nsec lives in the browser profile
and is only usable by web pages in that browser via `window.nostr`
(NIP-07). Consequences:

- Every device needs its own copy of the nsec — copied around via
  plaintext-ish channels, each copy another thing to protect.
- Mobile apps and other browsers can't use the identity unless the key is
  imported there too.
- The PIN (and proposed passkey) protection only guards the _local_ copy.

The long-term shape: **the extension becomes the single place the key lives**
(passkey-wrapped per proposal 1), and everything else talks to it remotely
via [NIP-46](https://github.com/nostr-protocol/nips/blob/master/46.md)
(Nostr Connect / remote signing, kind 24133 over relays).

## Proposal

Two complementary modes, sharing one policy engine:

### 4a. Signer mode (bunker) — the differentiator

The extension acts as a NIP-46 **signer**. Other devices/clients (mobile
apps, other browsers, CLI tools) connect with a `nostrconnect://` URI or
bunker string; signing requests arrive over relays, the user approves them
in the existing prompt UI, responses go back over relays. The nsec never
leaves the extension.

- Pairing UX in the options page: show `nostrconnect://` URI + QR code;
  incoming connection requests get an approval prompt (reuse `prompt.tsx`).
- New background module (e.g. `nip46Signer.ts`): relay pool management and
  kind-24133 request handling. `nostr-tools` ^2.7 already ships a `nip46`
  module — no new dependencies (verify API surface at implementation time).
- Requests are encrypted with NIP-44 conversation keys — reuses the existing
  `nip44` code paths and `secretsCache`.
- **Per-client policies**: keyed by the client's pubkey, mirroring the
  per-site permission levels in `common.ts` (`PERMISSIONS_REQUIRED`) and the
  policy engine from proposal 2c. "This phone app may sign kind 1 and kind 7,
  must prompt for kind 3."

### 4b. Client mode (proxy) — the thin extension

Optionally, `window.nostr` calls are **proxied to an external bunker**
(e.g. a hardware-bound signer, a home server, or the extension's own signer
mode on another machine). The browser then holds no key at all; the
extension becomes UI + policy layer + NIP-46 client. This is the answer for
users who don't want key material in any browser.

### Suggested order

Signer mode (4a) first — it makes the extension the hub and compounds with
passkeys (proposal 1): the strongest-protected copy of the key becomes the
one everyone else uses. Client mode (4b) is a smaller follow-up.

## Architecture notes

- Both modes sit **behind the existing permission layer**: NIP-46 request
  types (`sign_event`, `nip04_decrypt`, `nip44_decrypt`, …) map onto the
  same `PERMISSIONS_REQUIRED` levels used for NIP-07 calls, so prompts,
  batching (proposal 2b), and policies (proposal 2c) apply uniformly.
- Combined with proposal 1, high-risk remote operations can require a fresh
  passkey assertion — the "touch your key to sign remotely" model.

## Open questions / challenges

- **Awake problem**: the extension is not a server. Browser closed (or MV3
  service worker asleep, see proposal 3) = signer offline. Kind 24133 events
  are ephemeral; requests sent while asleep are missed unless relays retain
  them and we catch up with a `since` filter on reconnect. Decide:
  best-effort catch-up, "signer is offline" signaling to clients, or
  documented expectation that the signer machine stays on. MV3 lifecycle
  (WebSocket keepalive vs. worker idle timeout) must be validated in the
  proposal-3 spike.
- **Relay trust & metadata**: requests/responses are encrypted, but relays
  see timing and pubkey pairs. Allow per-session relay selection; consider
  dedicated/private relays for signing traffic.
- **Request authentication**: a leaked `nostrconnect://` URI or a spammy
  relay means arbitrary connection attempts — the pairing approval +
  per-client pubkey policies are the mitigation; rate-limit unknown clients.
- **This is the mobile story**: Firefox Android support was dropped
  (proposal 3). Signer mode replaces it — mobile Nostr clients (Amethyst
  et al.) already speak NIP-46, so the phone connects to the desktop
  extension instead of needing a ported extension. Better security (one
  key copy, passkey-wrapped) and testable with any NIP-46 mobile client.
- **Session auditability**: with remote signing, a log of what was signed
  for which client (ties into proposal 2's audit question) becomes
  important.

## Non-goals

- Not a hosted bunker service — no server component, no key custody.
- Not FROST/threshold signing — Frostr-style key splitting remains a
  possible later step for multi-device redundancy.

## Interactions with other proposals

- **Proposal 1 (passkeys)**: the key being remotely requested is exactly the
  key that should be hardware-wrapped; fresh-assertion policies gate remote
  ops.
- **Proposal 2 (batching & policies)**: remote clients are first-class
  policy subjects alongside web origins; bursts from a mobile app hit the
  same batched prompt UI.
- **Proposal 3 (MV3)**: the service-worker lifecycle design (3c) is a
  prerequisite for a reliable always-on signer.
