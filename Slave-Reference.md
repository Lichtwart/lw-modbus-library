# Slave reference

Every property a slave in a library entry can carry, what values are legal, and
which of them actually reach the device.

**Nothing validates a file in this repo.** There is no schema check in CI, and
the plugin only refuses a file that is not valid JSON or is missing the `type`
marker — every field below is taken verbatim. So this page is the check: a value
outside these ranges is accepted by the library, inserted into a device config,
and then rejected by the firmware, where it can take the **whole slave table**
down with it and not just the one bad field.

The ranges come from the firmware schema (`@lichtwart/cot-widget-types/core/v2`)
and from the validation the Modbus tab applies to a slave a user edits by hand.
A library entry has to pass the same rules, because after insert it _is_ a
hand-edited slave.

For how a file is laid out, what happens on insert, and the repo's own
constraints, see the [README](Readme.md). This page is only the field reference.

---

## Limits at a glance

| limit                              | value                      | what breaks past it                                   |
| ---------------------------------- | -------------------------- | ----------------------------------------------------- |
| slaves per device                  | **20**                     | the 21st is refused by the tab; firmware ignores it   |
| active read registers per slave    | **40**                     | firmware rejects the slave                            |
| interval writes (`resetRegisters`) | **5**                      | a 6th makes the device reject the whole slave         |
| register name length               | **19 characters**          | the name is rejected, taking the slave config with it |
| unit length                        | **7 characters**           | measurement unit is rejected                          |
| `conversion` / `custom` length     | **14 characters** each     | one shared firmware buffer                            |
| register address                   | **0 – 65535**              | 16-bit Modbus address space                           |
| slave address                      | **1 – 247**                | 0 is broadcast, 248–255 are reserved by the spec      |
| `scalingFactor`                    | **-1 000 000 … 1 000 000** | out of range                                          |
| `changeOfValue`                    | **0 – 1 000 000**          | out of range                                          |
| holding-register value             | **0 – 65535**              | 16-bit register value                                 |

---

## Entry wrapper

A device file is `{ type, tags, slave }`. Only `slave` is described on this
page.

| field   | required | value                                                                    |
| ------- | -------- | ------------------------------------------------------------------------ |
| `type`  | yes      | Exactly `"lw-modbus-slave"`. Anything else and the file is skipped.      |
| `tags`  | no       | `string[]`, ours, for the library's filter dropdown. Stripped on insert. |
| `slave` | yes      | The slave definition. Missing and the file is skipped.                   |

`entryId`, `lastUpdate` and `previous` belong to the **company** library, which
lives on a group managed object in the cloud. They never appear in a file here.
If you see them in a pasted payload, delete them — they are stripped on insert
anyway.

---

## `slave`

| field                   | required | type              | value                                                                                      |
| ----------------------- | -------- | ----------------- | ------------------------------------------------------------------------------------------ |
| `id`                    | yes      | integer           | **Reassigned on insert** (lowest free id on the target device). Put anything; `1` is fine. |
| `address`               | yes      | integer 1–247     | **Reassigned on insert** (highest used + 1). Put the manufacturer default anyway.          |
| `name`                  | yes      | string, non-empty | What the library lists and what the user sees in the menu. Be descriptive.                 |
| `requestInterval`       | yes      | integer ≥ 1       | Seconds between polls of this slave. The tab's own new slaves use `30`.                    |
| `isActive`              | yes      | boolean           | **Use `false`.** An entry arrives switched off so the user enables it deliberately.        |
| `isSavingOnStorage`     | no\*     | boolean           | Not implemented in the firmware. `false`, or leave it out — the tab omits it too.          |
| `description`           | no       | string            | Free text about the slave. Cloud-only.                                                     |
| `registers`             | yes      | array             | Active read registers, max 40. [See below](#registers--read-registers).                    |
| `resetRegisters`        | yes      | array             | Interval based writes, max 5. **Always present**, `[]` when empty.                         |
| `inactiveReadRegisters` | no       | array             | Same shape as `registers`, switched off. Cloud-only.                                       |
| `writeRegisters`        | no       | array             | One-off write targets. Cloud-only, and a different type union.                             |

\* The firmware schema marks `isSavingOnStorage` required, but the Modbus tab
creates slaves without it and the field does nothing yet. Both are accepted.

`resetRegisters` is the one array that must not be dropped when it is empty: it
is a required firmware field and is always written, as `[]`.

### Why `id` and `address` are yours to ignore

On insert the plugin assigns a fresh `id` (lowest free number on that device)
and a proposed `address` (highest used + 1), then drops the slave into the list
as a **pending change**. Nothing reaches the device until the user presses
**Save to device**, and the user is expected to correct the address to whatever
is really on the bus. Everything else in the file is used exactly as written.

---

## `registers` — read registers

One entry per value the device should poll. Each one becomes a **measurement
series** in the cloud, which is where most of the constraints come from.

| field           | required | type                                                 | value                                                                                                 |
| --------------- | -------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `name`          | yes      | string                                               | **Max 19 characters**, non-empty, unique within this slave's read registers.                          |
| `unit`          | yes      | string                                               | **Max 7 characters.** `""` is legal and common.                                                       |
| `address`       | yes      | integer 0–65535                                      | Register address as the manufacturer documents it.                                                    |
| `type`          | yes      | `"holding"` \| `"input"` \| `"discrete"` \| `"coil"` | Modbus register type.                                                                                 |
| `datatype`      | yes      | see [pairing](#datatype--type-pairing)               | What the raw register holds.                                                                          |
| `swap`          | yes      | `"none"` \| `"byte"` \| `"word "` \| `"word_byte"`   | Byte/word order. Note the trailing space.                                                             |
| `scalingFactor` | no       | float, -1 000 000 … 1 000 000                        | Raw value is multiplied by it and output with **two decimal places**. `0` or absent disables scaling. |
| `changeOfValue` | no       | number 0–1 000 000                                   | Send immediately when the value moves by at least this much. `0` or absent = off, poll interval only. |
| `conversion`    | no       | string, max 14 chars                                 | EPL rule key. [See below](#conversion--epl-rules).                                                    |
| `custom`        | no       | string, max 14 chars                                 | Free EPL string, attached to the measurement. No rule of ours reads it.                               |
| `description`   | no       | string                                               | Free text for the register. Cloud-only.                                                               |

**19 characters, not 20.** The type package's JSDoc says `@max 20`; that JSDoc
is wrong, confirmed against the firmware source with its author. A 20-character
name is _rejected_, not truncated, and it takes the whole slave config with it.

**The name is the series name.** It is what appears in charts and in the tab's
Live View, and Cumulocity cannot rename a series without losing its history — so
renaming a register later starts a new series. Pick the name once, properly.
Keep it unique per slave: two registers with the same name write into the same
series.

**Only `registers` reaches the device.** `inactiveReadRegisters` is a cloud-side
parking spot for registers the user switched off; it has the identical shape and
is simply never sent. Use it in a library entry for registers that are
documented but rarely wanted — the user can switch one on without typing it in.

### `datatype` ↔ `type` pairing

The firmware enforces this, and so does the tab:

| register `type`    | allowed `datatype`                                                                                                       |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| `coil`, `discrete` | **`bool` only** — these address single bits                                                                              |
| `holding`, `input` | anything **but** `bool`: `uint8` `uint16` `uint32` `int8` `int16` `int32` `int64` `float32` `float24s` `float24u` `pf32` |

### `swap`

| value         | effect                        |
| ------------- | ----------------------------- |
| `"none"`      | no reordering                 |
| `"byte"`      | swap bytes, `AB` → `BA`       |
| `"word "`     | swap words, `ABCD` → `CDAB`   |
| `"word_byte"` | reverse both, `ABCD` → `DCBA` |

**`"word "` has a trailing space.** That is the firmware schema, not a typo in
this document. `"word"` without it is invalid.

`"word "` and `"word_byte"` are **not valid for `int16` and `uint16`** — there
is only one word, so there is nothing to swap. Use `"none"` or `"byte"` there.

### `conversion` — EPL rules

A rule runs in the tenant's EPL (`cot-epl-scripts`, `core/v2/conversion.apama`)
and publishes extra series beside the raw one. Only these keys do anything;
anything else is passed through and silently ignored, so a typo costs you the
conversion with no error anywhere.

| key             | kind     | what it publishes                                                                       |
| --------------- | -------- | --------------------------------------------------------------------------------------- |
| `enocean-temp`  | suffix   | One series, `<register name>-Converted`, carrying the register's own unit and datatype. |
| `ecp202-output` | bitfield | Splits the register's bits into the fixed series listed below.                          |
| `ecp202-alarms` | bitfield | Splits the register's bits into the fixed series listed below.                          |

**Bitfield series names are fixed and permanent** — they are the EPL's contract,
not derived from your register name. Renaming the register moves only the raw
series.

| key             | series, in bit order                                                               |
| --------------- | ---------------------------------------------------------------------------------- |
| `ecp202-output` | `Compressor` `Defrost` `Fans` `ColdRoomLight` `Dripping` `StandBy` `HotResistance` |
| `ecp202-alarms` | `HighTempAlarm` `LowTempAlarm` `OpenDoorAlarm` `ManInRoomAlarm`                    |

Each bit is published **unitless and as 0/1**.

A bitfield register needs **`datatype: "uint16"`** and **no `scalingFactor`**.
With scaling on, the firmware divides before the rule sees the value and every
bit reads 0, while the measurement still arrives looking perfectly healthy.
Nothing enforces this — it is the easiest way to ship a broken entry.

Do not set both `conversion` and `custom` on the same register: the tab cannot
model what the combination publishes and will show no converted rows for it.

---

## `resetRegisters` — interval based writes

Writes `value` into `address` every `interval` seconds. The firmware calls the
field `resetRegisters` and the type `ModbusSlaveWriteRegister`; the UI calls the
concept **"Interval based writes"**. Same thing.

| field         | required | type                    | value                                              |
| ------------- | -------- | ----------------------- | -------------------------------------------------- |
| `name`        | yes      | string                  | Max 19 characters, unique within this list.        |
| `interval`    | yes      | integer ≥ 1             | Seconds between writes.                            |
| `address`     | yes      | integer 0–65535         | Target register.                                   |
| `type`        | yes      | `"coil"` \| `"holding"` | Writable register types only.                      |
| `value`       | yes      | integer                 | `coil`: `0` or `1`. `holding`: 0–65535.            |
| `description` | no       | string                  | Free text. Cloud-only, not in the firmware schema. |

Rules, all of them easy to get wrong in a file:

- **Max 5 per slave.** A 6th makes the device reject the whole slave.
- **No two entries may target the same `type` + `address`.**
- **The slave needs at least one active read register**, or none of them run.
  The device services reset registers only while it polls the slave, and it
  polls nothing until a read register is active. An entry with a non-empty
  `resetRegisters` and an empty `registers` is broken and looks fine.
- **It only writes.** To read that register as well, add a read register at the
  same address — that is a separate entry in `registers`, not a flag here.
- `interval` and `value` have no safe default. If the manufacturer does not
  document one, do not guess: leave the interval write out of the entry.

---

## `writeRegisters` — one-off write targets

Saved targets for the manual "write a value now" button in the tab.
**Cloud-only:** the device never receives these, so they cannot break a config —
they cost nothing on the wire and are pure convenience for whoever inserts the
entry.

| field         | required | type                     | value                    |
| ------------- | -------- | ------------------------ | ------------------------ |
| `name`        | yes      | string                   | Unique within this list. |
| `address`     | yes      | integer 0–65535          | Target register.         |
| `type`        | yes      | `"register"` \| `"coil"` | Note: `"register"`.      |
| `description` | no       | string                   | Free text.               |

`"register"`, not `"holding"` — this is a **different type union** from the two
arrays above. Three similarly named "write register" things exist in this
system; this is the one behind the manual write button, and it is the only one
the device never sees.

---

## What actually reaches the device

Only these fields are transferred by SmartREST. Everything else lives on the
managed object in the cloud and is used by the UI alone.

| level          | transferred                                                                                            |
| -------------- | ------------------------------------------------------------------------------------------------------ |
| slave          | `id` `address` `name` `requestInterval` `isSavingOnStorage` `isActive` `registers` `resetRegisters`    |
| read register  | `name` `unit` `address` `type` `swap` `datatype` `changeOfValue` `conversion` `custom` `scalingFactor` |
| interval write | `name` `interval` `address` `type` `value`                                                             |

Cloud-only, i.e. safe to fill in freely: every `description`,
`inactiveReadRegisters`, `writeRegisters`, and the entry's `tags`.

---

## Full example

Everything filled in, as a reference for the shape. Real files must be plain
JSON — the comments below are for reading only.

```jsonc
{
  "type": "lw-modbus-slave",
  "tags": ["Kühlstellenregler", "ECP202"],
  "slave": {
    "id": 1, //                       reassigned on insert
    "address": 1, //                  reassigned on insert; manufacturer default
    "name": "ECP202 Kühlstelle",
    "requestInterval": 30, //         seconds
    "isActive": false, //             always false in a library entry
    "isSavingOnStorage": false, //    not implemented in the firmware
    "description": "Eliwell ECP202 controller, RS-485",

    "registers": [
      {
        "name": "RoomTemperature", // 15 chars, becomes the series name
        "unit": "C",
        "address": 256,
        "type": "holding",
        "datatype": "int16",
        "swap": "none", //            "word " is invalid for int16
        "scalingFactor": 0.1,
        "changeOfValue": 0.5,
        "description": "Sensor 1, cold room",
      },
      {
        "name": "OutputState",
        "unit": "", //                a bitfield register has no unit
        "address": 260,
        "type": "holding",
        "datatype": "uint16", //      required by the bitfield rule
        "swap": "none",
        "conversion": "ecp202-output", // no scalingFactor — it would zero every bit
      },
      {
        "name": "DoorSwitch",
        "unit": "",
        "address": 12,
        "type": "discrete",
        "datatype": "bool", //        the only datatype a discrete input may have
        "swap": "none",
      },
    ],

    "inactiveReadRegisters": [
      {
        "name": "EvaporatorTemp", //  documented, arrives switched off
        "unit": "C",
        "address": 257,
        "type": "holding",
        "datatype": "int16",
        "swap": "none",
        "scalingFactor": 0.1,
      },
    ],

    "resetRegisters": [
      {
        "name": "WatchdogReset",
        "interval": 60, //            seconds
        "address": 300,
        "type": "holding",
        "value": 1,
      },
    ],

    "writeRegisters": [
      {
        "name": "ForceDefrost",
        "address": 301,
        "type": "coil", //            "register" | "coil" here
        "description": "Starts a manual defrost cycle",
      },
    ],
  },
}
```

---

## Checklist

Structure:

- [ ] valid plain JSON — no comments, no trailing commas
- [ ] `type` is exactly `"lw-modbus-slave"`
- [ ] `slave.name` filled in and descriptive
- [ ] `tags` filled in, short and reusable
- [ ] `isActive: false`
- [ ] `resetRegisters` present, `[]` if unused

Read registers:

- [ ] ≤ 40 in `registers`
- [ ] every `name` ≤ 19 characters, non-empty, unique within the slave
- [ ] every `unit` ≤ 7 characters
- [ ] every `address` 0–65535
- [ ] `datatype` matches `type` — `bool` only on `coil` / `discrete`, never on
      `holding` / `input`
- [ ] `swap` spelled exactly, `"word "` with its trailing space
- [ ] no `"word "` / `"word_byte"` on `int16` / `uint16`
- [ ] `scalingFactor` within ±1 000 000, and **absent** on every bitfield
      register
- [ ] every bitfield register is `uint16`
- [ ] `conversion` is one of the three known keys, or absent
- [ ] `conversion` and `custom` not both set on one register

Interval writes:

- [ ] ≤ 5 in `resetRegisters`
- [ ] no two share a `type` + `address`
- [ ] `interval` ≥ 1 and documented by the manufacturer, not guessed
- [ ] `value` is `0`/`1` for `coil`, 0–65535 for `holding`
- [ ] `registers` is non-empty whenever `resetRegisters` is

One-off write targets:

- [ ] `type` is `"register"` or `"coil"` — not `"holding"`
- [ ] names unique within `writeRegisters`
