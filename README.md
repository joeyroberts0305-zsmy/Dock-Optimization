**Dock Optimization Project**

**Executive Brief:** 

**What it does:**
Data-driven dock layouts that cut forklift travel, reduce congestion, and stay easy to run.
**Leaders get:**
**Travel reduction:** Optimized door ordering that minimizes total forklift feet.
**Fewer hotspots:** Soft congestion smoothing spreads heavy doors.
**Operational clarity:** Labeled dock diagram with color-banded clusters, break-door seams, and dimensioner (no-cross) bars.
**Auditability: **“Why here?” workbook—top counterpart doors and an R80 radius (how concentrated a lane’s flow is).
**Repeatable:** Works across service centers with different door counts, spacing, and no-cross zones. Soft congestion smoothing spreads heavy doors.


**Repository Layout**
Dock Optimization Project/
├─ data/
│  ├─ Inbound.csv        # CSV/XLSX also supported
│  └─ outbound.csv
├─ outputs/              # images, Excel, reports
└─ Dock_Optimization_V2.ipynb


**Quick start**
1. Drop your files in Dock Optimization Project/data/
    Inbound (auto-detected): INB_DATE, INB_LANE, INB_CUBE, INB_SHIPMENTS, INB_WEIGHT_LB
    Outbound (auto-detected): OUT_DATE, OUT_LANE, OUT_CUBE, OUT_SHIPMENTS, OUT_WEIGHT_LB
    Terminal/type columns are also detected so inbound to MCI (D/B) and outbound from MCI (O/B) are filtered correctly.

2. Open Dock_Optimization_V2.ipynb and Run All.

3. Outputs (in /outputs)

* dock_layout_MCI_updated.png – TO/FROM clusters (color bands), break doors, dimensioners
* fromX_toY_cube_clustered.xlsx – workflow spreadsheet (TO columns grouped & colored)
* why_here_report.xlsx – top counterparts, shares, R80 radius
* math_report.md/json – objective, parameters, baseline vs. optimized feet & time
* Heatmaps and capacity flags
  
** Techical summary **
  **Flow matrix 𝐹:** For each day, preserve inbound totals and allocate to outbound lanes by that day’s outbound mix. If same-day is sparse, allow ±1 day, then weekly, then a full-window proportional fallback.
  
<img width="568" height="124" alt="image" src="https://github.com/user-attachments/assets/621f5f08-e0c3-4279-8668-7749cbcf53ad" />

<img width="230" height="47" alt="image" src="https://github.com/user-attachments/assets/4f0d2ebc-476f-4dce-be5e-fe23986944a2" />

 **Distance matrix 𝐷:** Manhattan “along the dock”: aisle crossing + bay spacing × door delta + a turn penalty.
 
<img width="852" height="70" alt="image" src="https://github.com/user-attachments/assets/7113d247-bf10-4b72-b256-02935447c0a3" />

**No-cross spans** (dimensioners/ramps) add a detour if a path would cross them.

<img width="390" height="172" alt="image" src="https://github.com/user-attachments/assets/cb9d944d-4fa1-4d68-801f-11fd89acfc90" />

<img width="464" height="63" alt="image" src="https://github.com/user-attachments/assets/b19087b0-f908-4757-9b9e-9739a4295b0b" />

**Optimization objective:** Place inbound and outbound lanes on doors to minimize total 𝐹×𝐷, with optional soft penalties for congestion smoothing.
<img width="333" height="73" alt="image" src="https://github.com/user-attachments/assets/02b15f51-a33c-488b-9fcd-399f1bfb3288" />

**Congestion smoothing:**
<img width="485" height="55" alt="image" src="https://github.com/user-attachments/assets/991f6f7f-d62f-459d-8c0e-51c740c3aeac" />

**Soft clustering:**
<img width="760" height="109" alt="image" src="https://github.com/user-attachments/assets/3a9145a8-0da0-4edc-9428-319f8d6ae5a6" />

**Total objective:**
<img width="255" height="46" alt="image" src="https://github.com/user-attachments/assets/9cc84d77-92f3-4389-86a7-58961fc9759e" />

**Algorithm:** Alternate two Hungarian solves (inbound then outbound) and finish with pairwise swap “polish.” Converges quickly at 38×38.

**Break doors:** Pick peaks in smoothed outbound workload as natural seams for staging/tug flow.

**Submatrix selection:** If you have more lanes than doors, a greedy “max-coverage” step keeps the 𝑅×𝐶 lanes that capture most of the flow.

**Method Glossary: **
**Manhattan distance:** Approximate travel as “across the aisle, then along the dock.” Distance = crossing + spacing × door difference + turn penalty.

**Hungarian algorithm:** Finds the globally optimal one-to-one assignment for a given cost matrix in 𝑂(𝑛3). Here, we use it to place lanes on doors.

**Greedy (max-coverage)**: When trimming to a fixed dock size, repeatedly add the row/column with the largest marginal flow captured—fast and effective.

**Alternating optimization:** Solve inbound with outbound fixed, then outbound with inbound fixed; repeat a few rounds.

**Congestion smoothing:** Penalizes big peaks in adjacent door workloads so heavy traffic gets spread.

**Soft clustering:** Uses lane-lane similarity to gently encourage similar lanes to sit near each other (cleaner work zones).

**R80 radius:** For a door, how many nearby counterpart doors (and feet) cover 80% of its flow. Tight R80 = focused interactions; wide R80 = diffuse. 

**Site configuration:**
| Key                                   | Purpose                                  | Typical       |
| ------------------------------------- | ---------------------------------------- | ------------- |
| `N_INBOUND_DOORS`, `N_OUTBOUND_DOORS` | Physical door counts                     | e.g., 38 / 38 |
| `BAY_SPACING_FT`                      | Center-to-center spacing                 | 12–14 ft      |
| `AISLE_CROSSING_FT`                   | One cross between rows                   | \~60 ft       |
| `TURN_PENALTY_FT`                     | Handling overhead (ft)                   | \~10 ft       |
| `NO_CROSS_WINDOWS`                    | Dimensioners/ramps (door spans + detour) | site-specific |
| `NUM_PD_OUTBOUND_DOORS`               | Reserve last K outbound doors for P\&D   | 0–6           |
| `LAMBDA_CONG_*`, `LAMBDA_*_CLUSTER`   | Spread vs. compactness                   | 0–0.05        |

**FAQ**

* “TO” vs “FROM” labels? Bottom row = FROM (inbound to MCI). Top row = TO (outbound from MCI).
* Do cluster colors match top/bottom 1-to-1? No—TO and FROM are clustered independently. The Excel legend lists members per band.
* Door widths (8/9/10 ft) Only spacing and crossing/turn penalties affect the objective. Width can be modeled via capacities if needed.

  
