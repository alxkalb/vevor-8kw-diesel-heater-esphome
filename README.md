# VEVOR 8 kW diesel heater → ESPHome / Home Assistant

🇩🇪 [Deutsche Version](README.de.md)

Local control of a **VEVOR 12 V diesel air heater (8 kW, wired blue-LCD controller)** with an ESP32 and ESPHome: see the state, level and supply voltage in Home Assistant, switch the heater on and off, and change the power level (1–6). No cloud, no Bluetooth, no 433 MHz tricks. The ESP taps the single data wire between the controller and the heater and "presses the keys" for you. The original controller and its remote keep working.

> ⚠️ **This controls a fuel-burning appliance. Read [docs/safety.md](docs/safety.md) before you build or run anything.** Use at your own risk. This project is not affiliated with VEVOR.

![Level shifter schematic](docs/images/schematic.svg)

## Is this for my heater?

Probably **only if it looks like this one**. It was built and tested on exactly one unit:

- a VEVOR 12 V diesel air heater sold as "8 kW"
- wired controller with a **blue LCD** and the keys *up, down, on/off, gear (settings), OK*, connected by a **3-wire cable** (red 5 V, black GND, blue DATA)
- optionally a 433 MHz remote: its commands show up on the bus as the same frames as the controller keys

This heater does **not** speak the 4800-baud (or 25000-baud) UART protocol that other open-source projects document. It uses a slow pulse code with 12 ms bit cells, see [docs/protocol/protocol.md](docs/protocol/protocol.md) ([Deutsch](docs/protocol/protocol.de.md)). If your heater is different, the raw-capture helper in [`esphome/helpers/`](esphome/helpers/raw-capture.yaml) shows within minutes whether the same protocol is in use. Reports for other models are very welcome (see *Contributing*).

## Features

- State: standby, start requested, fan start-up, preheating, ignition, running, cooling down, fault
- Power level 1–6 (read and change)
- Supply voltage of the heater (read in standby, integer volts)
- Fault detection (fault frames, e.g. empty tank) and a "No signal" watchdog
- Buttons: **ON, OFF, Level Up, Level Down**. They only act when the heater state allows it.
- Fan, pump and glow plug status as diagnostic entities
- Fully local (ESPHome native API). Nothing talks to the internet.

## How it works

```
12 V supply ──┬── heater main unit ═══ 3-wire cable ═══ controller (display)
              │                                │
              └── XL4015 → 5.0 V ──┐        blue DATA wire (tapped)
                                    ├── level-shifter board (2× BS170) ── ESP32 (GPIO16 RX / GPIO17 TX)
                                    └── ESP32 VIN
```

Once per second the controller sends an 8-bit frame **A** and the heater answers with a 16-bit frame **B**. The ESP decodes both. To start, stop or change the level it sends the same extra frame the controller sends when you press a key (`A2` = on/off toggle, `A3` = level up, `A4` = level down), in the free time window after the controller's frame. Details: [protocol](docs/protocol/protocol.md). Overview: ![wiring](docs/images/wiring-overview.svg)

## What you need

| Part | Notes |
|---|---|
| ESP32 dev board | tested: ESP32 WROOM-32D (`esp32dev`) |
| 2× BS170 (TO-92) | N-channel MOSFETs |
| Resistors | 2× 1 kΩ, 2× 100 Ω, 1× 10 kΩ |
| XL4015 step-down module | 12 V → **5.0 V** for the ESP32 VIN and the pull-up |
| Perfboard, wire, case | see [docs/hardware](docs/hardware/README.md) |

Parts list, schematic, build and test steps: **[docs/hardware/README.md](docs/hardware/README.md)** ([Deutsch](docs/hardware/README.de.md)).

## Software setup

1. Build and test the board ([hardware guide](docs/hardware/README.md)). Do the loopback test first ([`esphome/helpers/loopback-test.yaml`](esphome/helpers/loopback-test.yaml)).
2. Copy [`esphome/secrets.yaml.example`](esphome/secrets.yaml.example) to `secrets.yaml` next to the heater YAML and fill in your values.
3. Use [`esphome/vevor-8kw-heater.yaml`](esphome/vevor-8kw-heater.yaml) (English entity names). The logic of the hardware-tested version is in [`esphome/vevor-8kw-heater.de.yaml`](esphome/vevor-8kw-heater.de.yaml) (German entity names, identical logic).
4. Flash it **while the heater is in standby**. Add the device to Home Assistant.
5. Check: *Heater Bus Active* = on, *Heater State* = Standby, *Heater Voltage* = about 12–13 V.
6. First start with someone present, watching the heater and the exhaust. The controller display shows what the heater does.

Tested with ESPHome 2026.9.1 and 2026.10.0, Home Assistant 2026.10.

### Entities

| Entity | Meaning |
|---|---|
| **Heater State** | Standby, Start requested, Fan start-up, Preheating, Ignition, Running, Cooling down, Standby (after fault), Fault …, No signal |
| **Heater Level** | 1–6 while the heater is active, otherwise unknown |
| **Heater Voltage** | supply voltage in V, only available in standby |
| **Heater Fault Status** | `No fault`, `Fault! E-08 (?)` (the number is a hint, see status), `No signal` |
| Heater ON / OFF / Level Up / Level Down | buttons |
| *Diagnostic:* Raw Status, Event, Fan, Pump, Glow Plug, Bus Active | for troubleshooting |

### Built-in safety logic

- Commands are sent **only in the free window** after a controller frame, never while the controller is talking.
- **ON** only from standby (or from the fault standby), **OFF / Level ±** only while the heater is active. Levels stay within 1–6. Up to 6 queued presses, a queued command expires after 8 s.
- Bus watchdog: no valid frame for 15 s → *No signal*, all queued commands are dropped.
- Fault detection: fault frames and "operation ended without an off command" are shown as *Fault!*.
- The heater's own protections (overheat, flame-out, low voltage) are untouched. The ESP only emulates key presses.

## Status

What is verified, what is a hypothesis and what is untested: **[docs/status/status.md](docs/status/status.md)**. Short version: start, stop, level ±, state/level/voltage reading, the "empty tank" fault and clearing it with ON were seen working on one unit. Other fault codes, long-term behaviour and other heater models are untested.

## Known limitations

- One unit tested. No guarantee for other VEVOR or "Chinese diesel heater" models.
- If the ESP, Wi-Fi or Home Assistant fails while the heater runs, the heater keeps running until you switch it off at the controller or remote.
- The "E-xx" number in the fault text is derived from a single observation (E-08, empty tank). Other codes are unverified.
- On/off is a toggle command on the bus. The firmware therefore only sends it when the observed state matches.

## Contributing

Reports are welcome, especially: other models, other fault codes (post the raw `B=…` frames and what the display showed), other levels and cool-down times. Open an issue and please **remove IP/MAC/Wi-Fi names from logs** before pasting.

## Credits and related projects

- [zatakon/vevor_heater_control](https://github.com/zatakon/vevor_heater_control) and [zatakon/esphome-vevor-heater](https://github.com/zatakon/esphome-vevor-heater): the starting point for this project (open-drain bus, level-shifter principle with two transistors). They target a different Vevor controller and protocol (4800 baud UART); the firmware here shares no code with them.
- [Ray Jones: Afterburner / BlueOwl](https://gitlab.com/mrjones.id.au/bluetoothheater): the well-known open-source work on the half-duplex bus of Chinese diesel heaters.
- The schematic in this repository is a new drawing with a different parts list (no voltage regulator on the board, BS170 instead of 2N7002).

## License

[MIT](LICENSE). The software and documentation are provided "as is", without warranty of any kind. VEVOR is a trademark of its owner; this project is independent and not endorsed by VEVOR.
