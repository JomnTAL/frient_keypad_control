# Frient KEPZB-110 Keypad for Alarmo (LED Sync Fix)

**A fork of [Darktoinon's original blueprint](https://github.com/Darktoinon/frient_keypad_control)** with critical fixes for the LED false-positive issue and improved code validation.

This blueprint manages multiple PIN codes and RFID badges to arm and disarm your Alarmo system using the Frient KEPZB-110 keypad, while synchronizing the status between the keypad and your alarm panel.

## What This Fixes

The Frient KEPZB-110 has an "optimistic" LED system: when you press a button, it immediately shows success (green for disarm, red for arm) **before** Home Assistant validates the code. This creates two problems:

1. **False confirmation**: Users think they disarmed the alarm when they didn't (keypad shows green, Alarmo stays armed)
2. **Lockout state**: After an invalid attempt, the keypad LED gets "stuck" opposite to Alarmo's actual state, requiring you to "re-arm" before you can actually disarm with a valid code

**This fork immediately re-synchronizes the keypad LED to match Alarmo's actual state whenever a code is invalid or missing.**

## Features

- Manage multiple PIN codes for arming/disarming
- Support for RFID badges (ISO14443 Type A / MIFARE Classic)
- Synchronize alarm states between Frient Keypad and Alarmo
- LED false-positive fix: Reverts keypad LEDs immediately when codes are invalid
- Code validation: Pre-validates PINs/RFIDs before attempting Alarmo actions
- Optional code requirements for arming/disarming
- Multilingual support (English/French) inherited from original

## Prerequisites

- Home Assistant with ZHA integration configured
- Alarmo custom component installed
- Frient KEPZB-110 keypad paired via ZHA
- ZHA Keypad Code configured in ZHA integration settings (found under Settings > Devices & Services > ZHA > Configure > [Your Keypad])

## Installation

Click the button below to import the blueprint directly into your Home Assistant instance:

[![Importer Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/JomnTAL/frient_keypad_control/refs/heads/main/frient_keypad_control.yaml)

Alternatively, you can manually import the blueprint by copying the YAML file to your blueprints folder: blueprints/automation/YOUR_USERNAME/frient_keypad_alarmo_fix.yaml

## Configuration Guide

### Step 1: Configure the ZHA Keypad Code

Before using the blueprint, you must set the "admin" code in ZHA that allows Home Assistant to control the keypad LEDs:

1. Go to Settings > Devices & Services > ZHA > Configure
2. Find your Frient keypad in the device list
3. Click Configure or Options
4. Set the Alarm control panel code (this is NOT a user PIN, but the code ZHA uses to sync keypad states)
5. Enter this same code in the blueprint's "Default PIN for Keypad LED Sync" field

### Step 2: Capture RFID Badge Codes

The Frient keypad transmits RFID codes only when you scan the badge AND press an action button. To capture your RFID codes:

1. Go to Developer Tools > Events
2. Enter zha_event in "Listen to events" and click Start Listening
3. On the keypad: Scan your badge, then immediately press Disarm, Arm Day, Arm Night, or Arm Away
4. Look for the event data: the code field will show your RFID value (format: +A1B2C3D4)
5. Copy this code including the plus sign into the blueprint's RFID List field

### Step 3: Set Up Authorized Codes

In the blueprint configuration, enter your codes as semicolon-separated values:

- For PINs: 1234;5678;0000
- For RFIDs: +A1B2C3D4;+12345678;+FFFFFFFF

Important: RFID codes must include the plus prefix exactly as captured from the events.

### Step 4: Configure Alarmo Users

For security, create matching users in Alarmo:

1. Open Alarmo > Users
2. Add a user for each PIN/RFID:
   - Name: e.g., "John" or "Badge 1"
   - Code: Exact match to blueprint entry (including the + for RFIDs)
   - Permissions: Set appropriate arm/disarm access

### Security Options

The blueprint includes toggles for:
- Require Code for Arming: Forces valid PIN/RFID to arm
- Require Code for Disarming: Forces valid PIN/RFID to disarm (strongly recommended)

## How It Works

Original behavior (problem): User presses Disarm without entering a code → Keypad LED turns GREEN → Alarmo rejects (no code) → Keypad stays GREEN → User confused and locked out.

Fixed behavior: User presses Disarm without code → Keypad LED turns GREEN briefly → Blueprint detects invalid code → Immediately forces keypad back to RED (matching Alarmo) → User knows attempt failed and can retry.

## Troubleshooting

**Keypad not responding to events**
- Verify the device is properly paired in ZHA (check Settings > Devices & Services > ZHA > Devices)
- Listen to zha_event in Developer Tools to confirm events fire when buttons are pressed
- Ensure battery level is above 20%

**RFID not working**
- Ensure you include the plus prefix in the RFID list (format: +12345678)
- Remember: Scan badge, then press action button. Scanning alone does not transmit data.

**LEDs don't sync properly**
- Verify the ZHA Keypad Code is correctly entered in the blueprint
- Increase the sync delay in the blueprint YAML if your network is slow (change milliseconds: 100 to 300 or 500)

**Can arm but not disarm (or vice versa)**
- Check Alarmo settings: Ensure "Require code to disarm" is enabled in Alarmo itself
- Verify the user code in Alarmo matches exactly what is in the blueprint lists

## Technical Details

The Frient KEPZB-110 maintains an internal state machine that updates LEDs optimistically. When a button is pressed, the keypad immediately updates its LED assuming success, then sends the Zigbee event to Home Assistant. This blueprint intercepts the event: if the code is valid, it executes the action and syncs the LED; if invalid or missing, it blocks the action and immediately forces the keypad LED back to match the current Alarmo state via alarm_control_panel service calls.

## Credits

- **Original author**: [Darktoinon](https://github.com/Darktoinon/frient_keypad_control) - Base logic, bidirectional sync, language support, and original concept
- **LED fix and validation improvements**: JomnTAL (this fork)
- **Community contributions**: Scootaash and Home Assistant community members for ZHA integration details

## License

GNU GPL v3 (same as original project)
