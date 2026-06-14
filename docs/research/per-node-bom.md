# Per-node bill of materials (2–6S charging node)

**2026-06-14.** Complete per-node BOM, grouped by block, with concrete example
parts and the open items to finalize before layout. Architecture is fixed:
**IP2366** (PD sink + buck-boost) + **BQ76907** (2–6S sense/balance/protect) +
**ESP32-C3** supervisor. See [`single-ic-feasibility.md`](single-ic-feasibility.md)
and [`cell-balancer.md`](cell-balancer.md) for the why. Tags: **FACT** =
datasheet/LCSC sourced; **UNVERIFIED** = needs confirm/sim. Quantities are per
node unless noted. Reference values are derated for this **≤3 A / ≤65 W** node —
the IP2366 reference design targets 140 W and is intentionally oversized.

## Block 1 — PD sink + buck-boost charge stage (IP2366)

| Ref | Qty | Part / value | LCSC | Note |
|---|---|---|---|---|
| U1 | 1 | **IP2366** (I²C ver.) QFN-40 5×5 | C20415848 | PD3.1 sink + 4-switch single-inductor buck-boost. **FACT** ~$0.96@100. **OUT OF STOCK at LCSC** → JLCPCB consignment / Injoinic reels. Internal gate drivers, no external driver IC. |
| Q1–Q4 | 4 | N-MOSFET, **40 V**, logic-level (Rds(on) spec @Vgs=4.5 V) | **AP40P04G C5443713** (40 V, PDFN5×6, stock ~34.6k, ~$0.22) | H-bridge (HG1/LG1 input leg, HG2/LG2 battery leg). 40 V needed: 6S=25.2 V + PD3.1 VBUS up to 28 V + overshoot → 30 V too tight. Reference part AER4051AE is not on LCSC. |
| Q5 | 1 | N-MOSFET, 40 V, input path | same class as Q1–Q4 | VBUS-path NMOS driven by VBUSG (reverse-block / input disconnect). Must carry full input current (~3 A). |
| L1 | 1 | **22 µH**, Isat ≥6–8 A, low DCR | see open items | Single buck-boost inductor (LX1↔LX2). Reference 22 µH/15A is oversized; need 6–8 A for 3 A worst-case. Many cheap 6×6 22 µH parts are only ~2 A — **verify Isat**. |
| R_sns1, R_sns2 | 2 | **5 mΩ** 1% 1206 alloy current-sense (low TCR) | **MA120610FR005MZ C252653** (stock ~4.9k, ~$0.078) | Input sense (CSP1/CSN1) + battery sense (CSP2/CSN2). At 3 A dissipates 45 mW. Match the value firmware expects. **FACT** matches reference R2/R4. |
| C_bulk_in/out | 4 | **22 µF** 1210 X7R — **50 V** VBUS side, 35 V battery side | LCSC basic 22 µF 50V/35V 1210 | DC-bias: a 35 V X7R loses >50% C near 28 V → use 50 V on VBUS side. |
| C_res | 2 | **100 µF 35 V** Al-polymer / solid | e.g. C44606614 | Low-ESR bulk for 250 kHz ripple. Could drop to 1× for the derated node if ripple sim allows (**UNVERIFIED**). |
| C_bst | 2 | 470 nF (or 100 nF) | LCSC basic | Bootstrap caps on BST1/BST2 (internal bootstrap, no external diode). |
| C_dec | ~4 | 100 nF 50 V 0603 | LCSC basic | VCC5V / VCCIO / VIO decoupling + general. |
| R_cfg | 3 | BAT_NUM, PSET, VSET resistors | — | **FACT**: BAT_NUM sets 2S..6S (e.g. 3.6k=2S … 18k/27k=6S); PSET sets power limit (~9.1k≈65 W); VSET sets per-cell full V (3.65/4.1/4.2/4.35/4.4 V). |

## Block 2 — USB-C input + input protection

| Ref | Qty | Part / value | LCSC | Note |
|---|---|---|---|---|
| J1 | 1 | **USB-C receptacle**, 16-pin, 20 V/5 A | **TYPE-C-31-M-12 C165948** (~$0.10) | CC1/CC2 to IP2366 CC pins. IP2366 handles sink CC on-die — **verify** whether external 5.1k Rd pulldowns are still required (datasheet/register). |
| D_esd | 1 | **USBLC6-2SC6** ESD array, SOT-23-6 | C111212 (ST) / C5180279 | ESD on CC1/CC2 + D+/D−. **FACT** 5A/5.25 V/~3.5 pF. |
| D_tvs | 1 | VBUS TVS, bidir | **SMBJ28A** (28 V standoff) — see note | Reference candidate SMBJ24CA clamps ~38.9 V; a 28 V-standoff part clamps tighter vs EPR 28 V. **Verify clamp < IP2366 VBUS abs-max** before lock. |

## Block 3 — Cell sense + balancing + protection (BQ76907 AFE)

| Ref | Qty | Part / value | LCSC | Note |
|---|---|---|---|---|
| U2 | 1 | **BQ76907RGRR** 2S–7S AFE, VQFN-20 3.5×3.5 | **C22458649** (~$1.46, stock ~992) | **FACT.** Per-cell 16-bit ADC, internal-FET passive balance ~35 mA, OV/UV/OCC/OCD/SCD + OT/UT, low-side CHG/DSG drivers, LDO, 400 kHz I²C. Covers full 2–6S. |
| R_in | up to 6 | **20 Ω** per-cell series input filter | LCSC basic 0603 | Sets balance current with internal FET (~35 mA @4.2 V). One per cell tap. |
| C_cell | up to 6 | 100 nF–1 µF per-cell filter | LCSC basic 0603 | Cell-input RC filter per BQ76907 app note. |
| Q_prot | 2 | CHG + DSG protection N-FET (low-side) | 40 V N-FET, e.g. same as Q1 class | Driven by BQ76907 CHG/DSG outputs for pack OC/SC/OV/UV cutoff. Size for 3 A. |
| RT1 | 1–2 | 10 kΩ NTC thermistor | LCSC basic | OT/UT sense. Place near cells/pack lead. |

## Block 4 — ESP32 supervisor + 3.3 V rail

| Ref | Qty | Part / value | LCSC | Note |
|---|---|---|---|---|
| U3 | 1 | **ESP32-C3-MINI-1-N4** | **C2838502** (~$2.10, stock ~8.1k) | **FACT.** RISC-V, Wi-Fi4 + BLE5 + ESP-NOW. C6 (C5736265, ~$2.82, Wi-Fi6/Thread) only if committing to Matter/Thread — not needed for ESP-NOW; footprint **not** guaranteed drop-in. |
| U4 | 1 | **TPS560430X3FDBVR** 3.3 V buck, 4–36 V in, 600 mA, SOT-23-6 | **C2071721** (~$0.63) | **MANDATORY** — IP2366 VCCIO is only ~30 mA; ESP TX peak is 300–350 mA. Fixed 3.3 V (no FB divider). 36 V in covers 6S (25.2 V) with margin. Power from **BAT** (not just VBUS) so the node reports with no brick attached. Alt: LMR51430XDDCR (C5185863, 3 A, adjustable) if rail load grows. |
| L2, C_buck | — | buck inductor + in/out caps | per TPS560430 datasheet | Standard buck support passives. |
| C_esp | — | 10 µF + 100 nF | LCSC basic | ESP32 module decoupling. |

**Sleep budget (UNVERIFIED):** IP2366 standby = 5 µA, but the ESP dominates idle
drain. Use deep-sleep + periodic wake reporting (and consider a load switch) to
avoid draining a charged pack left plugged into a powered node.

## Block 5 — Output / connectors + output protection

| Ref | Qty | Part / value | LCSC | Note |
|---|---|---|---|---|
| J2 | 1 | **JST B7B-XH-A** 7-pin (6S balance+sense) | **C144398** (~$0.12, stock ~15k) | **FACT** 3 A/contact (derate ~2.4 A) — this is the source of the product's ≤3 A cap. N+1 pins for N-cell tap. Use B2B-XH-A (C144397) for 2-pin power-only variant. |
| J3 | (opt) | 2-pin XH power, or double-up J2 power pins | C144397 | At 3 A a single XH contact is at its limit → consider doubling power pins for headroom. |
| F1 | 1 | ~4 A chip fuse (or polyfuse) on pack lead | LCSC basic | IP2366 has internal OCP/short detect (~30 ms OCP, ~40 µs short); fuse is board-survival backstop. **Bench-test** IP2366 dead-short behaviour before downsizing/omitting. An eFuse IC is overkill for 3 A. |

## Manager-node delta

Identical board **plus a display** on the shared I²C bus:

| Part | LCSC | Note |
|---|---|---|
| **SSD1306 0.96″ OLED 128×64, I²C** | bare-glass C# **UNVERIFIED** | v1 manager: 2 pins, shares the IP2366/AFE I²C bus, 3.3 V native. Cheapest. |
| ST7789 1.3″ 240×240 IPS SPI LCD | bare-glass C# **UNVERIFIED** | Premium color variant; more pins (SPI). |

Resolve bare-panel LCSC part numbers + FPC connector if integrating the display
onto the PCB rather than using a plug-in module.

## Cost snapshot (1-off LCSC, system overhead beyond power stage)

ESP32-C3 ~$2.10 · TPS560430 ~$0.63 · BQ76907 ~$1.46 · IP2366 ~$0.96@100 ·
USBLC6 ~$0.10 · TVS ~$0.03 · B7B-XH-A ~$0.12 · fuse <$0.05 — ICs dominate; the
per-node total lands in the "simple/elegant/cheap" target with a handful of passives.

## Open items to finalize

1. **IP2366 I²C register map is NDA-only.** Community IP2368 lib (addr 0x75;
   Vbat 0x50/0x51; Ibat 0x6E/0x6F; charge-stop SYS_CTL8=0x08; power/current
   SYS_CTL3=0x03) is for the **sibling** and may differ. Request the official
   IP2366 register doc + bench-verify ADC scaling before firmware. **Biggest risk**
   for the IR→current loop. (github.com/D-314/IP2368-Arduino-Library)
2. **MOSFET Rds(on) @Vgs=4.5 V** for AP40P04G (and any alt) — confirm logic-level
   Rds(on) and Vds≥40 V before locking the H-bridge. Decide single 40 V line vs
   split (40 V VBUS side / 30 V battery side).
3. **L1 Isat** — ripple/RMS sim across 2–6S and VBUS 5–28 V to size Isat (likely
   6–8 A vs reference 15 A); evaluate 10 µH for a smaller 6×6 high-current part.
4. **USB-C CC pulldowns** — confirm whether external 5.1k Rd is needed or on-die.
5. **VBUS TVS clamp** vs IP2366 VBUS abs-max — pick SMBJ28A vs SMBJ24CA after
   reading the abs-max.
6. **VBUS-path FET (Q5)** count/spec — confirm it's a 5th external N-FET, 40 V,
   carries full input current.
7. **22 µF bulk DC-bias** at 28 V — use 50 V VBUS-side; confirm IP2366 min input
   cap requirement.
8. **PSET value for 65 W** (~9.1k, table extraction was garbled) — confirm on a
   clean datasheet copy; or set via I²C.
9. **BQ76907 balance current** vs 20 Ω filter R — confirm against datasheet.
10. **Output fuse vs polyfuse** final call after IP2366 short-circuit bench test.
11. **Manager display** bare-glass LCSC C# + FPC if PCB-integrated.

## Sources

- IP2366 datasheet V1.20 — https://www.injoinic.com/api//static/uploads/20250529/20250529120342_6837dc9e98b68.pdf
- IP2366 LCSC — https://www.lcsc.com/product-detail/C20415848.html
- BQ76907 — https://www.ti.com/product/BQ76907
- ESP32-C3-MINI-1 LCSC — https://www.lcsc.com/product-detail/C2838502.html
- TPS560430 LCSC — https://www.lcsc.com/product-detail/C2071721.html
- B7B-XH-A LCSC — https://www.lcsc.com/product-detail/C144398.html
- USBLC6-2SC6 LCSC — https://www.lcsc.com/product-detail/C111212.html
