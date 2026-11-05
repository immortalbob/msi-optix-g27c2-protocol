# MSI Optix G27C2: DDC/CI and EDID (Windows `DISPLAY\MSI1462`)

Part of the [MSI Optix G27C2 protocol reference](README.md). Tags (**OBSERVED**, **HOST**, **PUBLISHED**,
**INFERRED**, **UNKNOWN**) are explained there. Times are local time of the observing PC (UTC-7). VCP codes are
written `0x10`; a VCP value is the 16-bit word the monitor returns, written `0x005A` (current) and `of 0x0064`
(maximum). Windows error codes are given in hex with their Windows names.

Short version (each point is tagged in the sections below): the monitor has no USB function, so DDC/CI over the
display cable is the only control channel. It answers standard MCCS codes for brightness, contrast, colour presets,
RGB gains, black levels, input and volume, but it does **not** return a capability string, so a host has to know the
codes in advance. Its I2C link is unreliable: roughly one read in four fails with a transmit error, and commands sent
with no idle time between them fail. Brightness and contrast writes work; a colour-preset write is acknowledged and
ignored. MSI numbers its colour presets and its inputs its own way, not as MCCS does.

## 1. Device

### 1.1 Identity (HOST)

| Item | Value |
|---|---|
| Model | MSI Optix G27C2: 27-inch class (60 × 33 cm), 1920 × 1080, up to 144 Hz (from the EDID, section 1.2) |
| Windows hardware id | `MONITOR\MSI1462` (display devnode `DISPLAY\MSI1462\...`) |
| Windows driver | the inbox "Generic PnP Monitor" (`monitor.inf`); no MSI monitor driver or colour profile was installed |
| Connection on the observed PC | DisplayPort, a second monitor beside the primary display |
| USB | none: Windows lists no USB device belonging to this monitor, so there is no USB control path |

The monitor's own name in Windows' device list is "Generic Monitor (Optix G27C2)"; the product name comes from the
EDID (section 1.2).

### 1.2 EDID (HOST)

Read from the registry copy Windows keeps of the monitor's EDID (256 bytes: a base block and one CTA-861 extension,
both checksums valid):

| Item | Value |
|---|---|
| Manufacturer id | `36 69` = `MSI` |
| Product code | `0x1462` (stored `62 14`) |
| Numeric serial | `1` (a placeholder) |
| Built | week 26 of 2019 |
| EDID version | 1.4 |
| Video input | `0xB5`: digital, 10 bits per colour, DisplayPort |
| Screen size | 60 × 33 cm (image size in the timings: 595 × 336 mm) |
| Gamma | 2.2 (`0x78`) |
| Features | `0x3B`: DPMS active-off, RGB 4:4:4 + YCbCr 4:4:4 and 4:2:2, preferred timing is native, continuous frequencies |
| Name descriptor (`FC`) | `Optix G27C2` |
| Range descriptor (`FD`) | `30 90 A2 A2 0A 01`: vertical 48 to 144 Hz, horizontal 162 kHz, maximum pixel clock 100 MHz |
| Serial descriptor (`FF`) | `0000000000001` (a placeholder) |
| Preferred timing | 1920 × 1080 at 60.00 Hz, pixel clock 148.50 MHz |
| Extension timings | 1920 × 1080 at 119.98 Hz (285.50 MHz) and at 143.99 Hz (325.05 MHz) |
| Standard timings | 1280 × 1024, 1440 × 900, 1680 × 1050 at 60 and 120 Hz; 1024 × 768 and 1920 × 1080 at 120 Hz |
| CTA-861 extension | revision 3, underscan and basic audio supported, one native timing; audio data block: LPCM, 2 channels, 32 / 44.1 / 48 kHz, 16 / 20 / 24 bit |

- **The serials are placeholders.** Both the numeric serial and the serial string are the same on every unit
  (INFERRED from their values). Two G27C2s on one PC cannot be told apart by EDID; key per-monitor state by the
  display path (Windows' target id) instead.
- **The range descriptor contradicts the timings.** Its 100 MHz pixel-clock limit is below the 285.50 and 325.05 MHz
  that the 120 and 144 Hz timings need (INFERRED: the range descriptor is not maintained; Windows uses the detailed
  timings and offers 144 Hz).
- **Two odd data blocks.** The CTA extension also carries two 3-byte blocks with the speaker-allocation tag
  (`86 86 6F` and `86 6F 80`) whose contents make no sense as speaker masks (INFERRED: malformed or mis-tagged; no
  effect seen).

## 2. Transport: DDC/CI through Windows

The monitor is controlled with DDC/CI (VESA's Display Data Channel Command Interface) over the display cable's I2C
lines, using the MCCS command set (**PUBLISHED**, VESA). On Windows the host uses the low-level monitor
configuration API in `dxva2.dll` (**PUBLISHED**, Microsoft):

| Step | Function | What the G27C2 does |
|---|---|---|
| 1 | `EnumDisplayMonitors` gives the `HMONITOR` | |
| 2 | `GetNumberOfPhysicalMonitorsFromHMONITOR`, `GetPhysicalMonitorsFromHMONITOR` | one physical monitor; its handle value was **0**, which is a valid handle (OBSERVED 2026-11-01). Code must not treat 0 as "no handle" |
| 3 | `GetCapabilitiesStringLength`, `CapabilitiesRequestAndCapabilitiesReply` | **fails every time** (section 3) |
| 4 | `GetVCPFeatureAndVCPFeatureReply(handle, code, type, current, maximum)` | answers, with failures (section 2.2) |
| 5 | `SetVCPFeature(handle, code, value)` | answers, with failures (section 5) |
| 6 | `SaveCurrentSettings` | never called (section 8) |
| 7 | `DestroyPhysicalMonitors` | |

Each `GetVCPFeatureAndVCPFeatureReply` call takes about 40 ms (OBSERVED; consistent with the DDC/CI reply wait,
**PUBLISHED**, ddcutil).

### 2.1 Where the host must run

DDC/CI is reachable only from the interactive user session. A process in session 0 (a Windows service running as
LocalSystem) sees one virtual 1024 × 768 screen named `WinDisc` and no physical monitor:
`GetNumberOfPhysicalMonitorsFromHMONITOR` fails with `0xC02625E5` (OBSERVED 2026-11-01 19:48, the same PC and
monitors as 19:47 from the user session). A service that wants to control the monitor needs a helper process in the
user's session.

### 2.2 Error codes and reliability

| Code | Windows name | Meaning here |
|---|---|---|
| `0xC0262582` (−1071241854) | `ERROR_GRAPHICS_I2C_ERROR_TRANSMITTING_DATA` | a transient I2C failure: retry. Also the permanent answer to the capability request |
| `0xC0262584` (−1071241852) | `ERROR_GRAPHICS_DDCCI_VCP_NOT_SUPPORTED` | the monitor does not implement the code: do not retry |
| `0xC02625E5` | (no physical monitor for this `HMONITOR`) | session 0, see section 2.1 |

- **About one read in four fails** with `0xC0262582`, also with plain reads 60 ms apart (OBSERVED 2026-11-02
  05:16 to 05:24, every snapshot of section 4.3 has failed reads; the same code read on the next try). Brightness
  `0x10` answered at 18:58 and failed at 19:47 on 2026-11-01 with the same error (OBSERVED). A failed read is a
  transient fault, not a missing feature.
- **Commands with no idle time between them fail.** On 2026-11-02 05:08 and 05:26, writes whose set followed the
  previous command's reply almost at once failed 1 to 3 ms after they began (OBSERVED). After the host measured a
  50 ms gap from the **end** of each command (a Get spends about 40 ms waiting for its reply, so a gap counted from the
  start was nearly zero), the next writes went through with no error (OBSERVED 2026-11-02 05:42). The DDC/CI
  specification has the host wait 40 to 200 ms depending on the operation, and ddcutil recommends longer sleeps and
  retries because I2C is unreliable (**PUBLISHED**, ddcutil).
- **An error on a set does not mean the set was lost.** At 05:08:01 a contrast set returned `0xC0262582` 3 ms in, and
  the value read back afterwards was the new one (OBSERVED). Always read back.
- **A minute without answers.** On 2026-11-03 at 16:21 every read failed for about a minute during normal use; the
  monitor then answered again with no intervention (OBSERVED; cause UNKNOWN). Behaviour while the monitor sleeps with
  the PC was not measured (UNKNOWN).
- MCCS commands may be honoured only from the active input on some monitors (**PUBLISHED**, general MCCS
  references); with the PC on another input, expect failures (UNKNOWN for this model: not tried).

## 3. No capability string

`GetCapabilitiesStringLength` failed with `0xC0262582` on every attempt: three tries 500 ms apart on 2026-11-01 at
18:58, and at every later start (OBSERVED). The same calls on the other monitor of the PC returned its string, so the
path itself works (OBSERVED). No capability string for this model was found in public sources either (**PUBLISHED**:
by absence; searched 2026-11-01).

Consequences:

- A host cannot learn the supported codes from the monitor. It must use a known list (section 4) and probe codes
  one by one.
- Probe only the codes listed here. Scanning every VCP code blindly can change settings, black the screen or stop
  DDC/CI until a power cycle on some monitors (**PUBLISHED**, Microsoft PowerToys' Power Display documentation). On
  this unit a read-only scan of `0x02`-`0xFF` that skipped the MCCS write-only controls went through without any
  visible effect (OBSERVED 2026-11-04, section 4.4); reads of other codes stay at the reader's risk.
- The MCCS version is still readable: `0xDF` returns `0x0201` = MCCS 2.1 (OBSERVED 2026-11-02 05:19).

## 4. VCP codes

### 4.1 Codes that answer (OBSERVED)

Values as read on 2026-11-01 and 2026-11-02 (current, maximum). "OSD" is the monitor's on-screen menu.

| Code | MCCS name | Value read | Meaning on this model |
|---|---|---|---|
| `0x0B` | colour temperature increment | `0x0032` of `0xFFFF` | 50 K steps for `0x0C` |
| `0x0C` | colour temperature request | `0x0046` of `0x00AA` | `3000 K + value × 50 K`: `0x46` = 6500 K, `0x7E` = 9300 K, maximum `0xAA` = 11500 K (section 4.2) |
| `0x10` | luminance (brightness) | `0x005A` of `0x0064` | 0 to 100, as the OSD shows it; writable (section 5) |
| `0x12` | contrast | `0x0032` of `0x0064` | 0 to 100, as the OSD shows it; writable |
| `0x14` | select colour preset | `0x0005` of `0x000B` | MSI's numbering, not MCCS's (section 4.2); writes ignored |
| `0x16` | video gain red | `0x0047` of `0x0064` | the OSD's User R, 0 to 100 |
| `0x18` | video gain green | `0x0045` of `0x0064` | the OSD's User G (by the standard; R was verified) |
| `0x1A` | video gain blue | `0x0041` of `0x0064` | the OSD's User B (by the standard) |
| `0x60` | input source | `3` | **3 while on DisplayPort**; MCCS 3 would be DVI 1, so MSI numbers its inputs its own way. The other inputs' numbers are UNKNOWN |
| `0x62` | audio speaker volume | `50` | the headphone output's volume (INFERRED from the MCCS name and the EDID's audio block) |
| `0x6C` / `0x6E` / `0x70` | video black level red / green / blue | `0x0032` of `0x0064` each | not mapped to an OSD item (UNKNOWN) |
| `0xD6` | power mode | `1` | on |
| `0xDF` | VCP version | `0x0201` | MCCS 2.1 |
| `0xE1` | manufacturer specific | `0x0000` of `0xFFFF` | UNKNOWN |
| `0xFF` | manufacturer specific | `0x0000` of `0x0001` | UNKNOWN |

The meaning of each standard code is **PUBLISHED** by VESA (MCCS); the values and their mapping to the OSD are
OBSERVED on this unit. The gains read 71 / 69 / 65 in every preset (OBSERVED), so they are the User preset's
settings and are kept while another preset is active.

### 4.2 Colour presets: MSI's numbering

The OSD offers only three colour temperatures: Cool, Warm and User (with R, G, B gains). The owner selected each one
while third-party code read the codes (OBSERVED 2026-11-02 05:16 to 05:24):

| OSD | `0x14` | `0x0C` | Colour temperature | MCCS name for that `0x14` value |
|---|---|---|---|---|
| Cool | `0x04` | `0x7E` (126) | 9300 K | 5000 K |
| Warm | `0x05` | `0x46` (70) | 6500 K | 6500 K |
| User | `0x06` | `0x46` (70) | 6500 K, adjusted by the gains | 7500 K |

- The MCCS names for `0x14` values (**PUBLISHED**, VESA) do not describe this monitor: "5000 K" is MSI's 9300 K Cool.
  Read `0x0C` for the actual temperature, or name the values Cool / Warm / User.
- The maximum `0x000B` suggests the MCCS range up to "User 1", but only `0x04` to `0x06` were seen (OBSERVED).
- Changing User R from 71 to 70 in the OSD moved only `0x16`, from `0x47` to `0x46` (OBSERVED): the gains use the
  OSD's 0 to 100 scale directly.
- **A late value.** Right after the owner selected Cool, `0x14` still read `0x05` twice; 85 s later it read `0x04`
  (OBSERVED 05:19:36 and 05:21:03). Whether the monitor updates its DDC copy late or the OSD applies a choice only
  when the menu closes is UNKNOWN. Read again after a delay before trusting a value that should have changed.

### 4.3 Manufacturer range `0xE0` to `0xFF`

Scanned once, read-only, on 2026-11-02 at 05:19:59 with the colour preset on Cool (OBSERVED):

| Result | Codes |
|---|---|
| answers | `0xDF` (`0x0201`), `0xE1` (`0x0000` of `0xFFFF`), `0xFF` (`0x0000` of `0x0001`) |
| `ERROR_GRAPHICS_DDCCI_VCP_NOT_SUPPORTED` | `0xE0`, `0xE4`, `0xE5`, `0xE9`, `0xEA`, `0xED`, `0xEE`, `0xEF`, `0xF1` to `0xF6`, `0xF9`, `0xFB`, `0xFC` |
| I2C error (inconclusive) | `0xDC`, `0xE2`, `0xE3`, `0xE6`, `0xE7`, `0xF0`, `0xF7`, `0xF8`, `0xFA`, `0xFD`, `0xFE` |

No code in the range changed with the colour preset, so there is no hidden colour control there (OBSERVED within the
limits of one scan). The inconclusive codes were re-read on 2026-11-04 (section 4.4): all "not supported" except
`0xDC`. `0xEF` also answered "not supported" in a separate read on 2026-11-01 (OBSERVED). No manufacturer code of this
model is documented anywhere public (**PUBLISHED**: by absence).

### 4.4 Every code that answers (OBSERVED 2026-11-04, 16:57 to 17:02)

A read-only scan of `0x02` to `0xFF` with up to three tries per code (80 ms between reads), skipping the MCCS
write-only controls (`0x04`-`0x0A`, `0xB0`) and the ASUS command codes; then two snapshots of the answering codes
17 s apart, identical for every code that answered in both (39 of 47). **47 codes answer**, 32 more than section 4.1
listed. The new ones:

| Code | MCCS name | Value read (current of maximum) | Note |
|---|---|---|---|
| `0x02` | new control value | `0x0002` of `0x00FF` | MCCS "no new control values" |
| `0x13` | backlight control | `0x03E1` of `0x0064` | the current value exceeds the maximum: not a plain level; meaning UNKNOWN |
| `0x17` | user colour vision compensation | `0x0001` of `0x0064` | UNKNOWN |
| `0x22`, `0x24`, `0x25`, `0x26`, `0x29` | CRT geometry codes in MCCS | `1`, `4`, `2`, `2`, `0x0F`, each of `0xFFFF` | a flat panel has no such geometry: INFERRED to be MSI's own settings under standard numbers (OSD candidates) |
| `0x51`, `0x55`, `0x56`, `0x57` | reserved in MCCS 2.1 | `0x32`, `0x32`, `1`, `0`, each of `0xFFFF` | INFERRED: MSI's own settings (OSD candidates) |
| `0x52` | active control | `0` of `0xFF` | |
| `0x68`, `0x69` | (not in MCCS 2.1) | `0x0780` (1920), `0x0438` (1080) | the panel's width and height (INFERRED from the values) |
| `0x8D` | audio mute | `2` of `2` | not muted |
| `0xAC` | horizontal frequency | `0x07AC` of `0xFF01` | |
| `0xAE` | vertical frequency | `0x177A` of `0xFFFF` | 6010 in 0.01 Hz = 60.1 Hz at the time of the scan (the signal, not the maximum refresh rate) |
| `0xB2` | flat panel sub-pixel layout | `1` of `8` | RGB vertical stripe |
| `0xB6` | display technology type | `3` of `4` | LCD (TFT) |
| `0xC0` | display usage time | `0x6FF5` | 28661 hours |
| `0xC6` | application enable key | `0x006F` | |
| `0xC8` | display controller id | `0x0012` | |
| `0xC9` | firmware level | `0x0220` | |
| `0xCA` | OSD | `2` of `2` | OSD enabled |
| `0xCC` | OSD language | `2` of `0xFF` | English in MCCS numbering |
| `0xDC` | display application | `4` of `5` | an OSD mode selector: INFERRED, the strongest candidate for MSI's Game or Pro mode |

Errors in the scan: `0x15` returned `ERROR_GRAPHICS_DDCCI_INVALID_DATA` (`0xC0262585`) on every try; `0x45` the I2C
transmit error on all three tries; `0x5B` and `0xBE` `ERROR_GRAPHICS_DDCCI_INVALID_MESSAGE_COMMAND` (`0xC0262589`).
The last two errors are not final answers: in the snapshots `0x68`, `0x69`, `0xAC` and `0xB6` produced the same
`0xC0262589` once each and answered on another try, so **every error except "not supported" is worth a retry on this
monitor** (OBSERVED). Every other code in `0x02`-`0xFF` answered "not supported" at once.

What these codes mean on the OSD is not yet known: the OSD-correlation rounds (one menu item changed at a time, the
47 codes read before and after) were prepared but not run. Candidates for MSI's own settings are `0xDC`, `0x13`,
`0x17`, `0x22`-`0x29`, `0x51`, `0x55`-`0x57`, `0xE1` and `0xFF`.

## 5. Writes

Every write was sent with the owner watching the OSD, one value at a time, with `SetVCPFeature` and read back with
`GetVCPFeatureAndVCPFeatureReply` (OBSERVED):

| Date and time | Code | Change | Result |
|---|---|---|---|
| 2026-11-01 20:56:46 | `0x10` | 90 → 80 (`0x005A` → `0x0050`) | **failed**: the set returned `0xC0262582`, a retry failed on its first read, the OSD still showed 90 |
| 2026-11-01 21:12:46 | `0x10` | 90 → 80 | **ok**: one set, no error, read back 80; the owner saw the screen dim and the OSD show 80 |
| 2026-11-01 21:13:59 | `0x10` | 80 → 90 | **ok**: one set, read back 90 |
| 2026-11-02 05:08:01 | `0x12` | 50 → 40 (`0x0032` → `0x0028`) | **ok despite an error**: the set returned `0xC0262582` 3 ms in, the read-back found 40, the OSD showed 40 |
| 2026-11-02 05:08:45 | `0x12` | 40 → 50 | **lost**: the set and both read-backs failed; the OSD stayed at 40 |
| 2026-11-02 05:09:48 | `0x12` | 40 → 50 | **ok**: one set, no error, the OSD showed 50 |
| 2026-11-02 05:26:16 | `0x14` | `0x05` → `0x04` (Warm → Cool) | **ignored**: the first set returned `0xC0262582`, the read-back found `0x05`, a second set went through **without an error**; `0x14` still read `0x05`, `0x0C` 6500 K, and the OSD stayed on Warm |
| 2026-11-02 05:42:16 and 05:42:56 | `0x12` | 50 → 40 → 50 | **ok**: one set each, no error, OSD 40 then 50 |

- **Brightness and contrast are writable** and take effect at once (OBSERVED).
- **The colour preset is effectively read-only.** A clean set of `0x04` to `0x14` was accepted on the bus and had no
  effect (OBSERVED). Whether other values, or `0x0C` (colour temperature request), are honoured is UNKNOWN: `0x0C`
  writes were not tried.
- **Gains, input, volume and power** were not written (UNKNOWN).
- **The 05:08 to 05:26 failures** were all sets that followed the previous command with almost no idle time (section
  2.2). With a 50 ms gap from the end of each command, every write since went through on the first set (OBSERVED
  2026-11-02 05:42).
- Whether a written brightness or contrast survives the monitor being switched off at its power button is UNKNOWN
  (`SaveCurrentSettings` was never called).

## 6. Sequences

### 6.1 Reading

1. Find the physical monitor (section 2), accepting handle 0.
2. Identify it by the EDID: manufacturer `MSI`, product `0x1462`, name `Optix G27C2` (`MSI1462` alone may cover other
   MSI models, INFERRED: the product code looks like MSI's vendor-wide number).
3. Do not wait for a capability string; read the codes of section 4.1.
4. For each read: on `0xC0262582` wait about 100 ms and retry once; on `0xC0262584` drop the code.
5. Keep at least 50 ms between the end of one command and the start of the next.

### 6.2 Writing

1. Read the current value (it may differ from what the host last saw: the OSD may have changed it).
2. Set the new value.
3. Wait about 50 ms (longer, about 200 ms, after an error on the set), then read back; try the read up to three
   times.
4. If the read-back shows the old value, or the set and every read-back failed, send the set once more and read
   back again. The value is absolute, so a repeated set is harmless.
5. Report "unknown" rather than "failed" when no read-back answered: the set may have landed (section 2.2).

This is what worked on the bench from 2026-11-02 05:42 on (OBSERVED).

## 7. What cannot be done

- **No capability string**, so no list of supported codes from the monitor (section 3).
- **No colour-preset write** (section 5).
- **No USB**, so no other control path (section 1.1).
- **No vendor software** was used to control the monitor in this work, and nothing here depends on any.

## 8. Hazards

| Hazard | Guard |
|---|---|
| Blind VCP scans | Read only the codes in section 4; scanning unknown codes can change settings or stop DDC/CI on some monitors (**PUBLISHED**, Microsoft PowerToys) |
| `SaveCurrentSettings` writes the monitor's non-volatile memory and takes about 200 ms (**PUBLISHED**, Microsoft) | Do not call it routinely; it was never called in this work |
| Frequent writes (for example brightness following ambient light) on a link that fails one command in four | Rate-limit writes (the bench wrote seconds apart, never in bursts); read back every write |
| Input switching (`0x60`) with MSI's own numbering | Untried; a wrong value can leave the screen on an empty input. Write only values read back from the monitor while on that input |
| Power mode (`0xD6`) writes | Untried; could switch the screen off with no way back through DDC/CI (INFERRED). Do not send without a physical recovery plan |
| Treating handle 0 as invalid | It is the G27C2's valid handle (section 2) |
| Running the host as a Windows service | Session 0 sees no monitor (section 2.1); use a helper in the user session |
| Two G27C2s on one PC | The EDID serials are placeholders; key by display path, not serial |

## 9. Test vectors

At the API level (`GetVCPFeatureAndVCPFeatureReply` returns type, current, maximum; values are the words of
section 4):

| Id | Input | Expected |
|---|---|---|
| T1 | Get `0x10` | set-parameter, current `0x005A` (90), maximum `0x0064` (100) (OBSERVED 2026-11-01 18:58) |
| T2 | Get `0x12` | set-parameter, `0x0032` of `0x0064` (OBSERVED) |
| T3 | Get `0x0B`, `0x0C` in Warm | `0x0032`; `0x0046` → 3000 + 70 × 50 = 6500 K |
| T4 | Get `0x0C` in Cool | `0x007E` → 3000 + 126 × 50 = 9300 K |
| T5 | `0x0C` maximum | `0x00AA` → 3000 + 170 × 50 = 11500 K |
| T6 | Get `0x14` | Cool `0x04`, Warm `0x05`, User `0x06`; never name `0x04` "5000 K" |
| T7 | Get `0x60` on DisplayPort | `3`; never name it "DVI 1" |
| T8 | Get `0xDF` | `0x0201` → MCCS 2.1 |
| T9 | Get `0xEF` | fails with `0xC0262584` (not supported): drop, do not retry |
| T10 | `GetCapabilitiesStringLength` | fails with `0xC0262582` on every try: fall back to section 4.1 |
| T11 | Set `0x10` to `0x0050`, read back | `0x0050` (OBSERVED 2026-11-01 21:12:46) |
| T12 | Set `0x14` to `0x0004`, read back | still `0x0005`: report "not taken" (OBSERVED 2026-11-02 05:26) |
| T13 | EDID bytes 8-11 | `36 69 62 14` → `MSI`, product `0x1462` |
| T14 | EDID manufacturer decoding | `0x3669` = `0 01101 10011 01001` → 13 `M`, 19 `S`, 9 `I` |
| T15 | Error code as a signed 32-bit value | `0xC0262582` = −1071241854; `0xC0262584` = −1071241852 |

## 10. Open items

| # | Question | How to settle |
|---|---|---|
| 1 | Input numbers for the two HDMI ports | read `0x60` with the PC connected through each HDMI port |
| 2 | Whether `0x0C` (colour temperature request) or other `0x14` values can be written | one write each, owner watching the OSD |
| 3 | Whether the gains `0x16`/`0x18`/`0x1A` can be written in the User preset | as item 2 |
| 4 | What `0x6C`/`0x6E`/`0x70`, `0xE1`, `0xFF` and the 32 codes of section 4.4 control (`0xDC`, `0x13`, `0x17`, `0x22`-`0x29`, `0x51`, `0x55`-`0x57` first) | change OSD items one at a time (Game Mode, Response Time, Zero Latency, Refresh Rate display, Alarm Clock, FreeSync, Pro Mode, Eye Saver, HDCR, Image Enhancement, Sharpness, Screen Size, Language, Transparency, OSD Time Out, Power Button, Navi Key) and re-read the 47 codes |
| 5 | ~~The codes that gave an I2C error in the `0xE0`-`0xFF` scan~~ **settled 2026-11-04** (section 4.4): all "not supported" except `0xDC`; `0x15` and `0x45` are the only codes without a clean answer | |
| 6 | Whether written brightness and contrast survive a power-button off and on | write, power-cycle the monitor, read |
| 7 | Whether the monitor answers DDC/CI while another input is active | switch inputs in the OSD and read |
| 8 | Volume (`0x62`) and mute behaviour with headphones | write with headphones connected |
| 9 | Why the capability request always fails while reads work | an I2C-level trace (Linux i2c-dev with ddcutil) |
| 10 | Whether the late `0x14` update after an OSD change is tied to closing the menu | read continuously while changing the preset |
