---
name: incutec-eda
description: Drive the incutec-eda headless KiCad core (verify/source/build/bench) for this repo. Use when designing, sourcing, ERC/DRC-checking, or benchmarking KiCad boards from the terminal — e.g. "check my board", "source this LCSC part", "build a schematic from this design", "score against OpenFC-Lite". KiCad files are the source of truth; the agent authors the design, the core renders/sources/verifies.
---

# incutec-eda

Headless agentic KiCad core. **You (the agent) are the designer** — you reason out
the parts and nets. The core only renders, sources, and verifies against real KiCad
files. It is circuit-agnostic: never expect it to know topologies.

Run from the repo: `cd core && PYTHONPATH=. python3 -m incutec_eda <cmd>`
(or `incutec-eda <cmd>` if `pip install -e core` was run). Every command prints
JSON. **Exit codes: 0 = pass, 1 = verify gate failed (fix the design), 2 =
tool/infra error (fix the call).** Full reference: `docs/USAGE.md`.

## Commands

- `verify-sch <file.kicad_sch>` / `verify-pcb <file.kicad_pcb>` — ERC/DRC gate.
  Gate on `errors` only (warning/total counts are not run-to-run stable).
- `source <Cxxxx> [Cyyyy ...] --project <dir>` — import LCSC parts (symbol +
  footprint + 3D) into `<dir>/libs/`. After sourcing, add that lib to the
  project's sym-lib-table so the `lib_id`s resolve.
- `build <spec.json> <out.kicad_sch>` — render your `{parts, nets}` design, then ERC.
- `bench <canonical.kicad_sch> [candidate.kicad_sch]` — stats for a reference
  board, or score a candidate against it (BOM + topological-net overlap, no human).

### Assist primitives (the repeating-step helpers — operate on existing sheets)

- `fields-audit <sch>` — report components missing LCSC/MPN/Manufacturer + LCSC
  and Manufacturer-spelling inconsistencies.
- `fields-set <sch> <out> <assignments.json>` — bulk-set fields;
  `{"R1":{"MPN":"…"}, "*":{"Manufacturer":"…"}}` (`*` = all real parts).
- `place <sch> <out> <specs.json>` — add components into a sheet, then ERC.
  `[{"lib_id":"Device:C","value":"100n","count":4},{"lib_id":"Device:R","ref_prefix":"R","value":"10k"}]`.
  Refs auto-numbered from the symbol's prefix (override with `ref_prefix`).
- `sheets-scaffold <root> <name...> [--overwrite]` — create + link hierarchical
  child sheets into a root.
- `sheets-ports <child.kicad_sch> <ports.json>` — add hierarchical-label ports;
  `{"VBUS":"output","GND":"bidirectional"}`.
- `source … --field MPN=… --field Manufacturer=…` — stamp researched fields at import.

These are the mechanical multipliers: you (agent) supply the values/structure from
research; the tool applies them uniformly. Run `verify-sch` after edits; review the
`git diff`.

## Authoring a design spec

```json
{
  "name": "divider",
  "parts": [{"ref":"R1","lib_id":"Device:R","value":"10k","lcsc":"C25804"},
            {"ref":"R2","lib_id":"Device:R","value":"10k"}],
  "nets":  [{"name":"VOUT","pins":[["R1","2"],["R2","1"]]}]
}
```

Rules:
- `ref` required + unique; `lib_id` must resolve in installed KiCad libs.
- A net connects every pin you list on it. Omitted pins float and ERC flags them —
  that is your feedback signal. Single-pin nets ERC-*warn*, not error.

## The loop

Author spec → `build` → read the ERC JSON → fix the spec → repeat until exit 0.
Then a human opens it in KiCad for layout. For benchmarks, `build` your design from
a spec, then `bench <canonical> <candidate>` to self-grade.
