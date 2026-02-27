# LBS FieldGuard

Field tool for Android and Windows that monitors the radio layer, network traffic,
and filesystem for SS7-based attacks, SIM OTA commands, Pegasus/NSO implant
indicators, and silent SMS tracking.

Releases (APK and Windows binaries): https://github.com/LBSINTER/LBS-FieldGuardPublic/releases


---


## Contents

- [What it does](#what-it-does)
- [Platforms and requirements](#platforms-and-requirements)
- [Installation](#installation)
- [Screens](#screens)
- [Architecture](#architecture)
- [Project layout](#project-layout)
- [Building from source](#building-from-source)
- [Native modules](#native-modules)
- [Signature database](#signature-database)
- [SS7 payload catalogue](#ss7-payload-catalogue)
- [Station probe protocol](#station-probe-protocol)
- [Auto-update system](#auto-update-system)
- [Running tests](#running-tests)
- [Releasing a new version](#releasing-a-new-version)


---


## What it does

LBS FieldGuard runs on an Android device or a Windows workstation and provides six
main functions:

**Byte-pattern scanner**
Scans the device filesystem for files matching known attack signatures. The database
covers SS7 RAT payloads, Pegasus/NSO Group stage-1 indicators, SIM OTA scripts,
and generic malware byte patterns.

**RIL monitor**
Hooks into the Android radio layer via a BroadcastReceiver registered at maximum
priority. Intercepts inbound SMS PDUs before they reach other apps and classifies
them: Type-0 silent SMS, SIM OTA commands (IEI 0x70/0x71), STK ProactiveCommand
(PID 0x7F), binary SMS, and Class-0 flash SMS. Generates an alert for any PDU that
matches a known attack pattern.

**Packet analyser**
Captures network traffic using the Android VPN TUN interface on mobile or
WinPcap/Npcap on Windows. Flags outbound connections to known NSO/Pegasus C2
addresses, traffic on SIGTRAN ports (M3UA, SUA, SCCP, TCAP), rogue GTP tunnel
creation, and ICMP redirects.

**PDU builder**
Constructs SMS-SUBMIT PDUs from templates for testing and verification. Supported
templates: Type-0 silent, STK ProactiveCommand, SIM OTA, binary, WAP Push, and
fully custom. Output is shown as a hex dump alongside a decoded field breakdown.

**SS7 payload catalogue**
Static database of 15+ known attack types with PID, DCS, and UDH classification,
plain-language descriptions, and mitigation references. Available from the Alerts
screen for quick lookup during an incident.

**Station probe**
Persistent TCP connection to the LBS station at 140.82.39.182:5556. Used to pull
signature database updates, push device alerts to the operations centre, and relay
telemetry in real time.


---


## Platforms and requirements

| Component         | Android              | Windows                  |
|-------------------|----------------------|--------------------------|
| OS version        | Android 8.0 (API 26) | Windows 10 1903 or later |
| Architecture      | arm64-v8a, x86_64    | x64                      |
| Extra dependency  | none                 | Npcap (npcap.com)        |
| Node.js (build)   | 18 or later          | 18 or later              |
| Java (build)      | 17                   | not required             |
| Visual Studio     | not required         | 2022 with UWP workload   |


---


## Installation

### Android

1. Download `LBS-FieldGuard-vX.Y.Z.apk` from the
   [releases page](https://github.com/LBSINTER/LBS-FieldGuardPublic/releases/latest).
2. On the device, open Settings and find "Install unknown apps" (the exact path
   varies by manufacturer — it may be under Apps, Privacy, or Biometrics & Security).
   Allow the file manager or browser you will use to open the APK.
3. Transfer the APK to the device and tap the file to install.

Via ADB:

```
adb install LBS-FieldGuard-vX.Y.Z.apk
```

Required permissions (requested at first run):

| Permission                    | Purpose                              |
|-------------------------------|--------------------------------------|
| `RECEIVE_SMS` / `READ_SMS`    | RIL monitor SMS intercept            |
| `READ_PHONE_STATE`            | IMSI and cell tower information      |
| `BIND_VPN_SERVICE`            | Packet capture via TUN interface     |
| `MANAGE_EXTERNAL_STORAGE`     | File scanner access                  |

The app does not require a network connection to function. The station probe and
auto-update check are both optional features and can be left inactive.

### Windows

1. Download `LBS-FieldGuard-vX.Y.Z-windows.zip` from the
   [releases page](https://github.com/LBSINTER/LBS-FieldGuardPublic/releases/latest).
2. Install [Npcap](https://npcap.com) if packet capture is needed.
3. Extract the zip and run `LBSFieldGuard.exe`.

The Windows build uses react-native-windows and renders the same interface as the
Android version. Npcap is only required if you want the packet analyser to work;
all other screens function without it.


---


## Screens

**Dashboard**
Top-level status view. Shows the probe connection state, signature database
version, active alerts count, and the platform (Android or Windows). On Android,
also shows RIL monitor status.

**Scanner**
Run an on-demand scan or schedule recurring scans. Results list matched files with
the signature name, category, severity, and matched byte offset.

**PDU Builder**
Select a template, fill in the fields, and build a PDU. The output shows the
assembled hex string and a side-by-side field breakdown. Useful for constructing
test vectors when verifying RIL monitor classification.

**Alerts**
Chronological list of all alerts generated since the session started. Each alert
includes the source (RIL, scanner, or packet analyser), severity, timestamp, and
a short description. Tapping an alert opens the relevant SS7 catalogue entry if
one exists.

**Probe**
Manual control for the station connection. Shows latency (PING/PONG), last
signature update time, and a log of raw probe messages. Allows manually requesting
a DB update.

**Settings**
Configure the station address and port, set scan schedule and target directories,
toggle RIL monitoring and packet capture, and view the current app version.


---


## Architecture

```
LBS FieldGuard (React Native 0.73)
+--------------------------------------------------+
|  Scanner         RIL Monitor      Packet         |
|  (TypeScript)    (TypeScript +    Analyser        |
|                  Java RIL Bridge) (TypeScript +   |
|                                   PCAPBridge.cs   |
|                                   on Windows)     |
|                                                   |
|  Signature DB    SS7 Catalogue    Probe Client    |
|  (JSON asset)    (TypeScript)     (TCP socket)    |
|                                                   |
|  Zustand store   React Navigation                 |
|  Dashboard  Scanner  PDU Builder  Alerts          |
|  Probe  Settings                                  |
+--------------------------------------------------+
          |                         |
          | TCP 5556                | GitHub Releases API
          v                         v
   LBS Station              LBS-FieldGuardPublic
   (140.82.39.182)          (version check + APK
   sig updates,              download)
   alert relay
```


---


## Project layout

```
LBS-FieldGuard/
  src/
    App.tsx                       root component, UpdateBanner mount
    config/
      build.ts                    APP_VERSION, RELEASES_API_URL, probe config
    scanner/
      SignatureDB.ts              byte-pattern loader and matcher
      FileScanner.ts              chunked filesystem scanner
    android/
      PDUCodec.ts                 SMS PDU encode/decode (GSM-7, binary, UCS-2)
      RILMonitor.ts               RIL event monitor and SMS PDU classifier
    network/
      PacketAnalyser.ts           packet capture handler and NSO/SS7 detector
    probe/
      ProbeClient.ts              TCP probe client (station protocol)
    ss7/
      PayloadCatalogue.ts         SS7 attack type database (15+ entries)
    store/
      appStore.ts                 Zustand global state
    types/
      index.ts                    shared TypeScript types
    utils/
      UpdateChecker.ts            GitHub releases API version check
      crypto.ts
      id.ts
    ui/
      RootNavigator.tsx
      components/
        Icon.tsx
        UpdateBanner.tsx          dismissible update notification banner
      hooks/
        useScreenSize.ts          responsive sizing hook
      screens/
        DashboardScreen.tsx
        ScannerScreen.tsx
        PDUBuilderScreen.tsx
        AlertsScreen.tsx
        ProbeScreen.tsx
        SettingsScreen.tsx
  android/
    app/src/main/
      AndroidManifest.xml
      java/com/lbs/fieldguard/ril/
        RILBridgeModule.java      SMS BroadcastReceiver NativeModule
        RILBridgePackage.java
      res/mipmap-*/
        ic_launcher.png           LBS site icon, all density variants
        ic_launcher_round.png
  windows/
    LBSFieldGuard/
      PCAPBridgeModule.cs         WinPcap/Npcap NativeModule (C#)
    LBSFieldGuard.Package/
      LBSFieldGuardPackage.cs
  assets/
    signatures/
      db.json                     bundled byte-pattern signature database
  __tests__/
    PDUCodec.test.ts
    SignatureDB.test.ts
    PayloadCatalogue.test.ts
  .github/
    workflows/
      build-apk.yml               release APK build + publish to FieldGuardPublic
      ci.yml                      PR / push checks (tests + Gradle debug build)
```


---


## Building from source

### Android APK

```bash
# Install JS dependencies
npm install

# Debug build — runs on a connected device or emulator
npx react-native run-android

# Release APK
cd android
./gradlew assembleRelease
# Output: android/app/build/outputs/apk/release/app-release.apk
```

For a signed release build, set these four properties in
`android/gradle.properties` or as environment variables:

```
FIELDGUARD_UPLOAD_STORE_FILE=path/to/your.keystore
FIELDGUARD_UPLOAD_KEY_ALIAS=your-alias
FIELDGUARD_UPLOAD_STORE_PASSWORD=...
FIELDGUARD_UPLOAD_KEY_PASSWORD=...
```

### Windows

```bash
npm install
npx react-native run-windows
```

Install [Npcap](https://npcap.com) on the build machine before running if you
want the packet capture features to be functional during development.

### GitHub Actions (automated)

Pushing a tag of the form `vX.Y.Z` triggers `.github/workflows/build-apk.yml`:

1. Checks out the repo, installs Node 20 and Java 17.
2. Runs `npm ci` and bundles JS with Metro.
3. Builds a signed release APK (or a debug-keyed APK if no keystore secret is set).
4. Creates a GitHub Release on `LBSINTER/LBS-FieldGuardPublic` and attaches the APK.
5. Archives the same APK to a release on the private repo.

Required repository secrets for signed builds and public publishing:

| Secret                        | Description                                             |
|-------------------------------|---------------------------------------------------------|
| `FIELDGUARD_KEYSTORE_BASE64`  | Base-64 encoded `.jks` keystore file                   |
| `FIELDGUARD_KEY_ALIAS`        | Key alias inside the keystore                          |
| `FIELDGUARD_STORE_PASSWORD`   | Keystore password                                      |
| `FIELDGUARD_KEY_PASSWORD`     | Key password                                           |
| `FIELDGUARD_PUBLIC_PAT`       | GitHub PAT with `repo` scope on LBS-FieldGuardPublic  |

If `FIELDGUARD_KEYSTORE_BASE64` is absent, a fresh debug keystore is generated at
build time (the APK is still installable but carries no release certificate).


---


## Native modules

### Android — RILBridgeModule (Java)

Location: `android/app/src/main/java/com/lbs/fieldguard/ril/`

- Registers a `BroadcastReceiver` for `android.provider.Telephony.SMS_RECEIVED`
  and `android.provider.Telephony.SMS_CB_RECEIVED`.
- Receiver priority: `Integer.MAX_VALUE` — intercepts PDUs before any other app.
- Emits `onRILMessage` events to JS with the full PDU as a hex string.
- Required manifest permissions: `RECEIVE_SMS`, `READ_PHONE_STATE`.

Register in `MainApplication.java`:

```java
packages.add(new RILBridgePackage());
```

### Windows — PCAPBridgeModule (C#)

Location: `windows/LBSFieldGuard/PCAPBridgeModule.cs`

- Uses SharpPcap and PacketDotNet (NuGet).
- Opens the first available Npcap adapter in promiscuous mode.
- Emits `onPacket` events with: `{ srcIp, dstIp, proto, srcPort, dstPort, payloadHex }`.
- Requires Npcap to be installed on the target machine for capture to start.

Register in `App.cpp`:

```cpp
PackageProviders().Append(winrt::make<LBSFieldGuardPackage>());
```


---


## Signature database

`assets/signatures/db.json` — compiled into the app bundle at build time.

Schema:

| Field        | Type   | Values                                                              |
|--------------|--------|---------------------------------------------------------------------|
| `id`         | string | unique identifier                                                   |
| `name`       | string | display name                                                        |
| `category`   | string | `stk`, `sim_ota`, `pegasus`, `ss7_map`, `ss7`, `gtp`, `malware`    |
| `severity`   | string | `info`, `low`, `medium`, `high`, `critical`                         |
| `pattern`    | string | hex bytes, space-separated                                          |
| `mask`       | string | optional byte mask: `FF` = must match, `00` = wildcard             |
| `description`| string | attack description                                                  |

To update signatures without rebuilding: the station probe pushes `MSG_SIG_UPDATE`
patches over the TCP connection. The app merges them into the in-memory database at
runtime. To ship a permanent update, replace `db.json` and release a new build.


---


## SS7 payload catalogue

`src/ss7/PayloadCatalogue.ts`

| ID                    | Layer    | Attack                                              |
|-----------------------|----------|-----------------------------------------------------|
| SILENT_TYPE0          | SMS-PP   | Type-0 silent SMS (PID 0x40)                        |
| FLASH_CLASS0          | SMS-PP   | Class-0 flash SMS (social engineering display)      |
| SIM_OTA_UDH70         | SIM-OTA  | SIM OTA command (IEI 0x70)                         |
| SIM_OTA_UDH71         | SIM-OTA  | SIM OTA response (IEI 0x71)                        |
| STK_PROACTIVE_7F      | STK      | STK ProactiveCommand (PID 0x7F)                    |
| STK_SEND_SMS          | STK      | SEND SHORT MESSAGE — data exfiltration             |
| STK_LAUNCH_BROWSER    | STK      | LAUNCH BROWSER — drive-by exploit delivery         |
| USSD_PUSH_HIJACK      | USSD     | MAP USSD push used for credential theft            |
| MAP_SRILSM            | SS7-MAP  | SRI-SM — IMSI harvesting                           |
| MAP_ATI               | SS7-MAP  | AnyTimeInterrogation — real-time location query    |
| MAP_ISD               | SS7-MAP  | InsertSubscriberData — call forwarding injection   |
| GTP_HIJACK            | GTP      | GTPv1 session spoofing                             |
| NSO_PEGASUS_STAGERHEX | SMS-PP   | Pegasus stage-1 binary SMS byte pattern indicator  |


---


## Station probe protocol

Wire format:

```
+--------+------------+--------------------+
| 2 bytes| 4 bytes    | N bytes            |
| MSG_TYPE| PAYLOAD_LEN| JSON payload       |
+--------+------------+--------------------+
```

Message types:

| Type   | Direction        | Description                                      |
|--------|------------------|--------------------------------------------------|
| 0x0001 | Client to station| PROBE_HELLO — device_id, platform, app version   |
| 0x0003 | Client to station| PING                                             |
| 0x0004 | Station to client| PONG — echoes timestamp for latency measurement  |
| 0x0005 | Station to client| SIG_UPDATE — signature DB patch (JSON)           |
| 0x0002 | Station to client| ALERT — station-pushed alert                     |

The station address and port are configurable in Settings (default:
140.82.39.182:5556). The connection is optional; all local detection features work
without it.


---


## Auto-update system

The app checks for newer versions on startup and shows a dismissible banner if one
is available. The banner links to the public releases page.

How it works:

1. `APP_VERSION` is embedded in the binary from `src/config/build.ts` at build time.
2. Thirty seconds after launch, `UpdateChecker.ts` fetches the GitHub Releases API
   url defined in `RELEASES_API_URL`.
3. The `tag_name` field of the latest release (e.g. `v1.0.3`) is compared against
   `APP_VERSION` using semver ordering.
4. If the release version is higher, the update banner appears at the bottom of
   every screen for the rest of the session.

The API endpoint and version constant are both in one file:

```typescript
// src/config/build.ts

export const APP_VERSION = '1.0.2';

export const RELEASES_API_URL =
  process.env['FIELDGUARD_RELEASES_URL'] ??
  'https://api.github.com/repos/LBSINTER/LBS-FieldGuardPublic/releases/latest';
```

To use a private or self-hosted endpoint, replace the URL or set the
`FIELDGUARD_RELEASES_URL` environment variable before building. The endpoint must
return JSON with at minimum:

```json
{
  "tag_name": "v1.0.3",
  "html_url": "https://github.com/LBSINTER/LBS-FieldGuardPublic/releases/tag/v1.0.3"
}
```


---


## Running tests

```bash
npm test
```

Test coverage:

- PDU encode/decode round-trip (GSM-7, binary, Type-0, STK)
- Signature DB loading and byte-pattern matching
- SS7 payload catalogue classification


---


## Releasing a new version

1. Bump `versionCode` (increment by 1) and `versionName` in
   `android/app/build.gradle`.
2. Update `APP_VERSION` in `src/config/build.ts` to match the new `versionName`.
3. Update `README.md` if anything significant changed.
4. Commit, tag, and push:

```bash
git add -A
git commit -m "chore: bump to vX.Y.Z"
git tag vX.Y.Z
git push origin main --tags
```

The GitHub Actions workflow detects the `v*` tag, builds the release APK, and
publishes it to `LBSINTER/LBS-FieldGuardPublic`. Devices running older versions
will see the update banner on their next launch once the release is live.

Before the first release from a new repository setup, add the
`FIELDGUARD_PUBLIC_PAT` secret (a GitHub PAT with `repo` scope on
`LBS-FieldGuardPublic`) to this repository's Settings > Secrets > Actions.
