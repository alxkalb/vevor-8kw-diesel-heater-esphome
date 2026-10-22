# Hardware: level shifter, power and wiring

🇩🇪 [Deutsche Version](README.de.md)

Read [../safety.md](../safety.md) first. You are working next to a 12 V supply and a fuel-burning appliance.

![Schematic](../images/schematic.svg)

## What the board does

The heater and its controller talk over **one data wire** (the blue wire of the 3-wire controller cable). The wire is an open-drain bus with a pull-up: devices only pull it low. The ESP32 may not see more than 3.3 V on a GPIO, so a small level shifter with two MOSFETs sits in between:

- **Receive (Q2):** the bus is the gate of Q2. Bus high → Q2 conducts → GPIO16 is pulled low. Bus low → Q2 is off → R1 pulls GPIO16 up to 3.3 V. The signal is therefore **inverted** (the firmware knows this).
- **Transmit (Q1):** GPIO17 high → Q1 conducts → the bus is pulled low through R3. GPIO17 low → Q1 off → the bus is released.
- **R5** (10 kΩ to 5 V) is the pull-up for the bus on the board side.

Only the DATA wire is tapped. The controller's red (5 V) and black (GND) wires are **not** used. Ground is shared through the common 12 V supply of the heater and the XL4015.

![Wiring overview](../images/wiring-overview.svg)

## Parts list

Order numbers are deliberately not listed (they change, and wrong numbers are worse than none). Search by the type designation.

| Qty | Ref | Part | Notes |
|---|---|---|---|
| 1 | – | ESP32 dev board, **ESP32-WROOM-32D** (PlatformIO/ESPHome board `esp32dev`) | Do not use a WROVER/PSRAM board: GPIO16/17 are used internally there. |
| 2 | Q1, Q2 | **BS170**, N-channel MOSFET, TO-92 | Pinout **Drain – Gate – Source** (flat side towards you, pins down, left to right). This differs from the 2N7000, which is Source–Gate–Drain. |
| 1 | R5 | 10 kΩ, 1/4 W | bus pull-up to 5 V |
| 2 | R1, R3 | 1 kΩ, 1/4 W | R1: GPIO16 pull-up to 3V3; R3: TX series resistor to the bus |
| 2 | R2, R4 | 100 Ω, 1/4 W | series protection at GPIO16 and GPIO17 |
| 1 | – | **XL4015** step-down module (adjustable) | 12 V → **5.0 V**; needs roughly 7 V or more at the input |
| 1 | – | Perfboard, pad per hole, 2.54 mm | the prototype used a 50 × 100 mm board (Reichelt H25PR050) |
| – | – | Wire, solder, heat-shrink | |
| – | – | Wire tap (IDC "Kabeldieb") that fits the thin controller wire, **or** solder a branch to the blue wire | |
| – | – | Enclosure, strain relief for the cables | any case that is dry, non-flammable and fits the board |

Tools: soldering iron, **multimeter** (needed), small wire cutters/strippers, a 12 V supply or the heater's supply for testing.

## Netlist (point-to-point on perfboard)

| From | To |
|---|---|
| +5 V (XL4015 out) | R5 pin 1; ESP32 **VIN** |
| R5 pin 2 | **DATA** node |
| DATA node | Q2 **Gate**; R3 pin 1; blue wire of the controller cable |
| R3 pin 2 | Q1 **Drain** |
| Q1 Gate | R4 pin 2; R4 pin 1 → ESP32 **GPIO17** |
| Q1 Source | GND |
| Q2 Drain | R1 pin 1; R2 pin 1 |
| R1 pin 2 | ESP32 **3V3** pin |
| R2 pin 2 | ESP32 **GPIO16** |
| Q2 Source | GND |
| GND | XL4015 GND (OUT−), ESP32 GND; XL4015 IN− goes to the 12 V supply minus, which the heater also uses |

There is no voltage regulator on the board: the ESP32 dev board generates its own 3.3 V, and the 3V3 pin is used for R1. Layout is free; keep wires short. No ground plane is needed.

## Step 1: set the XL4015 to 5.0 V *before* connecting anything else

A step-down module that is not adjusted passes a high voltage to its output and **will destroy the ESP32**.

1. Connect only the 12 V input (+/−). Leave the output unconnected.
2. Measure the output with a multimeter.
3. Turn the multi-turn trimmer slowly until the multimeter shows **5.0 V** (the author reached 5.01 V). The module's factory setting is at the top of the range, and the trimmer needs **many turns (10–20)** before the output moves noticeably. Watch the multimeter, not the number of turns.
4. Switch off, connect the output to the board (+5 V, GND), switch on and check the voltage again under load.

## Step 2: check the board without the ESP32 and without the heater

- Continuity: no short between +5 V and GND, none between DATA and GND.
- Check the BS170 pinout twice. A swapped pinout is the most common mistake.

## Step 3: loopback test (ESP32 on the board, heater bus **not** connected)

1. Do **not** connect the blue wire yet.
2. Flash [`esphome/helpers/loopback-test.yaml`](../../esphome/helpers/loopback-test.yaml) (own `secrets.yaml` needed). For the first flash over USB, do not feed 5 V from the XL4015 into VIN at the same time unless you know your dev board separates USB and VIN; power from USB only, then switch to the XL4015 and use OTA later.
3. In Home Assistant (or the web log) press **TEST Pulse 1s**:
   - *TEST RX Q2* = **OFF** at idle, **ON** during the 1 s pulse → Q1 and Q2 work.
   - ON all the time → Q2 does not switch (pinout, solder joint, R5 missing).
   - OFF all the time, no change during the pulse → Q1 does not switch.

## Step 4: connect to the heater bus

1. Heater **in standby**, controller connected as before.
2. Measure the DC voltage between the blue wire and GND with the multimeter. On the author's unit: about **4.85 V** idle, and while frames are on the bus the multimeter flickers down to about 2.35 V, the same with and without this board connected (it averages the 4/8 ms pulses, so this is a mean value, not the logic low level). Other projects report about 4 V and a noisy bus for this kind of heater. **If you see more than about 6 V, stop** and do not connect the ESP32.
3. Tap the blue wire (IDC clip or a soldered branch) and connect it to the DATA node. Keep the tap short and insulated.
4. Flash [`esphome/vevor-8kw-heater.yaml`](../../esphome/vevor-8kw-heater.yaml). Expected: *Heater Bus Active* = on, *Heater State* = Standby, *Heater Voltage* ≈ 12–13 V, no unreadable-frame events in *Heater Event*.
5. If frames are not decoded, use [`esphome/helpers/raw-capture.yaml`](../../esphome/helpers/raw-capture.yaml) and compare the raw values with [the protocol description](../protocol/protocol.md). The controller display must still work normally with the tap in place.

## Notes

- GPIO16/GPIO17 and the polarity (RX inverted, TX active-high) are fixed in the firmware. If you change pins, change both `remote_receiver` and `remote_transmitter`.
- The BS170 gate threshold is specified up to about 3 V. The tested unit idles at about 4.85 V, which is well above that. A much lower idle voltage would not switch Q2 reliably.
- Put the board in a case, with strain relief on all wires. The XL4015 module can get warm; it only supplies the ESP32, so it stays well below its limits.
- Do not run the cable of the ESP32 board along the heater exhaust or hot air outlet.
