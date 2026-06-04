# Boosty ⚡

**Need 5V at up to 5A from 3.7V batteries? You need a Boosty!**

<img width="1077" height="445" alt="Screenshot 2026-06-04 at 9 20 42 PM" src="https://github.com/user-attachments/assets/0458f440-bd75-407f-a729-06d569e7f5d9" />


A super accessible, pill-shaped boost converter breakout board for anyone who needs a boost. Designed by [Absurd Industries](https://absurdindustries.in) - open power electronics for everyone!

Boosty takes a single-cell lithium battery (3.0–4.2V) and boosts it to a rock-solid 5V rail capable of delivering serious current. Born out of the need to power 200 WS2812B LEDs on the [Supersaber](https://github.com/Absurd-Industries/supersaber) without praying to the thermal gods.

## Specs

| Parameter | Value |
|-----------|-------|
| Input voltage | 3.0–4.2V (single-cell Li-ion / LiPo) |
| Output voltage | 5.0V regulated |
| Max output current | 5A continuous |
| Peak switch current | 10A (with cycle-by-cycle protection) |
| Efficiency | ~90% typical at 2A load |
| Switching frequency | ~600 kHz |
| Light-load mode | PFM (auto power-save) |
| Protections | OVP, OCP, thermal shutdown, UVLO |
| Board shape | Pill-shaped breakout |
| Converter IC | TPS61088RHLT (Texas Instruments) |
| License | CERN-OHL-S v2 (copyleft) |

## When to use Boosty

- Powering addressable LED strips (WS2812B, SK6812, APA102) from a single Li-ion cell
- Portable USB power banks and charging circuits
- Raspberry Pi / SBC projects running on battery
- Any project that needs 5V and more current than those tiny SOT-23 boost converters can handle
- When your existing boost converter is running at 2.5x its rated current and you've been getting away with it (you know who you are)

## Schematic

Based on the TPS61088 typical application circuit adapted for 5V output from a single-cell lithium battery.

### Bill of materials

| Ref | Value | Package | Description |
|-----|-------|---------|-------------|
| U1 | TPS61088RHLT | VQFN-20 | Synchronous boost converter |
| L1 | 2.2 µH | — | Power inductor, 12A+ saturation (Cyntec PIMB065T-2R2MS or equivalent) |
| C1, C2, C3 | 22 µF | 0805/1206 | Ceramic X5R, 10V min — input bulk capacitors |
| C4, C5, C6 | 22 µF | 0805/1206 | Ceramic X5R, 10V min — output bulk capacitors |
| C7 | 0.1 µF | 0402/0603 | Ceramic — VIN pin bypass (place closest to pin) |
| C8 | 2.2 µF | 0402/0603 | Ceramic — VCC internal LDO cap |
| C9 | 0.1 µF | 0402/0603 | Ceramic — BOOT cap (BOOT to SW) |
| C10 | 47 nF | 0402/0603 | Ceramic — soft-start cap |
| C11 | 4.7 nF | 0402/0603 | Ceramic C0G — compensation cap |
| R1 | 178 kΩ | 0402/0603 | 1% — FB divider top (sets VOUT = 5.03V) |
| R2 | 56 kΩ | 0402/0603 | 1% — FB divider bottom |
| R3 | 270 kΩ | 0402/0603 | 1% — switching frequency set (~600 kHz) |
| R4 | 100 kΩ | 0402/0603 | 1% — current limit set (11.9A peak) |
| R5 | 10 kΩ | 0402/0603 | 1% — compensation resistor |
| SW1 | — | — | SPST toggle/slide switch (EN to VBAT) |

### Key connections

- **EN** → SPST switch → VBAT (hardware power control; or tie directly to VBAT for always-on)
- **FB** → midpoint of R1/R2 divider from VOUT to GND
- **FSW** → R3 → SW node (not ground!)
- **ILIM** → R4 → AGND
- **COMP** → R5 + C11 → AGND
- **SS** → C10 → GND
- **VCC** → C8 → GND
- **MODE** → float (PFM mode for light-load efficiency)
- **NC pins (11, 12)** → GND (thermal dissipation)
- **Thermal pad** → GND with thermal vias

## PCB layout notes

Layout matters at these currents. Follow TI's guidelines from the TPS61088 datasheet (SLVSCM8D), Section 10:

- Keep SW traces **short and wide** — this is the high-frequency switching node
- Place input caps (C1–C3) as close to VIN pin as possible
- Place output caps (C4–C6) as close to VOUT pins as possible
- C7 (0.1µF VIN bypass) should be the **closest** cap to the VIN pin
- Use a solid ground plane on the bottom layer
- Stitch the thermal pad to the ground plane with **multiple thermal vias** (at least 6–9 vias under the pad)
- Keep AGND and PGND connected at the IC but avoid routing noisy power currents through the analog ground path
- R1/R2 feedback divider should route cleanly to FB without crossing the SW node

## DC bias derating warning

Ceramic capacitors lose capacitance under DC bias. A 22µF 0805 X5R cap rated at 6.3V might only give you 12–15µF at 5V. Use **10V rated** capacitors (or higher) to maintain adequate effective capacitance. Check the manufacturer's DC bias curves for your specific caps.

## Design calculations

All calculations derived from the TPS61088 datasheet (SLVSCM8D, Rev D, August 2021).

**Output voltage** (Equation 7):
```
VOUT = VREF × (1 + R1/R2) = 1.204 × (1 + 178/56) = 5.03V
```

**Switching frequency** (Equation 5, at VIN = 3.6V):
```
RFREQ = 4 × (1/fSW - tDELAY × VOUT/VIN) / CFREQ ≈ 270 kΩ → ~600 kHz
```

**Current limit** (Equation 6, PFM mode):
```
ILIM = 1,190,000 / RILIM = 1,190,000 / 100,000 = 11.9A peak
```

**Soft-start time** (Equation 1):
```
tSS = VREF × CSS / ISS = 1.204 × 47nF / 5µA ≈ 11.3 ms
```

## Firmware power budgeting (optional but recommended)

If you're driving LEDs, consider adding a software current budget to protect the converter from accidental overload:

```python
MAX_MA = 4000  # safe continuous output budget

def safe_brightness(color, num_leds, max_ma):
    r, g, b = color
    ma_per_led = ((r / 255) + (g / 255) + (b / 255)) * 20
    total_at_full = ma_per_led * num_leds
    if total_at_full == 0:
        return 1.0
    return min(max_ma / total_at_full, 1.0)
```

This way your firmware automatically dims brighter colors to stay within the power budget. Open hardware with a seatbelt.

## What Boosty replaced

The Supersaber's original boost converter was a TPS61032 (4A switch, ~800mA rated output). It was being operated at roughly 2.5x its rated capacity through a combination of optimism and thermal luck. Boosty replaces that with a TPS61088 that has actual headroom — 10A switch current, synchronous rectification, and proper thermal management.

| | TPS61032 (old) | TPS61088 (Boosty) |
|-|----------------|-------------------|
| Switch current | 4A | 10A |
| Rated output @ 5V | ~800 mA | ~5A |
| Synchronous | No | Yes |
| Efficiency @ 2A | ~82% | ~90% |
| Running gold LEDs | 😓 redline | 😌 cruising |

## License

This project is licensed under **CERN-OHL-S v2** (CERN Open Hardware Licence, Strongly Reciprocal). You are free to use, modify, and distribute this design, provided that any modifications are shared under the same license.

## Contributing

Found a bug? Want to improve the layout? Have a better inductor recommendation? Open an issue or PR. This is community hardware — your contributions make it better for everyone.

## Credits

⚔️ Designed by [Absurd Industries](https://absurdindustries.in) — copyleft consumer electronics, repairable by design, documented for humans.
