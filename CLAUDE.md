# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Home Assistant custom integration for monitoring and controlling LG smart home devices (washers, dryers, refrigerators, air conditioners, etc.) via the LG ThinQ cloud API. The integration is based on the WideQ API and supports both v1 and v2 of the LG ThinQ API.

**Important Authentication Note**: Users must use an independent LG account (NOT a social network login like Google/Facebook) for the integration to work.

## Development Commands

### Setup
```bash
# Install development dependencies
./scripts/setup
# or
python3 -m pip install --requirement requirements.txt
```

### Running Home Assistant with the Integration
```bash
# Starts Home Assistant with the custom component loaded
./scripts/develop
```

This script:
- Creates a `config/` directory if it doesn't exist
- Sets `PYTHONPATH` to include `custom_components/`
- Runs Home Assistant with config from `./config`

### Testing
```bash
# Install test dependencies
pip install -r requirements_test.txt

# Run all tests
pytest

# Run tests with output similar to CI
pytest -qq --timeout=9 --durations=10 -n auto -o console_output_style=count -p no:sugar tests

# Run a single test file
pytest tests/test_<name>.py

# Run with coverage
pytest --cov=custom_components
```

### Linting
```bash
# Run ruff linter with auto-fix
./scripts/lint
# or
ruff check . --fix

# Format code with black
black custom_components tests

# Sort imports with isort
isort custom_components tests
```

### Code Quality Configuration
- **Ruff**: Primary linter (config in `.ruff.toml`), targets Python 3.10+
- **Black**: Code formatter with 88 character line length
- **isort**: Import sorting with black profile
- **flake8**: Additional linting (config in `setup.cfg`)
- **pytest**: Testing framework with pytest-homeassistant-custom-component

## Architecture

### Three-Layer Architecture

The integration uses a three-layer architecture:

1. **WideQ API Layer** (`custom_components/smartthinq_sensors/wideq/`)
   - Core ThinQ API communication and device abstraction
   - Located in `wideq/` subdirectory (based on WideQ project)
   - Key components:
     - `core_async.py`: Async HTTP client (`ClientAsync`), authentication (`Auth`, `Session`), and gateway communication
     - `device.py`: Base `Device` class and device monitoring (`Monitor` class)
     - `devices/`: Device-specific implementations (AC, washer, refrigerator, etc.)
     - `model_info.py`: Device model and capability parsing

2. **Device Wrapper Layer** (`custom_components/smartthinq_sensors/device_helpers.py`)
   - Bridges between WideQ devices and Home Assistant entities
   - Classes: `LGEBaseDevice`, `LGEWashDevice`, `LGERefrigeratorDevice`, `LGETempDevice`, `LGERangeDevice`
   - Provides common functionality like state lookups, temperature conversions, and entity naming
   - Factory function: `get_wrapper_device()` creates appropriate wrapper for device type

3. **Home Assistant Platform Layer** (`custom_components/smartthinq_sensors/*.py`)
   - Platform integrations: `sensor.py`, `climate.py`, `binary_sensor.py`, `fan.py`, etc.
   - Each platform file creates HA entities from wrapped devices
   - Main entry point: `__init__.py` handles setup, authentication, and device discovery

### Key Flow

1. **Authentication** (`LGEAuthentication` in `__init__.py`):
   - OAuth2 flow for login
   - Creates `ClientAsync` instance with region/language
   - Manages token refresh and session

2. **Device Discovery** (`async_setup_entry` in `__init__.py`):
   - Retrieves devices from ThinQ API via `ClientAsync`
   - Creates WideQ `Device` instances
   - Wraps in `LGEDevice` coordinator with device helpers
   - Discovers entities across all platforms

3. **Entity Updates** (DataUpdateCoordinator pattern):
   - `LGEDevice` (in `__init__.py`) is a `DataUpdateCoordinator`
   - Polls device status every 30 seconds (`SCAN_INTERVAL`)
   - Entities listen to coordinator updates
   - Device wrapper classes provide state translation

### WideQ Device Architecture

Each device type in `wideq/devices/` follows a common pattern:
- Device class (e.g., `WasherDevice`, `ACDevice`) inherits from base `Device`
- Status classes (e.g., `WasherStatus`, `ACStatus`) inherit from `DeviceStatus`
- Status classes parse raw API data into typed attributes
- Device classes provide high-level methods and property access

### Configuration Flow

- Implemented in `config_flow.py` (`SmartThinQFlowHandler`)
- Async UI-based configuration (no YAML config)
- Supports: initial setup, reauthentication, and config updates
- Stores: region, language, token, OAuth URL, client ID

### Supported Platforms

The integration creates entities across multiple HA platforms:
- `sensor`: Main device state, operational attributes
- `binary_sensor`: Completion status, error states
- `climate`: HVAC devices (AC, heat pumps)
- `fan`: Fan controls for air purifiers, fans
- `humidifier`: Dehumidifier controls
- `water_heater`: Water heater temperature control
- `switch`: On/off controls for certain features
- `select`: Dropdown selections for modes/options
- `button`: One-time actions
- `light`: Lighting controls

## API Versions

The integration supports both ThinQ API v1 and v2:
- API version is auto-detected during authentication
- v2 is preferred and uses different endpoints/auth flow
- Config option `CONF_USE_API_V2` controls which version is used
- Most constants for v2 are in `wideq/core_async.py` (e.g., `V2_GATEWAY_URL`, `V2_API_KEY`)

## Common Patterns

### Adding Support for a New Device Type

1. Create device implementation in `wideq/devices/<device_type>.py`:
   - Define status class inheriting from `DeviceStatus`
   - Define device class inheriting from `Device`
   - Implement status parsing and device methods

2. Add device wrapper in `device_helpers.py` if needed:
   - Create wrapper class (e.g., `LGEMyDeviceDevice`)
   - Update `get_wrapper_device()` factory function

3. Create platform file (e.g., `my_platform.py`):
   - Implement `async_setup_entry()` to discover entities
   - Create entity class inheriting from appropriate HA base class
   - Implement required properties and methods

4. Add platform to `SMARTTHINQ_PLATFORMS` in `__init__.py`

### Working with Device State

Device state flows through these layers:
1. Raw API data → WideQ `DeviceStatus` subclass
2. `DeviceStatus` → Device wrapper (adds HA-specific conversions)
3. Device wrapper → HA entity (exposes as HA state/attributes)

## Testing Notes

- Tests are in `tests/` directory
- Uses `pytest-homeassistant-custom-component` for HA test fixtures
- Test configuration in `setup.cfg` under `[tool:pytest]`
- Coverage configuration in `.coveragerc`

## Dependencies

Core runtime dependencies (from `manifest.json`):
- `pycountry>=23.12.11`: Country/language code handling
- `xmltodict>=0.13.0`: XML parsing for certain API responses
- `charset_normalizer>=3.2.0`: Character encoding detection

## Version Compatibility

- Requires Home Assistant 2025.1.0+ (defined in `MIN_HA_MAJ_VER`, `MIN_HA_MIN_VER`)
- Targets Python 3.10+ (per `.ruff.toml`)
- Version is defined in `manifest.json`
