# MSI Optix G27C2: DDC/CI protocol reference

An independent description of how a host controls the MSI Optix G27C2 monitor (1920 × 1080, up to 144 Hz): the
VESA DDC/CI codes it answers, how MSI numbers its colour presets and inputs, how reliable its I2C link is, which
writes it takes, and what its EDID says. The monitor has no USB connection, so DDC/CI over the display cable is the
only control path.

| Part | How the host reaches it | Document |
|---|---|---|
| DDC/CI (MCCS VCP codes), EDID | Windows' low-level monitor configuration API (`dxva2.dll`) over DisplayPort | [msi-optix-g27c2-ddc.md](msi-optix-g27c2-ddc.md) |

Coverage per code and change history: [STATUS.md](STATUS.md).

## Why this exists

Nothing about this model's DDC/CI behaviour was public: no capability string, no input table, no list of working
codes, no failure report. The monitor makes that gap matter, because it never returns a capability string, so a
program cannot discover what it supports. From 2026-11-01 to 2026-11-04 the monitor was read and written by
independently written code on one PC, with the owner changing one OSD setting at a time and watching the screen for
every write. This reference writes down what it answered, what it ignored, and what is still unknown.

## Scope

- DDC/CI through Windows: the API calls, the VCP codes and their values, error codes, timing and reliability,
  writes, and the EDID.
- Not covered: picture quality, firmware, and any MSI software (none was used).
- One unit was observed (built week 26 of 2019, connected over DisplayPort). Other firmware revisions may differ.

## How to read the tags

Every statement carries one of these tags.

| Tag | Meaning |
|---|---|
| **OBSERVED** | seen on this unit through Windows' DDC/CI API by independently written third-party code, with the date; includes the owner's report of what the OSD and the screen showed |
| **HOST** | read on the PC without talking to the monitor: Windows' device properties, driver binding and the copy of the EDID Windows keeps in the registry |
| **PUBLISHED** | from VESA's MCCS standard, Microsoft's documentation, or an open-source project, which is named |
| **INFERRED** | a conclusion drawn from the evidence, not read from any source; the reasoning is given |
| **UNKNOWN** | a gap, with a note on what would close it |

There was no bus capture: DDC/CI runs over I2C inside the display cable, which a USB capture cannot see.

## Sessions

| Date | What was done |
|---|---|
| 2026-11-01 18:57 | read-only probe of both monitors on the PC: the capability request (failed three times) and brightness and contrast reads |
| 2026-11-01 19:47 | the same reads from the user session and from session 0 (a Windows service's context): no monitor visible in session 0 |
| 2026-11-01 about 20:45 | first full read of the standard codes; handle value 0; input number on DisplayPort |
| 2026-11-01 20:56 to 21:14 | first writes: brightness 90 → 80 (failed, then worked with command pacing) and back |
| 2026-11-02 05:08 to 05:10 | contrast 50 → 40 → 50, with I2C errors that showed the pacing problem |
| 2026-11-02 05:16 to 05:24 | colour presets mapped: Cool, Warm, User and the R gain, read before and after each OSD change; the `0xE0`-`0xFF` range scanned once |
| 2026-11-02 05:26 | colour-preset write: accepted on the bus, ignored by the monitor |
| 2026-11-02 05:42 | contrast 50 → 40 → 50 again with corrected pacing: no errors |
| 2026-11-02 to 2026-11-04 | reads every few seconds during normal use, for reliability |

## Sources

1. The monitor's own answers through Windows' DDC/CI API, read and written by independently written code on the
   owner's PC, with the owner reporting the OSD and the screen.
2. Windows' device properties and stored EDID for the monitor.
3. VESA's Monitor Control Command Set (MCCS), Microsoft's documentation of the low-level monitor configuration
   functions, and the projects and documents listed under Credits.

No MSI software, firmware or source code was used or seen.

## Credits

- **VESA** publishes DDC/CI and the Monitor Control Command Set, which define the standard VCP codes used here.
- **Microsoft** documents the low-level monitor configuration API ("Using the Low-Level Monitor Configuration
  Functions", `GetVCPFeatureAndVCPFeatureReply`, `CapabilitiesRequestAndCapabilitiesReply`, `SaveCurrentSettings`)
  and, in PowerToys' Power Display documentation, the risks of scanning VCP codes blindly.
- **[ddcutil](https://www.ddcutil.com/)** documents DDC/CI timing and why I2C needs retries (its FAQ and
  performance-options pages).
- **MSI** is the maker of the monitor. Product names are MSI's.

Nothing from these sources is copied here; only the facts are used, with credit.

## Related references

The same PC's other devices are documented in the same way, each in its own repository:

- [msi-meg-z790-ace-protocol](https://github.com/immortalbob/msi-meg-z790-ace-protocol): MSI MEG Z790 ACE motherboard: Mystic Light, the NCT6687D fan and sensor controller, MSI software and BIOS
- [msi-coreliquid-s360-protocol](https://github.com/immortalbob/msi-coreliquid-s360-protocol): MSI MEG CoreLiquid S360 cooler: controller and LCD
- [lian-li-uni-fan-sl-v2-protocol](https://github.com/immortalbob/lian-li-uni-fan-sl-v2-protocol): Lian Li UNI FAN SL V2 and the UNI HUB SL V2 controller
- [lian-li-universal-screen-protocol](https://github.com/immortalbob/lian-li-universal-screen-protocol): Lian Li Universal Screen 8.8": LCD and LED ring
- [razer-peripherals-protocol](https://github.com/immortalbob/razer-peripherals-protocol): Razer DeathAdder V2, Huntsman V2, Kraken V3 Pro and Synapse 3
- [asus-xg34wcdg-protocol](https://github.com/immortalbob/asus-xg34wcdg-protocol): ASUS ROG Strix OLED XG34WCDG monitor: DDC/CI and the Aura USB interface
- [msi-rtx4090-gaming-x-trio-protocol](https://github.com/immortalbob/msi-rtx4090-gaming-x-trio-protocol): MSI GeForce RTX 4090 Gaming X Trio: NVML and NVAPI control on Windows

## Licence

The documents in this folder are released under the **Creative Commons Attribution 4.0 International licence
(CC BY 4.0)**: use, copy, translate and build on them for any purpose, with credit to this work. See `LICENSE`.
No scripts are published with this reference.

## How to check your own unit

On Windows, any tool that uses the `dxva2.dll` monitor functions can repeat these reads, for example NirSoft
ControlMyMonitor or a few lines of code around `GetVCPFeatureAndVCPFeatureReply`. On Linux, `ddcutil getvcp 10` and
friends do the same over i2c-dev.

1. Read, do not write, until you know what the codes do on your unit.
2. Read the codes of the document's section 4.1 only; do not scan blindly.
3. Expect about one read in four to fail with an I2C error: retry after about 100 ms, and keep 50 ms of idle time
   between the end of one command and the start of the next.
4. To map an OSD setting, read every code, change one OSD item, close the menu, wait a few seconds, and read again.

Contributions especially welcome: the input numbers of the HDMI ports, whether `0x0C` or the RGB gains can be
written, and what `0x6C`/`0x6E`/`0x70`, `0xE1` and `0xFF` do. See [STATUS.md](STATUS.md).

## Safety

DDC/CI writes change the monitor's settings directly. Apart from the writes listed in the document, nothing here was
tested; anyone sending commands to their own monitor does so at their own risk. Do not write the power mode or the
input source without a way to recover at the monitor's own buttons, and do not call `SaveCurrentSettings` routinely.
