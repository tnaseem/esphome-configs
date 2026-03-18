# Architecture Overview

This repository contains ESPHome YAML configurations for M5Stack AtomS3 Lite (ESP32-S3) devices integrated with Home Assistant.

## Design Principles

The project follows a **package-based modular design**. Each physical device file is intentionally minimal — it declares only device-specific substitutions and then assembles behaviour by including shared packages.

To repurpose a device, only the `role:` package reference needs to change.

## Directory Structure

```text
.
├── common/           # Shared, reusable configuration fragments
├── roles/            # Role packages — one role per use-case
├── esp32-01.yaml     # Device declaration: esp32-01
├── esp32-02.yaml     # Device declaration: esp32-02
├── STAIRLIFT_CODES.md  # Captured IR codes for reference
└── secrets.yaml      # Credentials (not tracked in Git)
```

## Package Layers

Every device composes exactly three packages:

| Package key | File | Purpose |
|-------------|------|---------|
| `base` | `common/base.yaml` | Board, framework (ESP-IDF), logger, HA API, OTA |
| `wifi` | `common/wifi.yaml` | WiFi with static IP, fallback hotspot, captive portal |
| `role` | `roles/<name>.yaml` | Device-specific functionality |

### `common/base.yaml`

Sets the board (`esp32-s3-devkitc-1`), selects the ESP-IDF framework, enables the serial logger, and configures the encrypted Home Assistant API and OTA update password. Credentials are injected via `secrets.yaml`.

### `common/wifi.yaml`

Connects to the home WiFi network using a static IP defined by the `${device_ip}` substitution. `power_save_mode: none` keeps latency low. Falls back to an AP hotspot with captive portal if the network is unavailable.

### Role packages

Each file under `roles/` is a self-contained role that provides a single device function. See [Roles](#roles) below.

## Roles

| File | Description |
|------|-------------|
| `roles/bluetooth-proxy.yaml` | Bluetooth proxy — extends HA Bluetooth range |
| `roles/remote-receiver.yaml` | IR receiver — captures raw IR codes for analysis |
| `roles/stairlift-control.yaml` | Stairlift UP/DOWN control via Home Assistant switches |
| `roles/stairlift-test.yaml` | Stairlift UP/DOWN control via physical button (offline testing) |

## Hardware

All devices use the **M5Stack AtomS3 Lite** (ESP32-S3):

| GPIO | Function |
|------|----------|
| GPIO35 | Onboard RGB LED (SK6812, GRB order) |
| GPIO41 | User button (active LOW with internal pull-up) |
| GPIO2 | Grove port signal pin (IR TX/RX) |

## Secrets

The `secrets.yaml` file is excluded from Git. It must define:

```yaml
wifi_ssid: "your-ssid"
wifi_password: "your-wifi-password"
fallback_password: "your-fallback-password"
api_encryption_key: "your-api-key"
ota_password: "your-ota-password"
```

## Flashing

```bash
# Flash over the air by IP address
esphome run esp32-01.yaml --device 192.168.1.22
esphome run esp32-02.yaml --device 192.168.1.23

# Compile only (no flash)
esphome compile esp32-01.yaml
```
