---
name: component-research
description: Central component researcher — survey ALL suppliers, read datasheets, study state-of-the-art / reference designs / academic principles / trends, compare price + availability + size, and produce a ranked, sourced research report for a component choice. Use when picking a part or topology for a board (e.g. "find the charger IC for the USB-PD LiPo charger", "research USB-PD sink controllers", "what MOSFET for this", "compare buck-boost charger ICs"). The agent does the research; incutec-eda's research-report scores + formats it.
---

# Component research

A structured workflow to choose a component (or topology) well. **You do the
research and judgment; `incutec-eda research-report` does the deterministic
scoring + report.** Output is a ranked, sourced markdown report written into the
board's own repo (never the tooling repo).

If a supplier/datasheet/site needs live access and no skill/API is configured,
**use Chrome** (`mcp__claude-in-chrome__*`): navigate, read the page, pull
price/stock/specs. Prefer the distributor skills first where they work.

## Inputs

- **Role + requirement** — what the part must do and the board context (e.g.
  "buck-boost battery-charger controller, 1–6S (≤25.2V), ~5A, USB-PD input ≤28V").
- **Constraints** — `target_qty`, `max_area_mm2`, `min_stock`, and scoring
  `weights` `{price, availability, size, fit}`.

## Workflow

1. **State of the art / principles / trends.** Before parts: what's the right
   topology and why? Survey reference designs (TI/ADI/MPS/etc. eval boards & app
   notes), academic/standards principles (e.g. USB-PD EPR 28V/5A = 140W; buck-boost
   needed to charge 6S from variable PD voltage), and current trends. Web/Chrome.
2. **Candidate parts.** Manufacturer parametric search + distributor search across
   **all** suppliers (`digikey`, `mouser`, `lcsc`, `element14`, `jlcpcb`) — Chrome
   for any without API access. Capture MPN, manufacturer, package, lifecycle.
3. **Datasheets.** Fetch (digikey skill or Chrome) and read each candidate's
   datasheet; extract the key specs that decide the choice.
4. **Pricing & availability.** Best unit price at `target_qty` and stock per
   supplier (price breaks, lead time). Multi-supplier — don't trust one source.
5. **Size / constraints.** Package + footprint area vs `max_area_mm2`; compliance,
   second-source availability, JLCPCB basic-vs-extended if relevant.
6. **Assemble + score.** Fill the candidate schema (below), then:
   `incutec-eda research-report spec.json <board-repo>/docs/research/<topic>.md`.
   It ranks by the weighted score and writes the report.
7. **Recommend.** Write the rationale into `spec.recommendation` (the ranking is
   computed; the *why* is yours — trade-offs, risks, second sources).

## Candidate schema (assemble this as `spec.json`)

```json
{
  "topic": "...", 
  "constraints": {"target_qty":100,"max_area_mm2":60,"min_stock":500,
                   "weights":{"price":0.4,"availability":0.3,"size":0.2,"fit":0.1}},
  "candidates": [
    {"mpn":"...","manufacturer":"...","role":"charger","package":"VQFN-32",
     "size_mm":[5,5],"key_specs":{"vin_max":"36V","ichg":"20A","cells":"1-8S"},
     "datasheet_url":"https://...","lifecycle":"active","fit":0.9,"notes":"...",
     "offers":[{"supplier":"DigiKey","sku":"...","url":"...","stock":4200,
                "currency":"USD","price_breaks":[[1,4.10],[100,3.20],[1000,2.55]]}]}
  ],
  "sota": ["principle/topology/trend findings, each with a source"],
  "recommendation": "your pick + why; trade-offs, risks, second source"
}
```

`fit` is your 0–1 judgement of how well the part meets the spec (the only score the
tool can't compute). Be honest; cite datasheet specs in `notes`.

## Output

A ranked report (`research-report`) in the **board repo**, with a comparison table,
per-candidate detail with supplier offers + datasheet links, the state-of-the-art
section, and your recommendation. It's a reviewable artifact, not a chat answer.
