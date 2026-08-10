# Proposal 2: Batched requests, cleaner prompts, policy-based approval

## Problem

Nostr clients fire many `signEvent` / `nip04.decrypt` / `nip44.decrypt` calls
in bursts. Each spawns a separate prompt showing raw event JSON. Users get
fatigued, stop reading, and click "approve" blindly — defeating the permission
system entirely.

## Proposal

Three coordinated improvements:

### 2a. Human-readable prompt rendering

Render pending requests semantically instead of as raw JSON:

- "Publish a note: 'gm frens ☕'" (kind 1)
- "Decrypt a DM from npub1xyz…" (`nip44.decrypt`)
- "Overwrite your contact list (312 follows)" (kind 3)
- "Update your relay list" (kind 10002)

Dangerous kinds (0, 3, 10002, key-related) get visually distinct, louder
treatment.

### 2b. Batched request UI

`PromptManager` already queues prompts; extend it to present a **single prompt
containing the pending batch**: a scrollable list with per-item approve/deny,
plus approve-all / deny-all. Optionally support a non-standard
`window.nostr.signEvents(events[])` batch method for clients that opt in.

### 2c. Policy-based auto-approval

Add per-site, per-operation policies with scope and expiry:

- **Always allow**: e.g., nip44 decrypt for kind 4/44 DMs, sign kind 7
  reactions — low-risk, high-frequency operations.
- **Always prompt**: e.g., signing kind 1 notes, kind 0 metadata, kind 3
  contact lists, anything key-related — high-impact operations the user must
  see every time.
- **Time-boxed grants**: "allow kind 1 for this site for the next hour."

Policy UI lives in the Options page (per-site permission list) and in the
prompt itself ("Always allow this for this site" checkbox with sensible
defaults per kind).

## Design principle

Prompt once per *trust decision*, not once per *action* (the passkey model:
enroll once, act freely within scope). Per-event prompts should be reserved
for high-impact operations.

## Open questions

- Sensible default policy matrix (which kinds default to allow/prompt/deny).
- Batch method: propose as a NIP, or ship as an extension-specific extra first?
- How to revoke/audit policies (a log of auto-approved signatures?).
