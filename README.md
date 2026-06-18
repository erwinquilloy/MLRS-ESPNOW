# MLRS-ESPNOW

ESP-NOW firmware for an mLRS ground-side bridge.

`mlrs_espnow_gcs_heltec_v2` runs on a **Heltec WiFi Kit 32 (v1/v2)** and turns it
into a wireless link between an mLRS Nomad (transmitting MAVLink over ESP-NOW)
and an **MFD Mini Crossbow** OSD (consuming MAVLink over UART). Live telemetry
is also rendered on the board's built-in SSD1306 OLED.

```
   mLRS Nomad ──ESP-NOW──▶ Heltec WiFi Kit 32 ──UART──▶ MFD Mini Crossbow
                                  │
                                  └── SSD1306 OLED (telemetry HUD)
```

## Features

- Bidirectional MAVLink bridge (downlink + uplink) between ESP-NOW and UART.
- Channel auto-scan (1 / 6 / 11 / 13) until an mLRS bridge is found.
- Sender MAC latching so only the paired bridge is accepted.
- Built-in MAVLink v1/v2 parser (no MAVLink library dependency).
- Decoded telemetry on OLED: flight mode, arm state, GPS fix/sats, lat/lon,
  altitude, ground speed, heading, battery V/%, and RSSI %.
- ArduPilot / iNav firmware autodetection from the HEARTBEAT autopilot field
  (mode names are mapped accordingly).
- Link status indicator (`LINK OK` / `NO DATA` / `waiting`) based on packet
  recency.

## Hardware

| Item | Notes |
| ---- | ----- |
| Heltec WiFi Kit 32 | v1 or v2 (micro-USB variants); both share the same OLED pinout used here |
| MFD Mini Crossbow OSD | MAVLink input @ 115200 8N1 |
| mLRS Nomad | Configured as the ESP-NOW bridge counterpart |

### Wiring

| Heltec pin | Connects to |
| ---------- | ----------- |
| `GPIO17` (TX2) | Crossbow MAVLink input |
| `GND`          | Crossbow GND |
| micro-USB      | 5 V power |

UART RX is intentionally disabled (`-1`) so that **GPIO16** stays free as the
OLED reset line.

## mLRS Nomad (Tx) setup

On the Nomad transmitter, the WiFi Bridge mode **must be set to `ESP-NOW`**
(not `WiFi UDP` / `WiFi TCP`). The Heltec only listens on ESP-NOW and will
never latch onto a Nomad that's broadcasting over UDP.

Quick path via the OLED menu:

```
Setup -> WiFi Bridge -> Mode -> ESP-NOW
```

After changing the mode, reboot the Nomad so the new radio mode takes effect.
The Heltec's channel auto-scan (1 / 6 / 11 / 13) will then pick the Nomad up
within a few seconds.

## Build & flash

### Arduino IDE

1. Install the **ESP32 Arduino core ≥ 3.0.0** via Boards Manager.
2. Install these libraries via Library Manager:
   - Adafruit SSD1306
   - Adafruit GFX Library
3. Board: **ESP32 Arduino → Heltec WiFi Kit 32**.
4. Open `mlrs_espnow_gcs_heltec_v2/mlrs_espnow_gcs_heltec_v2.ino`, then Upload.

### Configurable defaults

Set at the top of the `.ino`:

| Define | Default | Purpose |
| ------ | ------- | ------- |
| `CROSSBOW_BAUD` | `115200` | UART baud to the Crossbow |
| `TX_PIN` | `17` | UART TX pin |
| `OLED_UPDATE_MS` | `200` | OLED refresh interval |
| `OLED_SDA` / `OLED_SCL` / `OLED_RST` | `4` / `15` / `16` | OLED I2C pins (Heltec defaults) |
| `OLED_ADDR` | `0x3C` | SSD1306 I2C address |

The WiFi country is locked to `EU` (channels 1-13) and the radio is forced to
`802.11b` for maximum ESP-NOW range.

## Supported MAVLink messages

| Msg ID | Name | Fields used |
| ------ | ---- | ----------- |
| 0  | `HEARTBEAT`           | `type`, `autopilot`, `base_mode`, `custom_mode` |
| 1  | `SYS_STATUS`          | `voltage_battery`, `battery_remaining` |
| 24 | `GPS_RAW_INT`         | `fix_type`, `satellites_visible` |
| 33 | `GLOBAL_POSITION_INT` | `lat`, `lon`, `alt`, `vx`, `vy`, `hdg` |
| 109 | `RADIO_STATUS`       | `rssi` (treated as 0-100% per mLRS, not 0-254 SiK) |

All other messages are still forwarded over UART -- only these are decoded for
the on-board OLED.

## License

GPL v3.
