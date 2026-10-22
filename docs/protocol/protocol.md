# Bus protocol (VEVOR 8 kW heater, blue-LCD controller)

🇩🇪 [Deutsche Version](protocol.de.md)

Reverse-engineered from logic-level captures on one unit with the `remote_receiver` of ESPHome. Everything here describes **what was observed**; items that are only guessed are marked *(hypothesis)*. See [../status/status.md](../status/status.md) for the verification status.

## This is not UART

Other open-source projects for Vevor heaters (for example the ones listed in the main README) talk **4800 baud UART** to the controller. This heater does not. Its single data wire carries a slow pulse code. Connecting an ESPHome `uart:` at 4800 or 25000 baud decodes nothing useful here.

## Physical layer

- One wire, open drain with a pull-up, idle = high. Devices pull the bus low.
- Through the level shifter the ESP32 sees the bus **inverted**: bus high = GPIO16 low.
- Measured on this unit with a multimeter on the DATA wire: idle about **4.85 V**. While frames are on the bus the reading flickers between 4.85 V and about 2.35 V. Both values were the same with and without the level-shifter board connected. A multimeter cannot resolve the 4/8 ms pulses and shows a mean value, so 2.35 V is not a logic level. The real low level was not measured separately. Other projects report about 4 V and a noisy bus (3.3–6 V) for this kind of heater.

## Bit coding

Every bit is a 12 ms cell, **MSB first**. A bit is "bus high, then bus low":

| Bit | Bus high | Bus low | `remote_receiver` raw (inverted: − = bus high) |
|---|---|---|---|
| `1` | 8 ms | 4 ms | `-8000, +4000` |
| `0` | 4 ms | 8 ms | `-4000, +8000` |

Decoder tolerance in the firmware: the long part 6.5–9.5 ms, the short part 3.0–5.2 ms. `idle: 25ms` separates frames and `filter: 10us` removes glitches.

## Frames and timing

Once per second, in this order:

| Part | Direction | Length | Duration |
|---|---|---|---|
| **A frame** | controller → heater | 8 bit | 96 ms |
| gap (a low "start pulse") | | | about 30 ms |
| **B frame** | heater → controller | 16 bit | 192 ms |
| quiet | | | about 680 ms |

When the ESP32 sends a frame, it first holds the bus low for 30 ms (the start pulse), then sends the bits. In the firmware this is the first value `30000` in the `transmit_raw` list.

## A frames (controller → heater)

| Code | Binary | Meaning |
|---|---|---|
| `A6` | 1010 0110 | idle / keep-alive, sent every second |
| `A2` | 1010 0010 | **on/off toggle** |
| `A3` | 1010 0011 | level up |
| `A4` | 1010 0100 | level down |

- The upper nibble is always `A`. The firmware ignores 8-bit frames that do not start with `A`.
- `A2` from the controller key, from the 433 MHz remote and from the ESP32 look identical on the bus.
- On/off is a **toggle**. The heater does not tell the sender whether the command was "on" or "off"; the sender has to know the state. The firmware therefore sends it only when the B frame shows the matching state.
- In standby, the controller keys *Up, Down, OK, Gear* do not produce any extra frames on the bus.
- The heater **remembers the last level**; a start uses it.

## B frames (heater → controller), 16 bit

Written as hex `HHLL`.

### Standby

`60xx`: the low byte is the **supply voltage in volts** (integer). `0x0C` and `0x0D` alternate (12 V and 13 V); the firmware accepts `0x0A`–`0x1F`.

### Active (starting, running)

The low byte is `0x3F + level`: `0x40`–`0x45` = level 1–6. The high byte describes the phase. *(hypothesis, consistent with all captures)* The bits of the high byte are: `0x08` fan, `0x04` pump, `0x01` glow plug. A seen start sequence at level 3 (low byte `42`):

| B frame | Bits in hi byte | Interpreted phase (firmware text) |
|---|---|---|
| `6042` | none | Start requested |
| `6842` | fan | Fan start-up |
| `6942` | fan + glow | Preheating |
| `6D42` | fan + pump + glow | Ignition |
| `6D42` / `6942` alternating | | pump pulses while preheating continues |
| `6C42` | fan + pump | Running |

### Cooling down

Low byte `C0`: a cool-down phase. Seen after a normal stop: `64C0`/`65C0` → `6CC0`/`6DC0` → `69C0` → `68C0` → `600C` (back to standby, 12 V). `61C0` and `60C0` after a very short run, which went straight to standby.

Measured cool-down times (one unit): about 1 min 30 s after a run of about 1 min 55 s; about 5 min 32 s after a run of about 3 min. After a very short run the heater did not cool down at all.

### Fault

The high nibble `8` marks a fault frame. Seen when the tank ran empty (display: E08):

`8CC9` → `89C9` → `88C9` → `8009` (about 2 min 20 s from the first fault frame to the fault standby)

- Low byte & `0x3F` = `0x09` in that case. *(hypothesis)* The displayed E-number is this value minus 1, which fits only this one observation. The firmware shows `E-08 (?)` with a question mark for this reason.
- With `C0` set in the low byte (`8CC9`, `89C9`, `88C9`) the heater is still cooling down. `8009` is the fault standby.
- The fault stays until the next start. Pressing ON in the fault standby (`8009`) clears it and the heater starts normally (verified).

An alternative fault detection exists in the firmware: when an operation ends (low byte no longer `0x40`–`0x45`) at least 2 frames in a row without a seen `A2` just before, "Fault!" is shown. This catches a shutdown by the heater itself.

## Sending commands: the rules that worked

1. Never transmit while another frame is on the bus. A collision garbles both frames; the heater does not answer a garbled frame. Try again in the next second.
2. The working window is **about 100 ms after a regular B frame** that followed a controller A frame (not after the B reply to the ESP's own command).
3. Start pulse (bus low) 30 ms, then 8 bits, MSB first.
4. The heater answers an extra A frame with a B frame about 20–30 ms later. That B frame shows the new state (for example level + 1).
5. Level change: one `A3`/`A4` per step. The firmware queues up to 6 steps, one per free window, and checks the limits (1–6) on every step.

## Raw example (frame `A2`, as in `esphome/helpers/raw-capture.yaml`)

```
30000,                  # start pulse: bus low 30 ms
-8000,4000, -4000,8000, # 1 0
-8000,4000, -4000,8000, # 1 0
-4000,8000, -4000,8000, # 0 0
-8000,4000, -4000,8000  # 1 0   -> A2 = 1010 0010
```

(Complete frame, two bits per line for readability. In the real `code:` list everything is one flat list of numbers.)

## Comparison with kellerman/vevor_pwm

[kellerman/vevor_pwm](https://github.com/kellerman/vevor_pwm) (published earlier) describes the same kind of bus for "Vevor PWM" heaters: 8-bit commands to the heater, 16-bit status words from the heater, a 30 ms start pulse, about 12 ms per bit. Comparing its README with the frames seen here, **the values match when every bit is inverted** (their polarity convention is the opposite of the one used in this document). The table is my own comparison from their README. It was not tested on their hardware, and their library was not run here.

| This project (as decoded here) | vevor_pwm README | Meaning there |
|---|---|---|
| A frame `A6` | command `0x59` (default bits) | idle |
| A frame `A2` | `0x5D` (bit 2 set) | power on/off |
| A frame `A4` | `0x5B` (bit 1 set) | "power level toggle" (cycles 6 → 1 → 6; here: level down) |
| A frame `A3` | `0x5C` | not in their table (here: level up) |
| `600C` … `6009`, `600F` | `0x9FF3` … `0x9FF6`, `0x9FF0` | idle/standby with battery voltage (here: 12 V … 9 V, 15 V) |
| `6845`, `6945`, `6D45` | `0x97BA`, `0x96BA`, `0x92BA` | startup sequence (their values are for level 6; here: fan, preheating, ignition) |
| `6C40` … `6C45` | `0x93BF` … `0x93BA` | active, last nibble `F`…`A` = level 1…6 |
| `6CC0` | `0x933F` | shutdown in progress |
| `69C0` | `0x963F` | "clearing flame" |
| `68C0` | `0x973F` | cooldown |
| `0x8000` … `0x8FFF` | `0x7000` … `0x7FFF` | error states |

What this adds to the picture: the cool-down seen here as one phase (low byte `C0`) is a sequence of three named stages there, and an independent source describes the level nibble and the supply voltage the same way. What it does not settle: the fan/pump/glow-plug bit meanings and the E-number hint stay hypotheses here. Their README also lists a "ready" status word (`0xFA6C`) that was not seen in the captures of this project.

## Differences to the UART projects

| | this heater | zatakon/esphome-vevor-heater (XMZ-D2) |
|---|---|---|
| Coding | 12 ms pulse cell, A (8 bit) + B (16 bit) frames, 1 per second | UART 4800 baud, bytes |
| Direction | half duplex on one wire | half duplex on one wire |
| Interface | `remote_receiver` / `remote_transmitter` | `uart:` |

## How to capture your own heater

Flash [`esphome/helpers/raw-capture.yaml`](../../esphome/helpers/raw-capture.yaml) and watch the log for `Received Raw:` lines. Compare them with the table above. If your heater uses 8 ms / 4 ms cells you are probably on the same protocol. Please open an issue with a log excerpt (without IP/MAC/Wi-Fi names).
