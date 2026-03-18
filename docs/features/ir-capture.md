# IR Code Capture

## Overview

The `roles/remote-receiver.yaml` role turns a device into a raw IR capture tool. It was used to reverse-engineer the stairlift handset remote control — listening to each button press and printing the raw IR timings to the serial log so they could be replayed by the transmitter role.

## Hardware

| Component | Detail |
|-----------|--------|
| IR receiver | KY-022 (38 kHz demodulating receiver) |
| Receiver pin | GPIO2 (Grove port signal pin) |

The KY-022 output is active-LOW (the signal line idles HIGH and pulses LOW on receipt), so `inverted: true` is set on the pin to normalise the polarity before ESPHome processes the signal.

## Role Configuration

```yaml
remote_receiver:
  pin:
    number: GPIO2
    inverted: true   # KY-022 is active-LOW
    mode:
      input: true
  dump: raw
  filter: 50us
  idle: 10ms
```

| Setting | Value | Reason |
|---------|-------|--------|
| `inverted: true` | — | KY-022 idles HIGH; inverted so ESPHome sees marks as HIGH |
| `dump: raw` | — | Outputs raw µs timings, ideal for cloning an unknown protocol |
| `filter: 50us` | 50 µs | Ignores glitch pulses shorter than 50 µs (noise rejection) |
| `idle: 10ms` | 10 ms | Declares a burst complete after 10 ms of silence |

## How It Was Used

1. Flash `remote-receiver.yaml` onto a device and open the ESPHome serial log.
2. Hold the stairlift remote over the KY-022 sensor and press each button.
3. ESPHome logs a `Received Raw:` line for each burst, e.g.:

   ```
   [I][remote.raw:036]: Received Raw: -852, 1918, -462, 1917, -461
   ```

4. The sign convention with `inverted: true` is: **negative = mark** (IR LED on), **positive = space** (IR LED off).
5. Convert to transmitter convention (positive = mark) by negating all values:

   ```
   UP: [852, -1918, 462, -1917, 461]
   ```

6. Repeat for each button and record the patterns.

The full set of captured logs (both raw and Pronto format) is preserved in [`STAIRLIFT_CODES.md`](../../STAIRLIFT_CODES.md).

## Interpreting the Captured Codes

The stairlift remote uses a simple 5-element raw burst at 38 kHz with a ~35 ms repeat cadence. Direction is encoded in the **second space** and **final mark** duration:

| Element | UP | DOWN |
|---------|----|------|
| Mark 1 | ~852 µs | ~851 µs |
| Space 1 | ~1918 µs | ~1919 µs |
| Mark 2 | ~462 µs | ~462 µs |
| Space 2 | **~1917 µs** (long) | **~992 µs** (short) |
| Mark 3 | **~461 µs** (short) | **~851 µs** (long) |

## Re-using This Role

To capture codes from a different remote:

1. Set `role: !include roles/remote-receiver.yaml` in the device file.
2. Flash the device.
3. Press the remote buttons one at a time and copy the `Received Raw:` lines from the log.
4. Remember to negate values when moving from receiver (inverted) to transmitter convention.
