---
name: add-nissan-entity
description: 'Add or modify a Home Assistant entity in the NissanConnect integration. Use for sensors, binary sensors, buttons, climate entities, device trackers, vehicle capability gating, translation keys, entity documentation, and matching tests.'
argument-hint: 'Describe the entity, source data or action, and required vehicle capability'
---

# Add a Nissan Entity

Implement one NissanConnect entity end to end while preserving vehicle capability checks, stable entity identity, and the distinction between fetching data and waking a vehicle.

## Establish the Contract

1. Read the repository [guidelines](../../../AGENTS.md) and the "Update Time" and "Entities" sections of the [README](../../../README.md).
2. Extract or determine:
   - The Home Assistant platform and entity type.
   - The cached vehicle property or Nissan action that controls the entity.
   - The required `Feature` or a reliable non-`None` availability check.
   - The owning coordinator and any device class, state class, unit, or unknown-state behavior.
3. Inspect the closest entity in the target platform and its matching `tests/test_<platform>.py`. Use their public shape, setup path, and test style as the local pattern.
4. State one local hypothesis for how the entity should work and choose the cheapest test that would disprove it. Ask for clarification only when the code and request cannot determine the user-visible behavior.

## Trace the Data or Action

1. Locate the source property, action, and capability in `custom_components/nissan_connect/kamereon/` before editing the platform.
2. If the in-tree Kamereon layer does not expose the required data or action, extend that layer narrowly and test the behavior through mocks. Keep this API layer synchronous.
3. Select the coordinator by behavior:

   | Behavior | Coordinator or rule |
   | --- | --- |
   | Ordinary cached vehicle status | `DATA_COORDINATOR_FETCH` |
   | Daily or monthly journey history | `DATA_COORDINATOR_STATISTICS` |
   | Explicit vehicle-waking refresh or command | Preserve the existing poll/action path; never introduce it into a state property or listener |

4. Run every blocking login, fetch, refresh, or control call made from async Home Assistant code through `hass.async_add_executor_job`.

## Implement the Entity

1. Add the entity in the target platform's `async_setup_entry`, retaining the account-email lookup and per-VIN vehicle loop.
2. Gate creation with the verified `Feature` or source-value predicate. Do not create an entity that cannot work for that vehicle.
3. Derive the class from `KamereonEntity` and the matching Home Assistant entity class so coordinator listeners and device metadata remain consistent.
4. Set a descriptive `_attr_translation_key` and Home Assistant metadata as class attributes where possible.
5. Expose cached state through the platform-native property such as `native_value` or `is_on`. State properties and coordinator callbacks must not call Nissan services.
6. Implement commands with the platform's async method when the underlying operation can block. Refresh only the coordinator needed to reflect the result.
7. Preserve `KamereonEntity.unique_id`. Because the translation key participates in the unique ID, do not rename an existing key without an explicit entity-registry migration plan.

## Synchronize User-Facing Files

1. Add the same nested translation key to every JSON file in `custom_components/nissan_connect/translations/`. Preserve valid JSON and use the established wording in each locale. When the user does not provide translations, add best-effort, context-appropriate native wording for every locale and identify any uncertain wording in the final response; do not silently leave locales missing.
2. Add the entity to the appropriate list in the README and state any capability limitation concisely.
3. If introducing a new platform rather than an entity on an existing platform, add it to `ENTITY_TYPES` in `const.py` and create the matching platform test file.
4. Do not change release metadata unless the request explicitly includes a release.

## Test the Behavior

Update `tests/test_<platform>.py` or create it for a new platform. Cover the applicable cases:

- Setup creates the entity for a supported vehicle and omits it for an unsupported vehicle.
- Multiple vehicles remain isolated by account and VIN when setup behavior changes.
- State maps normal, missing, and unknown source values correctly.
- Coordinator-driven state changes are written when custom update handling is involved.
- Actions call the exact vehicle method with the expected arguments, do not block the event loop, and refresh only the intended coordinator.
- Tests use mocked sessions, vehicles, coordinators, and callbacks and never contact Nissan services.

## Validate and Finish

1. Run the narrow platform suite: `python -m pytest tests/test_<platform>.py`.
2. Run the complete suite: `python -m pytest tests`.
3. Run `git diff --check` and inspect the focused diff for accidental identity, polling, or unrelated changes.
4. Confirm all completion criteria:
   - Unsupported vehicles do not receive the entity.
   - State reads do not make network calls or wake the vehicle.
   - Existing unique IDs and config-entry data remain stable.
   - Every locale, the README, and matching tests are synchronized.
   - The narrow and complete test suites pass.