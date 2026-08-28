---
name: analyze-mynissan-app-update
description: 'Investigate and repair Nissan integration breakage after a MyNISSAN app or backend update. Use for new Android or iOS releases, OneID/WSO2/PKCE changes, OAuth client IDs, hidden login tenants such as regionCode, Kamereon token exchange, endpoint migrations, APK/XAPK/DEX/JNI interoperability analysis, redacted live probes, and Home Assistant regression validation.'
argument-hint: 'Describe the app version, failure, sanitized URL markers, or changed Nissan behavior'
---

# Analyze a MyNISSAN App Update

Recover the integration after a Nissan app or backend migration by deriving the smallest evidence-based protocol change from public artifacts and safe probes. Determine whether authentication, token exchange, vehicle endpoints, or entity consumption changed before editing code.

## Protect Accounts and Captures

1. Treat complete authorization URLs, query strings, callback URLs, cookies, and captured traffic as sensitive even when the user is unsure. Never repeat or search a full captured URL.
2. Classify values before using them:
   - Public static configuration: issuer and API hosts, OAuth public client ID, redirect URI, scopes, app package ID, platform name, and documented paths.
   - Ephemeral secrets: `state`, `code_challenge`, `code_verifier`, authorization code, `sessionDataKey`, cookies, access tokens, refresh tokens, and ID tokens.
   - Personal data: email, password, VIN, location, account identifiers, and raw vehicle responses.
3. Never send ephemeral secrets or personal data through chat, question tools, command arguments, environment variables, source files, logs, search queries, or test fixtures.
4. If a live account test is necessary, launch a temporary terminal program that reads email, password, and country with `getpass`. The user must type them directly into the terminal. Do not read, relay, or send those values yourself.
5. Disable HTTP debug logging during live tests. Report fixed stage/error codes instead of exception text or response bodies. Hash or mask VINs and redact coordinates.
6. Put downloaded apps, decompiled sources, cookie jars, and live-test scripts under a permission-restricted `/tmp` directory. Delete them and stop related processes before finishing.
7. Use only publicly distributed artifacts and endpoints for interoperability. Do not bypass access controls, signature checks, DRM, rate limits, or certificate validation.
8. Do not invoke vehicle-waking refreshes or remote commands during investigation unless the user explicitly requests that separately.

## Establish the Baseline

1. Read the repository [guidelines](../../../AGENTS.md), the current authentication implementation in `custom_components/nissan_connect/kamereon/`, and `tests/test_kamereon.py`.
2. Anchor on the exact failure stage: authorize page, credential form, authorization callback, identity-provider token, Kamereon token exchange, user discovery, vehicle discovery, status fetch, or Home Assistant entity setup.
3. Record the current contract as a table of method, host, path, headers, body or query fields, response fields, token role, and whether the call can wake a vehicle.
4. State one falsifiable local hypothesis and one cheap check. Prefer “only identity changed; vehicle hosts remain” or “the token exchange changed” over “rewrite the API.”
5. Preserve unrelated worktree changes. In particular, do not stage or commit customization files unless the user explicitly requests it.

## Search for Very Recent Implementations

1. Search public GitHub code, issues, pull requests, commits, branches, and recent forks using only non-sensitive markers such as the app name, public domain, package ID, protocol names, and error text.
2. Check active Nissan clients in multiple languages and Home Assistant, ioBroker, Homey, or Node-RED integrations. Inspect recent branches and forks directly because code indexes lag behind hour-old changes.
3. Check official App Store and Play Store metadata for package or bundle ID, version, build, publication time, and release notes.
4. Verify a candidate implementation by reading its actual request construction and token use. A mention of the new domain or a login outage is not an implementation.
5. If no implementation exists, say so and continue with protocol reconstruction. Do not wait for community code when public evidence supports a narrow repair.

## Acquire and Verify a Public App Artifact

1. Prefer the current Android APK/XAPK matching the official package ID because it is suitable for static interoperability analysis. Use iOS metadata and captures as corroboration, not as a source of reusable dynamic values.
2. Read official App Store or Play Store metadata first and record package ID, version, build, release date, publisher, and reported size.
3. A publicly accessible APK mirror may be used when the official store does not provide a direct unauthenticated artifact. Accept it only when package ID, version, build, and publisher match the official metadata; otherwise stop and report the artifact as untrusted.
4. Record the mirror source and SHA-256 hash. Verify the Android signing certificate against a previous known-good release when available, or corroborate the same artifact metadata through a second independent source. Do not treat a filename or mirror page title as sufficient verification.
5. Obtain the artifact without an account, access bypass, or executable installer. Download the APK/XAPK directly into the restricted temporary directory.
6. Verify version name, version code, package ID, size, file type, archive integrity, and signing-certificate identity before analysis.
7. For XAPK/APKS bundles, inspect the manifest and identify the base APK plus architecture and resource splits. Use a native split matching an available local architecture when native constants must be inspected.
8. Compare the current artifact with a previous known-good release when available. Focus on changed hosts, endpoint paths, auth classes, and native public configuration.
9. Never add the artifact or decompiled output to the repository.

## Reconstruct the Contract from the App

1. Inventory public hosts and endpoint markers from DEX string tables, resources, manifests, and native libraries. Compare them with `SETTINGS_MAP` before concluding that the vehicle API moved.
2. Identify auth and networking classes from DEX descriptors. Useful markers include `OneId`, `Login`, `OAuth`, `Token`, `Refresh`, `Repository`, `ViewModel`, Retrofit interfaces, and HTTP annotations.
3. Use targeted JADX decompilation before decompiling the whole app. If a full run is resource-heavy, locate the relevant DEX by class or log marker and decompile only that DEX or single class.
4. Trace each Retrofit method through its repository, view model, and token storage. Distinguish access token, ID token, and refresh token by their actual call sites rather than field names alone.
5. For native public configuration:
   - Find exported JNI getters with `nm -D` or `readelf`.
   - Disassemble only the relevant getter with `objdump`.
   - Read only the referenced `.rodata` bytes.
   - Extract public OAuth identifiers and redirect URIs only. Do not extract unrelated keys, private credentials, signing material, or certificate pins.
6. Independently validate a recovered public client ID against the authorize and token endpoints. An intentionally invalid authorization code should reach grant validation (`invalid_grant`); `invalid_client` indicates the client configuration is wrong.
7. Produce a contract matrix before editing. Include exact token provenance and header formatting, for example whether a downstream exchange consumes an identity-provider ID token without `Bearer` while vehicle APIs consume a Kamereon access token with `Bearer`.

## Preserve the Known-Good OneID Insight

Use the following as the current known-good baseline, then verify it again after every app update rather than assuming it remains true:

1. Start the MyNISSAN authorization-code flow with a newly generated PKCE verifier, S256 challenge, and state.
2. Load the Nissan login form and preserve all server-provided hidden fields.
3. Treat the user-selected Nissan account country as locale and routing input only.
4. Do not assume the hidden `regionCode` is an ISO country. It can be an internal OneID tenant such as `NG`.
5. Reproduce the page's JavaScript transformation: keep the visible `userName` as the email and construct the hidden technical `username` from the server-provided `regionCode` plus the email. Never replace that hidden tenant with `DE`, `GB`, or another account country.
6. Exchange the callback code and original verifier for the identity-provider token set.
7. Trace which identity-provider token is used for the Kamereon exchange. In the known-good Android 4.0.0 flow, the WSO2 ID token is sent in `Authorization` to the BFF access-token endpoint with `platform=Android`.
8. Use the resulting Kamereon access token as a Bearer token for existing vehicle APIs. Refresh the Kamereon token through its separate refresh endpoint and preserve the previous refresh token if no replacement is returned.

## Probe the Protocol Safely

1. Query public OIDC discovery metadata and retain only issuer, authorize, token, revoke, JWKS, grant, response, PKCE, and client-auth capabilities.
2. Generate a fresh verifier, challenge, and state for every probe. Never reuse values from a user's captured URL.
3. Disable automatic redirects. Follow authorize and login redirects one hop at a time and enforce the expected HTTPS scheme, hostname, and port.
4. Parse HTML with a structured parser. Record only form action paths, methods, input names, non-sensitive enum-like hidden values, and cookie names.
5. Before posting credentials, enforce the Nissan auth origin. Preserve hidden fields and overlay only the browser-controlled username and password fields required by the page JavaScript.
6. Establish an invalid-account baseline with a random `example.invalid` identity. Compare allowlisted error codes or a truncated hash of generic error text; never print the text itself.
7. Probe token endpoints with intentionally invalid synthetic codes or tokens and report only HTTP status, OAuth error code, and response-key shape.
8. Set `allow_redirects=False` on every credential and token POST. A cross-origin 307 or 308 can otherwise replay the POST body, authorization code, verifier, or refresh token.
9. Do not weaken TLS verification or origin checks to make a probe pass.

## Implement the Smallest Repair

1. Edit only after evidence identifies the controlling layer. Keep vehicle endpoints unchanged when current app artifacts still use them.
2. Store only verified static public configuration in `kamereon_const.py`. Never copy dynamic values from a captured authorization URL.
3. Keep login and API calls synchronous inside the Kamereon layer and invoke them from Home Assistant through `hass.async_add_executor_job`.
4. Centralize authenticated requests, token refresh, 401 retry, and redacted errors. Never include complete auth URLs, token responses, or credentials in exceptions or logs.
5. Preserve account and VIN partitioning. Deduplicate account-wide reauthentication when multiple vehicles share one session.
6. If required account configuration changes, update the config-entry version, migrate existing data conservatively, provide reauthentication, and validate option changes before persisting them.
7. Use the sibling [Kamereon feature skill](../add-kamereon-feature/SKILL.md) if vehicle endpoints or response parsing changed.
8. Use the sibling [Nissan entity skill](../add-nissan-entity/SKILL.md) if entity behavior or capability gating changed.

## Test in Increasing Trust Order

1. Add direct mocked protocol tests in `tests/test_kamereon.py` for:
   - PKCE generation and authorize parameters.
   - Login-form parsing and preservation of the server-provided tenant.
   - Technical username construction such as `NG/email` without hard-coding `NG` in production.
   - Callback origin and state validation.
   - Identity-provider token exchange and exact token role used by Kamereon.
   - Kamereon access and refresh token installation.
   - 401 refresh and request retry.
   - External form, redirect, and token-redirect rejection.
2. Test config-entry migration, reauthentication, options validation, and shared multi-vehicle sessions when those paths changed.
3. Run the narrow tests, then `python -m pytest tests`, then repeat the full suite with the Python version in `.tool-versions` when possible.
4. Run editor diagnostics, JSON validation, `git diff --check`, and explicit checks for untracked text files.
5. Search the diff for captured dynamic values and live vehicle data before committing.

## Run an Optional Live End-to-End Test

Only perform this after mocked tests and synthetic production probes pass, and only with explicit user approval.

1. Create the runner under `/tmp`; read account values with `getpass` directly in the terminal and keep them only in memory.
2. Report fixed stages: authentication, user discovery, vehicle discovery, passive status fetch, statistics fetch, and entity mapping.
3. Fetch only passive cloud data. Do not press buttons, control climate, honk, flash lights, start charging, or deliberately wake the vehicle.
4. Display only the minimum useful vehicle values. Hash the VIN, redact coordinates, suppress raw responses, and never show tokens or account identifiers.
5. Construct all Home Assistant platforms from the real cached vehicle object. Verify capability gating, translation keys, unique-ID uniqueness, device metadata, and readable state without invoking commands.
6. Close sessions, clear in-memory token references, delete runner and bytecode, remove downloaded artifacts, and verify no related process remains.

## Completion Criteria

- Public search results and app version are time-stamped and evidence-based.
- The changed protocol layer is identified before implementation begins.
- Client ID, redirect URI, hosts, token roles, hidden tenant, and payloads are independently corroborated.
- Account country and server-provided OneID tenant are not conflated.
- Credentials and tokens can only reach allowlisted Nissan origins, and sensitive POSTs never auto-redirect.
- No captured dynamic value, credential, VIN, coordinate, token, app artifact, or decompiled source enters the repository.
- Mocked protocol tests, Home Assistant tests, and the complete suite pass.
- A live test, when approved, retrieves real passive data and validates entity mapping without remote commands.
- All temporary artifacts and processes are removed before reporting results.