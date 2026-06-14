# Component research — USB-PD charger controller — 2-6S LiPo, 140W max, simple/elegant/cheap (SELECTED: IP2366)

**Constraints:** `{"target_qty": 100, "max_area_mm2": 80, "weights": {"price": 0.3, "availability": 0.1, "size": 0.25, "fit": 0.35}}`

## Ranked candidates (best unit price @ qty 100)

| # | MPN | Mfr | Score | Price | Stock | Pkg | Size mm² | Lifecycle |
|---|-----|-----|-------|-------|-------|-----|----------|-----------|
| 1 | IP2366 (I2C version) | INJOINIC | 0.8825 | $0.955 | 0 | QFN-40(5x5) | 25.0 | active |
| 2 | BQ25756RRVR | TI | 0.6747 | $4.162 | 12660 | VQFN-36(5x6) | 30.0 | active |
| 3 | IP2368_BZ_VGLIP | INJOINIC | 0.541 | $1.902 | 33 | QFN-48(7x7) | 49.0 | active |

## Per-candidate detail

### IP2366 (I2C version) — INJOINIC (SELECTED — integrated PD3.1 sink + buck-boost charge/discharge)
- specs: cells=2-6S (resistor-set), max_power=140W, topology=single-inductor synchronous buck-boost, protocols=PD2.0/3.0/3.1 + AFC/FCP/SCP/QC2/3/3+ in&out, chemistry=Li-ion/LiFePO4 (3.65-4.4V/cell), interface=I2C optional (resistor-config standalone), standby=5uA, adc=14-bit
- LCSC: stock 0, breaks [[1, 1.5087], [10, 1.2529], [30, 1.1126], [100, 0.9554], [500, 0.8846], [1000, 0.8522]] USD — https://www.lcsc.com/product-detail/C20415848.html
- datasheet: https://www.lcsc.com/product-detail/C20415848.html
- notes: CHOSEN. Explicit 2-6S @ 140W, single-inductor minimal BOM, resistor-configurable (no firmware needed), 5x5. Cheapest + smallest. No 1S (2S min) - spec product 2-6S. Verify 6S charge current (140W/25.2V ~= 5.5A) in datasheet. LCSC stock 0 -> source reels via Injoinic/broker (consignable).
- sub-scores: {'price': 1.0, 'availability': 0.0, 'size': 1.0, 'fit': 0.95}

### BQ25756RRVR — TI (modular alternative — buck-boost charger (needs external USB-PD sink))
- specs: vin=4.2-70V, charge_current_max=20A, cells=1-14S, pd=NONE - needs external USB-PD sink
- LCSC: stock 12660, breaks [[1, 4.8154], [10, 4.1625], [500, 3.2017], [1000, 3.119]] USD — https://www.lcsc.com/product-detail/C19272232.html
- datasheet: https://www.lcsc.com/product-detail/C19272232.html
- notes: Robust/over-spec path: 1-14S, 70V, 20A, well stocked. But needs a separate USB-PD sink IC (+cost/area/firmware). Overkill for a simple 140W charger; keep as fallback if IP2366 sourcing or 6S current proves inadequate.
- sub-scores: {'price': 0.23, 'availability': 1.0, 'size': 0.833, 'fit': 0.85}

### IP2368_BZ_VGLIP — INJOINIC (sibling — bigger/pricier integrated PD buck-boost)
- specs: battery_rail=4.5-25V, charge_current_max=5A, max_power=140W (rated), pd=integrated PD sink, interface=I2C
- LCSC: stock 33, breaks [[1, 2.0498], [20, 1.9678], [100, 1.9022], [500, 1.8366], [1000, 1.8038]] USD — https://www.lcsc.com/product-detail/C5203723.html
- datasheet: https://www.lcsc.com/product-detail/C5203723.html
- notes: 48-pin/7x7 sibling of IP2366. Pick only if you need its extra I/O (likely dual USB-C port / more status LEDs / current headroom). ~2x the price, ~2x the area. LCSC rail 4.5-25V left 6S ambiguous - IP2366 states 2-6S outright.
- sub-scores: {'price': 0.502, 'availability': 0.003, 'size': 0.51, 'fit': 0.75}

## Excluded (unbuyable lifecycle)

- SC8815QDER (Southchip): discontinued

## State of the art / principles / trends

- USB-PD 3.1 EPR: 140W = 28V @ 5A. The front-end must negotiate EPR to reach 140W.
- 2-6S Li-ion spans 8.4-25.2V; PD source (5-28V) straddles the battery voltage -> synchronous buck-boost is mandatory.
- Injoinic IP236x are integrated 'power-bank SoCs': single QFN does PD3.1 + buck-boost charge AND discharge to 140W, resistor-configurable (firmware optional). IP2366 explicitly supports 2-6S.
- IP2366 vs IP2368: IP2366 (QFN-40 5x5, ~$0.85-1.51, explicit 2-6S) is the lean choice; IP2368 (QFN-48 7x7, ~$1.80-2.05) only earns its extra pins with dual-port / extra I/O.
- Modular TI BQ25756 + USB-PD sink is the robust over-spec fallback (1-14S/70V/20A) at ~2-3x IC cost + a second chip + firmware.
- Sourcing: low/zero LCSC stock is NOT a blocker (JLCPCB consignment / direct Injoinic reels). DISCONTINUED lifecycle IS (SC8815).

## Recommendation

SELECTED: IP2366 (I2C version). Single ~$0.85-1.51 QFN-40 does PD3.1 negotiation + single-inductor buck-boost charge/discharge to 140W, explicit 2-6S, resistor-configurable so it can run firmware-free. Cheapest, smallest, fewest external parts = simple/elegant/cheap. Spec the product 2-6S (no 1S). Before layout: confirm in the IP2366 datasheet the max charge current at 6S (~5.5A for 140W into 25.2V) and the reference application (single-inductor, NMOS sizing). Keep TI BQ25756 + a PD sink as the documented fallback if 6S current or IP2366 sourcing falls short.
