# Status and change log

Part of the [MSI Optix G27C2 protocol reference](README.md). One row per code or area; the tags are explained in the
README.

## DDC/CI and EDID

| Code / area | Name | Read | Write | Open |
|---|---|---|---|---|
| physical monitor | handle from `GetPhysicalMonitorsFromHMONITOR` | value 0, valid (OBSERVED) | | |
| session | where DDC/CI works | user session only; none in session 0 (OBSERVED) | | |
| capability string | `GetCapabilitiesStringLength` | fails every time with an I2C error (OBSERVED) | | why |
| reliability | I2C transmit errors | about one read in four fails; retry works (OBSERVED) | sets fail without idle time; 50 ms after the end of a command works (OBSERVED) | behaviour while the monitor sleeps |
| `0x0B` / `0x0C` | colour temperature increment / request | 50 K; 6500 K, 9300 K (OBSERVED) | not tried | writability of `0x0C` |
| `0x10` | brightness | yes | **yes** (OBSERVED 2026-11-01) | persistence across a power-button cycle |
| `0x12` | contrast | yes | **yes** (OBSERVED 2026-11-02) | as above |
| `0x14` | colour preset: Cool `04`, Warm `05`, User `06` | yes; a late update after an OSD change once | **ignored** (OBSERVED 2026-11-02) | other values |
| `0x16` / `0x18` / `0x1A` | User RGB gains, 0-100 | yes (R mapped to the OSD) | not tried | writability |
| `0x60` | input | `3` on DisplayPort | not tried | HDMI numbers |
| `0x62` | volume | `50` | not tried | |
| `0x6C` / `0x6E` / `0x70` | black levels | `50` each | not tried | OSD mapping |
| `0xD6` | power mode | `1` (on) | not tried; hazardous | |
| `0xDF` | MCCS version | `0x0201` = 2.1 | | |
| `0xE0`-`0xFF` | manufacturer range | `0xE1`, `0xFF` answer; everything else "not supported" (the 11 inconclusive codes re-read 2026-11-04) | | meanings |
| `0x02`-`0xDF`, full scan | every code that answers (section 4.4) | **47 codes answer** (OBSERVED 2026-11-04): 32 new, among them `0xDC` 4 of 5, `0x13`, `0x17`, `0x22`-`0x29`, `0x51`, `0x55`-`0x57` (MSI's own settings, INFERRED), `0x68`/`0x69` 1920/1080, `0xAC`/`0xAE` frequencies, `0xB2`, `0xB6`, `0xC0` 28661 h, `0xC6`, `0xC8`, `0xC9` `0x0220`, `0xCA`, `0xCC`; `0x15` invalid data, `0x45` no answer | not tried | the OSD-correlation rounds (prepared, not run) |
| EDID | identity, timings, audio | complete (HOST) | | |

## Change log

- **2026-11-04 (evening): the full-range scan.** Read-only scan of `0x02`-`0xFF` with retries and two snapshots of the
  answering codes: 47 codes answer, 32 of them new (section 4.4); the 11 codes left inconclusive by the `0xE0`-`0xFF`
  scan are "not supported" except `0xDC`; the error classes `0xC0262585` (`0x15`) and `0xC0262589` (transient) added.
  Meanings of the new codes still unknown: the OSD-correlation rounds are listed in open item 4 and were not run.
- **2026-11-04, first edition.** Written from reads and writes through Windows' DDC/CI API by independently written
  code (2026-11-01 to 2026-11-04), an OSD-correlation session for the colour presets (2026-11-02), Windows' stored
  EDID, and the public sources credited in the README.
