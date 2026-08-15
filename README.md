# Charger

A distributed USB-C charging system for FPV battery packs. Two boards sharing
one charge, sense and balance core: a headless **charger node** that lives with
the pack, and a **manager** hub with a display and an encoder that drives
several nodes. No design exists yet; this page is the specification. If you
want to design it, say so on [Discord](https://discord.gg/v3sWmTcx3R).

[![Status](https://img.shields.io/endpoint?url=https://opendrone.be/api/status/Charger.json)](https://github.com/OpenDrone-hw/.github/blob/main/CONTRIBUTING.md#the-life-of-a-project)
[![Discord](https://img.shields.io/badge/Discord-join-5865F2?logo=discord&logoColor=white)](https://discord.gg/v3sWmTcx3R)

Nobody holds this board yet: claim it on Discord.

## Why

Field charging is one parallel board, one power supply and a lot of trust. The
distributed shape puts the charge controller with the pack instead: each node
does its own constant-current, constant-voltage and balancing, and the manager
becomes a user interface and a power budget rather than the thing doing the
charging. A failure is then one pack rather than the whole board.

USB-C PD is the input because it is the supply people already carry.

## Specifications

Targets, not measurements. The site imports this table once a board exists.

| | |
|---|---|
| Charger node | Headless, 2S to 6S, one per pack |
| Manager | 2S to 8S, display and encoder, drives several nodes |
| Input | USB-C PD |
| Charge controller | TI BQ25758 buck-boost, or a justified alternative |
| Balancing | Per cell |
| Chemistries | LiPo, LiHV, Li-ion, LiFe |

## Constraints

- Two boards, one shared charge, sense and balance core.
- USB-C PD input on both boards; the manager may add other inputs.
- Assembly by JLCPCB, LCSC basic parts preferred.
- Project-local KiCad libraries only, on the org template.
- Safety first: pack fault, thermal event and mid-charge disconnect are
  designed before the UI is.

## Prior art

An earlier schematic pass exists and was removed from `main` on 2026-08-13:
BQ25758 core, TLA2528 cell ADC, TCA9554A driving per-cell bleed FETs, ESP32
modules for ESP-NOW between manager and nodes. It is reference for the
thinking, not a design to continue from.

- [research/DESIGN.md](research/DESIGN.md): what that pass contained, per
  block and per board.
- [research/SCHEMATIC_REVIEW.md](research/SCHEMATIC_REVIEW.md): its review,
  verdict "not functional, do not fab", with the defects listed.
- The files themselves: tag `pre-reset-2026-08-13`.

## Open questions

- **How the manager and nodes talk.** Wired bus, and which one? This decides
  the connector and half the enclosure. The earlier pass assumed ESP-NOW.
- **Power budget.** How is a fixed USB-C PD budget divided across nodes that
  each want it, and who decides?
- **Cell count split.** The node is 2S-6S and the manager 2S-8S. Is that split
  right, or should they match?
- **Safety.** What happens on a pack fault, a thermal event, or a disconnect
  mid-charge. This is the only product in the line that can start a fire on
  purpose.
- **Enclosure.** Mechanical is as much of this product as the PCB.

## In the line

What pairs with what, and what is available:
[opendrone.be](https://opendrone.be).

## Contributing

KiCad files cannot be merged, so say what you intend to change before you do,
on [Discord](https://discord.gg/v3sWmTcx3R). How everything works:
[CONTRIBUTING.md](CONTRIBUTING.md).

## License

Hardware licensed under [CERN-OHL-S-2.0](https://ohwr.org/cern_ohl_s_v2.txt),
see [LICENSE](LICENSE).
