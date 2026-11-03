# 15.0 Civil3D – MWP cable pull, IEEE 1185-2019 – V15

Dynamo for Civil 3D graph that reads the Civil 3D profile reports of a duct route, calculates the
cable pulling tension and sidewall pressure in both directions (IEEE 1185-2019), and writes an Excel
calculation in the layout of the 24252 cable-pull calculation workbook.

## Package contents

| File | What it is |
|---|---|
| `15.0 Civil3D_MWP_PullCal_IEEE1185_V15.dyn` | The Dynamo graph (CPython3, no custom nodes or packages needed) |
| `15.0 Civil3D_MWP_PullCal_Template_IEEE1185_V15.xlsx` | Excel report template. **Keep it in the same folder as the .dyn** |
| `Example_Output/Example_Energinet_to_FRD2A_C_CC_P.xlsx` | Example export (Energinet → FRD2A_C_CC_P, graph default cable inputs) |
| `Source/15.0 Civil3D_PullCal_Python_Node_V15.py` | The main Python node code, for reading or editing outside Dynamo |

## How to run

1. Unzip the folder anywhere. Keep the `.dyn` and the template `.xlsx` together.
2. In Civil 3D, export the two profile reports for the route as Excel:
   the **XY profile report** and the **Z profile entity report**.
3. Open `15.0 Civil3D_MWP_PullCal_IEEE1185_V15.dyn` in Dynamo for Civil 3D.
4. Enter the cable data in the input nodes (group 02): number of cables, mass per cable,
   cable OD, duct ID, friction, gravity, starting tension, allowable tension, allowable sidewall
   pressure, allowable fill and cable formation. Set *Project cable assumptions confirmed* to True
   once the values are for this project.
5. Set **Select both reports | True then Run** to True and click Run. In the Browse window hold Ctrl
   and select **both** reports, then Open.
6. Set *Write new Excel report* to True. Optionally set the export folder
   (*Browse for Excel export folder*). With no folder chosen, the report is saved next to the template.
7. Each run saves a new time-stamped workbook `FRD2A_PullCal_YYYYMMDD_HHMMSS_xxxxxx.xlsx`.
   Open it in Excel; it recalculates on opening.

The *Excel report template* node already names the template. If the file is not found at that path,
the graph looks for `15.0 Civil3D_MWP_PullCal_Template_IEEE1185_V15.xlsx` next to the .dyn and then next to the reports.

## Batch run – several routes, same cable parameters

1. Set the cable inputs once.
2. Set **Select reports | True then Run** to True and click Run. In the Browse window Ctrl-select the
   XY **and** Z reports of every route you want (e.g. 10 routes = 20 files).
3. Reports are paired automatically by file name (Civil 3D `...xy.xlsx` / `...z.xlsx`); if names do not
   pair, by drawing path + parent alignment in the report header. Selection order does not matter.
4. Every route is calculated with the same cable inputs. With *Write new Excel report* = True you get
   **one workbook** `FRD2A_PullCal_All_Routes_<time>.xlsx`:
   - **Summary** tab – one row per route (click the route name to open its tab): max tension and sidewall
     pressure in both directions, allowables, PASS/FAIL, total bend, number of 90° bends, reduced allowable
     fill and fill check (all live formulas), a worst-case row for all routes, and every review item
   - **one Pull tab per route**, named after the route (24252 pull-sheet layout, forward + reverse tables)
   - **IEEE Inputs** and **IEEE 1185-2019 Method** – shared by all routes
   To get a separate workbook per route instead, set `BATCH_SEPARATE_WORKBOOKS = True` near the top of
   SECTION 13B in the Python node.
5. A route that cannot be calculated (e.g. parabolic vertical curves) is listed in the summary with the
   reason; the other routes still run.

Selecting just one XY + Z pair works exactly as before.

## The Excel report

| Sheet | Contents |
|---|---|
| **Summary** | Project, route, both pull directions against allowable tension and sidewall pressure (PASS/FAIL), conduit fill and jamming checks, and the graph's review items |
| **Pull** | 24252 pull sheet layout. Column Q = IEEE section type (column Q): Straight, Slope Up, Slope Down, Horizontal Bend, Concave/Convex Upward/Downward Bend (compound curves show Horizontal Bend, sags with the cable on the duct floor show their slope type, as Dynamo). First table = increasing station, second table = decreasing station. Columns L–AC as the 24252 sheet (T1, lengths, bend, radius, gradient, Straight / Uphill / Downhill / Bend / Vertical T2, T2, sidewall pressure) with max, allowable and check rows |
| **IEEE Inputs** | Cable inputs from Dynamo (amber) and the worked steps of the *IEEE 1185-2019 cable inputs* node: D/d, r, Wc, f′ = Wc × μ, W = n × m × g, fill, c. Below: the 24252 fill check |
| **IEEE 1185-2019 Method** | Formula reference sheet with the bend diagrams, linked to IEEE Inputs |

### Pull sheet – step columns (AE onwards)

Every section formula is a live Excel formula, broken into the same steps as the Code Blocks on the canvas:

- **Civil 3D geometry**: segment, entry/exit station, entry/exit angle, plan radius, profile radius, grade-break turn, Δz, 3D length
- **Sections sorted by IEEE type**
- **Grade break (no radius)**: T1 = T_prev × e^(f′ × turn)
- **1 Straight**: T2 = T1 + W f′ L
- **2/3 Slope up/down**: T2 = T1 ± W L (sin θ ± f′ cos θ)
- **4 Horizontal bend**: T2 = T1 cosh(f′θ) + sinh(f′θ) √(T1² + (W R)²) + W Δz
- **5–8 Vertical bends** (Concave/Convex, Upward/Downward), including the sag check:
  if T1/R < W cos θ the cable lies on the duct floor and the slope formula is used
- **Compound curve**: larger of the horizontal and vertical result; P uses 1/R = √(1/Rh² + 1/Rv²)
- **Sidewall pressure**: P = c × T2 / R
- **Check against Dynamo**: the tension calculated by the Python node and the difference for each row

### Conduit fill check (from the 24252 workbook)

- Percentage of conductor fill = n × (π d²/4) / (π D²/4)
- Number of 90° bends = total bend on the route (plan + vertical + grade breaks) / 90, rounded up
- Reduced allowable fill = (1 − 0.15 × number of 90° bends) × allowable fill
- Likelihood of jamming = 1.05 × D/d (3 cables: 2.8–3.0 = jam risk)

## Verification

- Tested on 15 FRD2A route report pairs, each with two sets of cable data (30 exports), opened and
  recalculated in Microsoft Excel. Every section row matched the Python node to within 1×10⁻⁷ N, with no
  formula errors and no file repair. The routes cover straights, slopes, horizontal bends, vertical bends,
  compound curves and grade breaks.
- Not yet run inside Dynamo for Civil 3D; the Python was run offline through the same entry point.

## Notes and limitations

- The graph's default cable inputs (17 × 60 mm cables in a 160 mm duct) give 239 % fill and are
  placeholders. Enter the real cable data before using the results.
- Profiles with **parabolic** vertical curves are not supported (the reports must use circular curves).
- Grade breaks without a radius use a capstan estimate (e^(f′ × turn)); no sidewall pressure is given there.
- Results are engineering estimates for review, not an installation approval.

## Version history

- **V15** – Batch run writes ONE workbook: Summary of all routes + one Pull tab per route + shared IEEE Inputs / Method.

- **V14** – Batch summary is an Excel workbook (cable parameters, PASS/FAIL colours, errors, links to each route report) instead of CSV.
- **V13** – Batch run: select the reports of several routes and run them all with the same cable parameters (one Excel per route + batch summary).
- **V12** – Pull sheet column Q shows the IEEE section type of every section (checked against the Dynamo section list on 30 test exports).
- **V11** – First release of the 24252-style calculation template.
