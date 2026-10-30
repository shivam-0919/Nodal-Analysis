# Nodal Analysis for Oil & Gas Wells

A Python implementation of **Nodal Analysis** for production system optimization. The project combines the well's inflow performance (reservoir to wellbore) with its outflow performance (wellbore to surface) to find the natural operating point, then studies how changes in skin, tubing size and wellhead pressure move that point.

The repository contains two Jupyter notebooks:

| Notebook | Fluid | Purpose |
|---|---|---|
| `Nodal_Analysis.ipynb` | Gas well | Plots an IPR curve against tabulated TPR curves for three tubing sizes to compare them visually |
| `IPR_TPR_Model.ipynb` | Oil well | Full computational model: Vogel IPR, Poettmann & Carpenter VLP, operating-point solver and sensitivity studies |

---

## Background

Nodal analysis splits a producing well at a "node" (here, the bottomhole) and matches two curves:

- **IPR (Inflow Performance Relationship):** flowing bottomhole pressure (Pwf) the reservoir can sustain at a given rate.
- **VLP / TPR (Vertical Lift Performance / Tubing Performance Relationship):** Pwf required to lift fluids to the surface at that rate.

The intersection of the two curves is the **operating point** (rate and Pwf at which the well naturally produces).

---

## 1. `Nodal_Analysis.ipynb` (Gas Well)

- IPR data taken from PetroWiki (Pwf vs. rate in Mscf/D).
- TPR data for **1.90", 2.375" and 2.875"** tubing.
- Plots IPR and the three TPR curves with `matplotlib`; the intersection of each TPR with the IPR shows the expected flow rate and how larger tubing increases deliverability.

**Libraries:** `numpy`, `pandas`, `matplotlib`

---

## 2. `IPR_TPR_Model.ipynb` (Oil Well)

### Inflow: Vogel IPR
`Vogel_IPR(Qo, Pwf, Pr, Pb)` generates the IPR curve for:
- **Saturated reservoirs** (Pr ≤ Pb): standard Vogel equation.
- **Undersaturated reservoirs** (Pr > Pb): composite linear (above bubble point) + Vogel (below bubble point) model, handling test points above or below Pb.

The flow efficiency `F` is computed from the skin-related pressure drop (`delta_P_skin`, e.g. from a DST).

### Outflow: Poettmann & Carpenter VLP
`Poettmann_and_Carpenter_Method(...)` marches from the wellhead to the bottom of the well in pressure increments, calculating at each step:
- Solution GOR (Rs), oil FVF (Bo)
- Gas pseudo-critical properties, Z-factor, Bg
- Average mixture density and the friction term
- Depth increment, until the total well depth is reached

This returns the Pwf required for each flow rate.

### Operating Point
IPR and VLP are smoothed with cubic splines (`scipy.interpolate.splrep/splev`), and `scipy.optimize.fsolve` finds their intersection via `calculate_intersection()`.

### Sensitivity Analyses
| Function | What it does |
|---|---|
| `skin(delta_P_skin_red)` | Effect of stimulation/skin reduction on the IPR and operating point |
| `tubing_effect(d_new)` | Effect of tubing inner diameter on the VLP and operating point |
| `Pwh_effect(Pwh_new)` | Effect of wellhead pressure on the VLP and operating point |
| `optimum_tubing(d_range)` | Loops through standard tubing sizes until the rate gain drops below 100 STB/day and reports the optimum size |

### Example Input Data
| Parameter | Value |
|---|---|
| Reservoir pressure (Pr) | 3000 psi |
| Bubble point (Pb) | 2130 psi |
| Test rate / Pwf | 250 STB/D @ 2500 psi |
| Skin pressure drop | 200 psi |
| Wellhead pressure | 100 psi |
| Tubing ID | 2.441 in |
| Well depth | 5000 ft |
| Temperature | 150 °F |
| Oil API | 35 |
| GLR | 273 scf/STB |

### Sample Results
- **Base case operating point:** ≈ 954.7 STB/D at Pwf ≈ 606 psi
- **After removing skin (200 psi):** ≈ 1594.7 STB/D at Pwf ≈ 594 psi
- **2" tubing:** ≈ 931.6 STB/D at Pwf ≈ 726 psi
- **Wellhead pressure reduced to 50 psi:** ≈ 970.2 STB/D at Pwf ≈ 516 psi
- **Optimum tubing size from the standard list:** 1.315 in

> Results depend on the input data and correlations used; treat them as illustrative.

**Libraries:** `numpy`, `pandas`, `matplotlib`, `scipy`

---


## Limitations & Notes

- Poettmann & Carpenter is an empirical multiphase-flow correlation with known accuracy limits; more modern correlations (Hagedorn-Brown, Beggs-Brill, mechanistic models) may be preferable for design work.
- The VLP calculation is a simple loop and can be slow for many rates.
- Spline smoothing (`s=9`) is a fitting choice and can shift the intersection slightly.
- In the `skin()` and `tubing_effect()` functions, some plotted VLP curves reuse the base-case spline, so verify the plotted curves against the printed operating points when extending the code.
