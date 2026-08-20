# esphome/loctekmotion-desk

Full ESPHome control for a LoctekMotion (TekDesk) sit-stand desk on an ESP8266
(NodeMCU v2), including height readout, cover-style up/down control, memory
presets, and the built-in sit/stand **alarm** reminder.

Unlike the common iMicknl `loctekmotion_desk_height` external component, this
config drives everything from a **custom UART parser** in an interval lambda, so
it decodes the raw 7-segment DISPLAY frames (height *and* menu text) off the same
`9B 07 12` frame stream. The stock component exclusively consumed the desk RX
stream and blocked reading the display, which made closed-loop alarm control
impossible.

## Hardware

The desk is branded **TekDesk 2.0** (sold by DeskStand), manufactured by
**LoctekMotion**. Components:

| Part | Model |
|---|---|
| Keypad / handset | HS01B-1 |
| Desk control box | CB38M2A-1 |
| Motors | EC38SP1D3-5-630/580 |

Control interface:

- ESP8266 NodeMCU v2, wired to the desk's RJ45 keypad bus through a logic-level
  shifter.
- Pins (see `substitutions:` at the top of the YAML):
  - `desk_tx_pin` GPIO5 — ESP TX -> shifter -> desk RX (command line)
  - `desk_rx_pin` GPIO4 — ESP RX <- shifter <- desk TX (data line)
  - `keypad_rx_pin` GPIO3 — hardware UART0, keypad TX passthrough (console freed via `logger: baud_rate: 0`)
  - `screen_wake_pin` GPIO14 — PIN20 display-wake line

## Protocol (reverse-engineered)

The desk keypad speaks 9600-baud 8N1 UART. Every frame is:

```
9B <len> <type> <payload...> <crc16> 9D
```

- `9B` start byte, `9D` end byte.
- `<len>` = number of bytes from `<len>` to `9D` inclusive (i.e. `type + payload + crc + end`).
- `<type>` = message type (constant per message class): `0x02` = button command
  (keypad -> control box), `0x12` = height / 7-segment display (control box ->
  keypad). Types `0x11`, `0x13`, `0x15` are also emitted by the control box with
  no payload; their meaning is not yet decoded.
- `<crc16>` = CRC16/MODBUS (poly `0xA001`, init `0xFFFF`) over `len + type +
  payload`, transmitted high byte first.

### Button commands (type `0x02`)

The payload is **two bytes**. A physical key sets one bit; this keypad variant
only ever exercises the **first** payload byte, so held combos OR their bits
into a single frame (verified live: Up+Down -> `03 00`, P1+P2 -> `0C 00`).

| Payload | Function |
|---------|----------|
| `00 00` | **Wake** / heartbeat (idle frames) |
| `01 00` | **Up** (raw motor while held; also increments a value *only while the alarm menu is in edit mode*) |
| `02 00` | **Down** (raw motor; decrements a menu value only in edit mode) |
| `04 00` | Memory preset **M1** |
| `08 00` | Memory preset **M2** |
| `10 00` | Memory preset **M3** (stand) |
| `20 00` | **M** — Memory/Set (program a preset) |
| `40 00` | **A** — Alarm: short tap = enter set-mode / confirm; **long hold (~4 s) = alarm OFF** (this panel variant only; not in the reference tables) |
| `00 01` | **Preset 4 (sit)** — documented upstream, no physical key on this 7-key panel; the control box still accepts it |

Payload bit `0x80` of byte 1 and bits `0x02`..`0x80` of byte 2 are unmapped and
undocumented. Reset/calibration on these desks is a **physical gesture** (hold
Down at the low limit), not a serial opcode — no reset command exists in the
type-`0x02` space across any of the referenced research.

Height and menu state are decoded from the `9B 07 12` display frames using a
7-segment font table:

```
 aaa       a=0x01
f   b      b=0x02  c=0x04
f   b      d=0x08  e=0x10
 ggg       f=0x20  g=0x40
e   c      dp=0x80
e   c
 ddd
```

Glyphs decode via the table above; e.g. `0x77`->`A`, `0x6D`->`S`/`5`,
`0x50`->`r`, `0x40`->`-`, `0x09`->`:`. `S` vs `5` is disambiguated by context (an
alphabetic neighbour or a `S-` set prompt forces `S`).

### Alarm safety

`0x01`/`0x02` are **native motor commands**. They only double as menu navigation
*while the alarm number is actively blinking (EDIT mode)*. If the menu times out
mid-sequence, a stray Up/Down drives the motor and the desk moves. This config
guards every alarm-adjust tap with three interlocks:

1. `alarm_editing == true` (the digit is provably blinking — real edit mode),
2. a fresh menu frame seen within 450 ms (`alarm_menu_ms` lockout),
3. desk height unchanged since the sequence started (instant abort on any motion).

The `set_alarm` script also chooses its step **direction once** by shortest
circular distance on the 1..99 ring, so it wraps correctly (e.g. 2 -> 98 goes
*down* 2,1,99,98 = 4 taps, not up 96) and never oscillates at the 99<->1 boundary.
Turning the alarm **off** uses a long `0x40` hold (~4 s), the same command as the
physical Alarm Hold.

## Home Assistant entities

| Entity | Notes |
|--------|-------|
| `sensor` **Height** | Desk height in cm. Keeps the last value across reboots and republishes it on boot. |
| `cover` Desk | Up/down/position control (internal, driven by the height sensor). |
| `switch` **Alarm** | Toggle: **On** arms the sit/stand reminder to *Alarm Timer* minutes (fires the keypad commands); **Off** turns the alarm off (long `0x40` hold). Also passively **syncs** to the physical keypad: the switch tracks the On/Off shown on the desk display (any live alarm menu => On, `0FF` => Off), state only, no commands fired. |
| `number` **Alarm Timer** | Target countdown in minutes (1..99, unit `min`), persisted. A manual keypad edit is written here once, when Edit mode ends (you press A or it times out), so HA history records only saved values. |
| `button` **Memory** | Program a memory preset (`0x20`). |
| `button` **Wake** | Wake the display (PIN20). |
| `text_sensor` Display | Live mirror of the 7-segment display (height, menu, `0FF`, `ASr`, etc.). Diagnostic-class, hidden by default. |

## Flashing

Add this config to your ESPHome dashboard and compile + OTA it as usual.
Requires a `secrets` file for wifi. The HA API runs plaintext by default; add an
`api_encryption_key` secret and enable `api.encryption.key` to harden it.

## References

Protocol details in this document were reverse-engineered against the physical
desk and cross-checked against the following prior work. The frame structure
(`9B … 9D`, length + type byte, CRC16/MODBUS) and the two-byte button payload
(including the `00 01` "Preset 4 / sit" command with no physical key) are
corroborated across all of these independent sources.

- **iMicknl / LoctekMotion_IoT** — the canonical community project (ESPHome
  packages, pin-outs, command list, height/7-segment decoding).
  <https://github.com/iMicknl/LoctekMotion_IoT>
  (`README.md` "Command list" and "Research" sections.)
- **nv1t / standing-desk-interceptor** — frame-structure reverse engineering:
  documents the `<len>`/`<type>` bytes, CRC16/MODBUS over length+type+payload,
  message types `0x02` (button), `0x12` (height), and the still-undecoded
  `0x11` / `0x13` / `0x15` status frames.
  <https://github.com/nv1t/standing-desk-interceptor>
- **VinzSpring / LoctekReverseengineering** — Python packet parser and
  7-segment (`TM1650`) decode table used to confirm the display-glyph mapping.
  <https://github.com/VinzSpring/LoctekReverseengineering>
- **Dude88 / loctek_IOT_box** — Arduino control implementation (Alexa/MQTT),
  independent confirmation of the button command bytes.
  <https://github.com/Dude88/loctek_IOT_box>
- **grssmnn / ha-flexispot-standing-desk** — MicroPython/MQTT integration,
  additional command-byte confirmation.
  <https://github.com/grssmnn/ha-flexispot-standing-desk>
- **minifloat, mikrocontroller.net thread** — original 7-segment height decoding
  research that the above projects build on.
  <https://www.mikrocontroller.net/topic/493524>

Findings unique to this build (not in the sources above): the `40 00` **Alarm
(A)** key present on this panel variant, the closed-loop alarm decoding off the
display frames, and the live verification that held key combos OR their bits
into one frame.
