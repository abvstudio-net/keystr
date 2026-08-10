# Proposals

Design proposals for evolving nos2x-fox. These are design notes, not committed
roadmaps — each needs discussion before implementation.

1. [Passkey key protection](01-passkey-key-protection.md) — replace PINs with
   hardware-backed passkeys; never store the nsec in plaintext.
2. [Batching & policies](02-batching-and-policies.md) — batched request UI,
   human-readable prompts, and policy-based auto-approval.

## Related reading

- [NIP-07](https://github.com/nostr-protocol/nips/blob/master/07.md) — window.nostr
- [NIP-46](https://github.com/nostr-protocol/nips/blob/master/46.md) — remote signing (long-term direction)
- [WebAuthn PRF extension](https://w3c.github.io/webauthn/#prf-extension)
- FROST threshold signing (Frostr) — future multi-device key splitting
