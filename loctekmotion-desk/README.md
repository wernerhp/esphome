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
| `cover` Desk | Up/down/position control (internal, driven by the height sensor). Reports live motion: **Opening** while the desk rises, **Closing** while it lowers, **idle** when stopped — regardless of whether the move came from the keypad, a preset, or HA. |
| `switch` **Desk Lock** | Toggle: **On** blocks every keypad/ESP command from reaching the desk (motors won't move) while still forwarding idle heartbeats so the display sleeps normally; **Off** restores normal control. Also toggled at the keypad by holding **Preset 1+2+3 together for ~2 s**. See [Desk Lock](#desk-lock). |
| `switch` **Alarm** | Toggle: **On** arms the sit/stand reminder to *Alarm Timer* minutes (fires the keypad commands); **Off** turns the alarm off (long `0x40` hold). Also passively **syncs** to the physical keypad: the switch tracks the On/Off shown on the desk display (any live alarm menu => On, `0FF` => Off), state only, no commands fired. |
| `number` **Alarm Timer** | Target countdown in minutes (1..99, unit `min`), persisted. A manual keypad edit is written here once, when Edit mode ends (you press A or it times out), so HA history records only saved values. |
| `button` **Memory** | Program a memory preset (`0x20`). |
| `button` **Wake** | Wake the display (PIN20). Note: the serial `00 00` heartbeat does **not** wake a sleeping panel; a real key event is required. See PROTOCOL.md. |
| `text_sensor` Display | Live mirror of the 7-segment display (height, menu, `0FF`, `ASr`, etc.). Diagnostic-class, hidden by default. |
| `text_sensor` Keypad Command | Readable label of the current keypress/combo (`Preset 1 (0x04)`, `Preset 1+2 (0x0C)`, `none`, etc.), debounced ~60 ms and reset to `none` between presses so it works as a clean automation trigger — including the desk-ignored free combos (1+2 / 2+3 / 1+3). Diagnostic-class, hidden by default; enable it in HA to build keypad automations. See [Keypad automations](#keypad-automations). |

## Alarm safety (summary)

`0x01`/`0x02` are **native motor commands** that only double as alarm-menu
navigation while the alarm digit is actively blinking (EDIT mode). If the menu
times out mid-sequence, a stray Up/Down would drive the motor. This config
guards every alarm-adjust tap with three interlocks (edit-mode confirmed, fresh
menu frame within 450 ms, desk height unchanged). Full detail and the
step-direction logic are in [PROTOCOL.md](PROTOCOL.md#alarm-safety).

## Desk Lock

`switch.desk_lock` (or holding **Preset 1 + 2 + 3** together at the keypad for
~2 s) puts the desk into a locked state. While locked:

- Every **command** frame from the keypad or the ESP (up, down, presets, memory,
  alarm) is dropped before it reaches the desk control box, so the motors won't
  move no matter what is pressed.
- Any command injected from HA (e.g. a held cover Up/Down) is **continuously
  cleared**, so a queued move can't survive the lock and fire the instant you
  unlock.
- Idle `00 00` heartbeats are still forwarded, so the desk's own display-off
  timer keeps running and the panel sleeps normally.

The keypad gesture and the HA switch stay in sync (either one reflects the
other), and the lock state is restored across reboots.

> **Note — no "LOCKED" on the panel.** The display can't show a lock indicator
> without a hardware change: the desk's TX line is wired straight to the keypad's
> RX line, so the ESP can't inject text onto the panel unless that link is cut
> and routed through the ESP (the same MITM approach used on the command line).
> Not done here; possible future mod.

## Keypad automations

Enable the **Keypad Command** text sensor (hidden by default) to drive Home
Assistant automations from physical keypresses. It publishes a readable label for
the current key/combo and returns to `none` between presses, debounced ~60 ms so
intermediate combos (e.g. `Preset 1+2` on the way to `Preset 1+2+3`) don't
false-fire. The three **free combos** the desk itself ignores — `Preset 1+2`,
`Preset 2+3`, `Preset 1+3` — make good dedicated triggers.

Example: announce "standing" when Preset 1 is pressed and "sitting" when Preset 2
is pressed (e.g. through a media player or notify service):

```yaml
automation:
  - alias: "Desk — announce standing"
    trigger:
      - platform: state
        entity_id: sensor.tekdesk_keypad_command
        to: "Preset 1 (0x04)"
    action:
      - service: tts.google_translate_say
        data:
          entity_id: media_player.office
          message: "Standing"

  - alias: "Desk — announce sitting"
    trigger:
      - platform: state
        entity_id: sensor.tekdesk_keypad_command
        to: "Preset 2 (0x08)"
    action:
      - service: tts.google_translate_say
        data:
          entity_id: media_player.office
          message: "Sitting"
```

Swap the `to:` value for any label in the sensor (`Preset 1+2 (0x0C)`,
`Memory (0x20)`, `Up (0x01)`, etc.) and the action for whatever you want —
lights, scenes, presence, logging. The exact `entity_id` of the sensor depends
on your `device_name` substitution.

## Flashing

Add this config to your ESPHome dashboard and compile + OTA it as usual.
Requires a `secrets` file for wifi. The HA API runs plaintext by default; add an
`api_encryption_key` secret and enable `api.encryption.key` to harden it.
