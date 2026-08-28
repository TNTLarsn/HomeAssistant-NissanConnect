---
name: Compare MyNISSAN releases
description: "Compare two MyNISSAN app releases for authentication and API contract changes. Use after a new Android or iOS release to detect changes in OneID, OAuth/PKCE, client IDs, hidden tenants, token exchange, Kamereon hosts, Retrofit endpoints, or native JNI configuration before editing the integration."
argument-hint: "<candidate version or artifact> [baseline version or artifact]"
agent: agent
---

Compare the candidate MyNISSAN release with a known-good baseline. Treat invocation arguments as a candidate version, release URL, APK/XAPK path, or sanitized failure description followed optionally by the baseline equivalent.

Follow the acquisition, privacy, static-analysis, protocol-probing, and cleanup rules in the [MyNISSAN app update skill](../skills/analyze-mynissan-app-update/SKILL.md). This prompt is a comparison task only: do not edit integration code, tests, documentation, configuration, or git history.

1. Resolve both sides of the comparison:
   - The candidate is required. If it is ambiguous, ask only for its version or public artifact location.
   - When no baseline is supplied, use the latest release with documented successful authentication, passive vehicle retrieval, and Home Assistant entity mapping. Prefer that compatibility evidence over the immediately preceding store version. Repository evidence currently establishes Android 4.0.0 build 1942 as the initial known-good baseline; replace it only after a later release is proven compatible.
   - State the inferred baseline and its repository, artifact, and live-test evidence explicitly. Do not call an untested predecessor known-good merely because it is older than the candidate.
   - Do not use a captured authorization URL as an artifact location or reuse any dynamic value from it.
2. Read official store metadata for each release and verify package or bundle ID, version, build, publisher, release date, release notes, and reported size.
3. Acquire Android artifacts only as permitted by the linked skill. Verify archive integrity, SHA-256, package metadata, and Android signing-certificate identity before comparing content. Mark a side `unverified` when its artifact cannot be trusted or obtained.
4. Compare the smallest relevant surfaces first, expanding only when a difference appears:
   - Manifest, package metadata, split layout, and network-security configuration.
   - Public Nissan, OneID/WSO2, BFF, Kamereon, user, notification, and vehicle hosts.
   - OAuth public client ID, redirect URI, scopes, PKCE method, authorize/token/revoke paths, platform value, and client-auth expectations.
   - Login-form action, input names, JavaScript username transformation, and server-provided hidden `regionCode` tenant behavior. Never interpret that tenant as an ISO account country.
   - Retrofit interfaces: method, host owner, path, headers, query fields, request body, response model, and token consumed.
   - Token sequence and roles: authorization code, identity-provider access/ID/refresh tokens, Kamereon access/ID/refresh tokens, Bearer formatting, expiry, refresh, revoke, and 401 retry.
   - Exported auth-related JNI symbols and only the `.rodata` referenced by public OAuth/configuration getters.
   - Vehicle endpoint paths, service IDs, response models, and command/poll semantics.
5. For obfuscated code, trace call sites from Retrofit method names, model types, log markers, token storage, and view-model/repository usage. Do not infer token roles from DTO field names alone.
6. Corroborate every claimed change with at least two compatible facts when possible, such as store metadata plus package metadata, DEX call site plus Retrofit annotation, or native getter plus a synthetic endpoint probe.
7. Use only synthetic public-protocol probes. Generate fresh PKCE values, disable automatic redirects, enforce expected origins, and report only status, OAuth error code, field names, and response-key shape.
8. Do not perform a credentialed live login, fetch a vehicle, wake a vehicle, or invoke a remote command as part of this comparison.
9. Classify the integration impact:
   - `none`: no relevant contract change found.
   - `authentication-only`: authorize page, client, redirect, PKCE, login form, hidden tenant, or identity-provider token changed.
   - `token-exchange`: identity-provider-to-Kamereon exchange, token role, refresh, revoke, or retry changed.
   - `vehicle-api`: hosts, paths, service IDs, payloads, response models, polling, or commands changed.
   - `entity-surface`: cached fields or capabilities changed in a way that affects Home Assistant entity creation or state.
   - `unknown`: evidence is incomplete or artifacts are unverified.
10. Remove all downloaded artifacts, extracted files, decompiled output, cookie jars, and temporary scripts before reporting.

Report exactly this structure:

```text
Comparison: COMPLETE | PARTIAL | BLOCKED
Baseline: <version/build/source and verification state>
Candidate: <version/build/source and verification state>
Impact: none | authentication-only | token-exchange | vehicle-api | entity-surface | unknown

Changed contracts:
| Surface | Baseline | Candidate | Evidence | Integration impact |
| --- | --- | --- | --- | --- |
| <only changed or unresolved surfaces> |

Unchanged critical contracts:
- <security- or compatibility-relevant invariants actually checked>

Recommended next action:
- <smallest code/test/research action justified by the evidence>

Evidence gaps:
- <missing artifact, unverified signature, unresolved obfuscation, or none>

Cleanup: PASS | FAIL - <temporary artifacts/processes removed or remaining>
```

When no relevant differences are found, keep `Changed contracts` empty except for a single `No relevant changes found` row. Never claim compatibility merely because string sets match; report which executable call paths and token roles were checked.