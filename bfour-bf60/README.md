# BFOUR BF-60 Meat Thermometer - ESPHome Integration

ESPHome configuration for integrating the BFOUR BF-60 Bluetooth meat thermometer system with Home Assistant.

## System Architecture

```
┌─────────────────┐     ┌─────────────────┐
│   BF-40 Probe   │     │   BF-40 Probe   │
│  (Black/White)  │     │  (Black/White)  │
│  7C:72:E7:...   │     │  54:FE:EB:...   │
│  Service: FFF0  │     │  Service: FFF0  │
└────────┬────────┘     └────────┬────────┘
         │   Wireless (proprietary)   │
         └────────────┬───────────────┘
                      ▼
            ┌─────────────────┐
            │  BF-60 Base     │
            │  "Grill BT5.0"  │
            │  02:CA:6A:...   │
            │  Service: FFB0  │
            └────────┬────────┘
                     │ BLE
                     ▼
            ┌─────────────────┐
            │  ESP32-C3       │
            │  (ESPHome)      │
            └────────┬────────┘
                     │ WiFi
                     ▼
            ┌─────────────────┐
            │ Home Assistant  │
            └─────────────────┘
```

The BF-40 wireless probes transmit temperature data to the BF-60 base station, which aggregates the data and exposes it via BLE.

---

## BF-60 Base Station

### Device Information

| Property | Value |
|----------|-------|
| Device Name | Grill BT5.0 |
| MAC Address | XX:XX:XX:XX:XX:XX (Random) |
| Firmware | Version1.0 |
| Role | Base station / data aggregator |

### BLE Services

| Service UUID | Description |
|--------------|-------------|
| `0x1800` | Generic Access (GAP) |
| `0x1801` | Generic Attribute (GATT) |
| `0xFFB0` | **Thermometer Data Service** |
| `0xFE59` | Nordic Secure DFU |

### Thermometer Service (0xFFB0)

### Characteristics

| UUID | Properties | Description |
|------|------------|-------------|
| `0xFFB1` | Write | Command/Control characteristic |
| `0xFFB2` | Notify | Temperature data & alerts |

## Protocol Documentation

### Temperature Data Packet (15 bytes)

Sent via notifications on `FFB2` approximately every 1.8 seconds.

```
Byte:  0    1    2    3    4    5    6    7    8    9   10   11   12   13   14
     [55] [00] [BH] [BL] [WH] [WL] [--] [--] [--] [--] [--] [--] [--] [--] [--]
```

| Byte(s) | Description |
|---------|-------------|
| 0 | Header: `0x55` |
| 1 | Packet type: `0x00` = Temperature data |
| 2-3 | Black probe temperature (big-endian, divide by 10 for °C) |
| 4-5 | White probe temperature (big-endian, divide by 10 for °C) |
| 6-14 | Reserved (typically `0xFF`) |

**Special Values:**
- `0xFFFF` for temperature bytes = Probe disconnected

**Example:**
```
55 00 01 0F 01 0D FF FF FF FF FF FF FF FF FF
       └─┬─┘ └─┬─┘
        │     └── White: 0x010D = 269 → 26.9°C
        └──────── Black: 0x010F = 271 → 27.1°C
```

### Alarm Packet (5 bytes)
#### Alarm Acknowledgment Packet (5 bytes)

Sent when the user presses the touch button on the base station to acknowledge/silence a beeping alarm.

> **Note:** This packet is NOT sent when the alarm condition is first met. The base station beeps automatically when temperature reaches the target, but this BLE packet is only transmitted when the user physically interacts with the device to silence it. For real-time alarm detection, ESPHome compares the current temperature against the target temperature.

```
Byte:  0    1    2    3    4
     [55] [07] [01] [ID] [CS]
```

| Byte | Description |
|------|-------------|
| 0 | Header: `0x55` |
| 1 | Packet type: `0x07` = Alarm acknowledgment |
| 2 | Acknowledgment status: `0x01` = Acknowledged |
| 3 | Probe ID: `0x02` = White, `0x03` = Black |
| 4 | Checksum |

**Examples:**
```
55 07 01 02 5F  → White probe alarm acknowledged (button pressed)
55 07 01 03 60  → Black probe alarm acknowledged (button pressed)
```

### Command Packets (Write to FFB1)

#### Set Temperature Unit

```
55 01 [UNIT] 00
```

| UNIT | Description |
|------|-------------|
| `0x01` | Celsius |
| `0x02` | Fahrenheit |

#### Set Target Temperature

```
55 02 [PROBE] [TH] [TL]
```

| Byte | Description |
|------|-------------|
| PROBE | `0x01` = Black probe, `0x02` = White probe |
| TH | Temperature high byte (upper 8 bits of 16-bit value) |
| TL | Temperature low byte (lower 8 bits of 16-bit value) |

> **Note:** TH and TL are the two bytes of a single 16-bit temperature value (big-endian), not separate high/low temperature limits. Temperature is in whole degrees (not divided by 10).

**Example:** Set black probe target to 75°C
```
75 decimal = 0x004B (16-bit)
  TH = 0x00 (high byte)
  TL = 0x4B (low byte)

Packet: 55 02 01 00 4B
```

## ESPHome Entities

### Binary Sensors
- **Grill Connected** - BLE connection status
- **Black Probe Connected** - Black probe insertion status
- **White Probe Connected** - White probe insertion status
- **Black Probe Alarm** - Black probe target temperature reached (software-detected)
- **White Probe Alarm** - White probe target temperature reached (software-detected)
- **Black Probe Alarm Acknowledged** - User pressed button to silence black probe alarm
- **White Probe Alarm Acknowledged** - User pressed button to silence white probe alarm

### Sensors
- **Black Probe Temperature** - Current temperature (°C)
- **White Probe Temperature** - Current temperature (°C)
- **Grill Signal Strength** - Bluetooth signal strength (0-100%)

### Controls
- **Black Probe Target Temperature** - Set target (0-300°C)
- **White Probe Target Temperature** - Set target (0-300°C)
- **Grill Temperature Unit** - Celsius/Fahrenheit (controls base station display)

> **Note on Temperature Units:** ESPHome always reports temperatures in Celsius. The "Temperature Unit" selector changes the display on the physical base station only. Home Assistant will automatically convert to your preferred unit based on your HA locale settings.

### Diagnostic
- **Grill Firmware Version** - Base station firmware
- **Black/White Probe Alarm Acknowledged** - Button press detection

## Features

- ✅ Real-time temperature monitoring
- ✅ Probe connection detection
- ✅ Target temperature alerts
- ✅ Temperature unit selection (°C/°F)
- ✅ Target temperature persistence across reconnects
- ✅ Home Assistant integration via ESPHome API

## Hardware Requirements

- ESP32-C3 DevKitM-1 (or compatible ESP32 with BLE)
- BFOUR BF-60 Meat Thermometer

## Installation

### Option 1: Dashboard Import (Recommended)

1. Flash the firmware to your ESP32 using [ESPHome Web](https://web.esphome.io/)
2. Connect to the device's WiFi AP (`bfour-bf-60`) and configure WiFi credentials
3. The device will appear in your ESPHome Dashboard for adoption
4. After adoption, update the `grill_mac` substitution with your BF-60's MAC address
5. To find your MAC address, use `bfour-discovery.yaml` first

### Option 2: Manual Installation

1. Add this to your ESPHome configuration:
   ```yaml
   packages:
     bfour: github://wernerhp/esphome/bfour-bf60/bfour-bf-60.yaml@main
   
   substitutions:
     grill_mac: "XX:XX:XX:XX:XX:XX"  # Your BF-60 MAC address
   ```
2. Add WiFi credentials to your `secrets.yaml`:
   ```yaml
   wifi_ssid: "your_wifi"
   wifi_password: "your_password"
   ```
3. Compile and flash to your ESP32

### Finding Your BF-60 MAC Address

1. Flash `bfour-discovery.yaml` to your ESP32
2. Power on your BF-60 base station
3. Check the ESPHome logs - the MAC address will be displayed when found

## Protocol Discovery Tools Used

- **nRF Connect** (Android/iOS) - Service discovery and characteristic analysis
- **LightBlue** (iOS) - BLE packet capture
- **ESPHome BLE Client** - Real-time packet monitoring

## Known Limitations

- Device uses random BLE address type
- No battery level available (probes don't expose this via base station)
- Nordic DFU service present but not utilized

## Related Devices

The BF-40 probes use a different protocol with service UUID `0xFFF0`. See BF-40 section below.

---

## BF-40 Wireless Probes

### Device Information

| Property | Black Probe | White Probe |
|----------|-------------|-------------|
| Device Name | BF-40 | BF-40 |
| MAC Address | XX:XX:XX:XX:XX:XX | XX:XX:XX:XX:XX:XX |
| Address Type | Public | Public |
| Role | Wireless temperature probe | Wireless temperature probe |

### BLE Services

| Service UUID | Description |
|--------------|-------------|
| `0x1800` | Generic Access (GAP) |
| `0x1801` | Generic Attribute (GATT) |
| `0xFFF0` | **Probe Data Service** |

### Probe Service (0xFFF0)

| UUID | Properties | Description |
|------|------------|-------------|
| `0xFFF1` | Read, Write | Configuration (has User Description) |
| `0xFFF2` | Read | Status/Data (has User Description) |
| `0xFFF3` | Write | Command input (has User Description) |
| `0xFFF4` | Notify | Temperature notifications (has CCCD + User Description) |
| `0xFFF5` | Read | Additional data (has User Description) |

### Notes on BF-40 Probes

- Each probe has its own BLE address and can be connected directly
- The BF-60 base station receives data wirelessly (not via BLE) from the probes
- For ESPHome integration, connecting to the BF-60 base is recommended as it provides aggregated data from all probes
- Direct probe connection may be useful for debugging or alternative setups

---

## License

This project is provided as-is for personal use. BFOUR is a trademark of its respective owner.

## Acknowledgments

Protocol reverse-engineered through BLE packet analysis in January 2026.
