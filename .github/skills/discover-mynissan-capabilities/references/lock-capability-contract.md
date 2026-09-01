# MyNISSAN Lock Capability Contract

Last verified: 2026-09-01

Use this reference for passive door/lock status and active remote lock/unlock work. Re-verify it against the latest original app before production code changes when the app release or backend contract has changed.

## Provenance

- Original Android package: `eu.nissan.nissanconnect.services` from Nissan Automotive Europe.
- Examined release: MyNISSAN `4.0.0-prod-release`, version code `1942`.
- Bundle SHA-256: `c18b8c6509a685e961b0ab349f339e3240df40a134bb4c5ca16ff9509804f2ea`.
- The base and ARM64 APKs passed v2/v3 signature verification and shared signing-certificate SHA-256 `5aabd9f68884830da39f505530ca5f75481723c0a694bf119bf7269c7ea36337`.
- Previous release `3.17.1-prod-release`, version code `1852`, had the same package and signer. Its lock-status gate resolved to the same UIDs as release 4.0.0.
- Official Apple and Google listings identify Nissan Automotive Europe and MyNISSAN/NissanConnect Services. Public Nissan material states that availability varies by country, model, year, grade, subscription, and equipment.

Artifacts, decompiled output, tools, and temporary directories were removed after analysis. Do not add app artifacts or decompiled code to the repository.

## Verified Capability Gates

The app converts Android string resources to UID strings and compares them with `Vehicle.enabledUIDs`. It does not decide lock support from `modelName` alone.

| App behavior | UID | App resource meaning | Scope |
| --- | --- | --- | --- |
| Passive vehicle lock status | `2021` | vehicle lock status | Default and cross-badged path |
| Passive remote status check | `886` | remote status check | CCS2 vehicle where `ccsGen == "1"` and `isCrossBadged == false` |
| Active remote lock/unlock | `27` | remote control lock/unlock | One service-catalog family |
| Active remote lock/unlock | `878` | remote control lock/unlock | Renumbered service-catalog family |
| Lock/unlock button visibility | `878` | vehicle lock status | CCS2-specific button gate |
| SRP security setup | `747` | CCS1.5 SRP navigation gate | `ccsGen == "2"`; the numeric value is also present under an older catalog name |
| SRP security setup | `909` | CCS2 SRP navigation gate | `ccsGen == "1"` and not cross-badged |

Original-app pseudologic, expressed without decompiled source:

```text
has_lock_status = uid_886 if ccsGen == "1" and not isCrossBadged else uid_2021
has_remote_lock_unlock = has_lock_status and (uid_27 or uid_878)

if ccsGen == "1":
    lock_button also requires uid_878
    has_srp_setup = not isCrossBadged and uid_909
elif ccsGen == "2":
    has_srp_setup = uid_747
else:
    has_srp_setup = false
```

`ccsGen`, `isCrossBadged`, and enabled service UIDs are vehicle/backend data. Treat them as capability metadata, not a static model catalog.

The integration exposes active control only when both the command and proven
SRP setup predicates pass. It currently configures owner vehicles only. The
original app has separate secondary-user OTP and registration calls using a
vehicle UUID and `X-VehicleIdType: UUID`; that path is intentionally not
enabled without equivalent end-to-end verification in this integration.

## Verified Transport

| Operation | Method and path | Side effect | Capability |
| --- | --- | --- | --- |
| Read door and lock status | `GET v1/cars/{vin}/lock-status` | Passive cloud read | `2021` or `886` according to the app gate |
| Request fresh lock status | `POST v1/cars/{vin}/actions/refresh-lock-status` | May wake the vehicle | Same status capability; never use during passive discovery |
| Request device OTP | `POST v1/users/{userId}/vehicles/{vin}/generate-device-otp` | Emails a six-digit code | Owner security setup |
| Register device | `POST v1/users/{userId}/vehicles/{vin}/register-device` | Registers a trusted device | Owner security setup |
| Check device registration | `GET v1/users/{userId}/vehicles/{vin}/devices/{deviceId}/registration-status` | Passive security read | Setup recovery |
| Remove registered device | `DELETE v1/vehicles/{vin}/devices/{deviceId}` | Removes trusted device | Explicit options-flow action |
| Enroll or update PIN | `POST v1/cars/{vin}/actions/srp-initiates` | Changes SRP verifier | Proven SRP setup capability |
| Request SRP challenge | `POST v1/cars/{vin}/actions/srp-sets` | Creates transient challenge | Before every command |
| Lock or unlock | `POST v1/cars/{vin}/actions/lock-unlock` | Remote vehicle command | `27` or `878`, plus successful security setup |
| Follow action | `GET v1/cars/{vin}/actions/status?actionId={id}` on the action-status polling service | Polls enrollment, challenge, or command state | Returned action identifier |

The app has distinct DTOs for `LockStatus`, `RefreshLockStatus`, and
`LockUnlock`. Do not conflate passive status with command eligibility. Device
OTP and registration use JSON envelopes with `Content-Type: application/json`;
vehicle actions use `application/vnd.api+json`.

The exact `LockUnlock` attributes are:

```json
{
  "action": "lock or unlock",
  "deviceId": "stable per-vehicle integration device ID",
  "duration": 0,
  "target": "doors_hatch",
  "srp": "transient command-bound proof"
}
```

The proof order is `{vin}/RLU/Lock` or `{vin}/RLU/Unlock`. The app's lock
button path does not issue a wake-up action before this command. Every
enrollment, challenge, and lock/unlock response must contain an action ID and
reach final status `COMPLETED`. `REJECTED`, `CANCELLED`, `TIMEOUT`, a missing
action ID, or a local timeout is failure. An early action-status `404` is
transient and remains inside the bounded polling loop.

## Verified SRP Contract

The native implementation uses the RFC 5054 2048-bit group, generator `2`, a
10-byte enrollment salt, a 32-byte random client private value, SHA-256, and
HMAC-SHA256. Integer serialization is significant:

```text
x = SHA256(salt || SHA256(user || ":" || PIN))
v = g^x mod N
A = g^a mod N
k = SHA256(minimal(N) || one_byte(g))
u = SHA256(minimal(A) || PAD(B, 256))
S = (B - k * g^x)^(a + u*x) mod N
K = SHA256(minimal(S))
proof = HMAC-SHA256(
    key=K,
    message=PAD(A, 256) || PAD(B, 256) || user || salt || order,
)
```

In particular, `K` hashes the minimal unsigned encoding of `S`, not a
256-byte padded encoding. The synthetic regression vector uses salt
`00010203040506070809`, user `synthetic-user`, PIN `1234`, order
`SYNTHETICVIN/RLU/Lock`, and proof
`94c4d6e6b06a724c8dffda642478672957039adb2979bc0fba6d6196bbe5ece6`.

## Persistence and Recovery

The integration creates one random stable device ID per vehicle and stores it
with a setup state (`registered`, `configured`, `enabled`, or `unregistered`).
OTP values, the four-digit PIN, SRP salt/verifier inputs, ephemeral keys, and
proofs are never persisted. The PIN is supplied through the Home Assistant
lock service `code` field for each command.

If setup stops after device registration, the options flow preserves only the
`registered` state and resumes at PIN enrollment after verifying registration.
Changing the PIN reuses the proven `srp-initiates` enrollment path. Removing
the trusted device uses the verified DELETE endpoint. The current app contains
an `SrpDelete` response model but no request producer or endpoint, so the
integration does not invent a separate server-side SRP deletion call.

## Issue #96 Implementation

[Issue #96](https://github.com/dan-r/HomeAssistant-NissanConnect/issues/96)
requires more than adding UID `886`: that UID restores passive status for the
CCS2 catalog. Active control is implemented through device registration, SRP
PIN enrollment, a fresh per-command challenge/proof, command submission, and
final action-status verification.

The public `feature-locks` branch was useful research material, but not
merge-ready evidence:

- It covers OTP/device registration, SRP enrollment/challenge, command submission, action polling, and a Home Assistant lock entity with mocked tests.
- It gates active control only on UID `27`, omitting verified CCS2 UID `878`.
- It inserts a vehicle wake-up before lock/unlock while explicitly stating that this call was not traced in the app's button path.
- It contains temporary device-ID behavior, unresolved SRP rejection/error-code handling, and logs raw request/response or SRP material.
- It stores the four-digit SRP PIN in config-entry data and does not yet establish the desired reauthentication, reset, multi-vehicle, and cleanup behavior.

The implementation in this branch corrects those gaps: both catalogs are
gated, `deviceId` is included, no wake-up or success fallback exists, the
native session-key encoding is reproduced, completion is required for every
action, multiple vehicles are isolated, and no PIN is stored or logged.

## Public Model Evidence

These examples are evidence of observed or advertised behavior, not a complete model guarantee:

| Vehicle | Evidence | Confidence |
| --- | --- | --- |
| Qashqai e-Power 2024/2025 EU | Public issue reports UID `886`; forcing the old local gate restored door/lock status from the existing endpoint | Strong observed support for passive status; active command still security-gated |
| Juke | Official Nissan UK material explicitly describes checking whether the Juke is locked | Strong advertised support, still dependent on grade/subscription |
| Ariya | Official Nissan UK material explicitly advertises remote lock/unlock | Strong advertised active capability; the actual UID and entitlement remain vehicle-specific |
| X-Trail 2024 T33 | Public issue reports app lock/unlock behavior | Observed active capability, but no sanitized UID in that report |

The current app's broad compatibility list includes LEAF from May 2019, Navara from July 2019, Juke from November 2019, Qashqai from July 2021, Ariya from July 2022, X-Trail from September 2022, Townstar, Primastar, newer Micra, and Interstar variants. This list proves app compatibility only. It does not prove lock status or remote locking for every grade, country, subscription, or production date.

## Renumbered Catalog and Future Features

A public Qashqai 2025 EU report observed activated UIDs in the 800-series, including `878` and `886`. The original app resolves these relevant mappings:

- `878`: remote control lock/unlock; also used for CCS2 lock/unlock button visibility.
- `886`: remote status check, selected by the CCS2/non-cross-badge lock-status gate.
- `909`: CCS2 SRP security-setup gate for non-cross-badged vehicles.
- `747`: CCS1.5 SRP security-setup gate; retain its deliberate enum alias.

Other publicly observed UIDs such as `420`, `835`, `836`, `851`, `874`-`877`, `879`-`885`, `888`, `889`, `891`, `893`, `894`, `915`, `916`, `5001`, and `5008` remain unmapped here. Do not infer their meaning from sequence or neighboring IDs. Resolve each through an original-app resource name plus its consuming call site before adding it to `Feature`.

## Implemented Boundaries

- Passive fetch accepts either verified status catalog UID and remains in the
  ordinary non-waking fetch path.
- Explicit refresh remains separate because it may wake the vehicle.
- Security setup and commands are excluded from every coordinator, property,
  startup path, and passive refresh.
- Sensitive POSTs are single-shot and redirect-free. Errors and logs contain
  stage names or backend error codes, not request URLs, identifiers, OTPs,
  PINs, SRP values, action IDs, tokens, or raw responses.
- A controlled live command remains a separate, state-changing verification
  step and requires fresh explicit user approval.
