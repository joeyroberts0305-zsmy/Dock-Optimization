**Dock Optimization Project**

**Executive Brief:**
This project builds a data-driven dock layout that lowers forklift travel, reduces congestion, and makes the floor plan easier to run.
What leaders get:

* Travel reduction: Optimized door ordering that minimizes total forklift feet traveled.
* Fewer hotspots: Soft congestion smoothing spreads heavy doors to safer loads.
* Operational clarity: A labeled dock diagram with color-banded clusters, break-door seams, and dimensioner (no-cross) bars.
* Auditability: A “why here?” workbook that shows top counterpart doors and an R80 radius (how concentrated each lane’s flow is).
* Repeatable process: Works across service centers with different door counts, spacing, and non-traversable areas.
How it works:

* Reads your inbound/outbound history and builds a flow matrix showing how much freight moves from each inbound lane to each outbound lane.
* Converts door positions to feet using site geometry and adds detour penalties for dimensioners/ramps you can’t drive through.
* Solves for the best door order with optional penalties that (a) smooth congestion and (b) keep similar lanes near each other.
* Exports visuals, spreadsheets, and a short math report to show savings and defend the plan.


**Repository Layout**
Dock Optimization Project/
├─ data/
│  ├─ Inbound.csv            # input (CSV/XLSX also supported)
│  └─ outbound.csv           # input
├─ outputs/                  # artifacts (images, Excel, reports)
└─ Dock_Optimization_V2.ipynb    # one-click end-to-end notebook


**Quick start**

1. Put data into Dock Optimization Project/data/

    * Typical columns (case/spacing don’t matter; auto-detected):

      * Inbound: INB_DATE (pickup/arrival), INB_LANE (origin or lane), INB_CUBE, INB_SHIPMENTS, INB_WEIGHT_LB

      * Outbound: OUT_DATE (dispatch/load), OUT_LANE (destination), OUT_CUBE, OUT_SHIPMENTS, OUT_WEIGHT_LB

   * The loader also detects terminal/type columns to filter MCI correctly: inbound to MCI (types D/B), outbound from MCI (types O/B).

2. Open the notebook Dock_Optimization_V2.ipynb and run all cells.

**Outputs land in outputs/:**

* dock_layout_MCI_updated.png — color-banded TO/FROM clusters, break-door marks, dimensioners

* fromX_toY_cube_clustered.xlsx — spreadsheet for workflow review (TO bands colored)

* why_here_report.xlsx — Workflow Comparison audit (top counterpart doors, shares, and 80% radius)

* math_report.md/json — objective, parameters, savings (ft and time)

* heatmaps and capacity flags
  
** Techical summary **

  * The model builds flows between inbound lanes (FROM) and outbound lanes (TO) by matching dates and allocating proportionally when exact matches are sparse.

  * Convert door positions to feet with geometric logic (bay spacing, aisle crossing, turn penalty, and detour penalties for dimensioners/no-cross zones).

  * Solve door order to minimize total travel with optional congestion smoothing and soft clustering so similar lanes sit near each other: 
  <img width="369" height="50" alt="image" src="https://github.com/user-attachments/assets/d0330a31-75f5-4576-a2b8-38b49521ff1f" />

  * Draw a dock diagram with color bands for TO and FROM clusters, break-door suggestions, and DIMENSIONER no-cross bars.

  * Produce an audit explaining why each door lives where it lives (top counterparts, shares, and an 80% “flow radius” around the weighted median door).

  * Create an Ops spreadsheet (TO on Y, FROM on X) with bands so supervisors can verify that the clustering lines up with the way they actually run.

Mathematical Logic for clustering and door assignment
1. **Flow matrix F (units: cube or shipments or weight)**
  A table where entry 𝐹𝑖𝑗 is the amount of work (cube/shipments/weight) that moves from FROM lane 𝑖 to TO lane 𝑗 over the period.
  Per day we preserve inbound totals and allocate them across outbound lanes in proportion to that day’s outbound mix. If same-day is sparse, we allow ±1 day, then weekly, then a full-window proportional fallback.
  F is the “demand” the dock layout must serve; it’s what we try to move the shortest distance.

   <<img width="568" height="124" alt="image" src="https://github.com/user-attachments/assets/621f5f08-e0c3-4279-8668-7749cbcf53ad" />
   
   Then allocated each day's inbound volume by the outbound mix for that same day:
   
   <img width="230" height="47" alt="image" src="https://github.com/user-attachments/assets/4f0d2ebc-476f-4dce-be5e-fe23986944a2" />
     
3. **Distance matrix 𝐷 (units: feet)**
  Baseline “Manhattan along the dock” with a fixed aisle crossing and turn penalty:
  A table where 𝐷𝑝𝑞 is the travel distance (ft) from inbound door 𝑝 to outbound door 𝑞.
    p is the inbound door index (after you place the lanes), q the outbound door index (after placement).
    Typical values: BAY_SPACING_FT=12, AISLE_CROSSING_FT=60, TURN_PENALTY_FT=10.
  Manhattan distance: We approximate forklift travel as “go across the aisle, then along the dock,” which is an L1 (Manhattan) metric:

  <img width="852" height="70" alt="image" src="https://github.com/user-attachments/assets/7113d247-bf10-4b72-b256-02935447c0a3" />
  
  No-cross zones (dimensioners, ramps, walls)
  No-cross (dimensioners/ramps): If the straight path between 𝑝 and 𝑞 would cross a blocked span [𝑎,𝑏) we add a detour (extra feet). This steers the optimizer to avoid illegal/unsafe shortcuts.
    Why it matters:𝐷 turns “who moves to whom” (from 𝐹) into feet and therefore time/cost.

  <img width="390" height="172" alt="image" src="https://github.com/user-attachments/assets/cb9d944d-4fa1-4d68-801f-11fd89acfc90" />

  For a candidate pair(𝑝,𝑞) if the segment between 𝑝 and 𝑞 intersects any [𝑎,𝑏) then: 
  <img width="464" height="63" alt="image" src="https://github.com/user-attachments/assets/b19087b0-f908-4757-9b9e-9739a4295b0b" />

  This makes “shortcuts” through dimensioners expensive in the solver and the diagram shows gold bars at those spans.
  
3. **Objective and solver**
  We place lanes by permuting door indices:
    𝜋(𝑖): door index assigned to inbound lane 𝑖 (FROM)
    𝜎(𝑗): door index assigned to outbound lane 𝑗 (TO)

Primary objective:

  <img width="333" height="73" alt="image" src="https://github.com/user-attachments/assets/02b15f51-a33c-488b-9fcd-399f1bfb3288" />
  
Optional penalties (all soft):

Congestion smoothing (spread heavy lanes) where 𝑊 in and 𝑊 out are door-ordered workloads, 𝑤 is a small window (e.g., 3).:

  <img width="485" height="55" alt="image" src="https://github.com/user-attachments/assets/991f6f7f-d62f-459d-8c0e-51c740c3aeac" />
  
Keep similar lanes near (soft clustering):

  <img width="760" height="109" alt="image" src="https://github.com/user-attachments/assets/3a9145a8-0da0-4edc-9428-319f8d6ae5a6" />
  
Total Objective:

  <img width="255" height="46" alt="image" src="https://github.com/user-attachments/assets/9cc84d77-92f3-4389-86a7-58961fc9759e" />

**Algorithm** 
It alternates two Hungarian solves + local swap polish:
1. Fix 𝜎 → compute row costs 𝐶 in = 𝐹⋅𝐷[:,𝜎]⊤ and solve for 𝜋.
2. Fix 𝜋 → compute col costs 𝐶 out= 𝐹⊤⋅𝐷[𝜋,:] and solve for 𝜎.
3. Pairwise swap passes on inbound and outbound doors to improve 𝐽aug.
4. 
This converges quickly and is robust for 38×38.

Break doors:  workloads 𝑊out, smooth them, and pick the top 𝑘 peaks (with a neighborhood suppression) as break-door separators—good seams for staging/tug flow.

4. **Submatrix selection(38x38)**
If the matrices are larger than the physical dock, we pick the 38 FROM and 38 TO lanes that capture the most total flow (“max coverage” greedy). This yields the working 38×38 used in the solver and outputs, while keeping most of the mass.

5. **Cluster bands (Ops Friendly View)**
I computed lane-lane similarities 𝑆out = 𝐹⊤𝐹(TO) and 𝑆in=𝐹𝐹⊤ (FROM), then:  
* Seriation: order lanes by the principal eigenvector (spectral 1-D embedding) so similar lanes are adjacent.
* Cuts: choose split points at low adjacency similarity (or force K clusters).
* Color bands: we color those contiguous segments in:
  * the Excel matrix (TO on Y, FROM on X),
  * the dock diagram (colored header bands for TO and colored footer bands for FROM).
* You can force the number of clusters or let the thresholds decide.

**Helpful Outputs:**
* Dock diagram: dock_layout_MCI_updated.png
  * Top row: TO (outbound) labels like “TO STL”, color-banded by TO clusters
  * Bottom row: FROM (inbound) labels like “FROM CHI”, color-banded by FROM clusters
  * Break doors: vertical ticks labeled “BREAK” at recommended seams
  * Dimensioners: gold horizontal “no-cross” bars centered between rows at specified door spans

* Why-here report: why_here_report.xlsx
  * For each door: top counterpart doors by flow and contribution (flow × distance), share of row/col, and an R80 radius (how many doors/feet cover 80% of that lane’s traffic around its weighted-median counterpart door).
  * Ops spreadsheet: fromX_toY_cube_clustered.xlsx
  * TO_cols_clustered sheet: TO columns grouped and color-banded with a legend
 *  raw_FROMxTO sheet: unclustered for audit
  
* Math report: math_report.md / .json
    *Parameters, objective, baseline vs optimized feet, savings (ft and travel-time equivalent), and solver history
* Heatmaps:
  * Log and linear scaled views of the 38×38 matrix so you can see both big lanes and “long tail”.
* Capacity flags:
  * Any day where INB exceeds ~22k lb (linehaul) or OUT exceeds ~48k lb (P&D) is written as a CSV.

**Customization for other Service Centers**
* Door counts: set N_INBOUND_DOORS, N_OUTBOUND_DOORS to your site’s physical capacity.
* Geometry: adjust BAY_SPACING_FT, AISLE_CROSSING_FT, TURN_PENALTY_FT per site standards (typical: 12–14 ft spacing; 9-ft doors; tweak crossing/turn penalties to match observed handling).
* Dimensioners / no-cross: populate NO_CROSS_WINDOWS with (lo, hi, name, detour_ft) per site.
* P&D reserve: set NUM_PD_OUTBOUND_DOORS>0 to reserve the last K outbound doors for high-volume city lanes (enforced by big-M penalties).
* Congestion vs clustering: tune LAMBDA_CONG_* and LAMBDA_*_CLUSTER. Raise congestion to spread hotspots; raise cluster to keep similar lanes tightly grouped.

**Cluster Coloring Logic**
* TO clusters group destinations with similar inbound supply patterns (columns with similar profiles).
* FROM clusters group origins that feed similar sets of destinations (rows with similar profiles).
* The diagram colors both: top band colors (TO) and bottom band colors (FROM). There isn’t a one-to-one mapping between TO-cluster #k and FROM-cluster #k; each side is clustered independently. The legend in Excel lists the members per band.

**Sanity Checks**
* Conservation check: F.sum() equals total allocated inbound across the period.
* Non-negativity: F.min() >= 0, D.min() >= 0.
* Sensitivity: Lower/raise BAY_SPACING_FT or NO_CROSS_WINDOWS detours and confirm door placements shift the way you expect.
* Heatmaps: Compare linear vs log: log reveals mid/small flows that linear can “wash out.”
* R80: For a given door, R80 should be “compact” (few doors/feet) if you see tight coupling, and larger when traffic is diffuse.

**FAQ:**

**Q: “TO MCI” vs “FROM MCI” labels?**
“FROM X” means arrives to MCI (inbound lanes).
“TO Y” means departs from MCI (outbound lanes).
The diagram’s bottom row is FROM (inbound to MCI). The top row is TO (outbound from MCI).

**Q: Can we use 10-ft or 8-ft doors?**
The geometry parameters only need accurate center-to-center spacing and realistic crossing/handling penalties; door width is operational but not required by the objective unless you want to add per-door capacities.

**Q: What about congestion or staging space?**
Congestion smoothing spreads heavy doors. You can also set break doors at seams to create natural staging zones.



**Configuration(per site)**
| Key                                   | Purpose                                      | Typical       |
| ------------------------------------- | -------------------------------------------- | ------------- |
| `N_INBOUND_DOORS`, `N_OUTBOUND_DOORS` | Physical door counts                         | e.g., 38 / 38 |
| `BAY_SPACING_FT`                      | Center-to-center spacing                     | 12–14 ft      |
| `AISLE_CROSSING_FT`                   | One cross between rows                       | \~60 ft       |
| `TURN_PENALTY_FT`                     | Handling overhead (ft)                       | 10 ft         |
| `NO_CROSS_WINDOWS`                    | Dimensioners/ramps (door span + detour feet) | see below     |
| `NUM_PD_OUTBOUND_DOORS`               | Reserve last K outbound doors for P\&D       | 0–6           |
| `LAMBDA_CONG_*`                       | Congestion smoothing weight                  | 0–0.05        |
| `LAMBDA_*_CLUSTER`                    | Soft clustering weight                       | 0–0.05        |



**Ops Notes:**
Labels: Bottom row on the diagram is FROM (inbound to MCI). Top row is TO (outbound from MCI). Labels read “FROM CHI”, “TO STL”, etc.

Clusters: TO clusters group destinations with similar inbound supply; FROM clusters group origins that feed similar destinations. They’re independent (no 1-to-1 mapping required).

Break doors: Suggested seams between large bands to improve staging and tug flow.

P&D: Set NUM_PD_OUTBOUND_DOORS > 0 to keep city lanes at the last K doors (loaded first a.m.; pickups later).

**Environment:**
Python ≥ 3.9
pandas, numpy, matplotlib, xlsxwriter, openpyxl, scipy

Glossary and Methods Explained:
**Flow matrix 𝐹:**
A table where entry 𝐹𝑖𝑗  is the amount of work (cube/shipments/weight) that moves from FROM lane 𝑖 to TO lane 𝑗 over the period.
Per day we preserve inbound totals and allocate them across outbound lanes in proportion to that day’s outbound mix. If same-day is sparse, we allow ±1 day, then weekly, then a full-window proportional fallback.
𝐹 is the “demand” the dock layout must serve; it’s what we try to move the shortest distance.

**Distance matrix 𝐷 (Manhattan/“along the dock”):**
A table where 𝐷𝑝𝑞 is the travel distance (ft) from inbound door 𝑝 to outbound door 𝑞.
Mahattan Distance: We approximate forklift travel as “go across the aisle, then along the dock,” which is an L1 (Manhattan) metric: 𝑑(𝑝,𝑞)=AISLE_CROSSING+BAY_SPACING⋅∣𝑝−𝑞∣+TURN_PENALTY 
**No-cross (dimensioners/ramps):** If the straight path between 𝑝 and 𝑞 would cross a blocked span [𝑎,𝑏) we add a detour (extra feet). This steers the optimizer to avoid illegal/unsafe shortcuts.
 𝐷 turns “who moves to whom” (from 𝐹) into feet and therefore time/cost.

**Assignment (Hungarian algorithm)**
A classic optimization that finds the minimum-cost one-to-one pairing between two sets (here: lanes → doors) given a cost matrix.
We alternate:
Fix outbound door order 𝜎, then use Hungarian on inbound costs to place inbound lanes 𝜋.
Fix inbound 𝜋, then Hungarian on outbound costs to place outbound lanes 𝜎.
Why Hungarian: It guarantees a globally optimal one-to-one assignment for the current cost matrix in 𝑂(𝑛3) time—fast and reliable for 𝑛≈38.

**Greedy algorithm (two places)**
Max-coverage selection (rows/cols): When there are more lanes than doors, we need the best 𝑅×𝐶 submatrix of 𝐹. The greedy heuristic repeatedly adds the row/column with the biggest remaining gain in covered flow. Fast, simple, and effective at capturing the majority of mass.
Fallback assignment: If the SciPy Hungarian isn’t available, we use a simple greedy: for each row, choose the cheapest unused column. It’s not globally optimal but works in a pinch.

**Alternating optimization (block coordinate descent)**
Optimize one side while holding the other side fixed, then swap. We iterate a few times.
Jointly solving for both inbound and outbound permutations is hard; alternating Hungarian solves exploit structure and converge quickly in practice.

**Congestion smoothing (penalty)**
A soft penalty added to the objective to discourage clustering too much load on adjacent doors. We smooth the door workloads with a small moving window and penalize large peaks.
Real docks need space; this helps spread hotspots and keeps aisles workable.

**Soft clustering (penalty**)
We compute lane-lane similarity (𝑆in=𝐹𝐹⊤Sin=FF⊤, 𝑆out=𝐹⊤𝐹Sout=F⊤F). If two lanes interact with many of the same partners, we add a penalty proportional to their distance in door numbers if they’re far apart.
Encourages similar lanes to sit near each other (shorter average travel, cleaner work zones) without hard constraints.

**Eigenvector seriation (spectral ordering)**

A simple, proven way to order lanes so similar ones sit next to each other. We compute a similarity matrix and order lanes by the top eigenvector (or first principal component).
Is used to visualize 𝐹 (clustered heatmaps) and to color-band TO/FROM clusters in Excel and the dock diagram.
Makes the “blocks of flow” pop out for humans; aids reviews and change-management.

**Break doors**
Natural seams between door groups where staging, tugs, or staffing breaks make sense. We pick peaks in smoothed outbound workload and mark them as BREAK.
Operations can use these seams to manage congestion and organize space.

**“R80” flow radius (explain the door’s neighborhood)**
For a given door, the minimum number of adjacent counterpart doors (converted to feet) that account for 80% of its flow around its weighted-median counterpart position.
Why: A quick diagnostic—tight 𝑅80 means focused interactions; wide 𝑅80 means diffuse traffic.

**Linear vs Log heatmaps**
Linear: Shows absolute magnitude—big lanes dominate the color scale.
Log: Compresses the dynamic range—reveals mid/small flows that matter operationally but would be invisible on a linear scale.

**Proportional allocation**
When daily pairing data aren’t complete, we allocate a day’s inbound totals across outbound lanes by that day’s observed outbound composition.
Preserves day-level totals, avoids making up non-existent pairings, and yields a realistic, dense 𝐹 for optimization.

**P&D reservation **
Reserve the last 𝐾 outbound doors for high-volume city lanes (enforced by big-M costs).
Ensures first-thing-AM deliveries are staged together, with pickups flowing back to nearby doors later.

**Executive interpretation of the Math**

Goal: Minimize total forklift feet (and thus time) by placing inbound and outbound lanes on doors so that heavy interactions are physically close, while respecting no-cross zones and avoiding congestion.

How it wins: Shorter 𝐹×𝐷 → fewer seconds of travel per move → more turns per hour → fewer touches and safer aisles. Penalties nudge the layout to be operationally practical, not just mathematically minimal.

**Micro-examples (sanity)**

**Manhattan distance:** Doors are 12 ft apart. From door 5 to door 8: along-dock is ∣5−8∣⋅12=36 ft. Add 60 ft aisle crossing and 10 ft turn penalty → 106 ft.

**Hungarian intuition:**
Suppose three inbound lanes each strongly ship to a different outbound. Hungarian finds the one-to-one mapping that minimizes total feet globally; a greedy pick for row 1 could block the best choice for row 2—Hungarian avoids that trap.

**Greedy max-coverage:**
If you can only keep 38×38 lanes from a much larger 𝐹, add the row/col with the biggest marginal flow until you hit 38 and 38. You keep most of the total mass and ignore long-tail noise.

**Tuning Opportunities**

**Geometry:** BAY_SPACING_FT, AISLE_CROSSING_FT, TURN_PENALTY_FT, plus NO_CROSS_WINDOWS → realism of 𝐷.

**Behavior:** LAMBDA_CONG_* and LAMBDA_*_CLUSTER → spread vs. compactness.

**Ops constraints:** NUM_PD_OUTBOUND_DOORS → morning delivery priority.

**Visualization:** cluster counts or thresholds → thickness of color bands.




   

   

   
   
  


  
