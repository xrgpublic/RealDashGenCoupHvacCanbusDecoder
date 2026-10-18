# Genesis Coupe BK2 HVAC CAN Bus Decoder for RealDash

Display the factory HVAC information from a **Hyundai Genesis Coupe BK2** on a custom **RealDash** dashboard, including when replacing the factory blue LCD with a 7-inch Android tablet.

**Project:** [RealDashGenCoupHvacCanbusDecoder](https://github.com/xrgpublic/RealDashGenCoupHvacCanbusDecoder/tree/main)  
**Original CAN research:** [GrayBoltWolf's 2015 R-Spec reverse-engineering post](https://www.reddit.com/r/genesiscoupe/comments/cqhh4p/genesis_coupe_bk2_can_bus_hvac_display_codes/)  
**2013 implementation and walkthrough:** [HVAC Display — Replacing Blue LCD on Double DIN Swap](https://www.reddit.com/r/genesiscoupe/comments/1x803rw/comment/phbqmzb/)

> [!IMPORTANT]
> The **2013 2.0T R-Spec manual-climate profile** is based on testing in the 2013 car. The **2015 R-Spec automatic-climate codes** were documented by GrayBoltWolf and have **not been independently tested with this project's RealDash configuration**. Select the configuration that matches your car and confirm the raw readings before trusting the dashboard.

## What this does

This project decodes HVAC data from the CAN wiring serving the OEM blue center LCD and makes it available in RealDash. Depending on the vehicle and climate-control type, it includes:

- HVAC temperature setting
- Outside temperature
- Vent/air distribution mode
- Outside-air versus cabin-air recirculation
- A/C compressor command/status field
- Fan speed
- Automatic climate state (where provided by the 2015 codes)

This is a **display/decoder project**, not an HVAC control replacement. It does not send commands to operate the climate system. The blue-screen HVAC bus should not be confused with OBD-II vehicle data.

## Vehicle compatibility

| Vehicle | Climate system | Source | Status |
| --- | --- | --- | --- |
| **2013 Genesis Coupe 2.0T R-Spec (BK2)** | Manual climate control | Project author's car | Tested by project author |
| **2015 Genesis Coupe R-Spec (BK2)** | Automatic climate control | GrayBoltWolf's CAN research | Codes published; RealDash profile needs vehicle testing |
| Other BK2 trims/years | Varies | May share some frames | Not confirmed by this project |
| BK1 / 2010–2012 | Varies | Different hardware may apply | Not supported/verified |

The profile differences matter: the 2013 and 2015 versions do **not** use identical temperature and air-intake interpretations.

## Files

On the [GitHub repository](https://github.com/xrgpublic/RealDashGenCoupHvacCanbusDecoder/tree/main), look for these files named in the 2013 installation post:

| File | Purpose |
| --- | --- |
| `HVACdash_Production.rd` | Importable RealDash dashboard layout |
| `genesis_coupe_bk2_hvac_Final.xml` | RealDash CAN description / decoder from the 2013 setup |

If additional model-specific XML or `.rd` files are present, choose the files for **your model year and HVAC type**. File names and layout may change as the repository evolves; consult the GitHub file listing.

## Hardware

The reference 2013 installation uses:

- A 7-inch Android tablet capable of running [RealDash](https://realdash.net/)
- A **Waveshare USB-CAN-A** USB-to-CAN adapter
- A powered or power-capable USB hub / USB-A to USB-C adapter, as required by the tablet
- Appropriate connector terminals, crimping equipment, insulated automotive wiring, and secure mounting
- A connection to the vehicle's original blue LCD harness

The original installation used **611-2.2 male terminals** to interface with the existing female LCD connector. A **DJ7104Y** connector housing was proposed as a possible plug-and-play solution, but fitment was **not confirmed** in the referenced post. Do not assume it is the correct mating housing without checking dimensions and pin positions.

### Basic connection

```text
Genesis Coupe blue LCD harness
     |  CAN High / CAN Low (and reference as required by adapter)
     v
Waveshare USB-CAN-A
     |  USB
     v
USB hub / OTG adapter
     |  USB data
     v
Android tablet running RealDash
```

Power the tablet through a suitable, fused automotive USB power supply. Do not use the LCD connector's 12 V pins as USB power without a correctly rated converter.

### OEM LCD wire-color reference

The original 2015 research describes these wires at the LCD:

| Wire color (original research) | Signal |
| --- | --- |
| Black | Ground |
| Red | Battery positive |
| Yellow | Ignition-switched 12 V |
| Thin red, center | CAN High |
| Blue | CAN Low |
| Tan/pink | Dash illumination positive |
| Gray | Dash illumination negative |

**These are reported colors, not a verified connector cavity/pin-number diagram.** Wire colors and connector variants may differ. Confirm the correct pins using the appropriate factory wiring diagram and a multimeter before connecting anything. Never connect battery/ignition 12 V to a CAN signal or USB data line.

## Quick start: RealDash

1. Download the appropriate RealDash dashboard (`.rd`) and CAN description (`.xml`) from [the repository](https://github.com/xrgpublic/RealDashGenCoupHvacCanbusDecoder/tree/main).
2. Connect the vehicle's **CAN H / CAN L** wiring at the blue LCD harness to the correct terminals of the USB-CAN adapter. Keep the original harness intact if possible; insulate and secure all connections.
3. Connect the adapter to the Android tablet through the USB hub/OTG adapter. Ensure the tablet supports the adapter and that USB data works.
4. Open **RealDash** and load/import the supplied `.rd` dashboard.
5. In RealDash, go to **Garage → your vehicle → Dashboard → Add** in the connection section, and select **Waveshare USB-CAN-A**.
6. Set the USB adapter's serial-interface baud rate as required by its configured mode. **2,000,000 bps** was used in the project author's working setup.
7. Set **CAN Speed = 100 kbps** for the LCD-side bus used in the 2013 installation. This setting is **separate** from the USB/serial baud setting. If your interface is connected to a different CAN bus, its CAN bit rate may differ.
8. Choose **CAN DESCRIPTION FILE** and import the matching `.xml` decoder.
9. Back out to the dashboard, switch the factory HVAC fan, vents, A/C, intake mode and temperature settings, and check whether the displayed values respond.

> **Baud-rate pitfall:** `2,000,000 bps` is the Waveshare adapter's **USB/serial communication setting**, *not* the vehicle's CAN bit rate. The reference LCD connection uses **100,000 bits/s CAN speed**. These settings are not interchangeable. The 2015 source also discusses a separate 500 kbps higher-speed bus; do not assume the LCD harness is that bus.

## CAN frame overview

The original research identifies three relevant **11-bit CAN identifiers**. Each example below is eight data bytes long and expressed in hexadecimal:

| Frame ID | Description | Relevant byte positions (zero-based) |
| --- | --- | --- |
| `0x131` | Cabin temperature setting / temperature-control position | 2013: byte `0`; 2015: byte `2` |
| `0x531` | Outside temperature | byte `2` |
| `0x132` | Vent, intake, A/C and fan | bytes `0`, `1`, `2`, `7` |

For example, the published `0x132` structure is:

```text
CAN ID: 0x132
Bytes:  AA BB XX 00 00 00 00 YY
Index:   0  1  2  3  4  5  6  7

AA = Vent/distribution mode
BB = Air intake / recirculation mode
XX = A/C compressor field
YY = Fan speed
```

The other bytes are shown as placeholders here; do not assume they are always zero in every captured packet.

## 2013 R-Spec — manual climate control

These values are from the project author's **2013 2.0T R-Spec**. This configuration **does not have automatic temperature regulation**, so the displayed Fahrenheit temperature is a *visual approximation of the knob position*, not a measured or thermostatically controlled target temperature.

### Temperature setting — `0x131`, byte `0`

Raw position values are **decimal 1–17** (hex `0x01–0x11`):

```text
Display °F = 62 + (V - 1) * 1.75
```

| Raw value (decimal) | Hex | Dashboard display |
| ---: | :---: | ---: |
| 1 | `01` | 62°F (coldest) |
| 9 | `09` | 76°F (middle) |
| 17 | `11` | 90°F (hottest) |

Intermediate values are linearly interpolated and may be fractional. **These are synthetic display values; do not interpret them as the actual cabin temperature.**

### Air recirculation — `0x132`, byte `1`

| Hex | Decimal | Meaning |
| --- | ---: | --- |
| `03` | 3 | Outside air / recirculation **Off** |
| `07` | 7 | Cabin recirculation **On** |

### Shared vent modes — `0x132`, byte `0`

| Hex | Decimal | Meaning |
| --- | ---: | --- |
| `11` | 17 | Head only |
| `12` | 18 | Head and floor |
| `13` | 19 | Floor only |
| `14` | 20 | Defrost and floor |
| `15` | 21 | Defrost only |
| `16` | 22 | Defrost and AUTO (documented in 2015 source; availability on manual system not confirmed) |

### Shared A/C compressor field — `0x132`, byte `2`

| Hex | Decimal | Meaning |
| --- | ---: | --- |
| `0C` | 12 | Off |
| `0D` | 13 | On |

This is the reported HVAC CAN field; it should not be assumed to prove that the compressor clutch is physically engaged at every instant.

### Shared fan speed — `0x132`, byte `7`

| Hex | Speed |
| --- | ---: |
| `F0` | 0 |
| `F1` | 1 |
| `F2` | 2 |
| `F3` | 3 |
| `F4` | 4 |
| `F5` | 5 |
| `F6` | 6 |
| `F7` | 7 |
| `F8` | 8 |
| `F9` | 9 |

For these values, `fan speed = V - 240` (where `V` is the unsigned byte interpreted as decimal).

### Outside temperature — `0x531`, byte `2`

```text
Outside temperature (°F) = V - 42
```

Example: `0x7F` = 127 decimal → **85°F**.

## 2015 R-Spec — automatic climate control (legacy research)

This mapping comes from [GrayBoltWolf's 2015 R-Spec CAN documentation](https://www.reddit.com/r/genesiscoupe/comments/cqhh4p/genesis_coupe_bk2_can_bus_hvac_display_codes/). It is preserved here as a reference for a separate automatic-climate setup. **A corresponding RealDash profile has not been validated on a 2015 car by this project's author.**

### Set temperature — `0x131`, byte `2`

```text
Set temperature (°F) = V + 60
```

| Hex | Decimal | Setpoint |
| --- | ---: | --- |
| `02` | 2 | 62°F / LO |
| `0C` | 12 | 72°F |
| `1E` | 30 | 90°F / HI |

The original LCD displays **LO** and **HI** at the extremes. The Fahrenheit value is an HVAC setpoint, not a measurement of actual interior air temperature.

### Air intake and AUTO — `0x132`, byte `1`

| Hex | Decimal | Meaning |
| --- | ---: | --- |
| `00` | 0 | Outside air |
| `01` | 1 | Outside air + AUTO |
| `04` | 4 | Cabin recirculation |
| `05` | 5 | Cabin recirculation + AUTO |

The **2015 AUTO states should not be substituted for the 2013 manual-climate `03`/`07` intake values**.

### Other 2015 fields

The 2015 source identifies the same general frame/byte arrangements for vent mode (`0x132`, byte `0`), A/C (`0x132`, byte `2`), fan speed (`0x132`, byte `7`), and outside temperature (`0x531`, byte `2`). Use the tables in the 2013 section above for those published values, but validate behavior on your 2015 vehicle.

## RealDash XML example: 2013 manual climate

This is the core 2013 decoder as posted in the installation thread, with the raw fields included for diagnostics:

```xml
<?xml version="1.0" encoding="utf-8"?>
<RealDashCAN version="2">
  <frames>
    <frame id="0x131">
      <value name="HVAC Cabin Set Temperature" units="F" offset="0" length="1" conversion="62+(V-1)*1.75" />
      <value name="HVAC Cabin Temperature Raw" offset="0" length="1" />
    </frame>
    <frame id="0x531">
      <value name="HVAC Outside Temperature" units="F" offset="2" length="1" conversion="V-42" />
    </frame>
    <frame id="0x132">
      <value name="HVAC Vent Mode" offset="0" length="1" enum="17:Head Only,18:Head and Floor,19:Floor Only,20:Defrost and Floor,21:Defrost Only,22:Defrost and AUTO,#:Unknown" />
      <value name="HVAC AC Compressor Mode" offset="2" length="1" enum="12:Off,13:On,#:Unknown" />
      <value name="HVAC Air Recirculation" offset="1" length="1" enum="3:Off,7:On,#:Unknown" />
      <value name="HVAC Fan Speed" offset="7" length="1" conversion="V-240" />
      <value name="HVAC Vent Mode Raw" offset="0" length="1" />
      <value name="HVAC Air Recirculation Raw" offset="1" length="1" />
      <value name="HVAC AC Compressor Raw" offset="2" length="1" />
    </frame>
  </frames>
</RealDashCAN>
```

RealDash receives values as **decimal numbers** in its enum declarations, even when the vehicle's CAN bytes are written in hexadecimal in this README. For example, hexadecimal `0x0D` is decimal `13` and maps to **On**.

> [!NOTE]
> RealDash's display widget type matters. If an enum signal is blank in a text gauge, test its matching `Raw` signal with a numeric gauge first. Use the appropriate gauge/image-state configuration for showing a label or icon; not every text gauge automatically renders enum strings.

## Troubleshooting

| Problem | What to check |
| --- | --- |
| No CAN packets | Confirm USB permission, adapter recognition, wiring polarity, ground/reference requirements, CAN speed, and that the HVAC controls are active. Do not swap wires or probe live circuits blindly. |
| RealDash requires `2000000` baud | That may be the adapter's serial-interface mode. Keep it distinct from the **100 kbps CAN** bit rate. |
| Temperature is wrong | The 2013 manual system reads byte `0`; the 2015 automatic system reads byte `2`. They use different conversions. |
| Air recirculation shows wrong states | 2013 `03`/`07` versus 2015 `00`/`01`/`04`/`05`. |
| Raw gauges work, labeled gauges are blank | Confirm enum syntax and widget support. Test the `Raw` signal in a numeric widget and configure UI states separately if needed. |
| Fan speed looks like `240`–`249` | Apply the `V-240` conversion to byte `7`. |
| Vent icon doesn't change | Confirm you're reading `0x132` byte `0` and matching the expected decimal enumerations (`17`–`22`). |
| Outside temperature is implausible | Verify `0x531` byte `2` and apply the `V-42` conversion. |
| 2015 automatic states don't work | Collect and compare raw CAN frames; the 2015 integration remains unverified in this repository. |

For troubleshooting, start by proving that **raw numeric frame values change** when you operate the HVAC controls, then add conversions and icon/label mappings one at a time.

## Safety and limitations

- **Listen-only is recommended.** This project is intended to read/display vehicle data; do not transmit arbitrary CAN frames into a live vehicle.
- Disconnect vehicle power appropriately before altering harness wiring, and observe automotive electrical safety practices.
- The wire colors above are a reference from a community post, **not** a guaranteed pinout for every car.
- Use proper strain relief, insulation, fusing and automotive-rated power conversion.
- Secure the tablet and wiring so they cannot interfere with driving or vehicle controls.
- Firmware, model year, market and HVAC package may affect CAN messages. Verify on your vehicle.
- The displayed manual-climate Fahrenheit value is **estimated from knob position** and does not measure cabin air temperature.

## Contributing

Contributions are welcome, especially:

- Testing and correcting the **2015 R-Spec AUTO climate RealDash profile**
- Reporting verified CAN captures from other Genesis Coupe years/trims
- Clear connector pinout photographs or verified plug-and-play connector parts
- Better dashboard layouts, icons, and documentation
- Troubleshooting findings and corrections to enum behavior in RealDash

When submitting an issue or pull request, include the **model year, trim, manual/auto HVAC type, CAN adapter, CAN bit rate**, and a short example capture of the relevant CAN frames (without unrelated personal/vehicle data).

## Credits and references

- **GrayBoltWolf** — original [2015 Genesis Coupe BK2 HVAC CAN bus reverse engineering](https://www.reddit.com/r/genesiscoupe/comments/cqhh4p/genesis_coupe_bk2_can_bus_hvac_display_codes/). Their work established the initial frame IDs and mappings used as the starting point for this project.
- **Important_Half_534** — [2013 Genesis Coupe R-Spec RealDash implementation and installation walkthrough](https://www.reddit.com/r/genesiscoupe/comments/1x803rw/comment/phbqmzb/), including revised manual-HVAC codes and the RealDash XML.
- **Source files and updates:** [xrgpublic/RealDashGenCoupHvacCanbusDecoder](https://github.com/xrgpublic/RealDashGenCoupHvacCanbusDecoder/tree/main)
- **Software:** [RealDash](https://realdash.net/)

## License

Check the repository for an explicit `LICENSE` file. Until a license is published, **do not assume the repository is licensed for unrestricted redistribution or modification**. The original community research is credited above.
