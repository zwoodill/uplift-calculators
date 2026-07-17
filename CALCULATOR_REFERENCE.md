# Off-Grid Systems Planner — Calculator Reference

Pseudocode + math behind every tab of `water-systems-planner.html`, plus how to use each.
Companion to `ONBOARDING.md` (HVAC-specific, for Lucas) and the design docs in
`SN002\thermals\`. This is a **design-trade tool**, not certified engineering — numbers
flagged "needs test / drawings / DAPIA" are assumptions until closed out.

---

## 0. How the whole thing works (read this first)

- **One self-contained HTML file.** No build step, no dependencies. Open in a browser or VS
  Code Live Preview. All state lives in memory and auto-saves to `localStorage`.
- **Two global objects:**
  - `S` = every **input** (setpoints, dimensions, materials, flags). Editing any field writes `S`.
  - `R` = every **result**, rebuilt from scratch on each change.
- **One pipeline.** Any input change calls `recalc()`, which runs top-to-bottom —
  occupancy → envelope → thermal → HVAC → seasonal → water demand → hot water → drainage →
  toilet → pump → power → battery → solar — then each tab's `render*()` paints `R` into the DOM.
  Because it's one pass, a change anywhere ripples everywhere (XPS thickness → wall R → heat
  load → HVAC watts → battery days, and the same thickness → wall thickness → interior area).
- **Unit convention.** All math is done in **imperial** (Btu/hr, °F, ft², gal). Values convert
  to metric **only at display** via `u*()` helpers (`uT`, `uDT` for temperature *differences*,
  `uR`, `uUo`, `uFl`, …). Toggle with the units button; `unitSys` defaults to metric.
- **Persistence.** `localStorage['uplift-water-planner-v4']`, payload schema `v:7`. Old v3
  saves migrate forward automatically. "Set as default" (password 1776) writes a baseline;
  "Reset to defaults" restores it.
- **Assumptions backend** (Variables tab, password 1776) exposes every hardcoded constant with
  a ⚑ NEEDS TEST / ✓ trusted chip — see §11.

Shared occupancy (used by several tabs), computed once at the top of `recalc()`:

```
occEq   = adults + guests + children × (childFactor/100)     // "water-equivalent" people (childFactor default 60%)
occHVAC = adults + guests + children × 0.7                    // heat-gain equivalent (thermal tab; OCC_CHILD_FRAC=0.7)
```
> **Two child models on purpose (verify intent):** water scales children by `childFactor` (0.60,
> user input) because kids use proportionally less water; heat gains scale by `OCC_CHILD_FRAC`
> (0.70, backend value) for metabolic output. Different physical quantities, so different factors
> are defensible — but they should stay *intentional*, not drift apart by accident.

> **Per-person heat is asymmetric on purpose:** heating credits **70 W** (sensible only — you
> can't count latent/moisture as useful heat), cooling counts **117 W** (total = sensible +
> latent, since the AC must remove both). This matches ASHRAE seated-adult figures.

---

## 1. Variables tab — shared inputs

**Purpose.** The single place for inputs used across every subsystem. Change it here and it
propagates everywhere.

**How to use.** Set occupancy (adults/children/guests + the child water factor), system
constants (bus voltage, autonomy days, budgets), and **Location + Season** (one shared climate
that drives both solar and thermal). The **Assumptions backend** dropdown at the bottom holds
the hardcoded physics constants (§11).

**Key behavior — Location + Season are shared state:**
```
climPSH(climate, season)  → peak sun hours   (feeds Solar)
CLIMATES[climate].heatF/coolF/means           (feeds Thermal design temps)
```
The same `S.climate` / `S.season` appear on the Thermal tab; changing either place updates both.

---

## 2. Envelope tab — assembly R, Uo, HUD zone check

**Purpose.** Turn the wall/roof/floor/window/door build-up into per-surface **effective R**
(accounting for steel thermal bridging), roll up to the whole-home **Uo**, and check it against
the HUD §3280.506 zone caps. The wall build-up also sets interior geometry.

**How to use.** Pick the calculation **basis** (Design vs HUD certificate) and target **zone**.
Choose a wall preset or fine-tune each layer. Enter the steel **member takeoff** (placeholder
until real structural drawings). Continuous interior board is the biggest lever.

**Math / pseudocode:**
```
# per surface: series of R-layers, with the framed cavity de-rated for steel bridging
R_assembly = R_films + R_cladding + R_gap + R_framedCavity + R_continuous + R_panel

# framed cavity layer (the steel-bridged zone):
if basis == "HUD":                       # §3280.509(b) parallel path
    ff = 0.15 (walls) or 0.10 (roof/floor)
    R_framedCavity via parallel: 1 / ( ff/(series+steel) + (1-ff)/(series+cavity) )
elif framingMode == "spacing":           # simple studs o.c.
    R_framedCavity = ASHRAE_90.1_AppA_table(nominalCavityR, studSpacing)   # ~45-50% of nominal
else:  # "takeoff" (welded truss frame)
    steelFrac = Σ(memberCount × length × faceWidth) / surfaceArea  × K      # K = weld/plate allowance
    R_framedCavity = AppA_table_by_steelFraction(cavityR, steelFrac)        # 16" o.c. ≈ 10.2%, 24" ≈ 6.8%

R_continuous = thickness × R_per_inch     # added in SERIES at full R — steel doesn't bridge it

# whole-home rollup
UA = Σ over {wall, roof, floor, window, door} of (area / R_eff)             # Btu/hr·°F
Uo = UA / A_envelope                                                        # Btu/hr·ft²·°F
pass = Uo ≤ UO_ZONE[zone]                 # Z1 0.116 · Z2 0.096 · Z3 0.079

# interior geometry from the build-up
wallThickness = bumper + sheath + gap + studDepth + continuous + panel
interiorL = 144 - 2·wallThickness ; interiorW = 90 - 2·wallThickness         # exterior box fixed 144×90×99"
```
**Outputs:** effective R per surface, UA, Uo + zone pass/fail, interior floor area & height,
assembly-comparison table.

**Fragile-pass assessment** (`uoFragility()`): a Uo under the cap is only a *solid* pass if a
modest, likely increase in the placeholder steel takeoff wouldn't flip it. The function bisects
for the **wall steel fraction at which Uo hits the zone cap** (`fracCrit`), compares to the
current takeoff fraction (`fracNow`), and classifies:
```
state = Uo > cap                                   → "fail"
      | headroom (fracCrit − fracNow) < 3 pts      → "fragile"   (design/takeoff basis)
      | margin (cap − Uo)/cap < 10%                → "fragile"   (HUD/spacing basis)
      | otherwise                                  → "solid"
```
Surfaced on the Envelope notice, the Summary Uo KPI, and the cockpit — green check only for
"solid"; amber for fragile *and* fail (a 0.0004-under-cap pass on placeholder data is not green).
**Caveats:** member takeoff is placeholder (needs drawings); floor/chassis bridge is first-pass.

---

## 3. Thermal tab — heating & cooling loads + seasonal energy

**Purpose.** Design heating/cooling loads on the *real* envelope (from §2), plus an annual
degree-day energy estimate. Feeds HVAC electrical draw.

**How to use.** Location + Season + Temperature basis (99% design extreme vs seasonal mean) set
the outdoor conditions; both design cases are always computed. Roof colour, wall colour, PV
coverage, ACH and latent fraction tune the loads. Pick the HVAC unit (shared with Energy tab).

**Heating (per design case):**
```
ΔT = indoorSet − outdoorDesign                       # HUD basis pins indoor 70°F
conduction = Σ (area/R_eff × ΔT) over wall,roof,floor,window,DOOR   # door term always included
infiltration = (basis=="HUD") ? 0.7 × (70−Tout) × perimeter        # §3280.509(a)
                              : 1.08 × (ACH × interiorVolume / 60) × ΔT
heatCert = conduction + infiltration                 # CERTIFIED: no internal-gain credit (§3280.510)
heatOper = heatCert − (occHVAC×70 + plugW)/0.293      # OPERATING: gains credited (energy estimate)
```

**Cooling (per design case):**
```
upRoof = (1−pvCov)×colourUplift[roofColor] + pvCov×3      # PV-shaded area gets +3°F residual
upWall = colourUplift[wallColor] / 2                      # walls: half (orientation averaging)
conduction = Σ area/R × (Tout + upObj − indoor)          # roof uses upRoof, walls upWall
solar   = windowArea × SHGC × solarFlux                  # daytime solar through glazing (Btu/hr)
gains   = (occHVAC×117 + plugW) / 0.293071               # W → Btu/hr — MUST convert to match the rest
sensible = conduction + infiltration + solar + gains     # all terms now Btu/hr
cool = sensible × (1 + latentPct/100)                    # humidity adder
# NB: every term in `sensible` is Btu/hr. conduction/infiltration/solar are already Btu/hr;
# only the internal-gains term starts in watts, so it is divided by 0.293071. (Verified in
# source: `intern=(occ*OCC_W.total+plugW)/BTUH_W`.) Same conversion the heating credit uses.
```
- `pvCov` is computed from **panel geometry**: `qty × panelL × panelW / roofArea`.
- Sizing cards always use the 99% extremes; the "at basis" cards follow the basis toggle.

**Seasonal energy (degree-day, envelope-driven):**
```
UAeff = UA_envelope + infiltrationConductance(ACH)
balancePoint Tb = indoorSet − internalGains / UAeff       # OUR envelope's heating threshold
HDD_Tb = HDD65 × ((Tb − winterMean)/(65 − winterMean))²   # rescale base-65 data to our balance point
heatKwh/yr = UAeff × HDD_Tb × 24 / 3412 / COP(winterMean)
coolKwh/yr = UAeff × CDD65 × 24 / 3412 × inflate / COP(summerMean)   # inflate = design cool / design conduction
```
The balance-point step is why a tighter envelope erases heating days entirely — base-65 alone
would miss that. Degree-days are approximate NOAA normals (⚑ flagged).

---

## 4. Water tab — demand, tanks, drainage

**Purpose.** Daily water demand from the fixture schedule, tank sizing, and DWV (drain-waste-
vent) sizing.

**How to use.** Edit the fixture table (flow gpm, minutes/day, % hot, per-person flag, grey
flag, DFU, trap size, autonomy flag). Set autonomy days and reserve % on Variables.

**Math:**
```
fixtureGal = gpm × min/day × (per-person ? occEq : 1)
totalFresh = Σ fixtureGal
hotDaily   = Σ fixtureGal × (showerLike ? hotFrac : pctHot/100)      # hotFrac from §5
greyDaily  = Σ fixtureGal over grey-flagged fixtures                 # hose & composting toilet add none

peakFlow = max( sum of 2 largest indoor gpm , largest hose gpm )     # simultaneous-use worst case
freshTank = autonomyFixtureGal × waterDays × (1 + reserve%)          # hose excluded (external supply)
greyTank  = autonomyGreyGal    × waterDays × (1 + reserve%)

# drainage
totalDFU = Σ fixture DFU ;  mainDrain = DFU≤6 ? 2" : DFU≤12 ? 3" : 4"     # §3280.610/.611
```
**Outputs:** fresh/hot/grey per day, peak flow, tank sizes, tank water mass (weight check),
DFU total + drain/vent sizes.

---

## 5. Hot Water tab — heater sizing, standby, capacity

**Purpose.** Size the tank + element, quantify standby loss, and count how many showers a tank
delivers. Feeds the energy budget.

**How to use.** Set inlet/setpoint/mix temps, tank size & shape, element watts, insulation.
Sub-tabs cover tank sizing, heater comparison, and insulation trade-off.

**Math:**
```
hotFrac = clamp( (showerMixF − inletF) / (setpointF − inletF), 0, 1 )   # thermostatic mix ratio
heatWh(gal, ΔT) = 8.34 × gal × ΔT / 3.412                               # lb×°F Btu → Wh

heatEnergyWh = heatWh(hotDaily, setpoint − inlet)                        # daily reheat energy
standbyW     = tankAreaFt² × (setpoint − ambient) / R_insul / 3.412      # continuous jacket loss
standbyWhDay = standbyW × 24
recoveryMin  = (heatWh(tankGal, ΔT) / elementW) × 60                     # cold-start full tank
consecShowers = tankGal / (showerGpm × min × FoI × hotFrac)              # showers before tank depletes
```
**Note:** %HOT for the shower row is *computed* (hotFrac, follows temps, not editable); for all
other fixtures it's a hand-entered usage assumption.

---

## 6. Toilet tab — composting liquid handling

**Purpose.** Size the urine container / leachate tank for the composting toilet.

**Math:**
```
urineGalDay = occEq × urineLitersPerPersonDay × 0.264      # L → gal
liquidGalDay = (mode=="urine-sep") ? urineGalDay : urineGalDay × leachateFrac%
containerGal = liquidGalDay × daysBetweenEmpties
flushWater = 0                                             # composting — no flush
```
**Caveat:** urine/person and leachate fraction are estimates; testing pins the real split.

---

## 7. Power tab — water-system energy + pump hydraulics

**Purpose.** The DC energy the *water system* draws (hot water + standby + pump) vs a budget,
and full pump selection from hydraulics.

**Pump (shower is the governing loop):**
```
dutyFlow = showerGpm × simulFactor                              # factor of ignorance for concurrency
friction = HazenWilliams(dutyFlow, headerID)×headerLen          # C=150 PEX
         + HazenWilliams(dutyFlow, ½"ID)×branchEquivLen
TDH = staticLift + residualHead + friction                      # total dynamic head (ft)
hydraulicW = dutyFlow × TDH × 0.1885                            # W per GPM·ft
pumpElecW  = hydraulicW / pumpEff
# selection: does the pump curve reach the duty point AND the peak-flow point?
canMeetPeak = pumpHead(dutyFlow) ≥ TDH  and  not stalled-on-static
runtimeMin = totalFresh / operatingPointFlow ;  pumpWhDay = pumpElecW × runtime/60
reqPsi = TDH / 2.3067 ;  pressureOK = reqPsi ≤ maxSysPsi         # §3280.609 (≤80 psi)
```

**Power balance:**
```
waterWhDay = heatEnergyWh + standbyWhDay + pumpWhDay
peakAmps   = elementAmps + pumpAmps        # simultaneous heater + pump draw
```
Flags fire when over budget, pump can't meet peak, or pressure exceeds the HUD ceiling.

---

## 8. Energy tab — loads, HVAC, battery, solar

**Purpose.** The whole-home DC balance: every electrical load, the computed HVAC draw, battery
autonomy, and solar harvest.

**Loads table + HVAC:**
```
houseWh = Σ (load.W × load.hrs)                    # editable table; the HVAC row is COMPUTED, locked
# HVAC row = thermal load ÷ COP, driven by Season:
mode = season=="winter" ? heating : season=="summer" ? cooling : off
COP  = override>0 ? override : type=="resist" ? 1 : max(1, curve(mode,Tout) × (type=="wave3"?0.85:1))
hvacElecW = thermalW(mode) / COP                   # resistance floor: COP never below 1
```

**Battery (pack is defined; autonomy is the output):**
```
installedWh = modules × moduleKwh × 1000
usableWh    = installedWh × DoD% × systemEff%
daysAchieved = usableWh / totalLoadWh              # totalLoad = water + house(incl. HVAC)
reqModules   = ceil( totalLoad × targetDays / (DoD% × eff% × moduleWh) )
```

**Solar:**
```
solarW      = panelW × qty × (1 + bifacialGain%)   # bifacialGain: flush=0, tested gap lever
solarWhDay  = solarW × PSH × derate%               # PSH from Location+Season
net = solarWhDay − totalLoadWh                     # + surplus recharges, − drains bank
```

**Daily profile chart:** each category's daily Wh spread over 24 h by a normalized shape
(meals / flat / evening / daytime). **The hour shapes are placeholders** (see the in-chart ℹ);
daily totals are real, intraday distribution is illustrative.

---

## 9. Summary tab — KPIs + decision cockpit

**Purpose.** At-a-glance rollup, plus a "cockpit" framing the design as constraints × levers ×
unknowns (so the calculator-drives-design ↔ design-drives-calculator loop stays explicit).

**Decision cockpit:**
- **Constraints (the corners):** HUD Uo vs zone cap (live pass/fail), fixed exterior box →
  interior area, roof fully consumed by panels, battery vs autonomy target.
- **Design levers (live sensitivity):** each lever is computed by *actually perturbing* the
  model one input at a time (`withS(key,val,fn)` swaps a value, recomputes, restores) and
  reporting the delta — e.g. "+¼″ continuous board → ΔUo, Δheat W".
- **Open unknowns:** count + pointers to the backend checklist.

---

## 10. Diagram tab — live dependency graph

**Purpose.** Visual of the calculation order and cross-subsystem hand-offs. Lanes = subsystems;
grey arrows = calc order within a lane; orange arrows = data passed between lanes.

> ⚠ **Being rebuilt (P7).** Current layout is slated for a full redo — treat as provisional.

---

## 11. Assumptions backend (cross-cutting) — Variables tab, password 1776

**Purpose.** Every hardcoded physics / reference / component constant in one editable, auditable
place — so nothing is buried in source, and every unknown is tracked.

**How to use.**
- Each row: a **status chip** — `⚑ NEEDS TEST` (assumption awaiting data) or `✓ trusted`
  (published/datasheet/decided). Click to toggle. Flagged rows get an editable **flag note**
  (what test/source will close it).
- Bottom **Open-unknowns checklist** merges flagged values + non-numeric tracked items (steel
  takeoff, belly construction, HVAC unit, …), each tagged by **what closes it**:
  `test · drawings · vendor data · reference data · decision`.
- **"Copy state for commit"** exports overrides + flags + notes as JSON. Edits here live in
  **this browser only**; paste that JSON to bake values into the shipped source (team-wide).

**Registers include:** air films, insulation R/inch, sol-air uplifts by colour, occupant heat
(+ child fraction), climate table (design temps, means, PSH, degree-days), HVAC COP curves +
Wave-3 derate, solar panel spec (Hyundai HiN-T595NI) + derate + bifacial gain.

---

## Appendix — key constants & sources

| Constant | Value | Source |
|---|---|---|
| HUD Uo zone caps | Z1 0.116 · Z2 0.096 · Z3 0.079 Btu/hr·ft²·°F | 24 CFR §3280.506 |
| Steel-framing effective R | ASHRAE 90.1 App A / IECC Table A103.3.6.2 | UpCodes / CFSEI |
| HUD framing fractions | 15% wall · 10% roof-floor | §3280.509(b) |
| HUD infiltration | 0.7 × T × perimeter | §3280.509(a) |
| Comfort-cool indoor design | 75 °F, ASHRAE Ch.22 (1989) | §3280.511 |
| Heat→Wh | 8.34 lb/gal · Btu / 3.412 | — |
| Btu/hr → W | × 0.293071 | — |
| Hazen-Williams friction | C = 150 (PEX) | — |
| Hydraulic power | GPM × ft × 0.1885 = W | — |
| Solar panel | Hyundai HiN-T595NI, 595 W STC bifacial | manufacturer datasheet |
| Battery | Pytes V5°, 51.2 V LFP, 5.12 kWh/module | manufacturer |

*Full regulatory method notes: Envelope & Thermal tabs' "Method & references" cards, and
`SN002\thermals\SN002_thermal_reconciliation.md`. Open regulatory questions:
`SN002\thermals\DAPIA_questions.md`.*
