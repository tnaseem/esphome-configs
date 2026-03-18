# Stairlift Control

## Overview

The stairlift control feature allows a Home Assistant user to start and stop a motorised stairlift by emitting IR (infrared) signals from an ESP32 device fitted with a KY-005 IR transmitter module.

The device impersonates the stairlift's handheld remote control by continuously re-transmitting the correct raw IR burst code at the same 35 ms repeat cadence used by the original remote.

## Hardware≈

| Component | Detail |
|-----------|--------|
| ESP32 board | M5Stack AtomS3 Lite (ESP32-S3) |
| IR transmitter | KY-005 (HW-483) connected to the Grove port |
| Transmitter pin | GPIO2 (Grove port signal pin) |

## IR Codes

Both direction codes use a 38 kHz carrier. Timings are in microseconds; positive values are marks (IR LED on), negative values are spaces (IR LED off).

| Direction | Raw code |
|-----------|----------|
| UP | `[852, -1918, 462, -1917, 461]` |
| DOWN | `[851, -1919, 462, -992, 851]` |

Direction is encoded in the **second space** and the **final mark**:

- **UP** — second space ~1917 µs (long), final mark ~461 µs (short)
- **DOWN** — second space ~992 µs (short), final mark ~851 µs (long)

Raw capture logs and Pronto hex representations are preserved in [`STAIRLIFT_CODES.md`](../../STAIRLIFT_CODES.md) at the repository root.

## Role Files

### `roles/stairlift-control.yaml` — Home Assistant control

Use this role for normal daily operation. It exposes two Home Assistant switches. No physical button is required.

**Home Assistant entities:**

| Entity | Behaviour |
|--------|-----------|
| `switch.esp32_01_stairlift_up` | Turn ON → start moving UP. Turn OFF → stop. |
| `switch.esp32_01_stairlift_down` | Turn ON → start moving DOWN. Turn OFF → stop. |

**Safety interlock:** Turning either switch ON will automatically turn the other switch OFF first, preventing simultaneous direction commands.

**Implementation notes:**

- A single `int` global (`stairlift_direction`: 0 = stopped, 1 = up, 2 = down) drives the transmitter loop.
- An `interval` component fires every 35 ms and emits the appropriate burst when the direction flag is non-zero.
- Both switches use `optimistic: true` so Home Assistant reflects the ON state immediately, making the OFF button (stop) visible.
- At the start of each `turn_on_action`, `stairlift_direction` is set to `0` before the interlock delay, closing the ~35 ms race window where a stale burst of the previous direction could fire.

```yaml
# Simplified excerpt
switch:
  - platform: template
    name: "Stairlift Up"
    optimistic: true
    turn_on_action:
      - globals.set: { id: stairlift_direction, value: '0' }  # silence immediately
      - switch.turn_off: stairlift_down_switch
      - delay: 50ms
      - globals.set: { id: stairlift_direction, value: '1' }
    turn_off_action:
      - globals.set: { id: stairlift_direction, value: '0' }
```

### `roles/stairlift-test.yaml` — Offline testing with physical button

Use this role when testing the stairlift without Home Assistant (e.g. during initial setup or debugging). No HA connection is required — all control is via the onboard button.

#### Button cycle

Each press of the GPIO41 button advances a `cycle_step` counter (mod 4):

| Step | Action |
|------|--------|
| 0 | Start DOWN — `stairlift_down_switch` turns ON |
| 1 | Stop DOWN — `stairlift_down_switch` turns OFF |
| 2 | Start UP — `stairlift_up_switch` turns ON |
| 3 | Stop UP — `stairlift_up_switch` turns OFF |

On the next press after step 3, the cycle wraps back to step 0.

#### Globals

| Global | Type | Purpose |
|--------|------|---------|
| `stairlift_direction` | `int` | Transmitter direction flag: 0 = stopped, 1 = UP, 2 = DOWN |
| `cycle_step` | `int` | Current position in the 4-step button cycle (0–3) |

#### Button implementation

The `on_press` handler runs a C++ `switch` statement on the current `cycle_step`, then increments it (wrapping at 4). This avoids four separate `if/else` chains and keeps the button logic compact:

```cpp
int step = id(cycle_step);
id(cycle_step) = (step + 1) % 4;
switch (step) {
  case 0: id(stairlift_down_switch).turn_on();  break;
  case 1: id(stairlift_down_switch).turn_off(); break;
  case 2: id(stairlift_up_switch).turn_on();    break;
  case 3: id(stairlift_up_switch).turn_off();   break;
}
```

The button uses `delayed_on: 50ms` / `delayed_off: 50ms` debounce filters to prevent a noisy press from advancing the cycle multiple times.

#### Safety interlock

Both UP and DOWN `turn_on_action` blocks include the same interlock pattern as `stairlift-control.yaml`:

1. Set `stairlift_direction = 0` immediately (silences the transmitter within one 35 ms interval tick).
2. Turn off the opposing switch.
3. Delay 50 ms.
4. Set `stairlift_direction` to the new direction.

Logger lines (`"Stairlift Up: started"`, `"Stairlift Down: started"`, etc.) are included to make serial log monitoring easy during testing.

## Flashing

```bash
# Flash the production stairlift role to esp32-01
esphome run esp32-01.yaml --device 192.168.1.22

# To test without HA, temporarily change the role in esp32-01.yaml:
#   role: !include roles/stairlift-test.yaml
esphome run esp32-01.yaml --device 192.168.1.22
```

## Testing Notes

- If the stairlift is already at the **top** of the rail, UP commands will have no physical effect regardless of the IR code being correct.
- Test UP direction from a mid-rail position.
- Monitor ESPHome logs (`esphome logs esp32-01.yaml`) to verify the interval is firing and bursts are being transmitted.
