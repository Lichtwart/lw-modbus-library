# Changelog

All notable changes to this repository are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and
this repository follows
[Semantic Versioning](https://semver.org/spec/v2.0.0.html) as applied to a
library of device definitions:

- **major** — a change to the file shape that older Lichtwart Core versions
  cannot read, or a removed device entry.
- **minor** — a new device entry, or a new optional field.
- **patch** — corrections to an existing entry (addresses, datatypes, units,
  tags) and documentation changes.

Note that this library is read live from `main`: a merged commit is in the UI
within about five minutes, regardless of which version it lands in. The version
number records what the library contained, it does not gate a release.

## [Unreleased]

## [1.3.0] - 2026-11-16

### Added

- `devices/QE-PLUS-3-Phase.json`: **QE 3-Phase PLUS** (Energiezähler, QE,
  Max-Demand). Same registers as QE-STD, plus the monthly 15-minute maximum
  demand (total and per phase, holding 668–674), its timestamp (holding 682,
  `uint32` with `swap "word "`) and the demand window (holding 24). The meter
  clock (RTC, holding 690–695) is included as inactive read registers and as
  write targets, because the RTC starts at 01.01.2000 after installation and
  the demand month follows it. `Status1` uses `swap "word "`, verified on a PLUS
  in the field (Bad Rappenau Nord, 12.11.2026). The demand registers use German
  names: `Ges_Spitzenleistung`, `Spitzenleistung_L1`–`L3`,
  `Spitzen_Zeitpunkt` and `Spitzen_AVG_Fenster`.
- `devices/ORNO-1-Phase.json`: **ORNO 1-Phase** (Energiezähler, ORNO). Voltage,
  current, active power, frequency and total active energy (kWh) as input
  registers, profile in use at Bad Rappenau Nord since 07/2026. `Wirkleistung`
  (input 260) has no `scalingFactor` (the earlier factor of 1 did nothing).

### Fixed

- `devices/QE-STD-3-Phase.json`: `Status1` (holding 238, `uint32`) now uses
  `swap "word "`. The meter sends 32-bit values low word first ("32b-LSW"),
  while the firmware reads `uint32` high word first under `swap "none"`, so the
  status bits came out in the wrong order. Confirmed by the firmware author
  (CORE-427) and on the meters at Bad Rappenau Nord (13. and 15.11.2026).

## 1.2.0

- feat: Finder 1 & 3 phase and QE-STD 3 Phase added
- refactor: Wago 3 phase tag added

## 1.1.0

### Added

- feat: **WAGO 879-3040**

## [1.0.0] - 2026-09-04

First versioned state of the library.

### Added

- `devices/SGB.json` — **SGB-Fuellleitung** (Leckanzeiger, SGB). Pressure
  reading as `int32` on holding 317, plus `Alarm` (coil 5184) and
  `Trockenfilter` (coil 5202) as change-of-value bools. 10 s request interval.
- `devices/Waveshare-8-DO.json` — **WaveShare 8xDO Type C** (Relais). Eight
  relay outputs, read back as coils and switchable through eight write
  registers. 30 s request interval.
- `devices/Waveshare-8-DIDO.json` — **WaveShare 8xDI/8xDO**. Eight digital
  inputs and eight relay outputs as read registers, with the eight outputs also
  exposed as write registers. 30 s request interval.
- `Readme.md` — how the plugin reads this repo, the hard constraints (public
  repo, GitHub API rate limit, CDN caching, silently skipped bad files), and how
  to add a device.
- `Slave-Reference.md` — the full reference for every property a slave can
  carry: legal values, ranges, the traps, and the pre-PR checklist. This is the
  quality gate, as nothing validates the files in CI.
- `LICENSE`.

### Changed

- `devices/SGB.json` reworked from the initial test entry: renamed from "SGB
  Leckanzeiger" to "SGB-Fuellleitung", the placeholder `test` coil write and the
  `CoreTemperature` / `DeviceStartAddress` registers replaced with the real
  pressure, alarm and dry-filter registers, and `tags` added.

[Unreleased]:
  https://github.com/Lichtwart/lw-modbus-library/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/Lichtwart/lw-modbus-library/releases/tag/v1.0.0
