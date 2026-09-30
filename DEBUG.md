# Debugging and protocol

How the protocol was decoded, what each register means, and what was tested. For wiring and normal use, see [README.md](README.md).

**WARNING:** reading register `0000` resets the heater's MCU (the heater restarts and loses the clock). It happened on a request for `0000`–`0005`, so it is one of those, most likely `0000`. Start register scans at `0010`.

## Debug tools

`ariston-esp32-debug.yaml` adds two text entities used to decode the protocol. Most people will not need them; to enable them, uncomment the `packages:` lines at the top of `ariston-esp32.yaml`.

- **Send frame (hex)**: type the command and data, e.g. `25 79 2D` (read register `79 2D`) or `33 79 2D C2 01` (write 45.0 °C). The ESP adds the header, the length and the checksum; the response shows up in the ESPHome log. Writes are allowed, so be careful.
- **Register scan**: e.g. `2000-2FFF` reads all registers in that range with command `23`, 6 per request, and logs the non-zero ones (tag `scan`). With the `25 ` prefix, e.g. `25 2100-21FF`, it reads one register per request with command `25` and logs every response (value + limits). `stop` stops it.

## Protocol

- UART 9600 8N1, 5 V logic. The heater stays silent until the ESP talks first.
- Frame: `C3 41 <cmd> <len> <data...> <checksum>` from the ESP, `C3 14 <cmd> <len> <data...> <checksum>` from the heater. Checksum = sum of all previous bytes modulo 256. The heater acknowledges every frame with `3C`.
- Registers are 2 bytes, low byte first (register `79 2D` = 0x2D79).
- Commands:
  - `52` init: `C3 41 52 03 02 0B 00 66` is answered with `C3 14 52 04 1D FE 00 13 5B`.
  - `23` read a list of registers (up to 6 per frame).
  - `25` read one register: value (2 bytes) followed by its limits. The limits depend only on the low byte of the register and repeat in every `xx yy` family, and `xxE0`–`xxFF` return random limits, so an answer does not prove that a register exists.
  - `33` write one register: `33 <reg lo> <reg hi> <value>`.

| Register | Meaning | Format |
|---|---|---|
| `05 21` | Power (on/off) | 1 byte |
| `06 21` | ECO | 1 byte |
| `0A 21` | Anti-legionella (the "bact" setting) | 1 byte |
| `DD 27` | Mode: 1 = MAN, 2 = P1, 3 = P2, 4 = P1+P2 | 1 byte |
| `79 2D` | Set temperature | °C × 10 |
| `60 2D` | Maximum temperature (TSAF, U4 in the hidden menu), read only tested | °C × 10 |
| `62 2D` | Anti-legionella target (65 °C), read only | °C × 10 |
| `6E 9E` | Temperature shown on the panel | °C × 10 |
| `60 10` | Average of the 4 temperatures | °C × 10 |
| `68 13` / `69 13` | Right tank: bottom / top thermistor | °C × 10 |
| `6A 13` / `6B 13` | Left tank: bottom / top thermistor | °C × 10 |
| `C4 4B` | Heating | 1 byte |
| `D3 4B` | Anti-legionella cycle running | 1 byte |
| `5E 47` | Minutes to the set temperature (5 min steps) | 1 byte |
| `DB 40` / `DA 40` | Clock hour / minute (FF = not set) | 1 byte each |

## The two tanks

The heater has two tanks, each with its own heating element and a probe with two thermistors. Cold water enters the right tank, the right tank feeds the left one from the bottom, and hot water leaves from the left tank. I found this by watching the thermistors and six DS18B20 sensors on the outside of the tanks while heating and while taking a shower: the bottom thermistor is the first to react both to the element and to incoming cold water. After a shower the heater first heats the left tank, then the right one.

## What I couldn't find

1. Anti-calc mode
2. Beep on/off
3. Setting the time and temperature for P1 and P2 (switching between MAN/P1/P2/P1+P2 works)
4. The number of available "showers" (1, 2, 3 or 4)

I scanned all registers `0010`–`FFFF` with command `23` before and after changing anti-calc and beep, and again before and after changing the P1/P2 times and temperatures on the panel: apart from the mode register (`DD 27`), no register changed. I also read the families `21xx`, `23xx`, `24xx`, `27xx`, `2Dxx` and `40xx` one register at a time with command `25`, before and after changing all of them: no value changed. These settings are either not readable this way or not exposed over this port. The number of showers seems to be calculated by the panel.
