## Code Review Results

**Scope:** Standalone audit of current `main` branch codebase (354 source files, ~268k LOC including build artifacts; Rust + Svelte/TypeScript)
**Intent:** Identify security vulnerabilities, correctness bugs, and development-velocity blockers; assess what AI and human contributors need to ship quality code
**Mode:** interactive

**Reviewers:** security, correctness, frontend, dev-velocity
- security -- cryptography, key handling, EVM, attachments, SSRF, Nostr/MLS protocol security
- correctness -- Rust async/concurrency, global state, MLS protocol bugs, message storage
- frontend -- Svelte stores, Tauri bridge, component lifecycle, async UI races
- dev-velocity -- CI quality gates, test coverage, docs, AI onboarding, project hygiene

No fixes were applied during this review; the findings below are the handoff for the team to prioritize.

### Triage Groups

| Group | Findings | Context | Preferred Resolution | Why |
|-------|----------|---------|----------------------|----|
| Cryptography and key-handling security | #1, #2, #4, #5, #6, #10 | All secrets flow through a small set of crypto helpers and global caches; flaws here compromise every account. | Fix the global ENCRYPTION_KEY and hardcoded salt first (#1, #2), then replace unsafe from_utf8_unchecked (#4), then require re-auth for exports (#5), then move secret derivation into Rust (#6). | These are the highest-severity findings and share the same root cause: secrets are exposed, cached, or derived unsafely. |
| MLS protocol correctness | #7, #8, #9, #13, #19, #20, #21, #22, #23, #26, #27, #28 | Group creation, keypackage publishing, welcome handling, and sync cursors have multiple interlocking gaps. | Implement the keypackage stub (#7), fix the kind collision (#8), clean up pending message rows (#9), then verify relay confirmation before committing group mutations (#13, #19, #20). | MLS is the core group messaging primitive; these bugs break group membership, duplicate messages, or leave stale state. |
| Message/event storage and deduplication | #9, #11, #14, #15, #24, #25, #29, #30, #31 | Flat event storage and in-memory state are not always consistent. | Fix pending row cleanup (#9), then make deduplication atomic (#11), then fix reaction/edit ordering (#24, #25) and the pending ID collision (#14). | These bugs produce duplicate messages, lost reactions, or out-of-order edits in normal chat usage. |
| EVM and transaction safety | #6, #32, #33, #39, #40 | Signing and transaction commands lack guardrails and explicit chain context. | Add user confirmation for signing (#6) first, then enforce chain_id on every transaction (#32), then validate addresses before use (#33). | Funds are at risk; the P1 confirmation finding is the biggest exposure. |
| Network and attachment SSRF / path traversal | #12, #16, #17, #18, #34, #35 | User-supplied URLs and attachment metadata drive filesystem writes and HTTP requests. | Add SSRF protection to net.rs downloads (#16) and image_cache (#18), then sanitize attachment extensions (#12), then normalize relay URLs (#17). | These are abuse vectors that can write outside the app directory or probe internal services. |
| CI and quality gates | #36, #37, #38, #43, #44, #45, #46, #47 | The project ships cross-platform releases without pre-merge validation. | Add a quality-gates job to CI (#36), then enable PR builds (#37), then remove the MCP bridge from release deps (#38), then fix the updater endpoint (#43). | Without CI gates, the highest-impact fixes above cannot be verified automatically. |
| Frontend architecture and store lifecycle | #3, #48, #49, #50, #51, #52, #53, #54, #55, #56, #57, #58, #59, #60, #61, #62 | Large components, uncancellable loads, and store cross-talk create a brittle shell. | Reset the profiles store on logout (#48), validate Tauri payloads (#49), then decompose the largest components (#50, #51, #52, #53, #54) and fix the DM load races (#55). | These affect user-visible reliability and make the codebase hard for AI and humans to reason about. |
| Project hygiene and AI onboarding | #42, #63, #64, #65, #66, #67, #68, #69, #70, #71, #72, #73, #74 | Legacy references, missing docs, and tooling gaps slow contributors. | Fix the ALCHEMY_ env prefix leak (#63), remove legacy-fix docs (#64), update deep-link scheme (#65), add an AI onboarding doc (#67), and add a justfile (#68). | These are low-risk but high-leverage for development velocity. |

### P0 -- Critical

| # | File | Issue | Reviewer | Confidence |
|---|------|-------|----------|------------|
| 1 | `src-tauri/src/crypto.rs:71` | Hardcoded Argon2 salt weakens key derivation | security | 100 |
| 2 | `src-tauri/src/crypto.rs:88` | Global ENCRYPTION_KEY is shared across accounts and pinned on first use | security | 100 |
| 3 | `src-tauri/src/mls.rs:160` | MLS keypackage publishing is a no-op stub | correctness | 100 |

- **#1** — A fixed, public salt means identical passwords produce identical keys across every install. An attacker who extracts the encrypted database can pre-compute or share rainbow tables, turning password guessing into a fast offline attack instead of a per-user brute-force. Generate a random 16+ byte salt on first install, store it in the OS keychain or a dedicated file outside the encrypted database, and pass it to argon2.hash_password_into.
- **#2** — internal_encrypt uses the first key it ever sees (from a PIN or Argon2) for all subsequent encryption, and stores it in a global OnceCell. Switching accounts, rotating PINs, or creating a second account does not rotate the key, so all accounts' secrets are encrypted under the same key derived from the first PIN used in the process lifetime. Scope the encryption key per account and derive it from the account-specific PIN each time; do not cache it globally. If caching is needed for performance, use a key cache keyed by active account and clear it on logout/switch.
- **#3** — Other devices cannot add this device to MLS groups because keypackages are never generated, published, or indexed. Group creation that depends on this device will fail silently or create groups with missing members. Implement real nostr-mls keypackage generation, publish the event to TRUSTED_RELAYS, and store the reference in the mls_keypackage_index table. Add tests that verify the published event is fetchable and indexed.

### P1 -- High

| # | File | Issue | Reviewer | Confidence |
|---|------|-------|----------|------------|
| 4 | `.github/workflows/main.yaml:1` | CI only builds releases; no test, lint, typecheck, or clippy gates | dev-velocity | 100 |
| 5 | `.github/workflows/main.yaml:3` | No CI runs on pull requests | dev-velocity | 100 |
| 6 | `src-tauri/src/account_manager.rs:478` | SQLite database is not encrypted at rest | security | 100 |
| 7 | `src-tauri/src/crypto.rs:159` | unsafe String::from_utf8_unchecked on decrypted ciphertext | security | 100 |
| 8 | `src-tauri/src/db.rs:2164` | Export of recovery phrase and EVM keys requires no re-authentication | security | 100 |
| 9 | `src-tauri/src/db.rs:3493` | Pending message leaves stale row after ID update | correctness | 100 |
| 10 | `src-tauri/src/lib.rs:4487` | EVM signing and transaction commands have no user confirmation | security | 100 |
| 11 | `src-tauri/src/stored_event.rs:45` | MLS welcome and keypackage share kind value 443 | correctness | 100 |
| 12 | `src-tauri/src/util.rs:300` | Hex decoder silently corrupts invalid input | correctness | 100 |
| 13 | `src/lib/utils/clear-account-state.ts:135` | Profiles store not reset on logout / account switch | frontend | 100 |
| 14 | `docs/plans/2026-06-27-001-feat-self-correcting-ai-testing-plan.md:1` | No integration or end-to-end test harness for the Tauri command contract | dev-velocity | 95 |
| 15 | `src-tauri/Cargo.toml:1` | Rust backend has only ~62 tests across 63 source files; high-risk modules lack coverage | dev-velocity | 90 |
| 16 | `src-tauri/tauri.conf.json:50` | Auto-updater endpoint points to legacy `covenant-application` repository | dev-velocity | 90 |
| 17 | `src-tauri/src/lib.rs:1550` | Message deduplication is non-atomic across DB and state | correctness | 75 |
| 18 | `src-tauri/src/mls.rs:580` | MLS add-member merges commit before publish succeeds | correctness | 75 |
| 19 | `src-tauri/src/mls.rs:1850` | MLS send marks success without relay confirmation | correctness | 75 |

- **#4** — CI only builds releases; no test, lint, typecheck, or clippy gates Add a `quality-gates` job before `publish-tauri` that runs `pnpm check`, `pnpm lint`, `pnpm test`, `cargo test`, and `cargo clippy -- -D warnings`; make the publish job `needs` it.
- **#5** — No CI runs on pull requests Add `pull_request:` to the `on:` trigger, or create a separate `pr.yaml` workflow that runs the same quality gates.
- **#6** — The profile database is opened with plain rusqlite::Connection::open. While individual secret fields are encrypted, message metadata, contact graphs, profile fields, group membership, allowlists, and event tags are stored in plaintext. Anyone with filesystem access can read this sensitive data without knowing the PIN. Enable SQLCipher (or libsql with encryption) for the bundled rusqlite build, or use a SQLCipher-backed connection, and derive the database key from the account PIN.
- **#7** — internal_decrypt uses String::from_utf8_unchecked on plaintext produced by ChaCha20-Poly1305. Authenticated encryption only guarantees integrity, not that the original plaintext was valid UTF-8. A corrupted or maliciously modified ciphertext can trigger undefined behavior and panic paths. Replace with fallible String::from_utf8 and return a DecodingError; treat non-UTF-8 plaintext as an integrity failure.
- **#8** — get_seed and export_evm_account_key_plaintext only check that the cached global ENCRYPTION_KEY is in memory. If a session is already unlocked, any code that can invoke these commands can extract the recovery phrase or EVM private key without re-entering the PIN. This violates the expectation that exporting secrets is a high-risk action. Add a mandatory `password: Option<String>` parameter to both export commands and require a fresh PIN/password before decrypting and returning secrets.
- **#9** — When a pending message is sent, its ID changes from a synthetic pending ID to the real event ID. save_event uses INSERT OR IGNORE, so the old pending=true row remains in the events table. After restart the UI loads both the pending and sent rows, showing duplicate messages. After confirming a send, DELETE the row whose id = pending_id, or use a single stable row ID from creation by updating the pending event in place instead of inserting a new row.
- **#10** — sign_evm_hash, wallet_build_and_send_transaction, evm_send_advanced_contract_call, and evm_send_squad_allowlisted_contract_call all sign arbitrary data or send funds once the app is unlocked. There is no biometric, PIN, or explicit confirmation step. A malicious frontend script or compromised webview can drain funds or sign any 32-byte hash (e.g., an asset transfer authorization). Add a user-confirmation Tauri dialog (or OS-level confirmation) for all signing and sending commands; require PIN for the first use per session and clear the confirmation after a timeout.
- **#11** — Code that branches on event_kind::MLS_WELCOME versus event_kind::MLS_KEY_PACKAGE cannot distinguish the two event types because both constants evaluate to 443. This causes mis-routing and mis-storage of welcome vs keypackage events. Assign distinct kind values (e.g., keep MLS_WELCOME as 443 and move MLS_KEY_PACKAGE to a separate value such as 444), then audit all match sites that reference these constants.
- **#12** — hex_string_to_bytes maps non-hex characters to 0 and ignores trailing characters on odd-length strings. A malformed encryption key, group ID, or EVM address will be accepted as a different value, causing wrong decryption keys, wrong group membership, or wrong addresses. Change the function signature to return Result<Vec<u8>, String>, reject non-hex characters, and require an even-length input. Update all callers to handle the error.
- **#13** — Profiles store not reset on logout / account switch Add `profiles.set({})` and `profileLoadingStates.set({})` to `clearAccountState` so the next account cannot see the previous account's cached Nostr profiles.
- **#14** — No integration or end-to-end test harness for the Tauri command contract Implement the planned two-phase harness (Playwright mock build + WebdriverIO real Tauri) or add a minimal Rust integration test suite using `tauri::test` and a small Tauri command smoke test.
- **#15** — Rust backend has only ~62 tests across 63 source files; high-risk modules lack coverage Prioritize unit tests for `mls.rs`, `message.rs`, `chat.rs`, `account_manager.rs`, `profile_sync.rs`, `evm/wallet_ops.rs`, `evm/advanced_contract_call.rs`, `voice.rs`, and `net.rs`; consider a test coverage badge in CI.
- **#16** — Auto-updater endpoint points to legacy `covenant-application` repository Update the updater endpoint to `https://github.com/covenant-gov/pacto-app/releases/latest/download/latest.json` and validate the public key matches the signing secret used in CI.
- **#17** — Two concurrent giftwraps for the same message can both pass the message_exists_in_db check before either is saved, then both be added to in-memory state. The duplicate ID check inside Chat::internal_add_message only protects the local Vec, not the DB, and cannot protect cross-task races. Acquire a per-message or per-chat async lock around the check-add-save sequence, or make insertion conditional on the DB INSERT result and reconcile failures by rolling back the in-memory add.
- **#18** — add_member_device calls engine.merge_pending_commit after client.send_event returns. If the evolution event fails to publish (e.g., all relays reject it), the local group advances to the next epoch while other members remain in the previous epoch, permanently breaking group state consistency. Confirm the evolution event was accepted by at least one relay before merging; on failure, leave the pending commit unmerged and return an error so the caller can retry.
- **#19** — client.send_event_to returns an Output that contains success and failed relay lists. The code only matches Ok/Err, so an Ok result with an empty success list (all relays rejected the message) is treated as a successful send. The UI shows sent but the message never reached the network. Inspect the Output's success list; if it is empty, treat the send as a failure, mark the message failed, and retry or surface the error to the user.

### P2 -- Moderate

| # | File | Issue | Reviewer | Confidence |
|---|------|-------|----------|------------|
| 20 | `.github/workflows/main.yaml:107` | Release body is a placeholder and every release is published as a draft | dev-velocity | 100 |
| 21 | `src-tauri/src/crypto.rs:38` | Panic paths on attacker-controlled or runtime data in crypto and messaging | security | 100 |
| 22 | `src-tauri/src/lib.rs:3961` | Login/create account commands return plaintext secrets to frontend | security | 100 |
| 23 | `src-tauri/src/message.rs:540` | Runtime panic in message attachment save path | correctness | 100 |
| 24 | `src-tauri/src/mls.rs:1050` | MLS sync processed counter double-increments | correctness | 100 |
| 25 | `src-tauri/src/net.rs:80` | Attachment download lacks SSRF protection | security | 100 |
| 26 | `src-tauri/src/profile_sync.rs:230` | Profile sync queue caches failures as successes | correctness | 100 |
| 27 | `src-tauri/src/stored_event.rs:120` | MESSAGE_EDIT is not a known kind in StoredEvent | correctness | 100 |
| 28 | `src/lib/app/post-login-sync.ts:11` | post-login sync fires uncoordinated fire-and-forget tasks | frontend | 100 |
| 29 | `src/lib/app/tauri-subscriptions.ts:75` | Tauri event payloads cast with `as` instead of validated | frontend | 100 |
| 30 | `src/routes/+page.svelte:515` | Svelte 5 project still uses legacy reactive statements with side effects | frontend | 100 |
| 31 | `src/routes/+page.svelte:713` | loadOlder advances offset by page size even when fewer messages are returned | frontend | 100 |
| 32 | `src/stores/auth.ts:96` | Auth errors surfaced as raw error.message strings | frontend | 100 |
| 33 | `src/stores/dm.ts:260` | reconcilePeerThreadInvites mutates another store inside `update` | frontend | 100 |
| 34 | `src/stores/profiles.ts:218` | loadProfile uses a fixed 500ms sleep for background fetch | frontend | 100 |
| 35 | `src/stores/safe.ts:80` | safe.ts asserts unsafe address string as viem Address | frontend | 100 |
| 36 | `eslint.config.js:1` | ESLint config uses only recommended presets and there is no formatting script or Prettier config | dev-velocity | 95 |
| 37 | `src-tauri/Cargo.toml:100` | `tauri-plugin-mcp-bridge` is a non-optional dependency in Cargo.toml despite being described as debug-only | dev-velocity | 95 |
| 38 | `vite.config.ts:14` | `ALCHEMY_` env prefix exposes the backend RPC secret to the webview bundle | dev-velocity | 95 |
| 39 | `vite.config.ts:42` | Frontend coverage thresholds are set but not enforced by CI; Svelte components are excluded from coverage | dev-velocity | 95 |
| 40 | `docs/README.md:1` | No tracked changelog or release notes file | dev-velocity | 90 |
| 41 | `docs/legacy-fixes/CATALOG.md:1` | Alpha-only legacy-fix docs and repair code still live in the codebase | dev-velocity | 90 |
| 42 | `src-tauri/src/lib.rs:6174` | Deep-link handler still references legacy `vector://` and `vectorapp.io` schemes | dev-velocity | 90 |
| 43 | `.cursor/rules/brief-inline-commentary.mdc:1` | `.cursor/rules` is missing critical policies for lock discipline, tests, env vars, and global state | dev-velocity | 85 |
| 44 | `docs/README.md:1` | No dedicated AI onboarding doc for adding Tauri commands, stores, or EVM/MLS features | dev-velocity | 85 |
| 45 | `src-tauri/src/account_manager.rs:1260` | Account switch races between account and ID caches | correctness | 75 |
| 46 | `src-tauri/src/db.rs:3128` | SQLite connection not returned on query error | correctness | 75 |
| 47 | `src-tauri/src/db.rs:4070` | Edit timestamps lack millisecond precision | correctness | 75 |
| 48 | `src-tauri/src/evm/wallet_ops.rs:370` | EVM transactions lack explicit chain_id | correctness | 75 |
| 49 | `src-tauri/src/lib.rs:1566` | Auto-accept of squad channel invites based on DM content only | security | 75 |
| 50 | `src-tauri/src/lib.rs:1600` | Giftwrap unwrap failures are silently dropped | correctness | 75 |
| 51 | `src-tauri/src/lib.rs:1904` | MLS live messages do not advance sync cursor | correctness | 75 |
| 52 | `src-tauri/src/message.rs:300` | Pending message ID can collide on rapid sends | correctness | 75 |
| 53 | `src-tauri/src/message.rs:530` | Attachment plaintext written before dedup/reuse decision | correctness | 75 |
| 54 | `src-tauri/src/mls.rs:350` | MLS group creation can omit members silently | correctness | 75 |
| 55 | `src-tauri/src/mls.rs:640` | Local MLS group metadata deleted before leave commit is published | correctness | 75 |
| 56 | `src-tauri/src/mls.rs:1750` | MLS operations re-enter Tokio runtime from blocking pool | correctness | 75 |
| 57 | `src-tauri/src/rumor.rs:341` | Attachment filename extension from attacker-controlled tag enables path traversal | security | 75 |
| 58 | `src-tauri/src/rumor.rs:430` | Reactions may reference wrong message with multiple e-tags | correctness | 75 |
| 59 | `src/components/channel/ChatView.svelte:205` | ChatView membership-version reactive reloads members repeatedly | frontend | 75 |
| 60 | `src/components/channel/ChatView.svelte:284` | ChatView sends all non-inbox messages with virtualBucket 'announcements' | frontend | 75 |
| 61 | `src/components/dm/DmThread.svelte:110` | DmThread side-effect reactive block processes wallet grants on every message update | frontend | 75 |
| 62 | `src/components/settings/ExportAllSecretsModal.svelte:44` | ExportAllSecretsModal keeps decrypted secrets in component memory | frontend | 75 |
| 63 | `src/routes/+page.svelte:620` | DM load in +page.svelte is uncancellable and races on rapid conversation switches | frontend | 75 |
| 64 | `src/stores/profiles.ts:87` | Module-level Tauri listeners in profiles.ts can stack across HMR | frontend | 75 |

- **#34** — loadProfile uses a fixed 500ms sleep for background fetch Replace the sleep with a promise-based waiter keyed by npub (e.g. backend resolves when the profile is ready, or a timeout with exponential backoff).
- **#35** — safe.ts asserts unsafe address string as viem Address Validate `entry.safeAddress` with `isAddress` before passing it to `getSafeState`; return early or record an error if invalid.
- **#39** — Frontend coverage thresholds are set but not enforced by CI; Svelte components are excluded from coverage Add a CI step that runs `pnpm test:coverage` and gates on the 80% thresholds; add component-level tests or remove the `src/**/*.svelte` exclusion so coverage reflects real UI code.
- **#40** — No tracked changelog or release notes file Add a `CHANGELOG.md` at the repo root and update the CI workflow to populate `releaseBody` from it.
- **#42** — Deep-link handler still references legacy `vector://` and `vectorapp.io` schemes Update the single-instance deep-link filter to `pacto://` or the current product domain; update the `TRUSTED_RELAYS` entries that still point to `asia.vectorapp.io` if they are meant to be Pacto relays.
- **#43** — `.cursor/rules` is missing critical policies for lock discipline, tests, env vars, and global state Add `.cursor/rules` files: keep `pnpm-lock.yaml` and `Cargo.lock` in sync; add tests for every new Tauri command; new Vite env vars must be `VITE_` or `ALCHEMY_`; avoid new `lazy_static`/`OnceCell` globals.
- **#44** — No dedicated AI onboarding doc for adding Tauri commands, stores, or EVM/MLS features Create `docs/AI_ONBOARDING.md` with step-by-step checklists for adding a Tauri command, adding a store, adding an EVM contract binding, and adding an MLS feature, plus how to test each.
- **#45** — switch_account calls set_current_account before clearing and reloading the chat/user ID caches. Any concurrent DB operation that uses the current account and the cache could read stale cache entries against the newly selected account's database. Clear the caches before changing the current account, or hold a single account-switch mutex so no DB operation can run between the account change and cache reload.
- **#46** — Functions such as get_dm_sent_received return Err before returning the connection to the pool. Because the pool holds only one connection per account, repeated errors exhaust the pool and block all subsequent DB operations. Introduce a RAII guard that returns the connection on Drop, or use a closure pattern that always returns the connection before propagating the result.
- **#47** — get_message_views builds edit history using event.created_at * 1000, ignoring the ms tag. Multiple edits in the same second cannot be ordered correctly and may display out of order. Use event.timestamp_ms() for edit timestamps, consistent with how message timestamps are reconstructed.
- **#48** — wallet_build_and_send_transaction constructs a TransactionRequest without .with_chain_id. If the RPC provider is misconfigured or the signer is on a different network, the transaction can be signed for the wrong chain and rejected or, worse, replayed on an unintended network. Call .with_chain_id(net.chain_id) on the TransactionRequest before sending, and verify the returned receipt's chain_id matches.
- **#49** — handle_text_message parses any incoming DM as a channel_in_squad invitation and auto-accepts the MLS welcome if the announcements_group_id matches a known local squad. The sender is not verified to be an admin or authorized inviter of that squad. A user who has the user's npub can send a crafted DM and force the user into a channel without consent. Before auto-accepting, verify the DM sender is a member of the announcements group (or otherwise authorized) and that the channel_group_id is a valid welcome from the welcomer known to the squad.
- **#55** — leave_group calls client.send_event and then immediately removes the group from mls_groups. If the event fails to publish, the local group is gone but the network still treats the user as a member, so the user will still receive group messages and cannot rejoin cleanly. Confirm the leave proposal was accepted by at least one relay before removing local metadata; on failure, return an error and keep the group.
- **#56** — send_mls_message and the live MLS handler use tokio::task::spawn_blocking and then immediately call rt.block_on. This defeats the blocking pool, risks deadlocks, and can panic when block_on is called from within the async runtime's own threads. Refactor MlsService so that engine calls happen in a dedicated thread that communicates via channels, or make the engine itself Send-friendly so it can be used directly in async commands without spawn_blocking/block_on.
- **#57** — process_file_attachment builds the local save path from the file-type tag by calling extension_from_mime. extension_from_mime's fallback returns the raw MIME subtype, which can contain path separators (e.g., "application/../../evil"). An attacker can craft a file attachment event that writes downloaded files outside the intended vector directory. Sanitize the extension to only allow alphanumeric and a small allow-list of safe characters (e.g., [a-zA-Z0-9]), and validate the final path is still inside the resolved base directory before writing.
- **#58** — process_reaction calls rumor.tags.find(TagKind::e()) which returns the first e-tag. If the reaction event references a root event plus a reply event, the reaction may be attached to the root instead of the intended message. Select the e-tag with the 'reply' marker, or fall back to the last e-tag if no marker is present, matching NIP-25 conventions.
- **#63** — DM load in +page.svelte is uncancellable and races on rapid conversation switches Track a per-npub request token or AbortController and ignore responses for conversations that are no longer active.
- **#64** — Module-level Tauri listeners in profiles.ts can stack across HMR Register all listeners in a single async setup function with a shared `Map<string, UnlistenFn>` and call previous unlisteners before re-registering.

### P3 -- Low

| # | File | Issue | Reviewer | Confidence |
|---|------|-------|----------|------------|
| 65 | `AGENTS.md:205` | AGENTS.md references a non-existent `install.sh` at the repo root | dev-velocity | 100 |
| 66 | `src-tauri/src/lib.rs:1` | lib.rs and db.rs violate single-responsibility | correctness | 100 |
| 67 | `src/components/dm/MessengerChatView.svelte:57` | MessengerChatView optimistic message IDs can collide | frontend | 100 |
| 68 | `src/components/parent/DashboardPollsPanel.svelte:90` | DashboardPollsPanel uses unsafe cast and voids a reactive dependency | frontend | 100 |
| 69 | `package.json:5` | No Makefile, justfile, or `scripts/` directory for common development tasks | dev-velocity | 95 |
| 70 | `src-tauri/src/image_cache.rs:528` | Image cache SSRF protection is vulnerable to DNS rebinding | security | 75 |
| 71 | `src-tauri/src/lib.rs:2789` | Relay URL validation allows arbitrary paths and query strings | security | 75 |
| 72 | `src-tauri/src/rumor.rs:632` | Dashboard poll vote ingestion does not verify group membership | security | 75 |
| 73 | `src/components/dm/DmThread.svelte:378` | DmThread copies npub with raw navigator.clipboard and no fallback | frontend | 75 |
| 74 | `src/components/settings/ProfileSection.svelte:260` | ProfileSection swallows clipboard errors when copying npub | frontend | 75 |

- **#65** — AGENTS.md references a non-existent `install.sh` at the repo root Remove the stale `install.sh` reference from AGENTS.md, or add the intended `install.sh` script if it is still needed.
- **#67** — MessengerChatView optimistic message IDs can collide Include a random component in the optimistic ID, e.g. `opt-${Date.now()}-${Math.random().toString(36).slice(2)}`.
- **#68** — DashboardPollsPanel uses unsafe cast and voids a reactive dependency Give `getPollBallotMap` a typed return and remove the `as Record<string, string>` cast; also remove the unnecessary `void pollBallotRefresh`.

### Actionable Findings

| # | File | Issue | Route | Notes |
|---|------|-------|-------|-------|
| 1 | `src-tauri/src/crypto.rs:71` | Hardcoded Argon2 salt weakens key derivation | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 2 | `src-tauri/src/crypto.rs:88` | Global ENCRYPTION_KEY is shared across accounts and pinned on first use | manual -> downstream-resolver | suggested_fix present; requires verification |
| 3 | `src-tauri/src/mls.rs:160` | MLS keypackage publishing is a no-op stub | manual -> downstream-resolver | suggested_fix present; requires verification |
| 4 | `.github/workflows/main.yaml:1` | CI only builds releases; no test, lint, typecheck, or clippy gates | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 5 | `.github/workflows/main.yaml:3` | No CI runs on pull requests | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 6 | `src-tauri/src/account_manager.rs:478` | SQLite database is not encrypted at rest | manual -> downstream-resolver | suggested_fix present; requires verification |
| 7 | `src-tauri/src/crypto.rs:159` | unsafe String::from_utf8_unchecked on decrypted ciphertext | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 8 | `src-tauri/src/db.rs:2164` | Export of recovery phrase and EVM keys requires no re-authentication | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 9 | `src-tauri/src/db.rs:3493` | Pending message leaves stale row after ID update | manual -> downstream-resolver | suggested_fix present; requires verification |
| 11 | `src-tauri/src/stored_event.rs:45` | MLS welcome and keypackage share kind value 443 | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 12 | `src-tauri/src/util.rs:300` | Hex decoder silently corrupts invalid input | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 13 | `src/lib/utils/clear-account-state.ts:135` | Profiles store not reset on logout / account switch | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 14 | `docs/plans/2026-06-27-001-feat-self-correcting-ai-testing-plan.md:1` | No integration or end-to-end test harness for the Tauri command contract | manual -> downstream-resolver | suggested_fix present; requires verification |
| 15 | `src-tauri/Cargo.toml:1` | Rust backend has only ~62 tests across 63 source files; high-risk modules lack coverage | manual -> downstream-resolver | suggested_fix present; requires verification |
| 16 | `src-tauri/tauri.conf.json:50` | Auto-updater endpoint points to legacy `covenant-application` repository | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 17 | `src-tauri/src/lib.rs:1550` | Message deduplication is non-atomic across DB and state | manual -> downstream-resolver | suggested_fix present; requires verification |
| 18 | `src-tauri/src/mls.rs:580` | MLS add-member merges commit before publish succeeds | manual -> downstream-resolver | suggested_fix present; requires verification |
| 19 | `src-tauri/src/mls.rs:1850` | MLS send marks success without relay confirmation | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 20 | `.github/workflows/main.yaml:107` | Release body is a placeholder and every release is published as a draft | manual -> downstream-resolver | suggested_fix present; requires verification |
| 21 | `src-tauri/src/crypto.rs:38` | Panic paths on attacker-controlled or runtime data in crypto and messaging | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 22 | `src-tauri/src/lib.rs:3961` | Login/create account commands return plaintext secrets to frontend | manual -> downstream-resolver | suggested_fix present; requires verification |
| 23 | `src-tauri/src/message.rs:540` | Runtime panic in message attachment save path | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 24 | `src-tauri/src/mls.rs:1050` | MLS sync processed counter double-increments | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 25 | `src-tauri/src/net.rs:80` | Attachment download lacks SSRF protection | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 26 | `src-tauri/src/profile_sync.rs:230` | Profile sync queue caches failures as successes | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 27 | `src-tauri/src/stored_event.rs:120` | MESSAGE_EDIT is not a known kind in StoredEvent | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 28 | `src/lib/app/post-login-sync.ts:11` | post-login sync fires uncoordinated fire-and-forget tasks | manual -> downstream-resolver | suggested_fix present; requires verification |
| 29 | `src/lib/app/tauri-subscriptions.ts:75` | Tauri event payloads cast with `as` instead of validated | manual -> downstream-resolver | suggested_fix present; requires verification |
| 30 | `src/routes/+page.svelte:515` | Svelte 5 project still uses legacy reactive statements with side effects | manual -> downstream-resolver | suggested_fix present; requires verification |
| 31 | `src/routes/+page.svelte:713` | loadOlder advances offset by page size even when fewer messages are returned | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 32 | `src/stores/auth.ts:96` | Auth errors surfaced as raw error.message strings | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 33 | `src/stores/dm.ts:260` | reconcilePeerThreadInvites mutates another store inside `update` | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 34 | `src/stores/profiles.ts:218` | loadProfile uses a fixed 500ms sleep for background fetch | manual -> downstream-resolver | suggested_fix present; requires verification |
| 35 | `src/stores/safe.ts:80` | safe.ts asserts unsafe address string as viem Address | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 36 | `eslint.config.js:1` | ESLint config uses only recommended presets and there is no formatting script or Prettier config | manual -> downstream-resolver | suggested_fix present; requires verification |
| 37 | `src-tauri/Cargo.toml:100` | `tauri-plugin-mcp-bridge` is a non-optional dependency in Cargo.toml despite being described as debug-only | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 38 | `vite.config.ts:14` | `ALCHEMY_` env prefix exposes the backend RPC secret to the webview bundle | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 39 | `vite.config.ts:42` | Frontend coverage thresholds are set but not enforced by CI; Svelte components are excluded from coverage | manual -> downstream-resolver | suggested_fix present; requires verification |
| 40 | `docs/README.md:1` | No tracked changelog or release notes file | manual -> downstream-resolver | suggested_fix present; requires verification |
| 41 | `docs/legacy-fixes/CATALOG.md:1` | Alpha-only legacy-fix docs and repair code still live in the codebase | manual -> downstream-resolver | suggested_fix present; requires verification |
| 42 | `src-tauri/src/lib.rs:6174` | Deep-link handler still references legacy `vector://` and `vectorapp.io` schemes | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 43 | `.cursor/rules/brief-inline-commentary.mdc:1` | `.cursor/rules` is missing critical policies for lock discipline, tests, env vars, and global state | manual -> downstream-resolver | suggested_fix present |
| 44 | `docs/README.md:1` | No dedicated AI onboarding doc for adding Tauri commands, stores, or EVM/MLS features | manual -> downstream-resolver | suggested_fix present |
| 45 | `src-tauri/src/account_manager.rs:1260` | Account switch races between account and ID caches | manual -> downstream-resolver | suggested_fix present; requires verification |
| 46 | `src-tauri/src/db.rs:3128` | SQLite connection not returned on query error | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 47 | `src-tauri/src/db.rs:4070` | Edit timestamps lack millisecond precision | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 48 | `src-tauri/src/evm/wallet_ops.rs:370` | EVM transactions lack explicit chain_id | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 49 | `src-tauri/src/lib.rs:1566` | Auto-accept of squad channel invites based on DM content only | manual -> downstream-resolver | suggested_fix present; requires verification |
| 50 | `src-tauri/src/lib.rs:1600` | Giftwrap unwrap failures are silently dropped | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 51 | `src-tauri/src/lib.rs:1904` | MLS live messages do not advance sync cursor | manual -> downstream-resolver | suggested_fix present; requires verification |
| 52 | `src-tauri/src/message.rs:300` | Pending message ID can collide on rapid sends | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 53 | `src-tauri/src/message.rs:530` | Attachment plaintext written before dedup/reuse decision | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 54 | `src-tauri/src/mls.rs:350` | MLS group creation can omit members silently | manual -> downstream-resolver | suggested_fix present; requires verification |
| 55 | `src-tauri/src/mls.rs:640` | Local MLS group metadata deleted before leave commit is published | manual -> downstream-resolver | suggested_fix present; requires verification |
| 56 | `src-tauri/src/mls.rs:1750` | MLS operations re-enter Tokio runtime from blocking pool | manual -> downstream-resolver | suggested_fix present; requires verification |
| 57 | `src-tauri/src/rumor.rs:341` | Attachment filename extension from attacker-controlled tag enables path traversal | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 58 | `src-tauri/src/rumor.rs:430` | Reactions may reference wrong message with multiple e-tags | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 59 | `src/components/channel/ChatView.svelte:205` | ChatView membership-version reactive reloads members repeatedly | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 60 | `src/components/channel/ChatView.svelte:284` | ChatView sends all non-inbox messages with virtualBucket 'announcements' | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 61 | `src/components/dm/DmThread.svelte:110` | DmThread side-effect reactive block processes wallet grants on every message update | manual -> downstream-resolver | suggested_fix present; requires verification |
| 62 | `src/components/settings/ExportAllSecretsModal.svelte:44` | ExportAllSecretsModal keeps decrypted secrets in component memory | manual -> downstream-resolver | suggested_fix present; requires verification |
| 63 | `src/routes/+page.svelte:620` | DM load in +page.svelte is uncancellable and races on rapid conversation switches | manual -> downstream-resolver | suggested_fix present; requires verification |
| 64 | `src/stores/profiles.ts:87` | Module-level Tauri listeners in profiles.ts can stack across HMR | manual -> downstream-resolver | suggested_fix present; requires verification |
| 65 | `AGENTS.md:205` | AGENTS.md references a non-existent `install.sh` at the repo root | gated_auto -> downstream-resolver | suggested_fix present |
| 67 | `src/components/dm/MessengerChatView.svelte:57` | MessengerChatView optimistic message IDs can collide | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 68 | `src/components/parent/DashboardPollsPanel.svelte:90` | DashboardPollsPanel uses unsafe cast and voids a reactive dependency | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 69 | `package.json:5` | No Makefile, justfile, or `scripts/` directory for common development tasks | manual -> downstream-resolver | suggested_fix present |
| 70 | `src-tauri/src/image_cache.rs:528` | Image cache SSRF protection is vulnerable to DNS rebinding | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 71 | `src-tauri/src/lib.rs:2789` | Relay URL validation allows arbitrary paths and query strings | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 72 | `src-tauri/src/rumor.rs:632` | Dashboard poll vote ingestion does not verify group membership | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 73 | `src/components/dm/DmThread.svelte:378` | DmThread copies npub with raw navigator.clipboard and no fallback | gated_auto -> downstream-resolver | suggested_fix present; requires verification |
| 74 | `src/components/settings/ProfileSection.svelte:260` | ProfileSection swallows clipboard errors when copying npub | gated_auto -> downstream-resolver | suggested_fix present; requires verification |

### Coverage

- **Total findings:** 74 (P0: 3, P1: 16, P2: 45, P3: 10)
- **Suppressed:** 0 findings below anchor 75
- **Residual risks:** 22
  - MLS engine (mdk-core / mdk-sqlite-storage) is a git dependency at a fixed commit; its cryptography, downgrade handling, and replay protection were not audited in this pass.
  - NIP-17 giftwrap security relies entirely on nostr-sdk; key compromise of a relay or relay MITM could expose metadata even if payload content is encrypted.
  - The global MNEMONIC_SEED is held as a standard String; the Rust allocator may move or copy it, and no secure zeroization is performed after logout or key derivation.
  - No rate limiting or key-derivation throttling on PIN unlock; an attacker with local filesystem access can attempt many PINs against the Argon2-derived key.
  - No certificate pinning for RPC, Blossom, or relay endpoints; a compromised system CA or local MITM can intercept traffic or return malicious payloads.
  - The ENCRYPTION_KEY is cached for the process lifetime and never rotated; legacy data encrypted under a weak or leaked PIN cannot be re-keyed.
  - The global ChatState (STATE) is a single tokio::sync::Mutex that is acquired frequently and held across async operations. Under heavy load or with many concurrent chats this could become a bottleneck or cause head-of-line blocking; a per-chat lock or actor model would reduce contention but requires a larger refactor.
  - The in-memory WRAPPER_ID_CACHE is cleared when sync finishes. A duplicate wrapper received after the cache is cleared but before the DB dedup path is exercised could be processed twice, leading to duplicate UI entries until the DB check catches it.
  - Reaction processing in lib.rs drops reactions whose referenced message is not yet in state. The TODO at lib.rs:1810 acknowledges this, but there is no persistent queue, so reactions received ahead of their messages during a partial sync are permanently lost.
  - Network errors in net.rs and wallet_ops.rs are swallowed or converted to boolean/String without structured retry metadata. This makes it hard to distinguish transient failures from permanent ones and can lead to silent data staleness in wallet balances and attachment reuse decisions.
  - The Nostr client (NOSTR_CLIENT) is a global std::sync::RwLock. While get_nostr_client clones the Arc without holding the lock, set/clear panic on poison and there is no explicit lifetime management across account switches. A poisoned lock after a panic would render the client unreachable.
  - Tauri event name casing/hyphenation (e.g. `typing-update`) cannot be fully verified from the frontend; a mismatch with Rust would silently drop typing events.
  - Plaintext localStorage caches for wallet summaries, Safe state, and treasury infra are scoped by npub but remain unencrypted; device access can leak account metadata.
  - Module-level listeners in `src/stores/profiles.ts` are assumed to be reset by the backend restart on logout; if logout ever stops restarting the app, listeners will continue updating stores.
  - Concurrent `parent-map-disk-cache.ts` read/write cycles could interleave if multiple dashboard fetches for the same parent race.
  - Svelte 5 legacy reactive semantics may cause subtle re-run bugs in components with many `$:` side-effect blocks.
  - MCP bridge crate may still be compiled and linked into release binaries even though the code path is cfg-gated, because it is a non-optional dependency in Cargo.toml.
  - Auto-updater endpoint references the legacy `covenant-application` repository, which may cause update checks to fail or return stale artifacts.
  - Release notes are a hard-coded placeholder and every release is drafted, so users see no changelog until a human manually edits the release.
  - Legacy `vector://` and `vectorapp.io` deep-link/relay references remain in the codebase, risking brand confusion and broken protocol handlers.
  - No pre-merge CI means regressions can reach `main` and immediately trigger a cross-platform release build.
  - AI contributors rely on AGENTS.md and scattered docs; there is no single onboarding checklist for common tasks.
- **Testing gaps:** 28
  - No unit tests for crypto.rs behavior with malformed, non-UTF-8, or truncated ciphertext; unsafe/from_utf8_unchecked path is untested.
  - No tests verifying that the Argon2 salt is unique per install or that two identical PINs produce different keys.
  - No tests for account key isolation: switching accounts should not decrypt one account's secrets with another account's key.
  - No tests for path traversal in attachment extension handling (e.g., MIME subtype containing '..' or path separators).
  - No tests for SSRF in attachment downloads or net.rs; private IP ranges, DNS rebinding, and URL scheme validation are not exercised.
  - No tests for authorization on export commands (get_seed, export_evm_account_key_plaintext) requiring re-authentication.
  - No tests for EVM signing and transaction confirmation boundaries; commands can be invoked without a PIN in tests.
  - No tests for auto-accept of channel-in-squad invites verifying the sender's authorization.
  - No unit tests for ChatState message ordering, duplicate handling, or reaction application under concurrent additions.
  - No tests verifying that pending message rows are cleaned up after a successful send.
  - No tests for the MLS keypackage publish path; the stub currently makes the feature untestable.
  - No tests for sync cursor advancement after live MLS message processing.
  - No tests for the DB connection return-on-error invariant, which would catch the leak identified in get_dm_sent_received.
  - No tests that verify send_event_to success list behavior for MLS and DM sends.
  - No tests for hex_string_to_bytes malformed input behavior.
  - No integration tests for account switching that verify cache consistency and connection pool state.
  - No unit tests for `subscribeAppEvents` event routing, listener cleanup, or payload validation.
  - No tests that verify `clearAccountState` clears every account-scoped store, including `profiles` and `profileLoadingStates`.
  - No tests for DM/channel message-load cancellation and ordering when the user switches conversations rapidly.
  - No tests for export-modals clearing secrets from component memory on close and unmount.
  - No tests for `getInvokeErrorMessage` / `friendlyMessage` mapping for backend auth and DM-send errors.
  - No tests for `currentNpubForPersistence` scoping: stores must not write to localStorage when the current npub is null or changed.
  - CI does not run `pnpm test` or `cargo test`.
  - No integration tests for Tauri commands; frontend unit tests only mock `invoke` from `@tauri-apps/api/core`.
  - Svelte components are excluded from the Vitest coverage report (`src/**/*.svelte`), so UI coverage is unknown.
  - No end-to-end harness exists; the self-correcting AI testing plan is documented but not implemented.
  - High-risk Rust modules (`mls.rs`, `message.rs`, `chat.rs`, `account_manager.rs`, `evm/wallet_ops.rs`, `evm/advanced_contract_call.rs`, `voice.rs`, `net.rs`) have little or no test coverage.
  - No Cargo `[dev-dependencies]` for test helpers (e.g., tempfile, tokio-test, pretty_assertions).
- **Failed reviewers:** 0

---

> **Verdict:** Not ready
>
> **Reasoning:** 3 P0 critical findings around cryptography/key handling (hardcoded Argon2 salt, global shared encryption key, unsafe UTF-8 conversion) and multiple P1 correctness gaps in MLS/message storage mean the codebase should not ship to users without targeted fixes. Additionally, CI has no quality gates, so regressions can reach `main` and trigger releases unchecked.
>
> **Fix order:**
> 1. P0 crypto/key-handling fixes (#1, #2, #4) — before any alpha build.
> 2. P1 MLS correctness (#7, #8, #9, #13, #19, #20) and message dedup (#11) — before group messaging is considered stable.
> 3. P1 EVM confirmation (#6) and CI quality gates (#36, #37) — before any release with real funds.
> 4. P2 frontend reliability and architecture (#48, #49, #55) and project hygiene (#63, #64, #65) — iterate in parallel.
