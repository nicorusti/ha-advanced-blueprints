# AGENTS.md - Coding Agent Instructions

## Project Overview

This is a **Home Assistant automation project** called **PV Excess Control**. It controls household
appliances (wallboxes, heat pumps, washing machines, etc.) based on excess solar power production.

The codebase consists of two main components:
- **Python (pyscript)**: `PV_Excess_Control/pyscript/pv_excess_control.py` — core control logic
  running inside Home Assistant's [pyscript](https://github.com/custom-components/pyscript) integration
- **YAML Blueprint**: `PV_Excess_Control/blueprints/automation/pv_excess_control.yaml` — HA Blueprint
  definition with all user-configurable inputs

There is no traditional build step, package manager, or test suite. Quality is enforced via pre-commit hooks.

## Directory Structure

```
ha-advanced-blueprints/
├── .github/workflows/          # CI: stale issue closer, dependabot
├── .pre-commit-config.yaml     # All linting/formatting hooks
├── pyproject.toml              # Ruff config (pyscript builtins)
├── PV_Excess_Control/
│   ├── blueprints/automation/
│   │   └── pv_excess_control.yaml   # HA Blueprint (~930 lines)
│   └── pyscript/
│       └── pv_excess_control.py     # Python logic (~1560 lines)
└── README.md
```

## Build / Lint / Test Commands

There are no build or test commands. All quality checks run via pre-commit.

### Install pre-commit hooks
```bash
pip install pre-commit
pre-commit install
```

### Run all hooks on all files
```bash
pre-commit run --all-files
```

### Run individual tools
```bash
# Ruff linter (with auto-fix)
ruff check --fix .

# Ruff formatter
ruff format .

# Type checking (strict mode)
mypy --strict --ignore-missing-imports PV_Excess_Control/pyscript/pv_excess_control.py

# Spell checking
codespell --ignore-words-list=fro,hass --skip="./.*,*.csv,*.json,*.ambr" --quiet-level=2 .

# YAML linting (excludes .github, .vscode, .devcontainer)
yamllint PV_Excess_Control/

# Python typing modernization (manual, run occasionally)
pre-commit run --hook-stage manual python-typing-update --all-files

# Prettier (YAML/markdown formatting)
prettier --write "**/*.yaml" "**/*.md"
```

### Testing
There is no test suite. Validate changes by:
1. Running `pre-commit run --all-files` and ensuring all hooks pass
2. Running `mypy --strict --ignore-missing-imports` on the Python file
3. Manual testing within a Home Assistant instance with pyscript installed

## Code Style Guidelines

### Python (pyscript)

#### Formatting
- **Ruff** handles all formatting and linting. No manual formatting decisions needed.
- Line length: Ruff defaults (88 characters).
- Use trailing commas in multi-line function signatures and collections.

#### Imports
- Standard library imports only (`collections`, `datetime`, `typing`). No third-party packages.
- `from typing import Union` is used for type annotations.
- Target Python 3.11+ (per `pyupgrade --py311-plus` configuration).

#### Pyscript Builtins
These globals are injected by the pyscript runtime and must NOT be imported:
- `state` — access HA entity states
- `log` — logging (`.info()`, `.error()`, `.debug()`)
- `service` — call HA services (`.call()`, `.has_service()`)
- `time_trigger` — decorator for scheduled triggers
- `task` — async task utilities

They are declared as builtins in `pyproject.toml` so Ruff does not flag them as undefined.

#### Type Annotations
- All function parameters and return types must be annotated.
- Use `Union[X, None]` pattern (the codebase predates `X | None` adoption, though
  `python-typing-update` may modernize this over time).
- mypy runs in `--strict` mode with `--ignore-missing-imports` (pyscript globals
  have no stubs).

#### Naming Conventions
- **Private/utility functions**: prefixed with underscore: `_get_state()`, `_turn_off()`, `_validate_number()`
- **Class names**: PascalCase: `PvExcessControl`
- **Instance/class variables**: snake_case: `export_power`, `appliance_switch_interval`
- **Constants**: snake_case (not UPPER_CASE) on the class: `min_excess_power = -10`
- **Log prefix**: formatted as `[{switch} {automation_id} (Prio {priority})]`

#### Docstrings
- Use triple-quoted multi-line docstrings with Sphinx-style `:param:` / `:return:` tags.
- Example:
  ```python
  def _get_state(entity_id: str) -> Union[str, None]:
      """
      Get the state of an entity in Home Assistant

      :param entity_id:  Name of the entity
      :return:            State if entity name is valid, else None
      """
  ```

#### Error Handling
- Wrap HA service calls and state lookups in `try/except` blocks.
- Log errors with `log.error(f"descriptive message: {e}")`.
- Return a safe fallback value (`None`, `False`, `return_on_error` param) rather than raising.
- Validate numerical values with range checks (`_validate_number` enforces -1000000 to 1000000).

#### Class Architecture
- `PvExcessControl` uses a class-level instance registry (`instances = {}` dict).
- Class variables hold shared state (sensor entity IDs, power history buffers).
- Instance variables hold per-appliance state (switch intervals, priority, runtime).
- The `__init__` method reuses existing instances when re-registered (idempotent).
- History buffers use `collections.deque(maxlen=3600)` for 3600-entry (1-second resolution,
  60-minute window) rolling histories. Since `deque` does not support slicing, use
  `list(d)[-n:]` when a slice is needed.
- Interval guards use `datetime` timestamps (`last_switch_change`, `last_current_change`)
  compared via `.total_seconds()`, not tick counters.
- The `_interval_to_ticks()` static method converts a seconds-based interval to a tick
  count for history slicing (1 tick = 1 second).

### YAML (Blueprint)

- Indentation: 2 spaces (enforced by yamllint and prettier).
- Blueprint inputs are organized into collapsible sections (`general_settings`,
  `power_sensors`, `home_battery_settings`, etc.).
- Each input has `name`, `description`, `default`, and `selector` fields.
- Descriptions use quoted multi-line YAML strings with markdown formatting.
- Use `selector` types: `entity`, `number` (with min/max/step/unit), `boolean`, `text`.

### General

- Files must end with a newline (enforced by `end-of-file-fixer`).
- No trailing whitespace (enforced by `trailing-whitespace` hook).
- Spell checking via codespell; `fro` and `hass` are in the ignore list.
- German umlauts in entity IDs are replaced with base vowels (`_replace_vowels()`).

## Pre-commit Hooks (Full List)

| Hook                    | Version        | Purpose                           |
|-------------------------|----------------|-----------------------------------|
| ruff                    | v0.13.1        | Python linting with auto-fix      |
| ruff-format             | v0.13.1        | Python formatting                 |
| codespell               | v2.4.1         | Spell checking                    |
| trailing-whitespace     | v6.0.0         | Remove trailing whitespace        |
| end-of-file-fixer       | v6.0.0         | Ensure files end with newline     |
| check-json              | v6.0.0         | Validate JSON files               |
| check-executables-have-shebangs | v6.0.0 | Shebang check (manual stage) |
| pyupgrade               | v3.20.0        | Modernize Python syntax           |
| yamllint                | v1.37.1        | YAML linting                      |
| prettier                | v4.0.0-alpha.8 | Multi-language formatting         |
| python-typing-update    | v0.7.2         | Typing syntax updates (manual)    |
| mypy                    | v1.18.2        | Static type checking (strict)     |

## Key Domain Concepts

- **PV Excess**: surplus solar power after household consumption is covered
- **Priority system**: appliances ranked 1-2000; higher number = higher priority;
  priority > 1000 means the appliance can turn on even without 100% excess coverage
- **Dynamic current control**: for wallboxes/chargers — ramps current up/down based on
  available excess, with configurable intervals and min/max current limits
- **Zero feed-in**: for systems that cannot export to grid; uses battery level and
  load thresholds instead of export power measurements
- **History buffers**: 60-minute rolling histories at 1-second resolution (3600 entries),
  stored in `collections.deque(maxlen=3600)`. Samples are appended directly each second
  with no intermediate buffering or averaging.
- **Solcast integration**: optional solar forecast for smart battery charging decisions
