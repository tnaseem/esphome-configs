# Hardware Test

## Overview

The `common/test-leds.yaml` package provides a quick smoke test for the M5Stack AtomS3 Lite hardware. Include it as a role package on any device to verify the onboard RGB LED and user button are functioning correctly before assigning a production role.

## What It Tests

| Component | GPIO | Test behaviour |
|-----------|------|----------------|
| RGB LED (SK6812) | GPIO35 | Pulses slowly (1 s transition, 2 s cycle) on startup |
| User button | GPIO41 | Press toggles the RGB LED on/off |

## Usage

Reference the package as a role in a device file:

```yaml
packages:
  base: !include common/base.yaml
  wifi: !include common/wifi.yaml
  role: !include common/test-leds.yaml
```

Flash the device, observe the LED pulsing, and confirm button presses toggle the light. Once hardware is verified, replace `test-leds.yaml` with the intended production role.

## Implementation Notes

- Uses the `esp32_rmt_led_strip` platform (ESPHome's RMT-based LED driver).
- No `rmt_channel` is specified; ESPHome allocates one automatically (required for recent ESP-IDF builds where manual channel assignment is deprecated).
- The button pin is configured `INPUT_PULLUP` with `inverted: true` because GPIO41 on the AtomS3 Lite is active LOW.
