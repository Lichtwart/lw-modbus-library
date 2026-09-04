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
