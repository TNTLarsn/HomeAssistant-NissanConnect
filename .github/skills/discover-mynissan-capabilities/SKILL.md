---
name: discover-mynissan-capabilities
description: 'Reverse-engineer and implement MyNISSAN capabilities from the verified original Android app. Always use before adding Nissan-backed functions or Home Assistant entities, or when exploring unknown Kamereon endpoints, service IDs, request payloads, token consumers, model/region/CAN-generation gates, passive status calls, refresh actions, or remote commands. Derive exact Retrofit/JNI contracts and corroborate them before code changes; never guess or enumerate private accounts.'
argument-hint: 'Choose inventory, investigate, implement, or live-verify and describe the capability, app release, model scope, or sanitized evidence'
---

# Discover MyNISSAN Capabilities

Use the publicly distributed original Android app as the primary interoperability specification before extending this integration. Reconstruct the exact app-to-Nissan contract, distinguish app-wide endpoint availability from vehicle-specific capability support, and implement only contracts supported by compatible independent evidence.

## Choose the Task Mode

Select and state one mode before using tools:

| Mode | Purpose | Repository edits | Production calls |
| --- | --- | --- | --- |
| `inventory` | Build a static inventory of app-reachable hosts, endpoints, service IDs, DTOs, and capability gates | None | None |
| `investigate` | Resolve one or more uncertain contracts with targeted static analysis and safe synthetic probes | None | Public or deliberately invalid synthetic probes only |
| `implement` | Reconstruct first, then add the smallest supported Kamereon and Home Assistant behavior | Yes | Synthetic probes only |
| `live-verify` | Confirm already-tested read-only behavior for an explicitly authorized account | Temporary runner only | Known passive calls after explicit approval |

Default to `implement` for a request to add a function or entity and to `investigate` for a request to research an endpoint. Do not enter `live-verify` merely because credentials would make discovery easier.

## Enforce the App-Evidence Gate

Before adding or changing any Nissan-backed function or entity:

1. Identify the latest official MyNISSAN Android package version, build, publication date, package ID, and publisher. Record when the metadata was checked.
2. Obtain a matching publicly distributed APK, XAPK, or APKS only when its provenance can be verified. Keep it in a permission-restricted directory under `/tmp`, never in the repository.
3. Verify package ID, version name, version code, publisher, file type, archive integrity, SHA-256 hash, and Android signing certificate. Compare the certificate with a known-good release when available; otherwise require a second independent artifact source.
4. Compare the candidate with the latest demonstrably compatible baseline. Android 4.0.0 build 1942 is only the initial baseline until a later compatible release is proven.
5. Reuse a prior extraction only when its recorded package, version, build, hash, signing identity, and relevant contract exactly cover the current task. Otherwise inspect the current verified artifact again.
6. Stop before production code changes if the artifact or relevant contract cannot be verified. A request, issue, static string, captured URL, familiar API pattern, or implementation in another client is supporting evidence, not a substitute for the original-app contract.

The gate is satisfied only when the contract matrix has two compatible evidence anchors where possible. Strong pairs include a Retrofit annotation plus its repository call site, a serializer DTO plus a synthetic response shape, a service-discovery consumer plus its feature gate, or a targeted JNI getter plus independent endpoint validation.

## Protect Users, Vehicles, and Artifacts

1. Use public artifacts only for interoperability. Do not bypass accounts, access controls, signatures, DRM, certificate validation, origin checks, redirects, rate limits, or anti-automation controls.
2. Never request, expose, search, persist, or place in commands authorization URLs, credentials, authorization codes, PKCE values, cookies, tokens, VINs, coordinates, account IDs, or raw personal responses.
3. Treat OAuth client IDs, redirect URIs, package IDs, hosts, and documented paths as public static configuration only after independent validation. Never copy a value from a user's dynamic capture into source.
4. Do not broadly decompile the app when targeted DEX, class, method, or native-getter analysis is sufficient. Inspect only networking interoperability surfaces; do not extract unrelated proprietary logic, private keys, signing material, credentials, or certificate pins.
5. Never fuzz paths, enumerate Nissan accounts, VINs, vehicles, model identifiers, or tenants, or infer hidden endpoints by brute force. Explore only call paths found in the verified app or public discovery metadata.
6. Never invoke a remote command, mutate vehicle state, or deliberately wake a vehicle during discovery or validation. Unknown wake behavior means the call is unsafe to probe.
7. Disable automatic redirects for authorization, credential, and token POSTs. Enforce expected HTTPS origins and keep TLS verification enabled.
8. Keep HTTP debug logging off. Report only allowlisted status codes, OAuth error codes, response-key shapes, and sanitized structural facts.

Follow the stricter authentication, artifact handling, and cleanup rules in the sibling [MyNISSAN app update skill](../analyze-mynissan-app-update/SKILL.md) whenever identity, token exchange, login forms, or a changed app release is involved.

For lock status, remote lock/unlock, CCS2 service UIDs, or issue #96, load the focused [lock capability contract](./references/lock-capability-contract.md). When implementation is requested, also load the [issue #96 implementation plan](./references/issue-96-implementation-plan.md).

## Reconstruct the Original-App Contract

Start at the app interaction or feature gate that owns the requested behavior, then inspect the nearest producer instead of collecting unrelated strings.

1. Inventory relevant hosts, base-URL keys, endpoint fragments, API versions, service IDs, Retrofit interfaces, serializers, repositories, view models, feature flags, and native public-configuration getters.
2. Resolve every Retrofit method through dependency injection or settings to its complete host and path. Record the exact HTTP method, headers, path variables, query parameters, body envelope, media type, timeout behavior, and response DTO.
3. Trace the method through its repository and UI consumer. Determine which token producer feeds it, how the authorization header is formatted, which errors are handled, and whether the call is passive, vehicle-waking, or state-changing.
4. Trace capability discovery separately from endpoint construction. Record numeric service IDs, server-advertised services, app-side region/model/CAN-generation gates, and fallback behavior.
5. For native public configuration, locate only the relevant exported JNI getter, disassemble that getter, and read only the referenced constant bytes. Independently validate recovered public OAuth configuration with safe synthetic requests.
6. Decode serializer annotations and representative response construction. Do not infer JSON field names from Kotlin, Java, or obfuscated symbol names.
7. Compare the same surface in the compatible baseline. Mark each method as added, removed, changed, or unchanged.

Produce this contract matrix before editing:

| Field | Required evidence |
| --- | --- |
| Operation | App call site and user-visible purpose |
| Transport | HTTP method, resolved host/base setting, API version, and exact path |
| Request | Headers, path/query fields, body schema, media type, and serializer names |
| Response | Success DTO, optional fields, error model, and relevant enum/timestamp semantics |
| Authentication | Actual token producer, consumer, and exact header formatting |
| Capability | Service ID and server/app gate, or explicit evidence that none exists |
| Applicability | Region, model, powertrain, and CAN generation only where explicitly gated |
| Side effects | Passive, may wake, refreshes vehicle, remote command, or unknown |
| Evidence | At least two compatible anchors where possible, with release/build provenance |

If any field controlling request safety or correctness remains inferred, mark it unknown and stop before implementation.

## Build a Defensible Capability Inventory

An app contains endpoints that it can call; it does not prove that every endpoint is available to every Nissan vehicle. Keep these scopes separate:

1. **App inventory:** All relevant request methods reachable in the examined app release, including dormant or gated methods when their call sites are proven.
2. **Backend advertisement:** Service IDs and capability metadata returned by vehicle discovery or documented public metadata.
3. **App applicability:** Explicit country, region, model, powertrain, subscription, or CAN-generation conditions found at consumers.
4. **Observed support:** Sanitized capabilities passively advertised for an explicitly authorized vehicle. This proves only that vehicle class and account context.
5. **Home Assistant eligibility:** Capabilities with stable semantics, a safe update path, and an appropriate capability gate that can become entities.

For each capability, report method, endpoint, service ID, side-effect class, app gate, observed-support scope, proposed Home Assistant platform, and confidence. Use `verified`, `partially verified`, or `unknown`; never turn absence from one artifact or account into an unsupported-model claim.

Do not claim complete coverage of "all vehicles" or "all models" unless an official catalog or explicit original-app rules enumerate that population. Phrase the result as "all app-reachable contracts in release X" and list residual model-coverage gaps.

## Explore Without Guessing

1. Form one falsifiable hypothesis about the unresolved contract and choose the cheapest probe that distinguishes it.
2. Prefer a second static anchor over a network call. Search targeted DEX descriptors, Retrofit annotations, serializer metadata, repository consumers, or a narrow native getter.
3. Use public OIDC discovery or intentionally invalid synthetic values only when they discriminate auth configuration. Generate fresh PKCE material and disclose no dynamic URLs.
4. Probe an authenticated endpoint only in `live-verify`, after mocked tests and synthetic checks pass and the user explicitly approves the exact passive calls.
5. Treat HTTP status alone as insufficient evidence for an endpoint schema. Do not use `HEAD`, `OPTIONS`, guessed IDs, or malformed production actions to discover behavior.
6. For refreshes and commands, prove request construction statically and test it with mocks. Never execute the production action as an exploratory probe.
7. If two candidate contracts remain plausible, inspect the nearest call-site or serializer boundary that differentiates them. Do not patch both variants or add speculative fallbacks.

## Implement One Proven Vertical Slice

In `implement` mode, state the falsifiable hypothesis, controlling path, evidence pair, and cheapest regression test before the first edit. Then implement only the proven slice.

1. Follow the sibling [Kamereon feature skill](../add-kamereon-feature/SKILL.md) for service IDs, synchronous request methods, response parsing, vehicle fields, token retry, and coordinator semantics.
2. Preserve the distinction between passive fetches, deliberate vehicle-waking refreshes, and explicit commands. Unknown wake behavior must not enter `fetch_all`, `refresh`, or a coordinator.
3. Follow the sibling [Nissan entity skill](../add-nissan-entity/SKILL.md) for capability gating, entity identity, translations, README updates, executor use, and platform tests.
4. Map a capability to a Home Assistant platform from its verified semantics, not its endpoint name. Do not create entities for dormant, subscription-blocked, unsafe, or structurally uncertain calls.
5. Implement broad inventories incrementally. Complete and validate one capability or tightly coupled capability family before opening another code path.
6. Immediately run the narrowest mocked test after the first substantive edit. If it falsifies the contract, return to the nearest original-app producer before changing more code.

## Validate in Increasing Trust Order

1. Add direct mocked tests for the exact service ID, method, resolved path, headers, query/body schema, response parser, feature gate, error behavior, and orchestration semantics.
2. Add the matching coordinator and platform tests for capability gating, state updates, unique IDs, executor use, and absence of network calls from entity properties.
3. Run the narrow API test, then the affected platform test, then `python -m pytest tests`.
4. Run editor diagnostics for every changed file, validate translation JSON, run `git diff --check`, and inspect untracked text files.
5. Search the diff for dynamic authorization values, personal data, raw responses, app artifacts, decompiled output, and accidental polling or identity changes.
6. Use the repository's [integration validation prompt](../../prompts/validate-nissan-integration.prompt.md) after code changes.

## Optional Passive Verification

Enter `live-verify` only after the implementation and mocked suite pass and the user explicitly approves a listed set of calls.

1. Create a temporary runner under `/tmp` that reads account values with `getpass` directly in the interactive terminal and retains them only in memory.
2. Call only endpoints already proven passive and non-waking. Never test buttons, climate control, locks, lights, horn, charging commands, or refresh actions.
3. Report fixed stage names and sanitized capability presence. Hash VINs, redact coordinates, suppress raw bodies, and never display tokens or account identifiers.
4. Verify entity construction from cached data without invoking commands or extra service calls.

## Clean Up and Report

Before finishing, close sessions, stop related processes, clear temporary credentials and token references, and remove every downloaded artifact, split APK, decompiled class, report containing extracted code, cookie jar, runner, bytecode file, and temporary directory.

Report:

```text
Mode: inventory | investigate | implement | live-verify
Result: PASS | PARTIAL | BLOCKED | FAIL
App baseline: <package version/build and provenance status>
Scope: <app-reachable contracts, capability family, or implemented entity>
Contract: <smallest verified transport/token/capability facts>
Model coverage: <verified gates and explicit unknowns>
Evidence: <concise static and synthetic/passive anchors>
Implementation: <files and behavior, or none>
Validation: <focused tests, full suite, diagnostics, and probes>
Security: <origins, redirects, secrets, vehicle side effects, and cleanup>
Residual risk: <specific unverified contract or model scope, or none>
```

Do not include credentials, token fragments, dynamic URLs, VINs, coordinates, raw responses, decompiled code, or claims broader than the evidence.