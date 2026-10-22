# Safety notes

🇩🇪 [Deutsche Version](safety.de.md)

This project connects a microcontroller to a **diesel-burning heater**. Mistakes can cause fire, carbon monoxide poisoning, fuel leaks, or damage to the heater. You build and use it **at your own risk**. The software and documentation come without any warranty (see [LICENSE](../LICENSE)). The author is not a certified heater installer. This project is not affiliated with VEVOR and has not been approved by them.

## Before you run the heater

- **Exhaust:** the exhaust must lead outside, gas-tight, with the outlet free and away from anything flammable. Never run the heater with a damaged or blocked exhaust or intake.
- **Carbon monoxide:** use a working CO detector in the room with the heater and in adjoining rooms. Remote start does not remove the need for ventilation and for checking the installation.
- **Fire:** keep flammable material (fuel canisters, paper, wood shavings, sawdust, solvents) away from the heater, the hot air outlet and the exhaust. A workshop or barn is a high-risk place for this.
- **Fuel system:** check lines, clamps and the tank for leaks. Use the fuel the manufacturer specifies.
- **Local rules:** installation, exhaust routing and operation of fuel-burning appliances are regulated in many places (buildings, vehicles). Follow the rules where you live.

## Remote start is a different risk than starting by hand

- **Do not rely on the ESP, Wi-Fi or Home Assistant to switch the heater off.** If any of them fails while the heater runs, the heater continues in its last state until you switch it off at the controller or the remote, or until the fuel runs out. The firmware cannot guarantee a shutdown.
- Think about whether it is acceptable to start the heater when nobody is present. If yes, add independent safeguards (see below).
- A timer or automation in Home Assistant that switches the heater off after N hours helps, but it also depends on the system working.
- **Never cut the power during the cool-down phase** except in an emergency. The heater cools down for about 1.5 to 5.5 minutes after a longer run (observed on one unit). Switching the supply off during that time leaves the combustion chamber hot and can damage the heater.
- Consider an independent hardware safeguard (for example a CO detector that you can hear or that notifies you, a fire detector, a supply that you can switch off from a safe place).

## Electrical points

- The heater bus idles at about 4.85 V on the tested unit (multimeter). Measure yours first. **Never connect the DATA wire directly to an ESP32 pin.** The ESP32 is not 5 V tolerant. Use the level shifter from this project and test it before connecting the heater ([loopback test](../esphome/helpers/loopback-test.yaml)).
- Set the XL4015 output to **5.0 V with a multimeter before** you connect the ESP32. A step-down module can deliver full input voltage if misadjusted and will destroy the ESP32.
- Tapping the controller cable can make the controller or heater misbehave if done badly. Test with the heater in standby. Keep the tap short and well insulated.
- The ESP32 only emulates key presses. It does not touch the heater's own protections (overheat, flame-out, low-voltage).

## What the firmware does and does not guarantee

- It only sends commands in states where they make sense, and only in a free time window. This reduces mistakes but does not make the system safe by itself.
- The state decoding is partly based on hypotheses. See [status](status/status.md) before trusting a text such as "Fault" or "Running" for decisions.
- Only one heater unit was tested.
