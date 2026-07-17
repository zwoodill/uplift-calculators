# Uplift Calculators — HVAC section onboarding (for Lucas)

Welcome! This repo hosts the SN002 off-grid systems planner. Your patch is **HVAC**:
turning the computed heating/cooling loads into realistic electrical draw. Everything you
need lives in one file and one clearly-marked block.

## The one file

**`water-systems-planner.html`** — a single self-contained HTML app (no build step, no
dependencies). Open it in a browser, or in VS Code use **Live Preview** for auto-reload.
There is no Node on the team machines; you don't need it.

## Your block: the HVAC section

Search the file for **`HVAC SECTION`** (boxed comment banner). It contains:

- **`COP_HEAT` / `COP_COOL`** — piecewise-linear COP-vs-outdoor-°F curves
  (currently from Charlie's model: heating −13 °F → 1.5 up to 60 °F → 3.8; cooling
  75 °F → 2.6 down to 115 °F → 1.4). Replace with the selected unit's published data.
- **`WAVE3_FACTOR`** — ×0.85 derate for the portable-unit case (an estimate; flagged).
- **`HVAC_TYPES`** — the unit-type dropdown. To add a unit: add a row here + a branch in
  `hvacCOP()`.
- **`hvacCOP(mode, ToutF)`** — COP lookup with a resistance floor (never below 1) and a
  manual override input.
- **`computeHvac()`** — season (Variables tab) picks the driving load: winter → heating
  *operating* load (internal gains credited), summer → cooling (with latent),
  spring/fall → off. Electrical W = thermal W ÷ COP, run 24 h.

The result feeds the locked "HVAC (computed)" row in the **Energy tab** loads table and
ripples into battery autonomy and solar makeup automatically.

Known gaps for you to close (also ⚑ flagged in-app):
1. Real unit selection + its published COP/capacity tables (replace Charlie's generic curves).
2. **Capacity collapse in deep cold** — below −13 °F the curve is extrapolated and real
   heat pumps lose capacity, not just COP. Model capacity, or spec resistance backup sizing.
3. Defrost-cycle and part-load behavior if you want to go further.

## Two ways values change (important)

- **Source defaults** (your job): edit the constants in the HVAC SECTION → PR → merged →
  everyone gets them.
- **Runtime tweaks**: the app has a password-gated "Constants & assumptions backend"
  (Variables tab ▸ dropdown at the bottom; ask Zach for the password). Those edits persist
  per-browser only — good for experiments, not for shipping. If an experiment proves out,
  move the value into the source and PR it.

## Verifying your change

1. Open the file in a browser; hard-refresh. The build stamp in the footer should not error.
2. Energy tab ▸ HVAC card: check thermal load, COP used, electrical W at:
   Minneapolis + winter + 99% design (worst case), Minneapolis + winter + seasonal mean,
   Austin + summer + 99% design. Sanity anchors as of build 49: ~1209 W / ~422 W / ~917 W.
3. Flip unit type to Resistance — COP must read 1.0. Set a COP override — it must win.
4. Season = spring/fall — HVAC row must go to 0.

## Workflow (protected main)

```
git clone https://github.com/zwoodill/uplift-calculators.git
git checkout -b hvac/<what-you-changed>
# edit water-systems-planner.html (HVAC SECTION)
git commit -am "HVAC: <what and why, cite the datasheet>"
git push -u origin hvac/<what-you-changed>
# open a Pull Request on GitHub → Zach reviews & merges
```

Direct pushes to `main` are blocked for collaborators — every change lands via a PR with
Zach's approval. A pre-commit hook stamps the build number/date automatically; don't edit
the stamp by hand. Cite your data source (datasheet PDF link or AHRI listing) in the PR
description — the app's backend links sources per value, and PRs should keep that standard.

## Context reading (optional but recommended)

- `SN002_thermal_reconciliation.md` and `SN002_unified_planner_architecture.md` in
  `OneDrive – Uplift Microhome\Projects\2026\SN002\thermals\` — the physics decisions and
  the phased plan (HVAC is phase P3).
- In-app: the Thermal tab's "Method & references" card explains the load model your COP
  divides into.
