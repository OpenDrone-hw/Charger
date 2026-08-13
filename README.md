# Charger

**Planned.** No design exists yet. This page is the specification: what we want
built and why. If you want to design it, say so on
[Discord](https://discord.gg/v3sWmTcx3R).

A distributed USB-C charging system for FPV battery packs. Two boards sharing
one charge, sense and balance core: a headless **charger node** that lives with
the pack, and a **manager** hub with a display and an encoder that drives
several nodes.

## Why

Field charging is one parallel board, one power supply and a lot of trust. The
distributed shape puts the charge controller with the pack instead: each node
does its own constant-current, constant-voltage and balancing, and the manager
becomes a user interface and a power budget rather than the thing doing the
charging. A failure is then one pack rather than the whole board.

USB-C PD is the input because it is the supply people already carry.

## Requirements

| | |
|---|---|
| Charger node | Headless, 2S to 6S, one per pack |
| Manager | 2S to 8S, display and encoder, drives several nodes |
| Input | USB-C, PD |
| Core | TI BQ25758 buck-boost charge controller, or a justified alternative |
| Balancing | Per cell |
| Assembly | JLCPCB, LCSC basic parts preferred |

## Open questions

- **How the manager and nodes talk.** Wired bus, and which one? This decides
  the connector and half the enclosure.
- **Power budget.** How is a fixed USB-C PD budget divided across nodes that
  each want it, and who decides?
- **Cell count split.** The node is 2S-6S and the manager 2S-8S. Is that split
  right, or should they match?
- **Safety.** What happens on a pack fault, a thermal event, or a disconnect
  mid-charge. This is the part that has to be designed first, not last, because
  it is the only product in the line that can start a fire on purpose.
- **Enclosure.** Mechanical is as much of this product as the PCB.

## Research

Earlier design and review work is in [research/](research/). Note that it
describes an unrouted schematic that has since been removed from `main`; it is
reference for the thinking, not a design to continue from. The files are
recoverable at the `pre-reset-2026-08-13` tag.

## Contributing

Issues and pull requests are welcome on any repo. KiCad files cannot be merged,
so say what you intend to change before you do, on
[Discord](https://discord.gg/v3sWmTcx3R).

How everything works: [CONTRIBUTING.md](CONTRIBUTING.md).

## License

Hardware licensed under [CERN-OHL-S-2.0](https://ohwr.org/cern_ohl_s_v2.txt),
see [LICENSE](LICENSE).
