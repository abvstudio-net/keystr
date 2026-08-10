# TODO

Next work items, in priority order. Each links to the design proposal that
covers it ([docs/proposals/](docs/proposals/)); check items off as they land.

## Up next

- [ ] **1. Fix lint tooling** — the eslint setup is broken: `.eslintrc.json`
      is legacy format but the repo has eslint 9, and `eslint-plugin-babel`
      is incompatible with flat config. Migrate to `eslint.config.js`, add
      `lint` / `format` / `check` scripts to `package.json`, and run them in
      CI. (Found while dropping Firefox Android support; no proposal —
      housekeeping.)

- [ ] **2. Kind-scoped accept/reject policies** —
      [proposal 5a](docs/proposals/05-upstream-feature-ports.md) (implements
      [2c](docs/proposals/02-batching-and-policies.md)). Adopt upstream's
      `policies[host][accept][type].conditions.kinds` model merged with our
      time-boxed grants; add "authorize/reject kind N forever" buttons to the
      prompt; migrate existing permissions on first run. Biggest standalone
      security/UX win, and it works on the current MV2 base.

- [ ] **3. Upstream quick wins** — proposal
      [5b](docs/proposals/05-upstream-feature-ports.md) `peekPublicKey`,
      5c notification audit log, 5e centered prompt windows, 5g open options
      on first install, 5f `ncryptsec` paste import. One PR-sized batch of
      small, independent features.

- [ ] **4. MV3 groundwork** —
      [proposal 3, phases 1–2](docs/proposals/03-cross-browser-mv3.md).
      Manifest templating with per-target output dirs
      (`dist/firefox|chrome|safari`), then the behavior-preserving state
      refactor: PIN cache → `storage.session`, storage-keyed prompt
      resolution, drop `'unsafe-eval'` from the CSP. Ships as MV2 still;
      de-risks the switch.

- [ ] **5. MV3 for all targets** — proposal 3, phase 3. Service worker on,
      Chrome Web Store submission, `safari-web-extension-converter` spike +
      Safari release runbook.

## Later (needs discussion / depends on the above)

- [ ] Passkey key protection — [proposal 1](docs/proposals/01-passkey-key-protection.md)
      (WebAuthn PRF spike per browser; depends on item 4's `storage.session`
      design).
- [ ] Human-readable prompts + batched request UI + result-in-prompt —
      proposals [2a/2b](docs/proposals/02-batching-and-policies.md) and 5d.
- [ ] Remote signer mode (NIP-46) —
      [proposal 4](docs/proposals/04-remote-signer-nip46.md); depends on MV3
      lifecycle work (item 4–5) and ideally passkeys.
