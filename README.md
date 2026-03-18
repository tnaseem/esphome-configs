# ESPHome Configurations

This repository contains the YAML configuration files for my ESP32-based smart home devices. 

By moving away from the Home Assistant ESPHome Add-on, this project allows for:
- **Version Control:** Tracking changes via Git.
- **Advanced Editing:** Using VS Code for better syntax highlighting and linting.
- **Modular Design:** Using ESPHome Packages for shared base configurations.

## 🛠 Hardware

- **Device:** M5Stack AtomS3 Lite (ESP32-S3)
- **Features:** Internal RGB LED (GPIO35), User Button (GPIO41)

## 📂 Project Structure

```text
.
├── common/
│   ├── base.yaml              # Shared board, framework, logger, API, and OTA config
│   ├── wifi.yaml              # WiFi + captive portal template (uses substitutions)
│   └── test-hardware.yaml     # AtomS3 Lite LED and button hardware test config
├── roles/
│   ├── bluetooth-proxy.yaml   # Bluetooth proxy role
│   └── remote-receiver.yaml   # IR remote receiver role
├── esp32-01.yaml              # Device: esp32-01 (192.168.1.22)
├── esp32-02.yaml              # Device: esp32-02 (192.168.1.23)
├── secrets.yaml               # (NOT IN GIT) WiFi and API credentials
└── README.md
```

## 🔧 Device Configuration

Each device file is minimal — it declares only what is unique to that device:

```yaml
substitutions:
  device_name: esp32-01
  device_ip: 192.168.1.22

esphome:
  name: ${device_name}
  friendly_name: ${device_name}

packages:
  base: !include common/base.yaml
  wifi: !include common/wifi.yaml
  role: !include roles/bluetooth-proxy.yaml
```

To repurpose a device, change the `role:` package reference to point at a different file under `roles/`.

## 🚀 Flashing from the Command Line

```bash
# Flash a specific device by IP
esphome run esp32-01.yaml --device 192.168.1.22
esphome run esp32-02.yaml --device 192.168.1.23

# Compile only (no flash)
esphome compile esp32-01.yaml
```

## 🔑 secrets.yaml

The `secrets.yaml` file is not tracked in Git. It must contain:

```yaml
wifi_ssid: "your-ssid"
wifi_password: "your-wifi-password"
fallback_password: "your-fallback-password"
api_encryption_key: "your-api-key"
ota_password: "your-ota-password"
```
