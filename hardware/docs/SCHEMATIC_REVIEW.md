# OpenDrone Charger: Schematic Review (SLAVE + MANAGER)

Synthesized from per-IC audits, adversarial verifications, subcircuit analyses, and cross-cutting checks. Findings whose verification returned `confirmed=false` / `INVALID` were dropped (listed at the end). Severities reflect the verified `final_severity`.

---

## 1. Executive Summary & Readiness Verdict

### SLAVE (`hardware/`, ESP32-C3, 2-6S / ≤3A charge-only)
**Verdict: NOT FUNCTIONAL, do not fab. Schematic is a corrupted, mid-edit work-in-progress.**

The dominant root cause is an annotation/UUID-linkage break: the root sheet `Charger.kicad_sch` (UUID `a9adf4ae…`) no longer matches the cell-instance reference paths or the `.kicad_pro` `top_level_sheets` UUID (`fb88b6f9…`). KiCad emits an annotation-error warning on netlist export and collapses all six cell instances onto one duplicate reference set, falsely shorting TAP1…TAP6 / AIN1…AIN6 and BAL_EN1…6. This means **every SLAVE "missing connection" finding below is partly an artifact of the corruption and partly real**: the entire power stage (gate drive, FET drains, bootstrap, input sense, program pins, 3V3 rail) is unwired in the exported netlist. The board must be re-annotated/re-linked in the GUI and re-netlisted before any of it can be trusted. Independent of the corruption, several power-stage SEV1s and a USBLC6-on-PD-VBUS SEV1 are genuine design errors. The logic-domain core (ESP32-C3 straps, EN RC, I2C, prog header) and the cell sense/balance topology are correct.

### MANAGER (`hardware-manager/`, ESP32-S3, 2-8S / ~10A, USB-PD 240W EPR bidirectional)
**Verdict: NOT FUNCTIONAL as drawn, multiple genuine SEV1 design errors, but the netlist itself is clean (no corruption).**

The MANAGER schematic is correctly annotated and its BQ25758 charge core + power tree are largely well-designed. However it has a cluster of hard showstoppers: ESP32-S3 held in reset (EN pulled low), two TPS54202 EN pins driven 5V over abs-max, USBLC6 ESD array on the 48V EPR VBUS rail, TPS54160 PWRGD tied to 12V (2× abs-max), the 3V3 buck output islanded off the rail, the EPR blocking FET being a 2N7002, the ACUV/ACOV divider swapped+mis-valued, the TLA2528 ADDR floating, and the EEPROM I2C bus pulled up at 1 MΩ. All are fixable but every one blocks bring-up or risks part damage.

**Neither board is ready for fabrication.** SLAVE needs re-annotation first, then a power-stage rewire; MANAGER needs ~10 connection/value corrections.

---

## 2. SEV1: Showstoppers

### SLAVE

**S1. Cell hierarchical sheet annotation/UUID break → 6× duplicate refs, false net shorts**
Root `Charger.kicad_sch` UUID `a9adf4ae…` ≠ cell-instance paths / `.kicad_pro top_level_sheets` UUID `fb88b6f9…`. KiCad reports "schematic has annotation errors"; netlist export collapses cell1…cell6 to one ref set {R121,R122,R123,R124,R125,C105,D105,Q109,Q110}. Result: AIN1…AIN6 all resolve to `{C105.1,R121.2,R122.1}`, TAP1…6 share `R121.1/R123.1/D105.1/Q110.2`, BAL_EN1…6 share `R124.1`. The cell.kicad_sch content itself is correctly annotated: this is a broken root↔instance link, not a wiring bug.
*Fix:* In KiCad GUI, run Tools ▸ Annotate Schematic (reset/re-link instances) so cell paths point at root `a9adf4ae…`, reconcile `.kicad_pro top_level_sheets` UUID, re-export netlist, confirm 54 unique cell refs and no warning. **Re-run this entire SLAVE audit afterward**: the exported netlist and pin data are unreliable for the cell hierarchy until then.

**S2. BQ25758 (U2) entire gate-drive bus unconnected: converter cannot switch**
HIDRV1(27), LODRV1(25), BTST1(26), HIDRV2(19), BTST2(20), LODRV2(21) all UNCONNECTED; correspondingly Q2/Q3/Q4/Q5 gate (pin 4) all UNCONNECTED. SW1={C3.2,L1.1,Q2.S,Q3.D,U2.28}, SW2={C4.2,L1.2,Q4.S,Q5.D,U2.18} bridges exist but nothing drives them.
*Fix:* HIDRV1→Q2.G, LODRV1→Q3.G (buck/SW1); HIDRV2→Q4.G, LODRV2→Q5.G (boost/SW2), via series gate R (2.2-10Ω, optional given BQ's integrated 3.4Ω/1.0Ω drivers). Mirror MANAGER U6.

**S3. Bootstrap caps C3/C4 BTST-side floating**
C3={2:SW1}, C4={2:SW2}; pin 1 of each on no net; BTST1/BTST2 unconnected. High-side drivers have no supply even after S2.
*Fix:* C3.1→U2.26 (BTST1), C4.1→U2.20 (BTST2). 100nF acceptable.

**S4. Buck high-side FET Q2 drain floating: no input rail path**
Q2.5 (drain) UNCONNECTED. Mirror Q4.5=VBATX (output) is correct. Buck leg has no input feed even after gate drive is added.
*Fix:* Q2.5 → +BATT/VAC input bus (the node at U2.32/33, with R3 sense in series: see S5).

**S5. Input (adapter) current sense broken: R3 and R10 each have a floating terminal**
R3 (5mΩ) = {1:+BATT} only; pin 2 on no net → the sense resistor is not in the current path at all. R10 (ACN filter) pin 1 floating; ACN_F has no source-side connection. ACP/ACN see no differential (abs-max ±0.3V). Output sense (R4/R11/R12/SRP/SRN) is the correct mirror to copy.
*Fix:* Insert R3 in series in the +BATT/VAC bus; Kelvin R9→ACP to one R3 terminal and R10→ACN to the other, mirroring R4/R11/R12.

**S6. FSW_SYNC (pin 36) floating: switching frequency undefined**
Datasheet §6.3.3.5 explicitly forbids floating FSW_SYNC. **Correction to original finding:** the resistor is NOT missing, R5=100k (Func "R-FSW", LCSC C25741) exists and sets fSW=300kHz (Table 6-3), but R5 pin 1 is orphaned and U2.36 is on no net. The intended R5.1→FSW_SYNC tie was never wired.
*Fix:* Connect R5 pin 1 → U2.36. R5.2 already to GND. Do not add a new resistor.

**S7. 3.3V buck U7 (TPS560430) essentially unwired**
U7 has only VIN(5)/EN(4)=+BATT, GND(2). CB(1)/BST, FB(3), SW(6) all UNCONNECTED. L2.1 (SW-side inductor) floating; C38 (boot cap) both pins floating; R55.2/R56.1 (FB divider midpoint) floating.
*Fix:* U7.6(SW)→L2.1; U7.1(CB)→C38, C38 other side→SW node; U7.3(FB)→R55/R56 divider midpoint. **FB divider also mis-valued:** 100k/43.2k → 2.65V, not 3.3V. Re-value to 135k/43.2k (or datasheet 156.25k/50k) for 3.30V (Vfb=0.8V).

**S8. System rail `/main/3V3` has no source; buck output island `+3.3V` is separate**
`/main/3V3` (ESP32 U8.3, ADC U3, expander U4, comparator U6.5, all pull-ups) is all sinks. The buck's intended output net `+3.3V` (C36/C37, L2.2, R55.1, U1.5) is a distinct net and is itself non-functional (see S7). Nothing powers the MCU/logic.
*Fix:* Merge `+3.3V` into `/main/3V3` (single rail) AND complete U7 per S7.

**S9. USBLC6-2SC6 (D2) VBUS pin on raw USB-PD VBUS (up to 28V): 5V part, will be destroyed**
D2 is a 5V ESD array (VRWM 5V, VBR 6V min); its VBUS pin sits on `/main/USBC_VBUS` upstream of Q1. First PD negotiation above ~6V breaks down the internal VBUS clamp and destroys D2 / loads the adapter so PD can't hold.
*Fix:* Keep D2 on D+/D− data lines only; protect raw VBUS with a ≥33V TVS (mirror D3 SMBJ30A on +BATT). Do not tie the USBLC6 VBUS pin to the high-voltage PD rail.

### MANAGER

**M1. ESP32-S3 (U15) EN/CHIP_PU pulled to GND: held in permanent reset**
EN_ESP = {R70.1, SW1.1, SW1.2, U15.3}; R70 (10k) other end on GND = pull-DOWN; SW1 (reset btn) also to GND. No pull-up to 3V3, no RC cap. EN sits at 0V → chip never enables.
*Fix:* Move R70 to pull EN UP to 3V3 (10k), add ~1µF EN→GND, keep SW1 EN→GND as reset.

**M2. No firmware programming interface (UART0 dead + native USB consumed) → board unbringable**
**(Upgraded from SEV2 per verification.)** U0RXD (pin36) UNCONNECTED; U0TXD (pin37)→`/main/TX` is a single-node dead-end (no header/bridge). Native USB pins IO19/IO20 (module pins 13/14) are used for ENC_B/ENC_SW; USBC_DP/DN reach only J1 + the USBLC6 (D1), never U15. No USB-UART bridge or programming header anywhere. Zero path to flash or talk to the MCU.
*Fix:* Route U0TXD/U0RXD to a proper UART/programming header (or on-board USB-UART bridge) as a TX/RX pair, and/or free IO19/IO20 from the encoder and wire USBC_DP/DN to them for native-USB flashing.

**M3. ACUV/ACOV divider taps swapped AND mis-valued: input charger never enables**
Chain is VAC_CHG-R19(1M)-ACOV(35)-R21(2.0k)-ACUV(34)-R20(15.4k)-GND. Datasheet requires ACUV on the UPPER tap, ACOV lower. As built, the ACUV-pin voltage at VAC=12/20/48/60V = 0.182/0.303/0.727/0.908V: always below the 1.1V UVLO threshold → device permanently reads undervoltage and never starts. Even with taps swapped, these values give UVP=64.3V / OVP=79.3V (both outside the usable window).
*Fix:* Swap so ACUV=upper node (R19/R21 junction), ACOV=lower node (R21/R20 junction); **and** recompute RAC2/RAC3 for the real window (e.g. UVP ~9-18V, OVP ~55-58V), do not just swap.

**M4. EPR blocking FET Q30 is a 2N7002 (115mA / 5Ω) in the TPS26750 5V/3A source path**
TPD4S480 EPR_BLK_G drives Q30 between VBUS_LV and USBC_VBUS; the TPS26750 internal 5V path sources up to 3A through it. TI Table 7-2 requires N-FET VDS≥30V, VGS≥15V, RDS(on)≤10mΩ @3A. 2N7002 (5Ω, 115mA, 225mW) drops 2.7-15V and dissipates 8-45W → instant destruction. ~1000× under-spec.
*Fix:* Replace Q30 with a 30-60V, low-Vth, ≤10mΩ, ≥3A N-FET that reaches full enhancement by VGS≈5-10V (EPR_BLK_G provides only ~5-12V above source at ~4µA). Verify turn-on time vs Qg.

**M5. EEPROM/identity I2C pull-ups R5/R6/R7 = 1 MΩ: PD config bus non-functional**
**Corrected value:** MPN 0402WGF1004TCE = LCSC C26083 = **1 MΩ** (not 100k; "1004"=100×10⁴). TPS26750 I2Cc reads its PD/EPR config from EEPROM U2 at boot. At 1 MΩ with ~20-50pF bus cap, rise time ≈17-42µs vs the 300ns spec: SCL/SDA never reach VIH; controller can't load config → no PD/240W.
*Fix:* Change R5/R6 (and R7) to 2.2k-4.7k (match main bus R38/R39=4.7k).

**M6. TLA2528 ADC (U7) ADDR pin (11) floating: undefined I2C address**
ADDR is sampled at power-up via a resistor to DECAP/GND (abs-max 2.1V). Floating → undefined latched address; the cell-sense ADC may not enumerate. Address can be reprogrammed in software, but only after addressing it at its (undefined) power-up address.
*Fix:* Add address-select resistor per Table 2 (≤5% tol), e.g. R1=0Ω DECAP→ADDR for 0x17, or R2=0Ω/DNP to GND for 0x10. Confirm no collision with U6 (0x6B), U8/TCA9554A (0x38), TPS26750 (0x20-0x23).

**M7. TPS54160 (U12) PWRGD (pin 6) tied to 12V output: exceeds 6V abs-max**
PWRGD (open-drain, abs-max −0.3 to 6V) sits directly on `/main/12V` (Vout=12.0V from R88=140k/R89=10k) with no series R, no MCU connection. 2× abs-max stresses/destroys the pin's clamp, and it's functionally useless tied to its own monitored rail.
*Fix:* Disconnect from 12V. Pull PWRGD up to 3.3V via 10-100k to an ESP32-S3 GPIO, or leave NC.

**M8. TPS54202 EN pins (U13 & U14) tied to 12V: exceeds 7V EN abs-max**
Both EN (pin 5) on `/main/12V`. TPS54202 EN abs-max −0.3 to 7V; 12V is ~5V over → stresses/destroys the EN clamp.
*Fix:* Float both EN (internal pull-up enables the device), or use a VIN→EN/EN→GND divider keeping EN ≤7V at 12V VIN while setting a UVLO point. (Contrast U12, which correctly uses a dedicated EN divider.)

**M9. USBLC6-2SC6 (D1) VBUS pin on the 48V EPR VBUS rail: destroyed on high-V negotiation**
D1.5 on `/main/USBC_VBUS` (raw connector VBUS, also feeds TPD4S480, Q1 80V FET, C57 100V). USBLC6 is a 5V part; enumerates fine at 5V SPR, then the internal VBUS-GND TVS conducts continuously the instant any >6V PDO (9V PPS … 48V EPR) is requested → destroys D1 / clamps the rail.
*Fix:* Keep D1 on DP/DN data lines; protect raw EPR VBUS with a 30V+ working-voltage TVS (e.g. SMAJ33A) at the connector, or move D1's VBUS pin to a regulated ≤5V node.

**M10. MANAGER 3.3V buck (U14) output islanded on net RAIL3: entire 3V3 rail unpowered**
U14 SW_3V3→L4.1, L4.2→net **RAIL3** = {C45.1, L4.2} only. The actual 3V3 loads (U15, LCD, U7, U8, all pull-ups), bulk C46, and FB-top R92.1 are on `/main/3V3`. RAIL3 and `/main/3V3` are not connected → buck output floats and has no feedback path. (5V/12V bucks are wired correctly by contrast.)
*Fix:* Merge RAIL3 into `/main/3V3` (post-inductor node) so loads and the FB divider sense the output.

**M11. Battery ideal-diode (U9/Q27) is charge-only and BLOCKS the bidirectional USB-PD discharge path**
U9 LM74700 ANODE=VBAT, CATHODE=TAP8; Q27 source=VBAT, drain=PACK_FUSED. The ideal diode conducts only VBAT→pack (charging) and the reverse comparator (−11mV, <0.45µs) shuts the gate off for pack→VBAT (discharge); Q27 body diode also opposes. U9/Q27 is the SOLE series element between pack and VBAT. The MANAGER is spec'd bidirectional (charge + USB-PD discharge) and the BQ25758 supports reverse power: this topology prevents discharge. EN cannot defeat the reverse comparator.
*Fix:* For bidirectional operation use a commanded back-to-back FET pair (gate driver you control, not a self-deciding ideal-diode), or add a separately-controlled discharge FET in parallel. If the node is intended charge-only, correct the README/CLAUDE.md bidirectional claim. Confirm intent in DESIGN.md before changing hardware.

---

## 3. SEV2: Major

### SLAVE

**S-A. No pull-ups on INT/PG/STAT open-drain outputs (U2)**
INT_CHG={U2.3,U8.20}, PG_CHG={U2.6,U8.21}, STAT_CHG={U2.4,U8.6}, each 2-member, no pull-up. Open-drain can only sink; lines read stuck-low unless ESP internal pull-ups are enabled (fragile for IRQ/status).
*Fix:* Add 10kΩ to 3V3 on all three (or document ESP32-C3 internal pull-ups as mandatory). **Note:** the original CE/R52 sub-claim is dropped, CE (pin 7) is a logic input, not open-drain; R52=100k to 3V3 is an acceptable default-disable pull-up.

**S-B. IOUT (9) and IIN (10) floating: datasheet forbids; no hardware current limit**
Both UNCONNECTED. EN_IOUT_PIN/EN_IIN_PIN reset to 1 (enabled), so at POR both are live control inputs. A floating IIN can drift above VIH_ILIM_HIZ and latch the device into Hi-Z (no switching/charging) before I2C is up.
*Fix:* Add programming R to PGND sized for ≤3A out / desired input limit, or tie both to PGND and clear EN_IOUT_PIN/EN_IIN_PIN in firmware. Floating not allowed.

**S-C. HUSB238A VBUS pin (16) missing 1µF VBUS-rail cap**
`/main/USBC_VBUS` has no ceramic (only ESD/zener/P-FET/pull-up). Every reference figure shows 1µF from the VBUS rail to GND for OVP/UVP sensing and dead-battery energy storage. (VDD pin5 IS decoupled by C36+C37=20µF: that requirement is met; only the VBUS-rail cap is missing.)
*Fix:* Add 1µF ≥35V X7R from VBUS (U1.16/J3 VBUS) to GND near the IC; a 10µF input bulk is also advisable for the P-FET inrush.

**S-D. SLAVE charge inductor L1 (SRP6540-100M, 10µH) current-under-rated**
Isat=Irms=4A, DCR=78.5mΩ. Charging up to 6S (25.2/26.1V) at 3.03A from ≤20V USB-PD forces boost; inductor current = Iin can reach ~10A at 9V in and >4A even at 20V in → saturation + ~1.1W DCR self-heat.
*Fix:* Use an ≥8A-Isat / ≥6A-Irms 10µH low-DCR part (e.g. SRP1265A-class). Size for worst-case boost peak, not 3A output.

**S-E. SLAVE logic buck TPS560430 (600mA) marginal for ESP32-C3 Wi-Fi TX**
Only 3V3 source; ESP32-C3 Wi-Fi TX ~335mA continuous with higher peaks + ~30-50mA of other loads. TX burst can hit the 600mA current limit → brownout/reset.
*Fix:* Move to a ≥1A logic buck (MANAGER uses TPS54202 2A), or bench-verify Wi-Fi TX is throttled with no brownout. Keep generous 3V3 bulk.

**S-F. No output bulk capacitance on SLAVE converter output (+VBAT/VBATX)**
Only SRP/SRN sense caps; output has no ceramic bulk for ripple/loop stability. Input +BATT only has C35(10µF)+C1(1µF).
*Fix:* Add 2-4× 10-22µF/50V X7R/X7S on the output near R4/+VBAT and matching input bulk per BQ25758 layout recommendations.

**S-G. SLAVE HUSB238A unused control pins**
INT_N(11), FAULT/OUT2(13), DBG_N(6), FLGIN(14) floating; EN_HVDCP/OUT1(7)→`/main/HVDCP_DIS` single-pin dangling. INT_N open = no PD interrupt (OK only if firmware polls). FLGIN floating is a robustness issue (downgraded to SEV3 below); HVDCP_DIS dangling suggests a dropped strap.
*Fix:* Route INT_N to a GPIO (or pull-up if polled). Terminate EN_HVDCP/OUT1 per intended HVDCP config. Re-verify after S1 re-annotation.

**S-H. SLAVE ACOV/V20 divider mis-netted (R15 dangling)**
R15.2 on `/main/ACOV`, R15.1 on single-pin net `/main/V20`. Half-connected resistor; ACUV=+BATT is the datasheet "unused" tie (OK).
*Fix:* Complete the divider if ACOV programming is intended, else remove R15. Re-verify after re-annotation.

### MANAGER

**M-A. PD config-port I2C value error**: see M5 (SEV1; the same R5/R6/R7=1MΩ defect).

**M-B. No input bypass cap (ANODE→GND) on any LM74700 ORing input**
USBC_PPHV/XT60_IN/SOLAR_IN have only the VCAP charge-pump cap (C72/C73/C74, VCAP-to-ANODE), zero ANODE-to-GND. Datasheet requires ≥22nF. Degrades 20mV forward-regulation stability and removes HF surge filtering.
*Fix:* Add ≥22nF (ideally 0.1µF) X7R from each ANODE node to GND at the controller.

**M-C. XT60 and Solar inputs have no TVS / transient clamp**
Only D11 (SMDJ51A) on VBUS_IN (the OR'd output). Hot-plug on XT60/long solar cable produces inductive overshoot directly across U4/U5 ANODE and Q4/Q5 VDS (LM74700 ANODE-GND 65V; SFS08 VDS 80V).
*Fix:* Add a TVS at each input connector: XT60_IN (~40-43V standoff for 8S) and SOLAR_IN (sized to panel Voc), clamping below 65V/80V.

**M-D. VBUS_IN TVS (SMDJ51A) clamps at 82.4V: above LM74700 65V and SFS08 80V abs-max**
Under high-current surge the clamp sits above the parts it protects, and VBR(min) 56.7V is below the 60V max operating input → leakage/heating near 60V.
*Fix:* If the bus can reach 60V, use a coordinated lower-clamp TVS (standoff above max operating, clamp <65V). If PD-EPR-limited to 48V, a 48V-class TVS clamping <63V is appropriate.

**M-E. 5V buck (TPS54202, 2A) cannot supply USB-C 5V source via TPS26750 PP5V**
PP5V (U1.28/29) is an input that must deliver up to 3A to VBUS + 315mA VCONN when sourcing 5V. With L3=4.7µH, peak inductor current at 3A ≈3.6A > the 2.5A min current limit → hiccup/shutdown.
*Fix:* Either don't advertise a 5V source PDO drawing >~1.5A through PP5V (document the 5V budget), or replace the 5V buck with a ≥4A part and resize L3/output caps.

**M-F. TPS26750 / TPD4S480 decoupling shortfalls**
- CVBUS: VBUS_LV has only C59=0.1µF; datasheet needs ≥1µF (4.7µF nom) on U1.26/27. Add 4.7µF/25V.
- CLDO_3V3: C1=1µF vs ≥5µF min. Increase to ≥10µF/6.3V. (Also feeds TPD4S480 VPWR.)
- CLDO_1V5: C4=1µF vs ≥4.5µF min. Increase to ≥4.7µF/6.3V.
- CPP5V: 5V rail has 46µF vs 120µF cSrcBulkShared. Add bulk to ≥120µF.
- USBC_VBUS: only C57=0.1µF/100V on the 48V node, no bulk. Add several µF of ≥63V X7R near the connector.

**M-G. Solar ORing Q5/Q6: single LM74700 (U5) across a back-to-back pair; ANODE on a drain**
U5 drives Q5+Q6 (common-source at SOLAR_MID) from one gate; ANODE=SOLAR_IN (Q5 drain), so V(AK) spans both FETs. Unlike the correct single-FET USB-C (Q3) and XT60 (Q4) paths, this desensitizes the −11mV reverse trip and degrades light-load forward regulation; reverse current is still blocked (by the series body diodes), just less accurately. Doubles series RDS(on)/gate charge; the second FET adds no function for a unidirectional input-only OR path.
*Fix:* Delete Q6 (and SOLAR_MID), wire as Q3/Q4: single FET S=SOLAR_IN, D=VBUS_IN, U5 ANODE=SOLAR_IN, CATHODE=VBUS_IN. (Verification trimmed the reverse-blocking failure claim; functional today but degraded/wasteful.)

**M-H. LMV331 OVP comparator (U11) input common-mode exceeded at the trip point**
U11 on 3V3; VICR ceiling ~2.57-2.6V (VCC−0.7 over temp). OV divider R65=127k/R66=10k (×0.073), OV_REF (TL432 U10)=2.495V → trip at VBAT=34.2V with OV_SENSE=2.50V (right at the ceiling). During a real fault OV_SENSE keeps rising: 8S LiHV (34.8V)→2.54V, 40V→2.92V, at/above VICR, where the comparator output is undefined exactly when OVP must assert. No hysteresis.
*Fix:* Run U11 from the 5V rail (VICR→4.2V, covers the swing), or rescale divider+reference so OV_SENSE stays ≤~1.5V across the full fault range; add hysteresis (IN+→OUT).

**M-I. TPD4S480 FLT (short-to-VBUS) not routed to the controller** *(SEV3 in source, listed here for grouping with PD)*
PD_FLT={R73.1,U16.9} pulls up to 3V3 only, goes nowhere. Hardware OVP still protects silicon, but firmware gets no fault notification and can't force detach.
*Fix:* Route PD_FLT to a free TPS26750 GPIO or an ESP32-S3 input.

**M-J. MANAGER 5V-rail inductor L3 (SWPA4030 4.7µH, Isat ~2.6A) saturation margin thin**
Near 2A load with ripple, peak ≈2.3-2.4A vs ~2.6A Isat.
*Fix:* Confirm 5V load <2A or use a 4.7µH part rated ≥3.5A Isat.

**M-K. DRV_SUP at 12V near DRV_OVP**: see DRV_SUP item; verification downgraded to **SEV3** (worst-case peak ~12.55-12.6V vs 12.8V min OVP, positive but thin margin). Listed in SEV3 below.

---

## 4. SEV3: Minor / Robustness / EMC

### SLAVE
- **NTC divider off-center (board temp):** R53‖R54 = 5k top vs RT2 10k → ~2.2V at 25°C, biases ADC range; two parallel 10k almost certainly unintentional. Use a single 10k top (DNP one), or set/document the intended value.
- **U17 BAT-pin RC filter omitted** *(MANAGER: see note below)*. *(SLAVE has no U17; this is a MANAGER item: listed under MANAGER.)*
- **VAC cap C1 (1µF/25V X5R) under-rated:** +BATT reaches 25.2V (6S) / up to 28V EPR; 25V X5R loses most of its C near rated V. Use ≥50V 1µF (0603/0805).
- **REGN/DRV_SUP share one 4.7µF (C6):** TI app circuit shows a 4.7µF on each pin. Prefer separate caps at REGN and DRV_SUP given 4-FET gate-drive current.
- **MODE (pin 17) floating:** *(downgraded from SEV2)* open pin = >27kΩ bin = buck-boost auto-detect, which matches the populated power stage, likely benign, but not the datasheet-prescribed method. Fit an explicit ≥27.0kΩ ±1% resistor MODE→PGND for deterministic operation.
- **HUSB238A FLGIN (14) floating:** *(downgraded from SEV2)* GATE-disable is opt-in (EN_FAULTIN) and INT_N is unconnected, so no functional misbehavior; only input-buffer leakage/robustness. Tie FLGIN→GND.
- **HUSB238A ADDR strap = 10k (or 1M) vs datasheet 900k:** will likely still resolve to 0x42 I2C mode, but off-spec. Set to 900k. *(Two agents reported R66 as 10k and 1M respectively: verify the actual value in the netlist; either way change to 900k.)*
- **F1 (Littelfuse 466, 5A, 32V / 50A interrupt):** thin voltage margin at 6S and inadequate interrupt capacity for a multi-cell LiPo short. Use a ≥40V (pref. 63V), high-interrupt-capacity fuse, or move short-circuit protection to Q18 + eFuse/current-limit.
- **INGATE Vgs clamp D5 (12V zener) at SI2319 ±12V Vgs abs-max:** ±5% tolerance can sit at/above the limit. Use a 9.1V/10V zener (e.g. MMSZ5239B/5240B).
- **OV comparator U6 (LMV331) has no hysteresis:** chatters at the ~25.8V trip, toggling the cutoff FET/CE. Add ~1-2MΩ from OV_OUT to IN+ for ~50-100mV hysteresis.
- **TLA2528 ADDR floating = 0x10 (valid):** documented config, but the leakage-sensitive option; keep the node clean or add the Table-2 resistor.
- **Bleed resistor R125 (RC2512 27R, 2W "7W"):** 0.7W at LiHV (≈35% of 2W), safe but hot; do NOT substitute a 1W part; spread the six with copper pour away from the dividers/ADC.

### MANAGER
- **U17 SiLM2660 BAT-pin RC filter (100R + 10nF) omitted:** only C57 0.1µF and C61 470nF on the node. Add 100R series into U17 BAT + 10nF BAT→GND per datasheet Fig 8 for charge-pump-reference robustness vs VBUS dv/dt.
- **U17 CP_EN slaved to CHG_EN/DSG_EN:** every enable incurs the full ~50ms tCPON charge-pump start before the FETs turn on. If latency matters, break CP_EN onto its own GPIO and pre-arm it; else document.
- **DRV_SUP at 12V near DRV_OVP (12.8V min):** worst-case peak ~12.55-12.6V leaves ~200-300mV positive margin (no OVP trip under datasheet-bounded worst case), but zero margin band and DRV_OVP has no hysteresis. Drop the gate-drive rail to ~9-10V (re-value R88/R89).
- **TPS54160 12V buck fsw too high for EPR input:** R83=57.6k → fSW≈1.84MHz. At Vin=48V min-on-time ≈136ns (vs 130ns floor, ~5% margin); at 51V ≈128ns → pulse-skipping / lost regulation at the TVS clamp voltage. Lower fsw to ~400-600kHz (R83≈200k→585kHz; tON≈427ns at 48V); recompute COMP. Also satisfies the Eq-13 short-circuit frequency-shift limit.
- **TPS54202 inductors undersized:** L3=4.7µH (5V, ~62% ripple), L4=3.3µH (3V3, ~73% ripple). Increase to ~10µH / ~6.8µH (same FNR4040 footprint) for ~30-40% ripple; re-check output-cap RMS.
- **Bootstrap caps C5/C6 = 100nF low for 71.5nC Qg:** ~0.72V (>14%) per-cycle droop on the 5V boot rail, pushing high-side VGS toward the SFS08 4.4V Miller plateau. Increase to 220-470nF (re-evaluate with the FET-choice fix, M-power below).
- **MODE resistor R11 = 3.0k at the buck-boost detect boundary:** ±1% can read above the ≤3.0kΩ threshold and misclassify as buck-only. Use 2.0-2.2k.
- **Input TVS D11 (SMDJ51A) clamp 82V > BQ25758 VAC abs-max 70V and 63V caps; VBR(min) 56.7V < 60V max input.** Coordinate a lower-clamp TVS below 70V (and 63V cap rating) with standoff above true max operating voltage. *(Same device as M-D.)*
- **VBUS_IN ceramic-cap voltage margin tight at 48V:** 63V X7R MLCCs lose ~30-40% C from DC bias at 48V, and D11 lets the rail momentarily exceed 63V during a surge. Budget for DC-bias derating; consider 100V MLCCs for the small ceramics.
- **Battery-side TVS standoff close to max LiHV pack:** SLAVE D12 SMBJ28A vs 6S LiHV 26.1V (~1.9V); MANAGER D10 SMBJ36A vs 8S LiHV 34.8V (~1.2V). If LiHV is supported, bump one step (SMBJ30A / SMBJ40A).
- **Balance-drive NPN (MMBT3904) over-dissipates on upper cells (MANAGER cell sheet):** when the 12V gate-clamp zener holds the NPN collector at TAPn−12V with no series limit, Ic≈26mA → Cell8 Vce=21.6V → ~0.56W in a 200mW SOT-23. Add a 2-4.7kΩ series collector/clamp resistor (caps dissipation to ~0.1W); re-check cells 6/7/8. **(This is effectively a SEV2-class thermal failure during normal top-cell balancing: fix before fab.)**
- **12V gate-clamp zener at SI2319 ±12V Vgs abs-max (MANAGER cell sheet):** use a 10-11V zener.
- **Cumulative-tap divider scheme (both boards):** per-cell V = difference of two large near-equal cumulative readings, so gain tolerance dominates: the 0.1%/25ppm thin-film parts are mandatory (a 1% substitution makes per-cell readings useless); bottom cell uses only ~12% of FSR. Document the differencing in firmware; consider a per-cell differential front-end on respin.
- **Always-on dividers create position-dependent self-discharge (both boards):** bottom cell drains ~3.5× the top (~802µA vs ~229µA at 4.2V on SLAVE). Don't leave the board connected to a pack at rest; consider a sense-enable switch or higher-impedance dividers.
- **MANAGER display FPC (J4) underspecified:** only 6 pins (3V3/GND/SCLK/MOSI/DC/CS); no LCD_RST net anywhere; no backlight pins; symbol(6)/value("12P")/footprint(8) pin counts disagree. Pick the exact module, reconcile pin counts, add RST + backlight.
- **MANAGER backlight FET Q29 shorts 5V→GND when on:** gate=IO9, drain=5V, source=GND, with no LED load in the path (no backlight pin on J4). Insert the backlight LED string + current-limit R in series; low-side switch the cathode.
- **TPS54202 IIN hardware limit R13=1.62k → ~12A ceiling:** non-binding upper clamp, generous for a PD input. Verify vs negotiated PD current; increase R13 if a lower ceiling is wanted.
- **TCA9554 part-variant risk:** netlist `part` field is "TCA9554PWR" (base, 0x20) while value/MPN are "TCA9554APWR" (0x38). The non-A part at 0x20 would collide with TPS26750 (0x20-0x23). Lock the assembled part to the A-variant.

---

## 5. Unconnected-Pin Disposition

### SLAVE

| Ref.Pin | Name | Disposition |
|---|---|---|
| U8.4/7/9/10/15/17/24/25/28/29/32-35 | NC | OK: module NC pins |
| U8.12 (IO0), .26 (IO18), .27 (IO19) | spare GPIO / native USB | OK: prog via UART0 |
| U2.5/11/15/16/31 | NC | OK: leave floating, do not tie PGND |
| U2.8 | TS/NC | OK if TS unused (but see R67 dangling, S-class) |
| U3.11 | ADDR | OK floating = 0x10 (SEV3 robustness) |
| U3.12 | NC | OK |
| U3.5/6 (AIN6/7) | spare ADC | OK: tied GND |
| U1.6 (DBG_N), .13 (FAULT/OUT2) | optional outputs | OK NC |
| U1.11 (INT_N) | PD interrupt | OK only if firmware polls (SEV2 S-G) |
| U1.14 (FLGIN) | digital input | **SUSPECT**: tie to GND (SEV3) |
| J3 SBU1/SBU2 | sideband | OK: unused for PD sink |
| **U2.19/20/21/25/26/27** | gate drivers / BTST | **BUG (S2/S3)**: must be wired |
| **U2.9/10/17/36** | IOUT/IIN/MODE/FSW | **BUG (S-B/SEV3/S6)**: program/tie |
| **Q2.4/.5, Q3.4, Q4.4, Q5.4** | FET gates/drain | **BUG (S2/S4)** |
| **U7.1/3/6** | CB/FB/SW | **BUG (S7)** |
| **L2.1, C38 both, R55.2, R56.1, R7/R8** | buck output net | **BUG (S7/S8)** |
| **R3.2, R10.1** | input sense | **BUG (S5)** |
| **R5.1** | FSW resistor | **BUG (S6)** |
| **C3.1, C4.1** | bootstrap | **BUG (S3)** |
| **D2.5** | USBLC6 VBUS | **BUG (S9)**: wrongly on 28V VBUS |
| U1.7 (EN_HVDCP) | →`/main/HVDCP_DIS` single-pin | **SUSPECT (S-G)** |
| R15.1 (→`/main/V20`) | ACOV divider | **SUSPECT (S-H)** |

*(All "BUG" rows on SLAVE are partly entangled with the S1 annotation corruption: re-annotate and re-export before treating as final.)*

### MANAGER

| Ref.Pin | Name | Disposition |
|---|---|---|
| U6.5/11/15/16/31 | NC | OK: leave floating |
| U17.3/13/15 | NC | OK |
| U17.10 (PACKDIV), .14 (PCHG) | pack-mon/precharge | OK: features disabled (PMON_EN/PCHG_EN=GND); confirm intent |
| U1.21 | NC | OK |
| U1.6/7/13/18/22/23/30/31 | spare GPIO / unused USB-LD | OK NC; datasheet prefers tie-to-GND (SUSPECT-lite) |
| U16.1/2/14/15 | SBU OVP channels | OK: SBU unused |
| U7.11 | ADDR | **BUG (M6)**: must be strapped |
| U7.12 | NC | OK |
| U15.36 (RXD0) | UART0 RX | **BUG (M2)**: no prog path |
| U15.37 (TXD0)→`/main/TX` | UART0 TX | **BUG (M2)**: dead-end net |
| U15.13/14 (IO19/20) | native USB | repurposed for encoder → blocks USB flashing (M2) |
| U15.3 (EN) | reset | **BUG (M1)**: wired to pull-down, not pull-up |
| U15 spare IO (8/21/35-42/45-48) | spare GPIO | OK NC |

---

## 6. Confirmed-Correct / INFO

**SLAVE - verified good:**
- ESP32-C3 (U8): power/GND/EPAD correct; EN reset RC (R57 10k + C41 1µF) exactly per datasheet; straps IO2/IO8/IO9 correct for SPI boot + Joint Download via SW1; UART0→prog header J2 correct; I2C (IO4/IO5) with 4.7k pull-ups; GPIO assignments sensible; supply decoupling present.
- I2C address map collision-free: HUSB238A 0x42(8-bit)/0x21, BQ25758 0x6B, TLA2528 0x10, TCA9554A 0x38; single 4.7k pull-up set.
- Output current sense (R4 5mΩ + SRP/SRN Kelvin via R11/R12 + VO_SNS) wired and valued correctly.
- ACUV=VAC / ACOV=PGND straps are the datasheet "unused programming" config.
- Cell sense/balance topology: tap-to-cell mapping, daisy-chained TAP_HI/TAP_LO, per-cell bleed (SI2319 S=TAP_HI → 27R → TAP_LO), XH-7 J5 pinout, gate level-shift (MMBT3904 + R123 default-off + D105 12V zener clamping Vgs to −12V), POR safe state (TCA9554 powers up as inputs → all bleed FETs off), TLA2528 DECAP/AVDD/DVDD decoupling, AIN RC settling, divider keeps AINx within FSR. **(All contingent on fixing S1.)**
- Input protection: reverse-polarity/OVP-cutoff FET Q18 (AP40P04G) + D11 gate clamp; XT60 OR-ing diode D4 (SS56) + rail TVS D3 (SMBJ30A); OV divider/TL432 reference (trip ≈25.8V, good for 6S Li-ion); Q1 (SI2319) ratings.

**MANAGER - verified good:**
- BQ25758 (U6) charge core fully and correctly wired: gate-driver→FET pairing (HIDRV1/LODRV1→Q7/Q8 buck, HIDRV2/LODRV2→Q9/Q10 boost), BTST1/2 caps, sense placement (5mΩ SRP/SRN + 5mΩ ACP/ACN with 470nF diff + 100nF CM + 10R filters), IOUT R12=4.99k→10.0A, FSW R10=57.6k→~450kHz, REGN/DRV_SUP caps, NC pins.
- **SFS08R03GNF FETs are NOT under-driven** (verification INVALID'd that finding): BQ25758 LODRV is supplied from DRV_SUP and HIDRV from BTST replenished from DRV_SUP = `/main/12V`, so gates swing ~12V, not 5V. RDS(on)≈3.3mΩ @ VGS=10V is met; FETs correctly matched. *(Re-evaluate the SEV3 bootstrap-cap droop with this in mind: droop is on the BTST cap, still worth 220-470nF.)*
- EPR voltage-translation topology correct: TPD4S480 divides 48V → ~20V to protect the 22V-rated TPS26750 VBUS pins; dead-battery RPD shorted to CC; CC OVP path protects controller CC pins.
- 240W back-to-back power path Q1/Q2 (SFS08, 80V/3.3mΩ) + SiLM2660 (U17) driver correct; VDDCP cap C61=470nF correctly placed VDDCP→BAT; CC cReceiver caps C67/C68=330pF; VBIAS C58=0.1µF/100V.
- Forward ORing orientation (USB-C Q3 / XT60 Q4 / Solar) correct; highest source wins, no back-feed.
- Two-stage rail tree (VBUS_IN → TPS54160 12V → TPS54202 5V/3V3) topology sound; setpoints 12.00 / 5.08 / 3.305V; TPS54160 selection/min-on-time/BOOT/SS/comp OK (except the high-fsw SEV3 and the M7/M8/M10 defects); 3V3 logic rail independent.
- C75 (1µF/25V) on U17 is **correctly rated** (verification INVALID'd the over-voltage finding): it is the LM74700 VCAP-to-ANODE charge-pump cap (floating differential, ≤15V abs-max), not a pack-voltage cap.
- U9 LM74700 cathode sensing the far side of fuse F1 = **SEV3 accuracy** only (verification downgraded from SEV2); reverse trip still functions, slightly desensitized.
- I2C address map collision-free (TLA2528 0x10, TPS26750 0x20-0x23, TCA9554A 0x38, BQ25758 0x6B; EEPROM 0x50 on the separate PD controller bus); main pull-ups 4.7k good.
- IO3 JTAG strap (R72 10k pull-down) = **INFO** (verification downgraded from SEV2): correct way to satisfy the no-internal-pull GPIO3; board boots fine, JTAG source simply unavailable.
- Boot straps IO0/IO45/IO46 correct for N8R2 3.3V flash.

---

## 7. Investigated: Not an Issue (dropped findings)

- **MANAGER SFS08 "under-driven at 5V gate"** (was SEV1): **INVALID.** DRV_SUP=12V feeds the LODRV buffers and BTST bootstrap; gates swing ~12V; FET fully enhanced. No thermal-runaway risk; no FET change needed.
- **MANAGER C75 1µF/25V "over-voltaged for 8S"** (was SEV2): **INVALID.** It's the LM74700 VCAP-to-ANODE charge-pump reservoir (floating, ≤15V stress). 25V rating is correct (~70% margin).
- **SLAVE BQ25758 MODE floating "undefined mode"** (was SEV2): downgraded to **SEV3.** Open pin auto-detects buck-boost, matching the populated stage; fit an explicit ≥27kΩ resistor for determinism.
- **SLAVE HUSB238A FLGIN floating** (was SEV2): downgraded to **SEV3.** GATE-disable is opt-in and INT_N is unconnected; only leakage/robustness. Tie to GND.
- **SLAVE BQ25758 CE/R52 "100k too high, needs 10k"** (sub-claim of the open-drain pull-up finding): **dropped.** CE is a logic input; 100k default-disable pull-up is fine.
- **MANAGER U6 DRV_SUP near OVP** (was SEV2): downgraded to **SEV3** (worst-case peak below the 12.8V min OVP).
- **MANAGER U9/Q27 fuse-in-sense-loop** (was SEV2): downgraded to **SEV3** (accuracy only; reverse trip still works).
- **MANAGER solar Q5/Q6 "reverse blocking fails"** (was SEV2): reverse blocking actually still works via series body diodes; net issue is degraded accuracy + a wasted FET (**SEV2/SEV3** efficiency/BOM), not a functional failure.
- **MANAGER IO3 JTAG strap** (was SEV2): **INFO.** Correct strap; JTAG source just unavailable.

---

### Cross-board metadata (housekeeping)
- **SLAVE `.kicad_pro` DOC_* variables are stale** (copied from an RP2354B "OpenFC-Lite / Betaflight" flight controller: DOC_MCU='RP2354B', DOC_FIRMWARE='Betaflight', etc.). Correct to the ESP32-C3 charger. (SEV2 metadata; safe to edit the `.kicad_pro` JSON.)
- **MANAGER `Manager.kicad_pro` has empty `text_variables`**: populate to mirror the corrected SLAVE schema. (SEV3 metadata.)