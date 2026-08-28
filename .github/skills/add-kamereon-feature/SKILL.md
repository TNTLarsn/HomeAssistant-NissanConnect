---
name: add-kamereon-feature
description: 'Add or modify a Nissan Kamereon API capability in the in-tree synchronous client. Use for Feature service IDs, vehicle status endpoints, refresh actions, remote commands, JSON:API request or response parsing, Vehicle fields, fetch_all orchestration, and mocked API tests.'
argument-hint: 'Describe the service ID, endpoint, method, payload or response fields, and intended consumer'
---

# Add a Kamereon Feature

Implement one Nissan API capability end to end without guessing the production contract, blocking Home Assistant's event loop, or turning a passive update into a vehicle-waking poll.

## Establish the API Contract

1. Read the repository [guidelines](../../../AGENTS.md), especially the coordinator and synchronous API-layer boundaries.
2. Extract from the request, an issue, or a captured and sanitized API example:
   - The numeric Nissan service ID and corresponding capability name, when one exists.
   - The HTTP method, base URL setting, API version, path, headers, query parameters, and JSON body.
   - Representative success, unsupported, and error responses.
   - Vehicle, region, CAN-generation, or model restrictions.
   - The intended Home Assistant or coordinator consumer.
3. Classify the operation before editing:

   | Operation | Expected shape | Orchestration rule |
   | --- | --- | --- |
   | Passive status fetch | Usually `GET`; updates cached `Vehicle` fields | Add to `fetch_all` only when it is suitable for every ordinary fetch interval and cannot wake the vehicle |
   | Journey or other on-demand data | Usually `GET`; returns domain values or objects | Keep out of `fetch_all`; call from the owning coordinator or consumer |
   | Explicit refresh | Usually action `POST`; asks the car for fresh state | Keep separate from passive fetches and invoke only through the polling or explicit-update path |
   | Remote control | Action `POST`; performs a user command | Keep out of every coordinator fetch loop and invoke only from the matching action entity or service |

4. If the service ID, endpoint, or schema is not supported by evidence, ask for that evidence. Do not invent URLs, IDs, payload fields, credentials, client secrets, or response shapes.

## Extend the Kamereon Layer

1. Add a verified service ID to `Feature` in `kamereon_const.py`. `Vehicle` already ignores unknown service IDs, so do not change discovery behavior merely to add one known value.
2. For fetched status, initialize every new `Vehicle` field to `None`, `False`, or an empty collection as appropriate before any request runs. A vehicle must have a stable object shape even when unsupported or before its first update.
3. Add the smallest synchronous method to `Vehicle` in `kamereon.py`:
   - Guard it with the relevant `Feature` before making a request.
   - Prefer `_get` or `_post` over direct OAuth calls so token refresh and retry behavior remain centralized in `_request`.
   - If the endpoint genuinely requires another HTTP method, extend `_request` explicitly and cover that method with tests; do not disguise it as `GET` or `POST`.
   - Select the existing base URL setting that owns the endpoint instead of embedding a complete production hostname.
   - Use the established JSON:API content type and body envelope when required by the endpoint.
4. Decode `resp.json()` once. Raise `ValueError` when the API body contains `errors`, then validate the required `data` and `attributes` shape before reading it.
5. Parse optional values with `.get()` and assign them on every successful fetch so absent values do not leave stale state behind. Convert API enums and timestamps to the existing enum and timezone-aware `datetime` patterns where applicable.
6. For a model- or API-version-specific response, keep one public operation and dispatch to narrow private or suffixed implementations. Preserve useful fallback values deliberately and test precedence.
7. Return parsed domain values for on-demand/history methods and API result bodies for commands. Routine status methods should update the vehicle cache rather than force consumers to parse transport data.
8. Never log authorization data, credentials, SRP material, VIN-independent personal data, or complete raw responses containing sensitive fields.

## Preserve Update Semantics

1. Add only passive, routine status methods to `fetch_all`.
2. Add a refresh method to `refresh` only when it is an intentional vehicle-waking poll covered by the configured polling semantics.
3. Keep commands out of both methods. A command must run only after an explicit user action.
4. Do not add service calls to entity state properties, coordinator listeners, icons, availability properties, or device metadata.
5. Keep the Kamereon implementation synchronous. Every async Home Assistant caller must run blocking login, fetch, refresh, or control work through `hass.async_add_executor_job` before refreshing only the coordinator needed to expose the result.

## Connect the Consumer

1. Wire only the consumer explicitly requested by the task. When no consumer is requested, stop at the tested API layer and summarize the newly available fields or method; do not infer or create an entity, coordinator path, or service.
2. Gate the consumer with the same verified `Feature`, or with a documented source-value predicate when the API provides no capability ID.
3. Preserve account-email and VIN partitioning when passing data through `hass.data` and coordinators.
4. When adding or changing an entity, follow the sibling [add-nissan-entity skill](../add-nissan-entity/SKILL.md) for identity, translations, README coverage, and platform tests.

## Test Without Nissan Services

Add or update focused tests for the API boundary. Mock `_get`, `_post`, the session, responses, and Home Assistant callbacks; never use real credentials, VINs, tokens, or network access.

Cover the applicable branches:

- An unsupported vehicle returns before issuing a request and retains safe default fields.
- A supported vehicle sends the exact method, URL path, headers, parameters, and JSON payload.
- A representative success response maps fields, enums, timestamps, and returned domain objects correctly.
- Optional fields clear or preserve prior values according to the documented contract.
- An API `errors` body raises the expected exception without partially applying state.
- Model- or API-version-specific dispatch selects the correct parser and fallback precedence.
- A command validates arguments, respects capability gating, and is not called by `fetch_all` or a passive coordinator refresh.
- Any new `_request` HTTP-method support retains token-expiry retry behavior.
- The consuming coordinator or entity runs blocking work outside the event loop and updates only the intended state path.

Create or extend `tests/test_kamereon.py` for every new or changed endpoint, request shape, or response parser. Add matching coordinator or platform tests only when consumption behavior changes.

## Validate and Finish

1. Run the narrow API-layer test, such as `python -m pytest tests/test_kamereon.py`.
2. Run the matching coordinator or platform test when a consumer changed.
3. Run the complete suite with `python -m pytest tests`.
4. Check diagnostics for every changed file, run `git diff --check`, explicitly check untracked text files, and inspect the focused diff.
5. Confirm all completion criteria:
   - The service ID and transport contract are evidence-based.
   - Unsupported vehicles make no request.
   - Passive fetching, vehicle-waking refreshes, and commands remain separate.
   - New vehicle fields always have safe pre-fetch defaults.
   - Async Home Assistant code never runs the synchronous API work on the event loop.
   - Request construction, parsing, errors, and the consumer are covered without network access.
   - Narrow checks and the complete suite pass.