# Repository Guidelines

## Project Context

- This repository is a Home Assistant custom integration for European NissanConnect Services vehicles. Use the [README](README.md) as the source of truth for supported regions, vehicles, entities, and user-facing polling behavior.
- Keep changes scoped to `custom_components/nissan_connect/` and its matching tests unless release metadata or user documentation must change too.

## Environment and Validation

- Use Python 3.11.3 as declared in [.tool-versions](.tool-versions).
- Install development dependencies with `python -m pip install -r requirements.test.txt`.
- Run the narrowest relevant test first, for example `python -m pytest tests/test_sensor.py`.
- Run the complete suite before finishing with `python -m pytest tests`.
- The [CI workflow](.github/workflows/main.yml) also runs Hassfest and the HACS validator. There is no repository-local lint or type-check command; do not invent one.

## Architecture

- `custom_components/nissan_connect/__init__.py` owns config-entry setup, account-scoped `hass.data`, vehicle discovery, coordinator creation, platform forwarding, migration, and unload.
- `custom_components/nissan_connect/coordinator.py` separates ordinary API fetches, vehicle-waking polls, and journey-statistics updates. Preserve the polling-versus-update semantics documented in the [README](README.md#update-time), including their battery-impact assumptions.
- Platform modules (`sensor.py`, `binary_sensor.py`, `button.py`, `climate.py`, and `device_tracker.py`) construct entities from vehicles and coordinators. `const.py` is the shared registry for domain keys, platform names, and default intervals.
- `custom_components/nissan_connect/kamereon/` is the in-tree synchronous Nissan API/domain layer. Run blocking login and vehicle API operations through `hass.async_add_executor_job`; never block Home Assistant's event loop.
- Runtime data is partitioned by account email and then by vehicle VIN. Preserve multi-account and multi-vehicle behavior when changing lookup keys or entity identity.

## Implementation Conventions

- Base new entities on `KamereonEntity` plus the matching Home Assistant entity class. Reuse its coordinator listener, device metadata, and account/VIN/translation-key-based unique ID unless a migration explicitly requires otherwise.
- Add entities in the platform's `async_setup_entry` and expose them only when the vehicle advertises the required `Feature` or source value. Unsupported vehicle capabilities must not create unusable entities.
- Use `_attr_translation_key` for entity names. Keep additions or renames synchronized across every JSON file in `custom_components/nissan_connect/translations/`.
- Keep fetches that do not wake the vehicle separate from explicit refresh/poll actions. Avoid extra service calls from entity properties or coordinator callbacks.
- Preserve existing config-entry data and unique IDs. When stored data must change, update `CONFIG_VERSION` and implement the migration in `async_migrate_entry`.
- Update the [README entity list](README.md#entities) when the user-visible entity surface changes.

## Tests

- Mirror platform changes in `tests/test_<platform>.py`; config-flow behavior belongs in `tests/test_config_flow.py`.
- Mock vehicles, sessions, coordinators, and Home Assistant callbacks. Unit tests must not contact Nissan services or depend on real credentials.
- Cover capability gating and coordinator-driven state changes, not only direct property values, when those paths change.
