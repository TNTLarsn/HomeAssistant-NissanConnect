# Issue #96 Implementation Plan

Status: implemented and mock-validated; controlled live command not performed

Goal: implement capability-gated Home Assistant lock/unlock for [issue #96](https://github.com/dan-r/HomeAssistant-NissanConnect/issues/96) without guessed protocol steps, leaked security material, or implicit vehicle wake-ups.

## Decision

The original-app reconstruction and implementation now cover issue #96's
complete software path:

- **Verified contract:** status UIDs `2021`/`886`, command UIDs `27`/`878`,
  SRP gates `747`/`909`, device registration, PIN enrollment, per-command
  challenge/proof, exact command payload, and final action polling.
- **Verified safety behavior:** no wake-up in the app's command path, no
  command retry or redirect following, no success without `COMPLETED`, and no
  stored or logged OTP, PIN, or SRP material.
- **Implemented Home Assistant behavior:** an opt-in, per-vehicle options flow
  and `LockEntity`, owner-only setup, transient service PIN, stable device ID,
  recovery, disable/re-enable, PIN update, and trusted-device removal.

Do not implement issue #96 as buttons. Use Home Assistant's `LockEntity` because lock/unlock are stateful device operations with an existing passive lock-status source.

## Phase 1: Land Passive Status Catalog Support (Complete)

1. Add verified UID `886` to `Feature` with a status-specific name.
2. Preserve UID `2021` and introduce a single lock-status capability predicate.
3. Apply the predicate to the passive GET and existing binary sensor.
4. Add direct parsing and capability tests for UID `2021`, UID `886`, neither, and command-only UIDs `27`/`878`.
5. Validate `tests/test_kamereon.py`, `tests/test_binary_sensor.py`, then the full suite.

This phase is independently releasable and addresses the missing Qashqai CCS2 lock-status entity.

## Phase 2: Reconstruct the Current Security Contract (Complete)

Using the verified MyNISSAN Android release, trace each stage from the lock/unlock UI call site:

1. Determine the exact app gate for UID `27` versus UID `878`, including `ccsGen`, cross-badge, region, subscription, and entitlement checks.
2. Trace device OTP generation, registration, registration-status lookup, device limits, replacement behavior, and reset/deletion calls through Retrofit interfaces and consumers.
3. Trace SRP PIN enrollment and deletion, challenge initiation, action-status polling service, final statuses, timeout behavior, and error mappings.
4. Inspect only the relevant exported native SRP getters/functions. Recover the group parameters and proof algorithm, then create fixed non-secret test vectors independently against the native implementation.
5. Trace exact lock and unlock order strings, action/target enums, request envelope, headers, token role, and whether the app issues a wake-up from the actual button path.
6. Explicitly disprove the prototype fallbacks: no speculative wake-up and no success when a response requiring completion lacks a verifiable outcome unless the app contract demonstrates that behavior.

Deliverable completed in
[`lock-capability-contract.md`](lock-capability-contract.md), including native
SRP serialization rules and a fixed independent synthetic vector.

## Phase 3: Implement the Synchronous Kamereon Layer (Complete)

1. Add active capability support for both UIDs `27` and `878` behind one semantic predicate.
2. Add narrow synchronous methods for device enrollment, registration state, SRP PIN lifecycle, challenge/proof generation, lock/unlock submission, and final action polling.
3. Keep all command methods out of `fetch_all`, `refresh`, entity properties, and coordinator listeners.
4. Route authenticated requests through the existing token-refresh path. Reject cross-origin redirects and retain TLS validation.
5. Use typed domain exceptions for unsupported capability, setup required, invalid OTP/PIN, rejected challenge, timeout, and command rejection.
6. Never log VINs, user IDs, device IDs, OTPs, PINs, salts, verifier values, ephemeral keys, proofs, action IDs, tokens, or raw responses.
7. Hold PIN and transient SRP material only as long as the proven workflow requires. Decide config-entry storage only after reviewing Home Assistant's credential-storage expectations; prefer user-supplied-on-demand or an established secure storage pattern over plaintext config data.

## Phase 4: Add Home Assistant Setup and Entity (Complete)

1. Add an explicit opt-in setup/reauth flow only for vehicles advertising UID `27` or `878`.
2. Support multiple vehicles independently; never select the first lockable vehicle implicitly.
3. Separate one-time OTP/device registration from SRP PIN creation, reset, disable, and recovery flows.
4. Add a `LockEntity` gated by active capability plus completed security setup.
5. Read `is_locked` only from cached passive status. Run setup and command calls through `hass.async_add_executor_job`.
6. After a command, follow only the app-proven completion/status path and refresh the minimum required coordinator. Do not add speculative wake-ups or broad polling loops.
7. Add translations for every locale, README capability/subscription notes, and stable unique IDs through `KamereonEntity`.

## Phase 5: Test in Increasing Trust Order (Complete)

Direct Kamereon tests must cover:

- UID `27` and `878` capability gating, with status-only UIDs rejected for commands.
- Exact OTP, registration, SRP enrollment/challenge, lock/unlock, and action-status requests.
- Fixed SRP enrollment and proof vectors, invalid server public values, and per-command proof binding.
- Origin and redirect rejection for every sensitive POST.
- Device already registered, invalid/expired OTP, wrong PIN, missing registration, rejected challenge, early not-indexed status, timeout, cancellation, and final success.
- No wake-up call unless directly proven by the app's lock/unlock path.
- No sensitive values in logs or exception messages.

Home Assistant tests must cover:

- Entity creation for UID `27` and `878`, omission for unsupported/status-only vehicles, and multi-vehicle isolation.
- Setup, disable, reset, reauthentication, and migration behavior.
- Executor use, cached state, unavailable/unknown handling, stable identity, and command failure propagation.
- No commands during setup, startup, coordinator refresh, or entity property access.

Run focused API tests, config-flow tests, lock-platform tests, affected binary-sensor tests, the complete suite under the declared Python version, diagnostics, JSON validation, and `git diff --check`.

Validation result on 2026-09-01: the focused Kamereon, SRP, config-flow,
lock-platform, migration, and binary-sensor slice passed 84 tests. The complete
suite passed 106 tests under Python 3.14.7 with the versions pinned in
`requirements.test.txt`. All eleven translation files are valid JSON and have
identical key structures.

## Phase 6: Controlled Live Verification (Not Performed)

Only after all mocked and synthetic checks pass, request explicit approval for each live stage:

1. Passive capability and registration-status reads.
2. Device OTP request and registration.
3. SRP PIN enrollment and a challenge that does not submit a vehicle command, only if proven non-mutating beyond security setup.
4. One user-confirmed lock or unlock command in a controlled setting, because issue #96 cannot be proven end to end without changing vehicle state.

Use a temporary runner with `getpass`, fixed allowlists, no redirects on sensitive POSTs, redacted stage-only output, and complete cleanup. Never run a remote command under the passive `live-verify` authorization used for status checks.

Earlier passive live verification succeeded. No OTP, enrollment, challenge, or
lock/unlock action was run as part of this implementation because those stages
need fresh, stage-specific user approval. Mocked transport and independent SRP
vectors cover the implementation without changing vehicle state.

## Acceptance Criteria

- [x] Both old and CCS2 service catalogs are capability-gated correctly.
- [x] Unsupported vehicles expose neither setup nor lock entity.
- [x] Passive lock state remains non-waking.
- [x] Lock/unlock requires explicit user setup, PIN, and action.
- [x] The exact original-app security chain is reproduced and covered by an
  independent vector.
- [x] No speculative wake-up, silent success, secret logging, plaintext PIN,
  or cross-vehicle state sharing remains.
- [x] Focused and full mocked tests pass under the declared dependency set.
- [ ] A real command is validated only after separate explicit authorization.
