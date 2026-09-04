# lw-modbus-library

Vetted Modbus device definitions for the **Lichtwart library** in the Modbus tab
of Lichtwart Core v2.

Every JSON file in `devices/` becomes one entry in the library list that users
pick from when they add a slave to a device. The plugin reads this repo
**directly from the browser** over GitHub's public API — there is no build step
and no deployment. A merged commit to `main` is live for every tenant on the
next library open.

## Hard constraints

- **The repo must stay public.** The browser request carries no token, and a
  token in a frontend bundle is not a token. Making the repo private breaks the
  library for everyone, with no error the user can act on.
- **The GitHub API rate limit is not a problem, but know where it sits.**
  Unauthenticated calls to `api.github.com` are capped at **60 per hour per
  source IP**. One library open costs exactly **one** of those — the folder
  listing. The device files themselves come from `raw.githubusercontent.com`,
  which is a CDN and not the rate-limited API, so the number of files in
  `devices/` does not enter into it. 60 opens per hour per IP, or per shared
  office IP behind one NAT. Nobody will hit that. An aggregated `library.json`
  built in CI is still worth doing eventually — not for the limit, but because
  it removes the API call altogether (a plain `raw.githubusercontent.com` fetch
  is not rate-limited at all) and opens the modal in one request instead of one
  per file.
- **`raw.githubusercontent.com` caches for 5 minutes**
  (`cache-control: max-age=300`). A merged change can take that long to appear
  in the UI. That is the CDN, not a bug in the plugin.
- A file that does not parse, or that is missing the `type` marker, is **skipped
  with a console warning** — the rest of the library still loads. So a bad
  commit costs one entry, not the whole list. It also means a broken file is
  invisible in the UI: check the browser console after a change.

## Adding a device

1. Create `devices/<Manufacturer>-<Model>.json`. The filename is not shown
   anywhere — the entry is listed by `slave.name` — but keep it descriptive.
2. Fill in the shape below, with [Slave-Reference.md](Slave-Reference.md) open
   beside it.
3. Open a PR. The review is the quality gate; there is no schema validation in
   CI yet.

The easiest way to author a file: configure the slave in the Modbus tab of a
real device, press **Copy** on the slave, and paste the clipboard JSON into the
file. The clipboard payload is exactly this format — add the `tags` array and
you are done.

## File shape

```jsonc
{
  "type": "lw-modbus-slave", // required marker, exactly this string
  "tags": ["Leckanzeiger", "SGB"], // ours, optional but please fill it in
  "slave": {
    "id": 2,
    "name": "SGB Leckanzeiger",
    "address": 2,
    "requestInterval": 20,
    "isActive": false,
    "description": "",
    "registers": [
      {
        "name": "CoreTemperature",
        "unit": "C",
        "address": 64,
        "type": "holding",
        "datatype": "int16",
        "swap": "none",
        "scalingFactor": 1,
      },
    ],
    "resetRegisters": [],
  },
}
```

| field   | required | notes                                                                  |
| ------- | -------- | ---------------------------------------------------------------------- |
| `type`  | yes      | Must be `"lw-modbus-slave"`. Any other value and the file is skipped.  |
| `tags`  | no       | Free text, used by the library's filter dropdown. Missing = no filter. |
| `slave` | yes      | The device definition. Missing and the file is skipped.                |

Keep tags short and reusable — they are shown as a flat list across all entries,
so `"Leckanzeiger"` is useful and `"SGB Leckanzeiger V2 2024"` is not.

**Every property a slave can carry — legal values, ranges, the traps, and the
pre-PR checklist — is in [Slave-Reference.md](Slave-Reference.md).** Nothing
validates a file, so that page is the check. Read it before your first entry;
the ranges are not obvious and a value out of range takes down the whole slave
config on the device, not just the one field.

## What happens on insert

When a user picks an entry, the plugin:

1. strips `tags` (and any other library-only field),
2. assigns a fresh `id` and a free `address` on that device,
3. drops the entry into the slave list as a **pending change** — nothing reaches
   the device until the user presses **Save to device**.

So `id`, `address` and `tags` are the three fields you do not need to get right.
Everything else is used verbatim.

## Checklist before opening a PR

The full checklist — structure, read registers, interval writes, write targets —
is at the end of [Slave-Reference.md](Slave-Reference.md#checklist). Work
through it; there is no CI that will do it for you.

## Versioning

The library is versioned in [CHANGELOG.md](CHANGELOG.md) — that file is the only
place the version lives, so there is nothing to keep in sync. Add your entry
under `## [Unreleased]` in the same PR as the device file; the version is cut
(and the git tag pushed) when a batch of entries is done.

Because the plugin reads `main` live, the version does not gate anything: it is
a record of what the library contained, not a release channel.
