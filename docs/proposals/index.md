# Proposals

Design proposals for evolving keystr. These are design notes, not committed
roadmaps — each needs discussion before implementation.

1. [Passkey key protection](01-passkey-key-protection.md) — replace PINs with
   hardware-backed passkeys; never store the nsec in plaintext.
2. [Batching & policies](02-batching-and-policies.md) — batched request UI,
   human-readable prompts, and policy-based auto-approval.
3. [Cross-browser Manifest V3](03-cross-browser-mv3.md) — one codebase,
   per-target builds for Chrome, Firefox, and Safari.
4. [Remote signer mode (NIP-46)](04-remote-signer-nip46.md) — the extension as
   a Nostr Connect bunker; keys live in one place, other devices sign remotely.
5. [Upstream feature ports](05-upstream-feature-ports.md) — cherry-pick the
   best of fiatjaf/nos2x: kind-scoped policies, `peekPublicKey`, notification
   audit, ncryptsec import.

## Related reading

- [NIP-07](https://github.com/nostr-protocol/nips/blob/master/07.md) — window.nostr
- [NIP-46](https://github.com/nostr-protocol/nips/blob/master/46.md) — remote signing (long-term direction)
- [WebAuthn PRF extension](https://w3c.github.io/webauthn/#prf-extension)
- FROST threshold signing (Frostr) — future multi-device key splitting
