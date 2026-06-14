# IC integration — IP2366 + BQ76907 + ESP32-C3

How the three play together. **Ownership split (defense in depth):** IP2366 owns
the *power*, BQ76907 owns *safety* (hardware, independent of firmware), ESP32-C3
owns *intelligence*. They meet on one I²C bus and one charge-current loop.

## 1. Charge-current path (the crux)

```
USB-C VBUS ─► IP2366 (PD sink + buck-boost) ─BAT+─► Pack+  (JST-XH main +)
                  │                                   │
                  │                              [ pack cells ]  ◄─VC0..VC6 taps (JST-XH balance)─► BQ76907
                  │                                   │
   IP2366 GND ◄───┴── [CHG][DSG NFETs] ◄─ [SRP/SRN sense R] ◄── Pack-  (JST-XH main −)
                            ▲ gate drive          ▲ current
                            └──────── BQ76907 ─────┘
```

Charge current loops IP2366 BAT+ → pack → pack− → **sense R → BQ76907's CHG/DSG
NFETs → ground → back to IP2366**. So **all charge current passes through the
BQ76907 protection FETs** — on a per-cell OV/OC/OT fault, BQ76907 opens CHG and
**cuts charging in hardware**, with no ESP or IP2366 involvement. That hardware
backstop is the whole reason BQ76907 is here. (Ensure IP2366 folds back gracefully
when its load drops on a protection trip.)

## 2. Control bus (one I²C, ESP is master)

```
ESP32-C3 (I²C master, 3.3V, pull-ups)
   ├── IP2366   (slave) — set charge-current setpoint, read status
   └── BQ76907  (slave, 400kHz) — read per-cell V / current / temp, command balancing
```

Confirm: (a) IP2366 vs BQ76907 addresses don't collide; (b) IP2366's I²C domain is
3.3V; (c) BQ76907 I²C references charger GND (= pack−) — same ground as the ESP.

## 3. Power tree

```
VBUS ─┐
      ├─(priority/diode-OR)─► TPS560430 buck ─► 3.3V ─► ESP32-C3 + BQ76907 VDD/REGSRC
BAT  ─┘
```

Buck input = VBUS **or** BAT, so the node stays alive on pack power alone (monitor /
balance / report when no USB). TPS560430 (4–36V in) covers both 20V PD and 25V 6S.
BQ76907's REGOUT LDO is light-load housekeeping only — **never power the ESP from it**
(25V→3.3V LDO would dissipate watts).

## 4. Responsibilities

| IC | Autonomous job | Host (ESP) interaction |
|----|----------------|------------------------|
| **IP2366** | PD negotiation + buck-boost CC/CV (config R: 6S, Pmax, 4.2V/cell, NTC) | ESP writes charge-current setpoint; reads pack V/I |
| **BQ76907** | Hardware protection (OV/UV/OC/SC/OT → CHG/DSG), per-cell 16-bit ADC, coulomb counter, balancing FETs | ESP reads cell V/I/T; commands per-cell balancing |
| **ESP32-C3** | — | Runs the IR→current loop, balancing policy, ESP-NOW radio, (manager) display |

## 5. The IR→current loop (the "smart" part)

1. ESP steps IP2366 charge current (I²C).
2. ESP reads ΔV per cell (BQ76907 16-bit ADC) and ΔI (BQ76907 coulomb counter).
3. Per-cell IR = ΔV/ΔI → estimate health/temperature headroom.
4. ESP sets IP2366 charge current = min(per-cell limit, 3A connector cap, thermal).
5. During charge, ESP enables BQ76907 balancing on the highest cells (top-balance).

## 6. Open integration details

- **Variable 2–6S.** Wire all 7 VC inputs to the balance connector; for packs with
  fewer cells the unused upper taps tie to Pack+ → read ~0V → ESP detects cell count
  and configures BQ76907. Confirm BQ76907's unused-cell handling per datasheet.
- **Balancing current.** BQ76907 internal balancing is ~tens of mA (fine for small
  FPV packs). If faster is required, confirm external-balance-FET support — features
  read internal-only.
- **Ground.** IP2366 charge return, BQ76907 VSS/sense, and ESP 3.3V buck must share
  one charger ground at the pack− node (below the cells, above the protection FETs
  depending on chosen sense topology — fix this on the schematic).
- **Fault coordination.** BQ76907 CHG-open vs IP2366 fold-back; ESP reads both fault
  states and reports them over the radio.
