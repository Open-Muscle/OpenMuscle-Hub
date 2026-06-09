# OpenMuscle Architecture and Repo Layout

Status: active decisions, recorded 2026-06-07.
Owner: Tory Moghadam (TURFPTAx).
Scope: how concerns are split across the Open-Muscle repos, and the conventions
that keep firmware, PC software, hardware, and VR cleanly separated.

This note supersedes section 2 of the earlier governance working draft. Where the
draft left items as NEEDS INPUT, the decisions are recorded here.

---

## Repo roles

| Repo | Role | Contains | License |
|------|------|----------|---------|
| [OpenMuscle-Software](https://github.com/Open-Muscle/OpenMuscle-Software) | PC/ML application and firmware workshop | `pc/` (ML, receiver, protocol, viz, CLI, web server including the VR bridge endpoints); `embedded/` (shared `om_lib` plus per-device firmware development); `data/`, `analysis/` | MIT |
| [OpenMuscle-FlexGrid](https://github.com/Open-Muscle/OpenMuscle-FlexGrid) | FlexGrid hardware only | KiCad, BOM, schematics (V3 and V4) | CERN-OHL-S-2.0 |
| [FlexGridV3-Firmware](https://github.com/Open-Muscle/FlexGridV3-Firmware) | Shipping FlexGrid V3 firmware | Promoted stable copy from `embedded/devices/` | MIT |
| FlexGridV4-Firmware (planned) | Shipping FlexGrid V4 firmware | Promoted when V4 hardware stabilizes | MIT |
| [OpenMuscle-LASK5](https://github.com/Open-Muscle/OpenMuscle-LASK5) | LASK5 hardware | KiCad, BOM, STLs | CERN-OHL-S-2.0 |
| [OpenMuscle-AR](https://github.com/Open-Muscle/OpenMuscle-AR) | VR/XR application | WebXR client (planned extraction; currently in OpenMuscle-Software), VR docs, future native Quest APK | MIT |
| [OpenMuscle-Library](https://github.com/Open-Muscle/OpenMuscle-Library) | Shared KiCad and CAD parts | Symbols, footprints, 3D models, mechanical parts reused across devices; consumed via the `OM_LIB` path variable | CC BY 4.0 (vendor parts per source) |
| [OpenMuscle-Hub](https://github.com/Open-Muscle/OpenMuscle-Hub) | Coordination and docs | Roadmap, this note, issue routing | docs |

## Decisions

### 1. Firmware home: develop together, promote on stable

Device firmware is developed together in `OpenMuscle-Software/embedded/`, which
holds a shared library (`lib/om_*.py`) and one folder per device under
`devices/`. Developing the devices together lets cross-device refactors of the
shared library land atomically.

When a device reaches a stable hardware milestone, its firmware is promoted to
its own standalone repo (for example FlexGridV3-Firmware) so it can version
against fixed hardware. The shared library stays in `OpenMuscle-Software` as the
canonical source.

Firmware is kept out of the hardware repos. Hardware (CERN-OHL-S-2.0) and
firmware (MIT) stay in separate repos so licenses do not mix and version cadences
stay independent.

### 2. VR split: client in OpenMuscle-AR, server bridge in the PC app

The WebXR client (currently `pc/src/openmuscle/web/static/vr/`) moves to
OpenMuscle-AR. The PC app keeps the FastAPI web server and its VR bridge
endpoints. The two communicate over the documented network seam (see below), so
the VR application can live and version on its own while the PC app remains the
data and inference source. The future native Quest APK also lives in
OpenMuscle-AR.

### 3. Software repo identity: the PC/ML app

After VR extracts, the shipping identity of OpenMuscle-Software is the PC/ML
application. It also remains the firmware workshop (`embedded/`), since firmware
is developed there and only stable copies are promoted out. No repo rename, to
preserve history and existing links.

## Interfaces (seams)

1. **PC to firmware:** the packet protocol, specified in
   `OpenMuscle-Software/docs/protocol.md` and implemented by `om_packet`.
2. **PC to VR:** the FastAPI server's VR bridge endpoints carry sensor frames out
   and predictions back. This contract is what allows the WebXR client to live in
   a separate repo. It must stay documented in `OpenMuscle-Software/docs/`.

## Conventions

### Firmware folder layout (in `OpenMuscle-Software/embedded/`)

```
embedded/
  lib/                      shared om_*.py modules (canonical source)
  devices/
    <device>_v<major>/      per device-revision firmware
    _template/              starting point for a new device
```

### Promoted firmware repo naming

`<Device>V<major>-Firmware`, for example `FlexGridV4-Firmware`.

### Versioning

- Hardware uses revision majors: V3, V4, and so on.
- Firmware uses semantic versioning inside its repo, for example `v4.0.0`, pinned
  to a single hardware major by the repo name.
- Hardware and firmware version numbers move independently.

## Open items

- Promote FlexGridV4-Firmware once V4 hardware stabilizes (do not promote a
  moving target).
- Confirm whether LASK5 firmware should follow the same promote-on-stable path
  into its own repo.
