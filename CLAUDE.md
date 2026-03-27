# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Home Assistant custom integration (`hive_local_thermostat`) that provides local control of Hive thermostat devices (SLR1, SLR2, OTR1) via MQTT through Zigbee2MQTT — no cloud dependency. It is a fork of the retired upstream integration with additional patches.

## Development Commands

All commands use `uv` for dependency management:

```bash
scripts/setup       # Install dependencies (uv sync)
scripts/develop     # Run Home Assistant dev server on port 8123
scripts/lint        # Run ruff + mypy
```

Individual tool commands:
```bash
uv run ruff check custom_components/hive_local_thermostat        # Lint
uv run ruff check custom_components/hive_local_thermostat --fix  # Lint + auto-fix
uv run ruff format custom_components/hive_local_thermostat       # Format
uv run mypy custom_components/hive_local_thermostat              # Type check
uv run pytest                                                     # Run tests
uv run pytest --cov                                              # Tests with coverage
```

CI validates with `hassfest` (official HA validation) and HACS action on every PR.

## Architecture

All integration code lives in `custom_components/hive_local_thermostat/`.

**Data flow:**
1. `__init__.py` loads the config entry, validates HA version (≥2025.4), registers services, creates `HiveCoordinator`, subscribes to the MQTT topic, and publishes an initial "get" to request state.
2. `coordinator.py` owns all device state. It parses incoming MQTT JSON payloads from Zigbee2MQTT, derives HVAC mode/preset/boost state, and publishes commands to `<topic>/set`. This is the single source of truth.
3. Entity files (`climate.py`, `sensor.py`, `number.py`, `select.py`, `button.py`, `binary_sensor.py`) read from and write through the coordinator. They never interact with MQTT directly.
4. `services.py` defines 4 custom services (boost_heating, cancel_boost_heating, boost_water, cancel_boost_water) that delegate to coordinator methods.

**Model differences:**
- **SLR2**: Heating + hot water — exposes `select` platform for water mode and water-related sensors/buttons.
- **SLR1 / OTR1**: Heating only — no water controls. Platform list in `__init__.py` excludes `select` for these models.

**Key MQTT topics** (user-configured base topic, e.g. `zigbee2mqtt/HiveReceiver`):
- Subscribe: `<topic>` — receives device state
- Publish get: `<topic>/get` — requests current state
- Publish set: `<topic>/set` — sends commands

**Boost tracking**: The coordinator tracks boost start timestamps and remaining duration. Values >65000 from the device are treated as invalid and recalculated.

**Schedule/AUTO mode**: AUTO maps to Hive's schedule mode (`heat` + `temperature_setpoint_hold = False`). Schedule mode visibility is a config option.

## Key Files

| File | Purpose |
|------|---------|
| `coordinator.py` | All MQTT parsing, state derivation, and command publishing |
| `__init__.py` | Entry setup, service registration, model-based platform selection |
| `config_flow.py` | UI config: MQTT topic, model, schedule visibility options |
| `const.py` | Domain, model constants, config keys, defaults |
| `services.yaml` | Service schema definitions for HA UI |
| `translations/en.json` | All user-facing strings |

## Code Standards

- **Strict mypy** type checking is enforced — all new code must be fully typed.
- **Ruff** is configured with extensive rules; run format + check before committing.
- **pytest-homeassistant-custom-component** is used for testing HA integration patterns.
- Minimum HA version is `2025.4.0` (enforced in `__init__.py` and `hacs.json`).
