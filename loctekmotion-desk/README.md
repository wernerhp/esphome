# esphome/loctekmotion-desk

Full ESPHome control for a LoctekMotion (TekDesk) sit-stand desk on an ESP8266
(NodeMCU v2): height readout, cover-style up/down control, memory presets, and
the built-in sit/stand **alarm** reminder.

Unlike the common iMicknl `loctekmotion_desk_height` external component, this
config drives everything from a **custom UART parser** in an interval lambda, so
it decodes the raw 7-segment DISPLAY frames (height *and* menu text) off the same
`9B 07 12` frame stream. The stock component exclusively consumed the desk RX
stream and blocked reading the display, which made closed-loop alarm control
impossible.

For the full wire protocol, frame format, command table, and reverse-engineering
notes, see **[PROTOCOL.md](PROTOCOL.md)**.

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

## Home Assistant entities

| Entity | Notes |
|--------|-------|
| `sensor` **Height** | Desk height in cm. Keeps the last value across reboots and republishes it on boot. |
| `cover` Desk | Up/down/position control (internal, driven by the height sensor). |
| `switch` **Alarm** | Toggle: **On** arms the sit/stand reminder to *Alarm Timer* minutes (fires the keypad commands); **Off** turns the alarm off (long `0x40` hold). Also passively **syncs** to the physical keypad: the switch tracks the On/Off shown on the desk display (any live alarm menu => On, `0FF` => Off), state only, no commands fired. |
| `number` **Alarm Timer** | Target countdown in minutes (1..99, unit `min`), persisted. A manual keypad edit is written here once, when Edit mode ends (you press A or it times out), so HA history records only saved values. |
| `button` **Memory** | Program a memory preset (`0x20`). |
| `button` **Wake** | Wake the display (PIN20). Note: the serial `00 00` heartbeat does **not** wake a sleeping panel; a real key event is required. See PROTOCOL.md. |
| `text_sensor` Display | Live mirror of the 7-segment display (height, menu, `0FF`, `ASr`, etc.). Diagnostic-class, hidden by default. |

## Alarm safety (summary)

`0x01`/`0x02` are **native motor commands** that only double as alarm-menu
navigation while the alarm digit is actively blinking (EDIT mode). If the menu
times out mid-sequence, a stray Up/Down would drive the motor. This config
guards every alarm-adjust tap with three interlocks (edit-mode confirmed, fresh
menu frame within 450 ms, desk height unchanged). Full detail and the
step-direction logic are in [PROTOCOL.md](PROTOCOL.md#alarm-safety).

## Flashing

Add this config to your ESPHome dashboard and compile + OTA it as usual.
Requires a `secrets` file for wifi. The HA API runs plaintext by default; add an
`api_encryption_key` secret and enable `api.encryption.key` to harden it.
