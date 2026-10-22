# Project status: what is verified, what is a guess

🇩🇪 [Deutsche Version](status.de.md)

State of this document: first publication. **One heater unit** (VEVOR 12 V, "8 kW", blue-LCD controller) with one ESP32-WROOM-32D, ESPHome 2026.9.1 and 2026.10.0, Home Assistant 2026.10.

## Verified on the hardware

| What | Evidence |
|---|---|
| Level shifter works in both directions | loopback test, bus frames decoded |
| Decoding of the 8-bit A and 16-bit B frames | live log, once per second, hours of operation |
| Standby recognised, supply voltage read from `60xx` | 12/13 V alternating |
| Start from standby with `A2` sent by the ESP32 | heater starts, display follows |
| Stop with `A2` while running | heater goes to cool-down |
| Level up/down with `A3`/`A4` | heater changes the level, B frame shows the new level |
| Start sequence B frames (`6042` … `6C42`) | captured at level 3 |
| Cool-down times | about 1 min 30 s after about 1 min 55 s run; about 5 min 32 s after about 3 min run |
| One long run | about 2 h 52 min at level 5 until the tank was empty; the heater handled the dry run itself (fault frames, cool-down, standby), display showed E08 |
| Fault frames for "empty tank" | `8CC9` → `89C9` → `88C9` → `8009` |
| **ON from the fault standby (`8009`)** | clears the fault and starts normally; after OFF the heater cooled down and returned to the normal standby |
| Controller display and 433 MHz remote still work with the tap in place | `A2` from both seen on the bus |
| Bus voltage with a multimeter on the DATA wire | idle about 4.85 V; during traffic the reading flickers between 4.85 V and about 2.35 V, with and without the level-shifter board connected (so R5 does not change the idle level noticeably). A multimeter averages the pulse train, so 2.35 V is a mean value, not a measured logic level |
| Entity cleanup (most entities marked *diagnostic*) | flashed and running cleanly |

## Hypotheses (consistent with the captures, but not proven)

| What | Why it is only a hypothesis |
|---|---|
| Hi-byte bits `0x08` fan, `0x04` pump, `0x01` glow plug | fits all captures; no direct measurement of the pump or the plug |
| Names of the start phases (Fan start-up, Preheating, Ignition, Running) | derived from the bit pattern above |
| E-number = (low byte & `0x3F`) − 1 | one data point (E08 ↔ value 9). Other numbers shown as `E-xx (?)` are guesses |
| `C0` in the low byte = cool-down | seen in all stops and in the fault cool-down, but not with every possible state |
| Integer volts in the low byte of `60xx` | only 12/13 seen; the exact scaling is not known |

## Not tested yet

- Other fault codes (overheat, flame-out, low voltage, ...). Their frames and what the firmware shows are unknown. Only the empty-tank fault (E08) was seen, including clearing it with ON.
- Long runs at levels 1 and 2; runs in cold weather; behaviour at an unstable 12 V supply.
- What happens when the ESP32 reboots, Wi-Fi or Home Assistant fail **during** a run (expected: the heater keeps running; the firmware decodes the state again after the next B frame; not tested).
- Long-term reliability (weeks of operation, e.g. memory or timing drift in the ESP32).
- Other VEVOR models and other controller versions.
- `esphome compile` of the English variant. In the authoring environment only `esphome config` (validation) could be run. The logic ran on the hardware in the German-named variant [`vevor-8kw-heater.de.yaml`](../../esphome/vevor-8kw-heater.de.yaml); the English file only renames entities/strings and adds comments (mechanically compared).

## Known weaknesses of the approach

- On/off is a toggle without acknowledgement on the bus. A wrong state assumption (for example after a lost frame) would toggle the wrong way. The firmware limits this by sending `A2` only if the last B frame fits the wanted action.
- The firmware cannot stop the heater if the ESP32, Wi-Fi or Home Assistant fail. See [../safety.md](../safety.md).
- State names are interpretations. Use *Heater Raw Status* when in doubt.

## Reports wanted

If you test another model, or a fault shows up on its own, please post the raw `B=…` frames and what the display showed. Do not provoke faults on purpose. Remove IP/MAC/Wi-Fi names from logs first.
