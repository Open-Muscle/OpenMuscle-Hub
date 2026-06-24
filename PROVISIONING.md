# OpenMuscle Wi-Fi Provisioning v1.0

**Status:** DRAFT for sign-off by phone + firmware teams. Sign-off via the `.claude-mail/openmuscle` coordination board.
**Companion to:** [PROTOCOL.md](./PROTOCOL.md). This document covers how a factory-fresh source gets onto the user's Wi-Fi; PROTOCOL.md covers everything that happens after.
**Owner:** firmware team (device side) + phone team (app side).
**Reference implementation:** to land in FlexGridV4-Firmware (device) and OpenMuscle-Connect (phone).

The unified plan calls manual `settings.json` editing "not the normal path." This spec is the normal path: a user pulls a device out of the box, opens the Connect app on their phone, and the device joins their Wi-Fi without anyone touching a config file.

## 1. Goals and non-goals

**Goals**
- A user with the Connect app and a factory-fresh source completes Wi-Fi setup in under 60 seconds, with one form (SSID + password) on the phone.
- Works without the Connect app too: any phone or laptop can complete the handshake by connecting to the device's AP and visiting `http://192.168.4.1/` (captive-portal-friendly).
- One device, many phones: any phone can re-provision a device that was previously provisioned, with no per-phone state on the device.

**Non-goals (deferred to v1.1+)**
- Authenticated provisioning (pre-shared QR token). v1.0 trusts the LAN.
- BLE-based provisioning. Wi-Fi AP only in v1.0.
- Multi-device batch provisioning. One device at a time.
- TLS on the provisioning channel. Plaintext over the device's own AP; the device only stays in AP mode briefly and the user is physically present.

## 2. State machine

A source has three Wi-Fi states:

| State | Trigger | Behavior |
|---|---|---|
| `unprovisioned` | `config/settings.json` has no `wifi_ssid` (or it is the empty string) at boot. | Start AP mode (section 3). Run HTTP server (section 4). Wait for `POST /provision`. |
| `provisioned` | `wifi_ssid` is set. | Start STA mode. Run normal discovery + cmd + data per PROTOCOL.md. |
| `reprovision_requested` | User explicitly cleared creds (BOOT-hold button + menu, OR `POST /reprovision`). | Clear `wifi_ssid` and `wifi_password`, persist, soft reset. On next boot, enter `unprovisioned`. |

A device MUST NOT silently retry STA forever and starve provisioning. If STA fails to join within `sta_join_timeout_s` (default 20 s) AND the device has never previously joined this SSID successfully, it MUST fall back to `unprovisioned` so the user can re-enter creds.

If STA fails AFTER a previous successful join (the user's router is rebooting, the password changed, etc.), the device keeps retrying STA in the background indefinitely. It does not fall back to AP because the user's existing setup is the source of truth; falling back to AP would break running hubs.

## 3. AP mode

When a source is in `unprovisioned`, it starts a Wi-Fi access point with these parameters:

| Parameter | Value |
|---|---|
| SSID | `OM-<dev>-<id-tail>`, e.g. `OM-flexgrid-d7af0b`, `OM-lask5-01`. The `<dev>` is the same string the source uses for the announce `dev` field; the `<id-tail>` is the trailing portion of the device id after the type prefix. |
| Password | none (open AP). |
| Channel | 1 by default; the firmware MAY pick a less-congested channel based on a scan. |
| Device IP | `192.168.4.1` (ESP32 default). |
| DHCP server | on. Phone receives `192.168.4.x`. |
| Max stations | 4. |
| Beacon hidden | no. The SSID is broadcast so phones see it in their normal Wi-Fi picker. |

**Why open AP.** The provisioning channel is plaintext to a single endpoint on a tiny LAN that exists for at most a few minutes during onboarding, with the user physically present beside the device. Adding a shared password adds friction without meaningfully changing the threat model (anyone in radio range can still capture the handshake during the brief window). A pre-shared QR token to authenticate the phone is the v1.1 hardening path.

**LED + OLED indication while in AP mode.**
- The status LED blinks blue at 1 Hz so the user knows the device is in provisioning mode.
- The OLED (where present) shows: device id on line 1, `WIFI SETUP` on line 2, the AP SSID on line 3, `192.168.4.1` on line 4. This is the printable cheat sheet for a user who is not using the app.

## 4. Provisioning HTTP server

While in `unprovisioned`, the source runs a minimal HTTP/1.1 server on TCP port 80 bound to the AP-side interface. It serves four endpoints. All bodies are JSON unless noted.

### 4.1 `GET /info`

Read-only. Returns the same shape as the cmd-channel `get_info` ack, plus a `state` field.

```json
{
    "v":     "1.0",
    "type":  "info",
    "id":    "flexgrid-d7af0b",
    "dev":   "flexgrid",
    "fw":    "v4.0.0",
    "caps":  ["sensor", "status", "cmd", "imu"],
    "matrix": [15, 4],
    "state": "unprovisioned"
}
```

The phone calls this first so the user can confirm "yes, this is the band I just unboxed" before sending credentials.

### 4.2 `POST /provision`

Body:

```json
{
    "ssid":     "MyHomeWiFi",
    "password": "correct-horse-battery-staple",
    "hub_host": "10.0.0.102",
    "hub_port": 3141
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `ssid` | string | yes | 1..32 bytes, UTF-8. |
| `password` | string | yes | 8..63 bytes for WPA2; empty string allowed for open networks. |
| `hub_host` | string | no | If present, the device persists this as a default subscriber to attempt after joining STA. Otherwise the device relies on normal hub discovery. Useful when the phone wants the device to immediately stream to a paired PC. |
| `hub_port` | int | no | Pairs with `hub_host`. Default 3141. |

Response on success:

```json
{
    "v":      "1.0",
    "type":   "provision_ack",
    "status": "ok",
    "id":     "flexgrid-d7af0b",
    "next":   "switching_to_sta",
    "reboot_in_ms": 1500
}
```

Response on error:

```json
{
    "v":      "1.0",
    "type":   "provision_ack",
    "status": "error",
    "data":   {"message": "ssid_too_long"}
}
```

After sending the success response, the device MUST:
1. Persist `ssid` + `password` (and optional `hub_host` + `hub_port`) to `config/settings.json`.
2. Wait `reboot_in_ms` (default 1500 ms) so the HTTP response actually drains over TCP before the radio drops.
3. Soft reset (`machine.soft_reset()`). On reboot the device will be `provisioned` and start STA.

The device MUST NOT attempt to verify the credentials before persisting. If the user typed the password wrong, STA will fail to join and the device falls back to `unprovisioned` per section 2; the phone retries.

### 4.3 `POST /reprovision`

No body required. Clears `wifi_ssid` and `wifi_password` from `settings.json`, then soft resets. Useful when a previously-provisioned device is moved to a new network and the user wants to re-enter credentials. Phones reach this endpoint over the STA-mode normal LAN; it does not require AP mode.

Response:

```json
{
    "v":      "1.0",
    "type":   "reprovision_ack",
    "status": "ok",
    "reboot_in_ms": 1500
}
```

### 4.4 `GET /` (captive-portal fallback)

Returns a self-contained HTML page (no external assets, no JavaScript dependencies beyond a single inline `<script>`) with an SSID + password form. The form POSTs to `/provision`. This is the path for users without the Connect app.

The HTML SHOULD include:
- The device id and `dev` from `/info` so the user can confirm they are on the right device.
- A list of nearby SSIDs the device scanned (`GET /scan` results, see 4.5).
- A submit button that calls `/provision` via `fetch()`.
- Plain CSS only; the page renders correctly on any phone browser.

### 4.5 `GET /scan` (optional)

Returns the list of SSIDs the device sees, sorted by signal strength. Hubs and the captive-portal HTML can use this to pre-populate the SSID picker so the user does not have to type their SSID.

```json
{
    "v":     "1.0",
    "type":  "scan",
    "data":  {
        "networks": [
            {"ssid": "MyHomeWiFi",   "rssi": -42, "auth": "wpa2"},
            {"ssid": "guest-2.4",    "rssi": -67, "auth": "wpa2"},
            {"ssid": "neighbor",     "rssi": -82, "auth": "wpa2"}
        ]
    }
}
```

Scan is best-effort; the device MAY take up to ~3 seconds to complete a scan and SHOULD cache results for ~30 seconds to avoid back-to-back scans on a page refresh.

## 5. DNS captive portal

To make the captive-portal flow seamless on iOS and Android, an `unprovisioned` source SHOULD run a tiny DNS server on UDP 53 that answers every query with `192.168.4.1`. The phone's captive-portal detection probes a well-known URL (`captive.apple.com`, `connectivitycheck.gstatic.com`, etc.) and gets the device's HTML, triggering the OS's "Sign in to network" prompt.

This is OPTIONAL because some MicroPython builds lack a usable UDP socket-server primitive at the required level. Devices that omit the captive-portal DNS MUST still serve `GET /` correctly so the user can complete provisioning by typing `192.168.4.1` into a browser.

## 6. Phone-side handshake

The Connect app's onboarding flow:

1. **Scan for unprovisioned devices.** Trigger a Wi-Fi scan and filter SSIDs by the `OM-*` prefix. Present the matches to the user with their `dev` type inferred from the SSID pattern.
2. **User picks a device.** The app connects to the picked AP via `WifiNetworkSpecifier` through `ConnectivityManager.requestNetwork()` (Android 10+, API 29+). This requests an app-scoped local-only network: the OS shows a one-time approval dialog, the app gets a `Network` handle, and the phone KEEPS home Wi-Fi or cellular as the default route (background data is not lost). HTTP calls in this flow are bound to the device network via `Network.openConnection` or `bindProcessToNetwork`. On unregister after provisioning, the OS auto-reverts; the phone never left home Wi-Fi as default. **Do not use `WifiNetworkSuggestion`** for this step: Suggestion is a hint for the system to maybe auto-join later, has no deterministic connect-now / talk / disconnect semantics, and Android deprioritizes no-internet suggested networks.
3. **Verify identity.** Once connected to the AP, GET `http://192.168.4.1/info` and surface the returned `id`, `dev`, `fw` to the user. The user taps "Yes, this is my device." The app SHOULD cross-check the SSID's id-tail against the returned `/info` `id` and warn on mismatch; a spoofed `OM-*` AP would fail this check.
4. **Gather credentials.** If the app has `ACCESS_FINE_LOCATION` granted AND location services are on (Android API 27+ requires both to read the saved SSID; API 33+ tightens further), pre-fill the SSID field with the phone's currently-saved home Wi-Fi name. Otherwise fall back to manual entry or the `GET /scan` picker (step 5). Always ask for the password.
5. **Optional scan.** GET `/scan` to surface nearby SSIDs as a picker if the user is not on the home Wi-Fi at the time of provisioning, or if step 4's auto-fill was unavailable.
6. **Post credentials.** POST `/provision` with `ssid` + `password` only. The phone SHOULD OMIT `hub_host` / `hub_port`: during the AP session the phone holds a `192.168.4.x` AP address and does not reliably know its future home-LAN IP, so passing a hub address at this point would be wrong. Discovery on the home LAN (step 8) provides the correct address. `hub_host` / `hub_port` are intended for known-static PC targets, not the phone.
7. **Wait for handoff.** After the 200 OK, the device reboots and leaves AP mode. Because the phone connected via `WifiNetworkSpecifier`, the default Wi-Fi was never replaced; the phone simply unregisters the device network and continues on home Wi-Fi. No manual reconnect step.
8. **Confirm join.** The phone listens on UDP 3140 for the device's announce. Within ~10 seconds of the reboot the announce should arrive. If it does not within `provisioning_timeout_s` (default 60 s), the phone surfaces a failure ("device did not join your Wi-Fi; try again") and re-enters step 2.

If the same user pulls multiple devices out of the box in sequence, the app should remember the user's SSID + password for the session and skip step 4 on subsequent devices.

## 7. Conformance

A v1.0-compliant **source** MUST:

1. Boot to `unprovisioned` if `wifi_ssid` is missing or empty.
2. Broadcast its AP SSID as `OM-<dev>-<id-tail>`, open auth, on `192.168.4.1`.
3. Serve `GET /info` and `POST /provision` per section 4. Other endpoints are SHOULD.
4. Persist creds and soft-reset on successful `POST /provision`.
5. Fall back to `unprovisioned` if STA fails to join on a never-before-joined SSID within `sta_join_timeout_s`.
6. Persist all credentials to a file path that is in `.gitignore` (do not bake creds into source-tree defaults).
7. Indicate provisioning state on the status LED (blue blink while in AP, solid green once joined).

A v1.0-compliant **phone hub** MUST:

1. Detect `OM-*` SSIDs as unprovisioned devices.
2. Walk the user through the handshake in section 6.
3. Provide a "reprovision this device" action that calls `POST /reprovision` over the user's normal LAN once the device is back on STA.

## 8. Open for v1.1

- **QR pre-shared token.** Each device prints a one-time pairing token on its back (or shows it on the OLED). The phone reads it via QR and includes it as an `Authorization` header. The device rejects `POST /provision` calls without a valid token. Closes the "anyone within radio range" gap during the AP window.
- **BLE provisioning.** Alternative to AP for devices that have BLE peripherals. The Connect app discovers via BLE scan, writes credentials to a GATT characteristic. Useful for devices without enough flash for an HTTP server, or in dense Wi-Fi environments where AP creation is unreliable.
- **Multi-device batch.** Phone provisions a queue of devices in sequence with the same SSID + password, surfacing progress in a single UI.
- **TLS.** Self-signed TLS on the provisioning channel once MicroPython's ssl module gets server-mode certificate support that fits in ESP32-S3 RAM.

## 9. Change log

- **2026-06-23 v1.0 DRAFT.** Initial draft. Open AP, plaintext HTTP on 192.168.4.1, `POST /provision` handshake. Captive-portal DNS optional. Phone-side flow specified in section 6. Conformance MUSTs for source + phone in section 7. Awaiting sign-off from phone team before freeze.
- **2026-06-24 v1.0 (sign-off corrections, doc-only).** Phone signed off (board #0042) with three required corrections, all in section 6: (step 2) the Android connect-to-AP API is `WifiNetworkSpecifier` via `ConnectivityManager.requestNetwork`, NOT `WifiNetworkSuggestion`; the latter has no deterministic connect-now semantics for this flow. (step 4) SSID auto-fill is gated on `ACCESS_FINE_LOCATION` + location services on; the spec now describes the fallback to manual entry or `GET /scan`. (step 6) the phone OMITS `hub_host` / `hub_port` because it does not know its future home-LAN IP during the AP session; post-join discovery on UDP 3140 provides the address. The open-AP threat model in section 3 is still pending Tory's ratify before device-side implementation lands.
