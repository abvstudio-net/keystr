# Proposal 1: Passkeys replace PINs; nsec never stored in plaintext

## Problem

- The optional PIN feature encrypts stored keys with AES-GCM-256, but a PIN is a
  weak, phishable, brute-forceable secret. It also lives entirely in software.
- Without a PIN, the nsec is stored in `browser.storage` effectively
  unprotected — readable by any process or person with profile access.

## Proposal

Use **WebAuthn/passkeys (hardware-backed 2FA)** as the key-protection mechanism
instead of a PIN:

1. **Enrollment**: the user registers a passkey (YubiKey, Touch ID, Windows
   Hello, Android/iOS authenticator). Using the WebAuthn **PRF extension**
   (`hmac-secret` on FIDO2 devices), the authenticator produces a stable secret
   that is used to derive an AES-GCM-256 wrapping key.
2. **Storage**: the nsec is stored **only encrypted** with the wrapping key.
   Plaintext nsec never touches `browser.storage`, ever.
3. **Unlock**: signing/decryption requires a fresh passkey assertion
   (touch/biometric) to re-derive the wrapping key, decrypt the nsec in memory,
   use it, and discard it. Cache lifetime (currently 10 min for PIN) becomes a
   tunable policy — possibly "never cache" for high-security users.
4. **Migration**: existing PIN-encrypted and plaintext keys are migrated on
   first run: the user is prompted to enroll a passkey and re-wrap keys. PIN
   mode may be kept as a fallback or removed (decision needed).

## Open questions / challenges

- **WebAuthn from an extension**: `navigator.credentials.create/get` requires a
  secure context with an `rpId`. Extension pages can use WebAuthn, but the
  relying-party story (which origin the credential binds to) needs a spike —
  likely an extension page with its own origin as RP ID.
- **PRF extension support**: supported on YubiKeys (`hmac-secret`) and platform
  authenticators, but not universal. Fallback behavior for authenticators
  without PRF: gate decryption behind an assertion + a separate stored salt?
  (Weaker — needs care.)
- **Service-worker lifecycle**: decrypted key material cannot survive worker
  restarts; `browser.storage.session` could hold the decrypted key for the
  cache window (MV3-safe equivalent of the current PIN cache).
- **Multiple devices**: passkeys are device-bound (synced passkeys aside).
  Users with two machines need per-device enrollment or a synced passkey.

## Non-goals

Passkeys **cannot** sign Nostr events directly (WebAuthn = ECDSA/P-256; Nostr =
Schnorr/secp256k1; keys are RP-bound and non-exportable). The passkey is a
*wrapper* around the nsec, never the identity itself.
