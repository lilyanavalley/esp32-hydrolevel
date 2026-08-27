# esp32-hydrolevel

HomeAssistant-connected ESP32-C3 liquid level sensor node.

Reads a **XKC-Y25-NPN capacitive liquid level sensor** (binary state: liquid present / absent) and reports the state to a [Home Assistant](https://www.home-assistant.io/) instance via MQTT.  The device appears automatically in HA through MQTT auto-discovery.

## Features

- Digital capacitive liquid-level detection (XKC-Y25-NPN or compatible NPN open-collector sensor)
- Home Assistant MQTT auto-discovery – no manual HA configuration required
- MQTT 3.1.1 **and** MQTT 5.0 support (selected via broker URI scheme)
- Plain and TLS-encrypted MQTT (`mqtt://` vs `mqtts://`, `mqtt5://` vs `mqtt5s://`)
- Username/password broker authentication
- Certificate-based mutual TLS (mTLS) – CA cert, client cert + key
- Software debounce with configurable window
- Configurable heartbeat publishing interval
- All settings via `.env` – nothing hard-coded in firmware
- Future-ready structure for additional sensors (temperature, humidity, pH, etc.)

## Hardware

| Pin | Connection |
|-----|------------|
| 3.3 V | Sensor VCC (red) |
| GND | Sensor GND (black) |
| GPIO 4 (default) | Sensor OUT (yellow) |

The default GPIO can be changed in `.env` (`HYDROLEVEL_SENSOR_GPIO`).

> **Sensor wiring note** – the XKC-Y25-NPN has an NPN open-collector output.
> A built-in pull-up is applied by the firmware.  Output LOW = liquid present,
> output HIGH = no liquid.  Set `HYDROLEVEL_SENSOR_ACTIVE_HIGH=false` (default).
> For sensors with inverted logic, set `HYDROLEVEL_SENSOR_ACTIVE_HIGH=true`.

## Prerequisites

1. [Rust](https://www.rust-lang.org/tools/install) (stable, 1.77+)
2. [ESP-IDF v5.x](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/get-started/index.html) with the `riscv32imc-esp-espidf` target
3. [`espflash`](https://github.com/esp-rs/espflash) for flashing
4. [`ldproxy`](https://github.com/esp-rs/embuild/tree/master/ldproxy) (`cargo install ldproxy`)

```sh
rustup target add riscv32imc-esp-espidf
cargo install espflash ldproxy
```

## Configuration

Copy `.env.example` to `.env` and fill in your values:

```sh
cp .env.example .env
$EDITOR .env
```

All `HYDROLEVEL_*` variables are read at **build time** and baked into the
firmware binary.  The `.env` file is ignored by git.

### Key variables

| Variable | Description |
|----------|-------------|
| `HYDROLEVEL_WIFI_SSID` | Wi-Fi network name |
| `HYDROLEVEL_WIFI_PASSWORD` | Wi-Fi password |
| `HYDROLEVEL_MQTT_BROKER_URI` | Broker URI (see schemes below) |
| `HYDROLEVEL_MQTT_USERNAME` | Broker username (optional) |
| `HYDROLEVEL_MQTT_PASSWORD` | Broker password (optional) |
| `HYDROLEVEL_MQTT_CLIENT_ID` | Unique MQTT client ID |
| `HYDROLEVEL_MQTT_CA_CERT_PATH` | Path to CA cert PEM (TLS, optional) |
| `HYDROLEVEL_MQTT_CLIENT_CERT_PATH` | Path to client cert PEM (mTLS, optional) |
| `HYDROLEVEL_MQTT_CLIENT_KEY_PATH` | Path to client key PEM (mTLS, optional) |
| `HYDROLEVEL_HA_DEVICE_ID` | Unique HA device ID |
| `HYDROLEVEL_HA_DEVICE_NAME` | Human-readable HA device name |
| `HYDROLEVEL_SENSOR_GPIO` | GPIO pin number for sensor output |
| `HYDROLEVEL_SENSOR_ACTIVE_HIGH` | `true` / `false` for signal polarity |
| `HYDROLEVEL_SENSOR_DEBOUNCE_MS` | Debounce window in milliseconds |
| `HYDROLEVEL_PUBLISH_INTERVAL_MS` | Heartbeat interval (0 = off) |
| `HYDROLEVEL_OTA_URL` | Optional HTTP(S) URL for startup OTA firmware update |
| `HYDROLEVEL_OTA_AUTO_APPLY` | `true` enables startup OTA apply from `HYDROLEVEL_OTA_URL` |

### MQTT broker URI schemes

| Scheme | Protocol | TLS |
|--------|----------|-----|
| `mqtt://host:1883` | MQTT 3.1.1 | No |
| `mqtts://host:8883` | MQTT 3.1.1 | Yes |
| `mqtt5://host:1883` | MQTT 5.0 | No |
| `mqtt5s://host:8883` | MQTT 5.0 | Yes |

## Building

```sh
cargo build --release
```

## Flashing

```sh
cargo run --release
# or
espflash flash --monitor target/riscv32imc-esp-espidf/release/hydrolevel
```

## Home Assistant

The sensor appears automatically under **Settings → Devices & Services → MQTT**
as device **"Hydrolevel Sensor"** (or whatever `HYDROLEVEL_HA_DEVICE_NAME` is
set to) with a `binary_sensor` entity named **"Liquid Level"**.

- **State ON** – liquid is present
- **State OFF** – liquid is absent

The device registers an availability topic so HA correctly shows the sensor as
"Unavailable" if the ESP32 goes offline unexpectedly.

## OTA updates

This project now includes:

- OTA-capable partition table (`ota_0` / `ota_1` + `otadata`)
- Optional startup OTA flow controlled by `HYDROLEVEL_OTA_URL`
- GitHub Actions release image generation (`hydrolevel-<tag>-ota.bin`)

When `HYDROLEVEL_OTA_AUTO_APPLY=true` and `HYDROLEVEL_OTA_URL` is set,
firmware will fetch the image at startup, write it to the next OTA slot, and
reboot into the new slot.

## GitHub Actions

`.github/workflows/ci-release.yml` provides:

- Rust CI on pull requests and branch pushes (format check, build, test compile)
- Release firmware image build on tags like `v0.5.0`
- Rust dependency caching via `Swatinem/rust-cache`

### Secrets used by CI/release

The workflow reads repository secrets matching the `HYDROLEVEL_*` names in
`.env.example` (for example `HYDROLEVEL_WIFI_SSID`,
`HYDROLEVEL_MQTT_BROKER_URI`, `HYDROLEVEL_HA_DEVICE_ID`, etc.).

For release-tag builds, required secrets must be configured; CI builds use safe
fallback defaults when secrets are not present.

## Extending for additional sensors

The codebase is structured for easy extension:

1. Add a new module in `src/` (e.g., `src/temperature.rs`)
2. Declare new `HYDROLEVEL_*` variables in `.env.example` and document them
3. Add the corresponding configuration struct fields in `src/config.rs`
4. Publish additional HA discovery payloads from `src/mqtt.rs`
5. Uncomment the module declaration in `src/main.rs`

## License

MIT – see [LICENSE](LICENSE).
