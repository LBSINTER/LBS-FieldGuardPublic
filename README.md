# LBS FieldGuard

Field tool for Android (and Windows) that watches the radio layer, network
traffic and filesystem for SS7 attacks, SIM OTA commands, silent SMS and
known Pegasus / NSO indicators.

This repo only hosts the public builds. The source lives in a private repo;
contact below if you need access.

**Latest build:** `LBS-FieldGuard-v1.1.1.apk` (in this repo, root directory).

Other downloads (always-current mirror): https://fieldguard.lbs-int.com/#download

---

## Install (Android)

Minimum: Android 8.0 (API 26). Architectures: arm64-v8a, x86_64.

1. Download `LBS-FieldGuard-v1.1.1.apk`.
2. On the phone, allow "install unknown apps" for the file manager or browser
   you used to open it (path varies by manufacturer — Settings > Apps, or
   Settings > Privacy on most phones).
3. Tap the APK to install.

Or via adb:

```
adb install LBS-FieldGuard-v1.1.1.apk
```

> Note: the v1.1.x builds are debug-keyed. If you have an older release
> installed and the upgrade fails, uninstall first.

First launch will ask for SMS, phone state, VPN and storage permissions:

| Permission                 | Used for                         |
|----------------------------|----------------------------------|
| RECEIVE_SMS / READ_SMS     | RIL monitor — SMS PDU intercept  |
| READ_PHONE_STATE           | IMSI, cell tower info            |
| BIND_VPN_SERVICE           | Packet capture (TUN interface)   |
| MANAGE_EXTERNAL_STORAGE    | File scanner                     |

Deny any of them and the rest still work. The app does not need an internet
connection to scan or to inspect SMS PDUs.

## Install (Windows)

Grab the Windows zip from the LBS download page above, install
[Npcap](https://npcap.com) if you want packet capture, then run
`LBSFieldGuard.exe`. Same screens as the Android build.

---

## What it does

**Scanner** — runs the bundled signature DB against files on the device.
Covers Pegasus stage-1 markers, SIM OTA scripts, SS7 RAT payloads, and
generic byte patterns. Manual or scheduled.

**RIL monitor** — registers a `BroadcastReceiver` on `SMS_RECEIVED` at
`Integer.MAX_VALUE` priority and decodes the PDU before any other app gets
it. Flags Type-0 silent SMS, SIM OTA (UDH 0x70 / 0x71), STK ProactiveCommand
(PID 0x7F), Class-0 flash, and binary SMS.

**Packet analyser** — VPN-based capture on Android, Npcap on Windows. Looks
for known NSO / Pegasus C2 hosts, SIGTRAN ports (M3UA, SUA, SCCP, TCAP),
rogue GTP tunnels and ICMP redirects.

**PDU builder** — assemble SMS-SUBMIT PDUs from templates (Type-0, STK,
SIM OTA, binary, WAP Push, custom). Outputs the hex string and a per-field
breakdown. Useful for testing the RIL monitor.

**SS7 / STK / SIM-OTA catalogue** — browseable database of ~67 known attack
patterns across 13 groups, with PID / DCS / UDH classification and a short
description for each.

**Secure Comms** (new in 1.1.x) — end-to-end encrypted DMs and groups,
running against the LBS comms relay. Open registration; profiles, group
types, media share.

**Station probe** — optional persistent TCP link to the LBS station for
signature DB updates and alert relay. Off by default; configurable in
Settings.

---

## What's new since 1.0.6

- **1.0.7** — PC Bridge desktop companion; PDU catalogue expanded to 67
  patterns / 13 groups.
- **1.1.0** — Secure Comms (DMs, group types, media share, open
  registration).
- **1.1.1** — In-app updater now talks to the LBS update server first
  (faster, no GitHub rate limits); native update notifications.

---

## Building from source

Source isn't in this repo. If you have access to the private source repo:

```
npm install
cd android
./gradlew assembleRelease
# Output: android/app/build/outputs/apk/release/app-release.apk
```

For a properly signed build, set these in `android/gradle.properties`:

```
FIELDGUARD_UPLOAD_STORE_FILE=path/to/your.keystore
FIELDGUARD_UPLOAD_KEY_ALIAS=...
FIELDGUARD_UPLOAD_STORE_PASSWORD=...
FIELDGUARD_UPLOAD_KEY_PASSWORD=...
```

Windows: `npx react-native run-windows` (needs the UWP workload in VS 2022).

---

## Contact

Bug reports, audit / commercial queries: **nicholai@lbs-int.com**

© LBS INT.
