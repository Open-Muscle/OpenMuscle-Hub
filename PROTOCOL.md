# OpenMuscle Packet Protocol v1.0

**Status:** FROZEN as of 2026-06-23. Implementations code against this document.
**Canonical home:** this file in OpenMuscle-Hub.
**Reference implementation:** [FlexGridV4-Firmware](https://github.com/Open-Muscle/FlexGridV4-Firmware) (see `lib/discovery.py`, `lib/commands.py`, `lib/subscribers.py`, `lib/network_manager.py`).
**Owner:** firmware team (changes go through the `.claude-mail/openmuscle` board).

This spec covers how OpenMuscle sources (sensor and labeler devices) announce themselves on a LAN, how hubs (Android phone, PC web app, native VR app) subscribe to them, and how the resulting sensor and label streams are framed. The wire format is JSON-encoded UTF-8 over UDP and TCP. There is no binary mode and no WebSocket framing in v1.0.

## 1. Terms

| Term | Meaning |
|---|---|
| **Source** | A device that emits sensor or label data (FlexGrid V3/V4, LASK5, future OpenHand, quest_hand). |
| **Hub** | A consumer that subscribes to one or more sources (Android Connect app, PC `openmuscle web`, native Quest app). |
| **Role** | A semantic tag a hub assigns to a subscribed source: `left`, `right`, `labeler`. Sources are role-agnostic; hubs decide. |
| **Frame** | One JSON message on the wire (announce, ack, sensor, label, status). |
| **Envelope** | The outer JSON object that wraps every frame: `{v, type, id, ts, ...}`. |

## 2. Network ports (the frozen contract)

| Purpose | Transport | Port | Direction | Notes |
|---|---|---|---|---|
| Discovery announce | UDP broadcast | **3140** | source -> 255.255.255.255 | New in v1.0. Was 3141 in pre-spec firmware; split out so data and discovery cannot interfere. |
| Data frames (sensor, label, status) | UDP unicast | **3141** | source -> subscribed hub | Hub picks its own listen port and tells the source in the `subscribe` verb; 3141 is the convention and the default. |
| Command channel | TCP | **8001** (FlexGrid), **8002** (LASK5) | hub -> source | Per-device-type; advertised in announce `services.cmd`. |
| mDNS (best-effort) | UDP multicast | 5353 | both | Optional. `_openmuscle._udp.local` service. Hubs that use mDNS find sources by hostname `<device-id>.local`. |

**Why two UDP ports.** Hubs that listen on the data port get a clean stream of only the sources they subscribed to. Putting announces on the same port forces every hub to type-filter every packet and creates back-pressure on busy capture sessions. The split is the v1.0 design; type-filtering on a shared port is a fallback only.

## 3. Envelope

Every frame is a JSON object with these required fields:

```json
{
    "v":    "1.0",
    "type": "<frame-type>",
    "id":   "<source-or-hub-id>",
    "ts":   164587,
    "data": { ... },
    "meta": { ... }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `v` | string | yes | Protocol version. `"1.0"` for this spec. |
| `type` | string | yes | Frame discriminator. For data frames, the source's device type (`"flexgrid"`, `"lask5"`, `"quest_hand"`). For control frames, the control type (`"announce"`, `"ack"`, `"cmd"`). |
| `id` | string | yes | Stable identifier of the emitting device or hub (e.g. `"flexgrid-d7af0b"`, `"pc-native-discovery"`). |
| `ts` | int | yes | Sender-local timestamp in milliseconds. Not wall-clock; only ordering and rate measurement are guaranteed. Hubs stamp arrival time on receipt. |
| `data` | object | yes | Type-specific payload. May be `{}` for status-only frames. |
| `meta` | object | no | Auxiliary metadata: battery, RSSI, calibration state, subscriber list, firmware version. |
| `seq` | int | sensor frames | Rolling 16-bit (0..65535) sequence number per (source, stream). Hubs use it to detect drops; wrap is normal. |
| `msg_id` | int | cmd + ack | Correlation ID set by the hub on each command, echoed in the ack. |
| `status` | string | ack only | `"ok"` or `"error"`. |

## 4. Versioning policy

The `v` field is the version of the envelope and the wire contract. Hubs and sources both inspect `v` on every received frame.

- **Non-breaking changes** stay on `"v": "1.0"`. Adding a new device `type`, a new optional `meta` field, a new command verb, or a new optional field inside an existing `data` payload is non-breaking. Old hubs see it and ignore it.
- **Breaking changes** bump `v` to `"1.1"` or higher. Changing the set of required envelope fields, changing the semantics of `ts` or `id`, changing existing frame shapes, or removing fields is breaking.
- **Graceful degradation rule.** A receiver MUST NOT crash on an unrecognized `type`, an unrecognized verb, or an unrecognized optional field. It MAY warn and discard. A receiver MUST reject a frame whose `v` is incompatible (major bump) and SHOULD log the version mismatch with the peer address so the operator can see what is out of date.

There is no automatic downgrade. If two ends disagree on major version, the operator updates one of them.

## 5. Discovery

### 5.1 UDP broadcast beacon (the reliable path)

Each source broadcasts an announce frame to `255.255.255.255:3140` once per `announce_interval_s` (default 1 second).

Hubs bind a UDP socket to `0.0.0.0:3140` and receive announces from every source on the same subnet. The source's IP comes from the packet's source address; it is never inside the payload.

### 5.2 Announce payload

```json
{
    "v":          "1.0",
    "type":       "announce",
    "id":         "flexgrid-d7af0b",
    "role":       "source",
    "dev":        "flexgrid",
    "fw":         "v4.0.0",
    "transports": ["wifi"],
    "caps":       ["sensor", "status", "cmd", "imu"],
    "matrix":     [15, 4],
    "services":   {"cmd": 8001},
    "ts":         164587,
    "boot_seq":   7
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Stable device id. |
| `role` | string | yes | Always `"source"` for v1.0. Reserved for future roles like `"gateway"`. |
| `dev` | string | yes | Device type (`"flexgrid"`, `"lask5"`, `"openhand"`, etc.). Used by hubs to pick the right data parser. |
| `fw` | string | yes | Firmware version, e.g. `"v4.0.0"`. |
| `transports` | string[] | yes | Available transports. `["wifi"]` in v1.0. BLE is reserved for v1.1+. |
| `caps` | string[] | yes | Capability tags: `sensor`, `status`, `cmd`, `imu`, `label`, `haptic`, `battery`. Hubs key UI off this. |
| `matrix` | int[2] | grid sources only | `[cols, rows]` for grid sources. Absent for non-grid sources like LASK5. |
| `services` | object | yes | Map of capability -> port. v1.0 has exactly one entry: `cmd` -> the TCP cmd port (8001 for FlexGrid, 8002 for LASK5). |
| `boot_seq` | int | yes (added) | Monotonic restart counter, incremented once per boot and persisted on the device. Lets a hub detect a fresh boot vs same session and flush cached subscription state. See 5.5. Sources predating this field MAY omit it; hubs MUST treat absent as `0`. |

The data port and announce port are protocol-fixed (3141 and 3140); only the per-device-type cmd port is advertised.

### 5.3 Beacon suppress when subscribed (REQUIRED hub address cache)

To keep the broadcast channel quiet during capture, a source MUST stop broadcasting announces while it has at least one subscriber, and MUST resume within one `announce_interval_s` of the subscriber list becoming empty.

This means a late-joining hub will NOT see an announce while another hub is already subscribed. To stay reachable in that window, every hub MUST keep a persistent address cache of devices it has seen and SHOULD re-probe known devices via `get_info` on startup. PC `openmuscle web` and Android Connect both implement this; new hubs must do the same.

The cache key is the device `id`. The cached entry holds at minimum: last known IP, cmd port, last contact timestamp.

### 5.4 mDNS (optional best-effort)

A source MAY register itself as `<device-id>.local` and SHOULD advertise the `_openmuscle._udp` service. MicroPython builds without an mDNS C module no-op gracefully; the broadcast beacon is the reliable path.

Hubs MAY use mDNS to resolve a known device that has gone silent, as an alternative to the address cache. mDNS is never the only discovery path; the spec requires the cache.

### 5.5 Active discovery probe (hub -> source over UDP)

In addition to the periodic broadcast beacon (5.1), a source MUST listen on its announce port and respond to an active discovery probe with an immediate announce. This lets a hub that just joined a new Wi-Fi network pull a fresh announce in under 100 ms instead of waiting up to one `announce_interval_s` for the next periodic beacon, and also covers the case where a known device went quiet because someone else has been subscribed for a while (5.3).

Probe payload (hub -> 255.255.255.255:3140):

```json
{
    "v":    "1.0",
    "type": "cmd",
    "data": { "verb": "discover" }
}
```

A source receiving a well-formed `discover` MUST respond with the same announce payload it would have sent on the next periodic beacon (5.2). It SHOULD send the response twice:

1. Broadcast to `255.255.255.255:3140` so any hub bound to the announce port on the same link picks it up alongside the probing hub.
2. Unicast to the prober's `(host, port)` so a hub that probed from an ephemeral port (and is therefore not bound to 3140) still receives the reply.

Sources MUST NOT respond to malformed packets, packets on the announce port that are not `type:"cmd"` with `verb:"discover"`, or their own broadcast announces echoed back.

Hubs SHOULD send a `discover` probe on each of:
- App foreground / "pull-to-refresh".
- Wi-Fi network change event (Android `NetworkCallback`, iOS `NWPathMonitor`).
- A user-driven "find devices" action.

`discover` is the only verb defined on the announce-port UDP channel. All other verbs are on the TCP cmd channel (section 6.3).

### 5.6 boot_seq and stale-subscription detection

The `boot_seq` field in 5.2 is a monotonic per-device counter, incremented and persisted to flash exactly once per boot. Hubs MUST track the most recently observed `boot_seq` per device-`id` and treat a strictly-increasing jump as a signal that the device rebooted and any subscription state cached on the hub for that device is stale. On a jump, a hub MUST:

1. Treat any cached subscription for that device as gone.
2. Re-subscribe via the cmd channel (6.3) before resuming data assumptions.

Hubs MUST NOT use `boot_seq` to gate which devices to display or which announces to accept; it is solely a stale-state invalidator.

A device that pre-dates this field MAY omit `boot_seq` from its announce; hubs MUST treat absent as `0`. A device that resets its persistence (factory reset, manual clear) MAY restart its `boot_seq` from `0`; the resulting jump-down does NOT invalidate hub state (only strict increases do). Wrap is not specified for v1.0; counters are 64-bit-safe on the wire as JSON integers and 32-bit-safe on-device.

## 6. Command channel

The cmd channel is TCP, newline-delimited JSON, one command per line, one ack per line. WebSocket framing is not used in v1.0; native hubs open a raw TCP socket. Browser hubs that need this in the future will go through a server-side bridge.

### 6.1 Command envelope (hub -> source)

```json
{
    "v":      "1.0",
    "type":   "cmd",
    "id":     "pc-native-discovery",
    "msg_id": 42,
    "data": {
        "verb":      "subscribe",
        "host":      "10.0.0.102",
        "port":      3141,
        "transport": "wifi",
        "hub_id":    "pc-native-discovery"
    }
}
```

The top-level `id` is the hub's identity (same hub MAY omit it; the reference firmware accepts cmd frames with or without). The inner `data.hub_id` is the same identity, kept inside `data` for compatibility with the subscribe verb's payload shape; sources match them when both are present.

### 6.2 Ack envelope (source -> hub)

```json
{
    "v":      "1.0",
    "type":   "ack",
    "status": "ok",
    "msg_id": 42,
    "data": {
        "verb":             "subscribe",
        "accepted":         true,
        "subscriber_count": 2,
        "max_subscribers":  4
    }
}
```

On error, `status` is `"error"` and `data.message` carries a human-readable reason.

### 6.3 Verbs

#### subscribe

Register the hub as a Wi-Fi (or future BLE) recipient of this source's data frames.

| Field | Type | Required | Notes |
|---|---|---|---|
| `verb` | string | yes | `"subscribe"`. |
| `host` | string | **optional** | The hub's data-receive IP. If omitted, the source uses the TCP peer address. Phone clients SHOULD omit; PC clients SHOULD send explicit `host` so they continue to work behind NAT and on multi-homed machines. |
| `port` | int | yes | The hub's data-receive UDP port. Convention is 3141. |
| `transport` | string | no | `"wifi"` (default) or `"ble"` (reserved). |
| `hub_id` | string | no | Free-form hub identity for diagnostics. Logged on the source. |

Ack `data`: `{verb, accepted, subscriber_count, max_subscribers}`. `accepted=false` means the subscriber list is full.

Re-subscribing the same `(host, port, transport)` triple is idempotent and refreshes the heartbeat timestamp; it does not consume a new slot.

#### unsubscribe

Remove the hub from the subscriber list. Same `host` / `port` / `transport` fields as `subscribe`. Ack `data`: `{verb, removed, subscriber_count}`.

#### heartbeat

Refresh the hub's last-seen timestamp. Same `host` / `port` / `transport` fields. Sent over the SAME TCP cmd channel, not UDP, so a half-closed TCP is visible immediately. Cadence is 1 Hz. Ack `data`: `{verb, refreshed}`.

A source MUST drop a subscriber whose last heartbeat is older than `heartbeat_timeout_s` (default 5 seconds, source-configurable). On drop, if no subscribers remain, the source resumes the announce beacon within one `announce_interval_s`.

#### get_info

Read-only: source replies with capabilities and current state. No effect on subscription.

Ack `data`:
```json
{
    "verb":        "get_info",
    "id":          "flexgrid-d7af0b",
    "dev":         "flexgrid",
    "fw":          "v4.0.0",
    "matrix":      [15, 4],
    "caps":        ["sensor", "status", "cmd", "imu"],
    "subscribers": [
        {"host": "10.0.0.102", "port": 3141, "transport": "wifi",
         "hub_id": "pc-native-discovery", "age_ms": 312}
    ],
    "role":        "right",
    "power_mode":  "stream",
    "imu":         {"chip": "icm42688-tokmas",
                    "gyro_dps_per_lsb":  0.0610,
                    "accel_g_per_lsb":   0.000488}
}
```

Optional reply fields (sources that implement them include; pre-this-spec firmware omits):

| Field | Type | Description |
|---|---|---|
| `role` | string | Hub-pushed spatial role: `"left"` / `"right"` / `"labeler"`. Absent when the device has no role assigned (= the phone should prompt the user to tag). See `set_role` below. |
| `power_mode` | string | Current device-side state-machine mode: `"stream"` / `"standby"` / `"charge"`. Lets the phone show the same mode in its UI. See `set_role` below for the cmd that complements `power_mode` cycling on the device. |
| `imu` | object | Per-IMU-chip scale dict: `{chip, gyro_dps_per_lsb, accel_g_per_lsb}`. Same shape as the announce `imu` field; lets a hub that discovers via cache or mDNS skip the announce step. |

`get_info` is the canonical way for a hub to re-probe a known device whose beacon is silent (because another hub is subscribed).

#### set_role

Push a spatial-role tag to the source. The role is persisted on the device and drives both the idle-LED hue (BLUE left / RED right / GREEN labeler / WHITE unassigned) and the hub's ability to assign Left || Right slots to bilateral captures without re-asking the user every session.

Cmd `data`:
| Field | Type | Required | Notes |
|---|---|---|---|
| `verb` | string | yes | `"set_role"` |
| `role` | string | yes | One of `"left"` / `"right"` / `"labeler"` / `""` (clear). Unknown values are clamped to `""` on the source. |

Ack `data`: `{verb, role}` echoing the resolved role (after clamping).

Sources that pre-date this verb reject it with `status="error"` (unknown verb). Hubs MUST treat that as "device cannot persist role; assign role hub-side only" and continue without further set_role calls for that session.

#### set_scan_rate

Change the sensor scan interval. `data.interval_ms` is an int in the range 5..2000. Out-of-range values reply `status="error"`. Ack `data`: `{verb, interval_ms}`.

#### start_stream, stop_stream

Pause and resume sensor frame emission without losing the subscriber list. Useful for trainer apps that want to gate streaming. Ack `data`: `{verb, streaming: true|false}`.

#### reboot

Schedule a soft reset of the source. Ack is sent before the reboot fires. Ack `data`: `{verb, rebooting: true}`.

### 6.4 Subscriber cap

A source enforces a per-device subscriber cap (default 4). This bounds UDP fan-out cost. The cap MUST cover at least three simultaneous hubs (phone + PC + VR); 4 leaves one spare slot for a second phone or a logging tool. Future hardware revisions may raise it; the announce `meta` (if present) can carry the current cap.

## 7. Data frames

Unicast UDP to each subscriber's `(host, port)`. One JSON object per UDP packet. Frames carry a `seq` field; hubs detect drops by gap.

### 7.1 FlexGrid (`type: "flexgrid"`)

```json
{
    "v":    "1.0",
    "type": "flexgrid",
    "id":   "flexgrid-d7af0b",
    "ts":   164587,
    "seq":  62344,
    "data": {
        "matrix": [
            [col0_row0, col0_row1, col0_row2, col0_row3],
            [col1_row0, col1_row1, col1_row2, col1_row3],
            ...
        ]
    },
    "meta": {"vbat": 3.94, "pct": 78, "rssi": -54}
}
```

- `data.matrix` is **column-major on the wire**: outer length is the column count, each inner array length is the row count. V3 is 16x4; V4 is 15x4 (15 cols x 4 rows). Hubs MUST consult the announce `matrix: [cols, rows]` for the source's actual dimensions rather than assume.
- The wire layout in this section is separate from the **CSV feature flatten** in section 8.3, which is row-major. Devices emit column-major matrices; hubs that flatten for capture or training do so row-major. See section 8.3.
- ADC values are 12-bit unsigned (0..4095).
- `data.rows` and `data.cols` MAY be included for legacy parsers; they MUST agree with the matrix shape if present.
- `data.imu` is **optional**: when the source has an IMU and chooses to ride it on the sensor channel, the field carries the latest sample at whatever rate the sensor loop fires. Shape: `{"accel": [ax, ay, az], "gyro": [gx, gy, gz]}` (signed 16-bit raw counts, units per the device's IMU variant). Hubs that want smooth orientation visualisation SHOULD parse this; the slower `meta.imu` path (section 7.4) stays as the back-compat snapshot and as the carrier of variant + temperature. Non-breaking; consumers that ignore `data.imu` are unaffected.

### 7.2 LASK5 (`type: "lask5"`)

```json
{
    "v":    "1.0",
    "type": "lask5",
    "id":   "lask5-01",
    "ts":   164587,
    "seq":  4012,
    "data": {
        "values":   [0.42, 0.18, 0.91, 0.0],
        "joystick": {"x": 2048, "y": 2048}
    }
}
```

- `data.values` is a flat array of calibrated finger-target floats in `[0.0, 1.0]`. The reference firmware emits 4 values; spec does not cap this at 4.
- `data.joystick` is optional; absent on hardware without a joystick.

### 7.3 Quest hand tracking (`type: "quest_hand"`)

Synthesized server-side by the PC app from WebXR frames (browsers can't speak UDP). Carried for completeness because hubs trained on FlexGrid+LASK5 must also accept Quest sources without surprise.

```json
{
    "data": {
        "values":      [px, py, pz, rx, ry, rz, rw, ...],
        "handedness":  "left" | "right",
        "joint_names": ["wrist", "thumb-metacarpal", ...],
        "hands": {
            "handedness": "left",
            "joints": [
                {"name": "wrist", "pos": [x, y, z], "rot": [x, y, z, w], "radius": 0.02, "valid": true}
            ]
        }
    }
}
```

- `values` follows the LASK5 convention: flat, canonical joint x channel order, 7 floats per joint `[px, py, pz, rx, ry, rz, rw]`. The label width is locked at the first label packet; later packets are padded or truncated to keep CSV rectangular.
- Tracking-lost joints MUST be emitted as zero position + identity quaternion with `valid: false`. Omitting a joint shifts every later joint into the wrong CSV column and silently misaligns labels.
- Empty payloads (full-hand tracking-lost) are dropped at the receiver.

A `<capture>.labels.schema.json` sidecar is written on the first `quest_hand` frame of a recording, mapping `label_0..label_N` back to `(joint, channel)`.

### 7.4 Status frames

```json
{
    "v":    "1.0",
    "type": "<device-type>",
    "id":   "<device-id>",
    "ts":   164587,
    "data": {},
    "meta": {
        "vbat":              3.94,
        "pct":               78,
        "rssi":              -54,
        "free_mem":          145000,
        "uptime_s":          1234,
        "subscribers":       2,
        "imu":               {"variant": "tokmas", "accel": [...], "gyro": [...]},
        "reset_cause":       3,
        "reset_cause_name":  "soft"
    }
}
```

Status frames are unicast on the same UDP data port as sensor frames, at ~1 Hz. Empty `data` plus populated `meta` is the discriminator: sensor pipelines route on `data` shape, status pipelines route on `meta`.

Status `meta` field semantics:
- `vbat` is battery voltage as a float in volts (NOT millivolts; this is what the reference firmware emits).
- `pct` is battery state-of-charge in percent (int, 0..100).
- `rssi` is Wi-Fi signal strength in dBm.
- `free_mem` is MicroPython heap free in bytes.
- `uptime_s` is seconds since boot.
- `subscribers` is the current subscriber count.
- `imu` carries the IMU snapshot when the device has one; `variant` distinguishes genuine InvenSense from the TOKMAS rebrand for downstream consumers that need to interpret raw counts.
- `reset_cause` is the MicroPython machine.reset_cause() integer; `reset_cause_name` is the human-readable form (`"pwr"`, `"hard"`, `"soft"`, `"wdt"`, `"deep"`). Hubs SHOULD log this on reconnect so a silent watchdog reset is visible.

These field names are the canonical contract; hubs key UI off them. Fields not listed here are reserved.

## 8. Multi-device data model

OpenMuscle is designed for multi-source capture (two FlexGrid bands plus a labeler, plus optionally a Quest source) from day one.

### 8.1 Sources are role-agnostic; hubs assign role

A source does not know whether it is on the user's left or right arm. The HUB assigns a role to each subscribed source after subscription. The role is hub-local state, not on-wire.

Defined roles in v1.0: `left`, `right`, `labeler`. Reserved for v1.1: `reference`, `auxiliary`.

A hub MAY persist the role assignment per device `id` so the user does not re-tag on every session.

### 8.2 Multi-source semantics

A hub maintains a set of subscribed sources, each independently:
- Each source heartbeats independently on its own TCP cmd channel.
- Each source streams independently on its own UDP unicast.
- The hub aligns frames across sources by their arrival timestamp (the hub's wall clock), not by the source-local `ts`. Source-local `ts` clocks are not synchronized.

When a source drops (heartbeat timeout or explicit unsubscribe), the hub's data pipeline MUST handle missing-source frames gracefully (zero-fill or skip per the application's choice) rather than blocking on the missing source.

### 8.3 CSV schema v2 (capture, on-disk LONG format)

When a hub writes capture CSVs, the on-disk format is **LONG**: one row per source frame, role-tagged. Sources stream unsynchronized (8.2), so the hub cannot put Left and Right onto a single row without resampling; one row per source preserves the raw timing.

Header:

```
ts_hub_ms, role, device_id, R0C0, R0C1, ..., R{rows-1}C{cols-1}, label_0, ..., label_M
```

- `ts_hub_ms` is the hub's arrival time in **epoch milliseconds** (UTC). Epoch is used so timestamps remain comparable across runs; monotonic-since-boot is not.
- `role` is the hub-assigned role (`left`, `right`, `labeler`).
- `device_id` is the source `id` (preserves traceability when capturing multiple devices of the same role).
- Feature columns are named `R{r}C{c}` where `r` is the row index and `c` is the column index. For a 15x4 V4 band this gives 60 named feature cells: `R0C0, R0C1, ..., R3C14`.
- **Flatten order is ROW-major, NOT column-major.** Iterate rows outer, columns inner. Feature index `k` holds `matrix[c][r]` where `r = k // cols` and `c = k % cols`. The on-wire matrix in section 7.1 is column-major; the flatten reorders it on write.
- Label columns come from the labeler source's frame (`lask5` `data.values` or `quest_hand` flat `data.values`), named `label_0..label_M` and emitted as floats (LASK5 calibrated targets are floats in `[0, 1]`; Quest joint poses are floats too).
- Line endings are CRLF, matching what both the canonical Python `csv.writer` and the phone's writer produce.

This is the byte-canonical layout; the reference is the phone's `tools/make_golden_csv_v2.py` (in OpenMuscle-Connect) and the PC `CaptureWriter` (in OpenMuscle-Software). Both produce identical bytes for identical inputs.

**Forearm-orientation label columns** (ratified 2026-06-26, board #0228; source spec #0209 v0.2). When a VR / `quest_hand` labeler is active in the capture, the hub writes two additional label columns derived from the tracked hand:

| Column | Type | Description |
|---|---|---|
| `forearm_roll_deg` | float, `[-180, 180]` | Continuous forearm roll, **gravity-relative**: `0` = palm-up (palm normal aligned with world up), `±180` = palm-down, `±90` = palm vertical. Gravity-relative because Quest hand tracking does not expose the elbow, so a true anatomical zero is not derivable; any anatomical re-zeroing is a fixed offset applied downstream. |
| `palm_up` | int, `0/1` | Derived flag: `1` = palm faces up. Monotone in `forearm_roll_deg`, so consumers can re-derive it from the roll column without reading a threshold from this spec. |

Presence convention: these columns are **OMITTED entirely** from captures that have no VR/quest labeler (not written as NaN or sentinel). `meta.json`'s `label_source` is the truth for whether they exist. Because schema-v2 files can therefore have differing column sets, **consumers MUST reindex each capture against the full schema-v2 column set before stacking captures** — a naive concat silently NaN-fills or misaligns.

Reference derivation: `openmuscle/forearm.py` in OpenMuscle-Software (position-based, wrist + knuckle joints vs gravity; handedness flips the palm normal).

### 8.4 Left || Right feature concat (derived TRAINER matrix)

The 120-column Left-then-Right concatenation is the **derived trainer matrix**, not the on-disk CSV column count. It is formed by grouping CSV rows (section 8.3) by `ts_hub_ms` and concatenating Left features then Right features:

```
trainer_row = features_left (60 cols) || features_right (60 cols) || label vector
```

A 15x4 V4 band gives 60 features per side, so a bilateral trainer matrix has 120 feature columns. The on-disk CSV remains LONG (~60 feature columns plus role + ids); the pivot to 120-column wide is the trainer pipeline's responsibility.

Pivot rules:
- Group CSV rows by `ts_hub_ms`. Drop groups missing a side.
- For the labeler value, use the labeler row nearest to the group's `ts_hub_ms` (within a configurable window; the PC reference uses the existing `TemporalMatcher`).
- A trainer that sees only one distinct `role` in the input MUST detect that and skip the pivot, training a 60-feature single-source model. Pooling Left and Right rows into a role-agnostic 60-feature model is a silent failure mode and MUST be guarded against.

This ordering (Left then Right) is contractual. Trainers, inference servers, and any future cloud pipeline MUST use it. Single-band captures stay 60 features wide.

### 8.5 One-limb mirroring

For users with only one arm, the recommended capture topology is two FlexGrid bands on the remaining arm (different positions, e.g. proximal + distal) plus a labeler. During labeling, the labeler position-mirror MUST be applied to the label vector so that the trained model still produces bilateral output.

**Same-arm band-to-slot mapping.** When two same-arm bands feed the Left and Right slots of section 8.4's 120-feature trainer matrix, the deterministic convention is:

| Physical position | Trainer slot |
|---|---|
| Proximal (closer to the elbow) | Left |
| Distal (closer to the wrist) | Right |

Hubs MUST surface this convention in their role-assignment UI so two operators tagging the same physical setup produce identical feature vectors. Without it, "proximal-as-left" and "distal-as-left" would both be valid local choices and the resulting trainer matrices would be incompatible.

Mirroring convention for the label vector: for finger-target labels, the per-finger value is the same on both sides (a closed fist closes both hands). For joint-pose labels (Quest), the `x` coordinate is negated and the quaternion is reflected across the YZ plane. Per-joint mirroring tables MAY be carried in a sidecar JSON.

A hub MUST tag mirrored captures with `mirror: true` in the per-capture sidecar (`<capture>.meta.json`, section 8.6) so downstream training does not double-apply mirroring.

### 8.6 Capture sidecar files

A v2 capture writes the LONG CSV (section 8.3) plus one or two JSON sidecars, all sharing the capture's base filename:

| File | Owner | Contents |
|---|---|---|
| `<capture>.csv` | hub writer | The LONG capture CSV per section 8.3. |
| `<capture>.meta.json` | hub writer | Capture-level metadata: `schema` (`"v2"`), `mirror` (bool, per 8.5), `label_source` (`"lask5"` / `"quest"` / `"manual"`), `roles` (object: `device_id` -> role token), `created_ms` (epoch ms). |
| `<capture>.labels.schema.json` | hub writer, Quest captures only | Maps `label_0..label_N` back to `(joint, channel)` for Quest sources. Written on the first `quest_hand` frame of a recording. |

Hubs writing v2 captures MUST emit `<capture>.meta.json` so the trainer can read `mirror` (to NOT double-apply one-limb mirroring) and `roles` (to map device ids to roles without parsing every CSV row). The PC and phone writers use the same key names; trainers and analyzers can read either source identically.

The two sidecars (`meta.json` and `labels.schema.json`) coexist; neither overwrites the other.

## 9. Backward compatibility

A v1.0-compliant hub SHOULD accept these legacy formats from older firmware that has not been updated:

1. **Bare JSON array.** A UDP payload that is a JSON array, not an object, is a legacy FlexGrid V3 frame. The array is the matrix. The hub synthesizes envelope fields: `type="flexgrid"`, `id="flexgrid-legacy-<src-ip>"`, `v="0.x"`.
2. **Python dict literal.** A payload parseable by `ast.literal_eval` but not by JSON is a legacy LASK5 / SensorBand format. Map `id`, `values`, optional `joystick` into the v1.0 envelope.
3. **Pre-split announce on 3141.** Pre-spec firmware broadcasts the announce on UDP 3141, not 3140. A v1.0 hub MAY listen on both ports during a transition window and route by frame `type`. This is a transition aid; new firmware MUST use 3140.

Backward-compat is a hub responsibility. v1.0 sources MUST emit only v1.0 frames on the correct ports.

## 10. Conformance

A v1.0-compliant **source** MUST:

1. Broadcast announce frames to `255.255.255.255:3140` at the configured interval.
2. Stop broadcasting while at least one subscriber is active; resume within one interval of the list becoming empty.
3. Accept TCP connections on the advertised cmd port (8001 / 8002) and handle the verbs in section 6.3.
4. Honor the subscriber cap and reply `accepted=false` cleanly when full.
5. Drop subscribers whose heartbeat is older than the timeout.
6. Send data frames as unicast UDP to each subscriber's `(host, port)` with a monotonic `seq`.
7. Survive an invalid command line without dropping the TCP connection (reply with `status=error`).
8. Catch `BaseException` (not only `Exception`) at the cmd-server accept-loop boundary, and supervise the listener so it restarts if it dies. Without this, an asyncio interruption (e.g. KeyboardInterrupt from a dev tool) silently kills the listener while the rest of the device keeps running, and hubs see a black hole.

A v1.0-compliant **hub** MUST:

1. Bind UDP 3140 for announce and UDP 3141 (or its own chosen port) for data; do not collide on a single port.
2. Maintain a persistent address cache keyed by device `id`, and re-probe known devices via `get_info` on startup.
3. Send heartbeats on the SAME TCP cmd channel as the subscribe verb, at 1 Hz.
4. Re-subscribe if the source drops it (subscribe again, do not assume the prior subscription survived).
5. Tag each subscribed source with a role (`left`, `right`, `labeler`); persist the tag per device id.
6. Stamp arrival time on receipt and use that, not source-local `ts`, for cross-source alignment.
7. Reject frames whose `v` is a major mismatch; log and continue on minor or unknown fields.

## 11. Open for v1.1

The following are intentionally NOT in v1.0; they are tracked here so implementers know what to leave room for.

- **BLE transport** alongside Wi-Fi. `transports: ["wifi", "ble"]` in the announce, BLE-specific subscribe with characteristic handles in `data`.
- **Time sync.** Source-local `ts` is not synchronized across devices. A future spec may add a hub-driven time-sync verb.
- **Subscribe filters.** Letting a subscriber request only sensor or only IMU frames to save bandwidth. Useful for VR latency budgets; deferred because port-split already separates the heaviest streams.
- **Authenticated subscribe.** No auth in v1.0; LAN trust model. Future versions may add a pre-shared token.
- **Provisioning handshake.** SoftAP / captive-portal credential push from the phone to a factory-fresh source. Tracked in PROVISIONING.md (separate doc, owned by firmware + phone teams).

## 12. Change log

- **2026-06-23 v1.0** Initial freeze. Port split (3140 announce, 3141 data) and multi-device data model baked in. Reference impl: FlexGridV4-Firmware. Authors: firmware team via OpenMuscle coordination board.
- **2026-06-24 v1.0 (pre-push fixes, doc-only, non-breaking)**: section 7.1 + 7.4 status `meta` field names corrected to match the reference firmware (`vbat`, `pct`, `rssi`, `free_mem`, `uptime_s`, `subscribers`, `imu`, `reset_cause`, `reset_cause_name`; not `battery_mv` / `scan_hz`). Section 8.3 corrected to **row-major** R{r}C{c} feature flatten and clarified as the on-disk LONG capture format. Section 8.4 reframed as the derived TRAINER matrix (120 cols = pivot result, not on-disk CSV column count) with pivot rules. Section 8.5 adds the deterministic proximal->left / distal->right mapping for two same-arm bands. Section 8.6 (new) specifies the `meta.json` + `labels.schema.json` capture sidecars and pins their key names so phone and PC writers stay interoperable. Section 6.1 cmd example gains a top-level `id`. Sign-offs in: phone (#0026, #0042, #0053), vrpc (#0032, #0057). Overseer ratified (#0036, #0058, #0068). Reference: `make_golden_csv_v2.py` (phone byte golden) and `web/state.py` + `CaptureWriter` (PC writer).
- **2026-06-25 v1.0 (data.imu, non-breaking optional)**: section 7.1 gains optional `data.imu = {"accel": [ax,ay,az], "gyro": [gx,gy,gz]}` for sources that ride IMU samples on the sensor channel at the sensor rate, for smooth orientation viz. Variant + temperature stay in `meta.imu` (section 7.4) as the canonical metadata path. Decision: board #0163 (Tory). Reference impl: FlexGridV4-Firmware imu_loop publishing into a shared cache that sensor_loop reads and passes through `network_manager.send_sensor(... imu=...)`.
- **2026-07-02 v1.0 (forearm-orientation label columns, non-breaking optional)**: section 8.3 gains the ratified `forearm_roll_deg` (float [-180,180], gravity-relative, 0 = palm-up) + `palm_up` (int 0/1) label columns, written only when a VR/`quest_hand` labeler is in the capture; OMITTED otherwise (no sentinel), with `meta.json` `label_source` as the presence truth and a hard consumer-reindex-before-stacking requirement. Ratified by Tory (board #0228); source spec #0207/#0209 v0.2 (vrpc); derivation reference `openmuscle/forearm.py`. Does not touch sections 5.5/5.6/6.3 (frozen per #0288/#0316).
