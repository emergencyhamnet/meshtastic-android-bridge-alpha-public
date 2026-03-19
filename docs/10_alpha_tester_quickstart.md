# Alpha Tester Quickstart

This guide is for installing and validating the Android Meshtastic Bridge alpha build on a tester phone.

## 1. Package contents

Expected alpha package files:
1. `meshtastic-bridge-alpha-YYYY-MM-DD.apk`
2. `SHA256SUMS.txt`

## 2. Install options

### Option A: Install from phone file manager

1. Copy the APK to phone storage (Downloads is fine).
2. Open the APK from Files app.
3. If prompted, allow install from unknown sources for the file manager app.
4. Complete install.

### Option B: Install over USB (ADB)

1. Enable Developer options on phone.
2. Enable USB debugging.
3. Connect phone by USB.
4. Run:

```powershell
adb devices
adb install -r meshtastic-bridge-alpha-YYYY-MM-DD.apk
```

## 3. First-run setup checklist

1. Open app.
2. Grant runtime permissions:
   1. SMS send
   2. SMS receive
   3. Notifications (Android 13+)
3. In Android Settings, grant Notification Access to Meshtastic Bridge.
4. Connect Meshtastic node over USB OTG.
5. Tap Start.

## 4. User Settings baseline for alpha

1. Set Reply routing mode to:
   1. Shared Channel for simple single-caller tests
   2. Channel-bound for per-caller private-channel tests
2. Set Shared channel explicitly (default expected: Private 1).
3. Add preferred caller rows:
   1. short name
   2. full international number
   3. optional RCS display name
   4. optional private channel (channel-bound mode)

## 5. Operator reply formats

Supported:
1. `shortname - message` (primary)
2. plain text `message` when one effective caller target is resolvable
3. `REPLY <messageId> <message>` (explicit message-id path)

Deprecated/ignored:
1. `REPLY <shortname> <message>`
2. `REPLY <message>`

## 6. Verify install and runtime

From app UI:
1. Start/Stop buttons respond.
2. Status box updates.
3. Activity log shows inbound/outbound entries.
4. System Settings shows active mode and channel summary.

From logs (optional):
```powershell
adb logcat -s BridgePoC
```

## 7. Fast rollback

If needed:
```powershell
adb uninstall org.emergencyham.bridge
adb install -r <previous_known_good.apk>
```