# Frient KEPZB-110 Keypad for Alarmo (with LED Sync Fix)

[![Home Assistant Blueprint](https://img.shields.io/badge/Home%20Assistant-Blueprint-blue)](https://my.home-assistant.io/badges/blueprint_import.svg)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

**A fork of [Darktoinon's original blueprint](https://github.com/Darktoinon/frient_keypad_control)** with critical fixes for the LED false-positive issue and improved code validation.

## What This Fixes

The Frient KEPZB-110 has an "optimistic" LED system: when you press a button, it immediately shows success (green for disarm, red for arm) **before** Home Assistant validates the code. This creates two problems:

1. **False confirmation**: Users think they disarmed the alarm when they didn't (keypad shows green, Alarmo stays armed)
2. **Lockout state**: After an invalid attempt, the keypad LED gets "stuck" opposite to Alarmo's actual state, requiring you to "re-arm" before you can actually disarm with a valid code

**This fork immediately re-synchronizes the keypad LED to match Alarmo's actual state whenever a code is invalid or missing.**

## Features

- **Multiple PIN codes** support for arming/disarming
- **RFID badge support** (ISO14443 Type A / MIFARE Classic)
- **Bidirectional sync**: Keypad ↔ Alarmo state synchronization
- **Code validation**: Pre-validates codes before attempting Alarmo actions
- **LED correction**: Instantly fixes keypad LEDs when codes are rejected
- **Optional code requirements**: Toggle whether arming/disarming requires codes

## Prerequisites

- Home Assistant with [ZHA integration](https://www.home-assistant.io/integrations/zha/)
- [Alarmo](https://github.com/nielsfaber/alarmo) custom component installed
- Frient KEPZB-110 keypad paired via ZHA
- **ZHA Keypad Code**: Must be configured in ZHA integration settings (see below)

## Installation

### 1. Import the Blueprint

In **Blueprints**, import URL: https://raw.githubusercontent.com/JomnTAL/frient-keypad-led-fix/main/frient_keypad_alarmo_fix.yaml

### 2. Create the Automation

1. Go to **Settings** → **Automations & Scenes** → **Create Automation** → **Use Blueprint**
2. Select "Frient KEPZB-110 Keypad for Alarmo (with LED Sync Fix)"

### 3. Configure the ZHA Keypad Code

Before the blueprint can control the keypad LEDs, you must set the "admin" code in ZHA:

1. **Settings** → **Devices & Services** → **ZHA** → **Configure**
2. Find your Frient keypad in the list
3. Click **Configure** or **Options**
4. Set the **Alarm control panel code** (e.g., `1234`)
5. Enter this same code in the blueprint's "Default PIN for Keypad LED Sync" field

*Note: This code is only used to sync the keypad's LED status, not for user authentication.*

## Configuration Guide

### Capturing RFID Badge Codes

The Frient keypad transmits RFID codes only when you **scan + press an action button**:

1. Go to **Developer Tools** → **Events**
2. Enter `zha_event` in "Listen to events"
3. Click **Start Listening**
4. On the keypad:
   - Scan your RFID badge
   - **Immediately press** Disarm, Arm Day, Arm Night, or Arm Away
5. Look for the event data:
   ```yaml
   event_type: zha_event
   data:
     command: arm
     args:
       arm_mode: 0  # 0=disarm, 1=home, 2=night, 3=away
       code: "+A1B2C3D4"  # Your RFID code with + prefix
   ```
6. Copy the code (including the `+`) into the blueprint's **RFID List** field

### Setting Up Codes

Enter codes in the blueprint as **semicolon-separated values**:

- **PINs**: `1234;5678;0000`
- **RFIDs**: `+A1B2C3D4;+12345678;+FFFFFFFF`

**Important**: RFID codes must include the `+` prefix exactly as captured from ZHA events.

### Alarmo User Setup

For security, create matching users in Alarmo:

1. Open **Alarmo** → **Users**
2. Add a user for each PIN/RFID:
   - **Name**: e.g., "John", "Badge 1"
   - **Code**: Exact match to blueprint entry (including `+` for RFIDs)
   - **Permissions**: Set arm/disarm access as needed

### Security Options

- **Require Code for Arming**: If enabled, the keypad will only arm when a valid PIN/RFID is provided
- **Require Code for Disarming**: **Strongly recommended** - prevents unauthorized disarming

## How It Works

### Original Behavior (Problem)
```
User presses Disarm (no code) → Keypad LED turns GREEN → Alarmo rejects (no code) → Keypad stays GREEN → User confused
```

### This Fork's Behavior (Fixed)
```
User presses Disarm (no code) → Keypad LED turns GREEN → Blueprint detects invalid code → Immediately forces keypad back to RED (matching Alarmo) → User knows attempt failed
```

## Troubleshooting

### "Keypad not responding to events"
- Verify the device is properly paired in ZHA (check **Settings** → **Devices & Services** → **ZHA** → **Devices**)
- Listen to `zha_event` in Developer Tools to confirm events are firing when you press buttons
- Ensure battery level is above 20%

### "RFID not working"
- Ensure you include the `+` prefix in the RFID list (format: `+12345678`)
- Remember: **Scan badge → Press action button**. Scanning alone does not transmit data.

### "LEDs don't sync properly"
- Verify the **ZHA Keypad Code** is correctly entered in the blueprint
- Increase the sync delay in the blueprint YAML if your network is slow (change `milliseconds: 100` to `300` or `500`)

### "Can arm but not disarm (or vice versa)"
- Check Alarmo settings: Ensure "Require code to disarm" is enabled in Alarmo itself
- Verify the user code in Alarmo matches exactly what's in the blueprint lists

## Technical Details

The Frient KEPZB-110 maintains an internal state machine that updates LEDs optimistically. When a button is pressed:
1. Keypad immediately updates its LED (assuming success)
2. Zigbee event is sent to Home Assistant
3. This blueprint intercepts the event:
   - **Valid code**: Allows action, syncs LED to match result
   - **Invalid/missing code**: Blocks action, forces keypad LED to match current Alarmo state via `alarm_control_panel` service calls

## Credits

- **Original author**: [Darktoinon](https://github.com/Darktoinon/frient_keypad_control) - Base logic, bidirectional sync, and original concept
- **LED fix & validation logic**: JomnTAL (this fork)
- **Community contributions**: Scootaash and Home Assistant community members for ZHA integration details

## License

GNU GPL v3 (same as original project)
