# NearShoreML: brief for Claude

This fork of GeoOcean's BlueMath_tk is where DEME builds NearShoreML: a hybrid SWAN + ML surrogate that reproduces SWAN's offshore-to-nearshore wave transformation. Train it once per site on a selected set of SWAN cases, then predict the full 30-year nearshore series in minutes.

Source of this brief: the deck `Wave_Modelling_ML_Flowcharts_v2_2026-10-16.pptx` (7 slides), agreed with Panos on 16 Oct 2026. Every item marked "to discuss" or "placeholder" is still open: do not decide it silently. Pick the stated default, say so in your output, and flag it.

The person driving this work is not a programmer or a SWAN modeller. Explain results in plain words, and back every claim with the file, line or number you checked.

---

## 1. Why: the current workflow (slide 1)

How a nearshore wave climate is produced today:

- Requested point or area. Offshore: use offshore data directly (ARGOSS, Aktis or another dataset). Nearshore, simple site or fast answer needed: ray modelling (exposed site, no breaking, simple bathymetry). Otherwise SWAN modelling is required, in one of three ways:

| # | Approach | Pros | Cons |
|---|---|---|---|
| 1 | Full time series (mostly used) | Most accurate offshore to nearshore; full 2D spectra on the boundary; water level and wind included | Weeks of run time; large amounts of output data |
| 2 | Transformation matrix (rare) | Quick: tens to thousands of SWAN cases (depending on the variables) | Spectral detail lost, only bulk parameters (Hs, Tp, Dir, spreading); separate sea and swell runs, summed, add error; interpolation errors |
| 3 | Hybrid SWAN + ML (under development: **this repo**) | Goal: full time series accuracy at matrix cost | |

Hybrid SWAN + ML, the goal:
- Run SWAN for a selected set of cases only.
- ML learns offshore to nearshore from those cases.
- ML predicts the full 30-year series in minutes.
- Scored against one full SWAN time series (section 3).

## 2. Tool options (slide 2)

Two choices: **what** the ML sees at the boundary (spectra, bulk per partition or bulk) and **where** (spatially varying boundary or one super-point).

All options: SWAN with the relevant physics on, including wind input and depth-induced breaking.

| | Full spectra (ML predicts local 2D spectra) | Bulk per partition (ML sees wind sea + each swell; output to check) | Bulk parameters (ML predicts local Hs, Tp, Dir) |
|---|---|---|---|
| **Spatially varying boundary** (a spectrum at each boundary location) | **1** Varying spectra as ML input. SWAN: spatially varying spectra | **1.5** Varying bulk per partition as ML input. SWAN: the real varying spectra. No input PCA needed. Later: also fits bulk forecast input | **3** Varying bulk parameters as ML input. SWAN: separate sea and swell runs * |
| **Super-point** (one spectrum for the whole edge) | **2** Super-point spectrum as ML input. SWAN: uniform spectrum (or keep varying spectra) | **2.5** Super-point bulk per partition as ML input. SWAN: super-point spectrum. No input PCA needed | **4** Super-point bulk parameters as ML input. SWAN: separate sea and swell runs * |

\* Combining separate sea and swell runs misses breaking of the combined wave field, so Hs can be unrealistic. Does not apply to 1.5 and 2.5: SWAN still runs the combined spectrum.

**Fallback rule (order to discuss):** test option 1 → if it passes, adopt it; if it fails, test 1.5 → 2 → 2.5 → 3 → 4. If all fail: revise the approach or the inputs. "Passes" means within the agreed criterion against the full SWAN benchmark (section 3).

## 3. Benchmark and pass criterion (slide 3)

One full SWAN time series is the reference for every option and for "passes".

Run the full series once, then choose the training time steps and compare with the full set:

1. **Full SWAN benchmark.** Run SWAN as a full time series once at the pilot site. Paradip: a full run exists (to confirm).
2. **Pick training time steps.** Choose N time steps from that same series (MDA on the hindcast). No new SWAN runs needed.
3. **Train the surrogate.** Fit GP / RBF on the N selected SWAN cases (sections 4 and 5).
4. **Predict the full series.** The surrogate predicts every time step of the benchmark period.
5. **Score against the benchmark.** Compare predicted and full SWAN series, time step by time step. Within the criterion: adopt. Else: next option.

**Candidate metrics (placeholders, to agree with the team):** Hs RMSE and bias; top 1 % Hs error (storm peaks); Tp and Dir error; workability % error. Thresholds: none yet. Agree the numbers before the first test.

**Also testable on the benchmark, with no new SWAN runs:** MDA picks vs random picks; N = 50, 100 or 200 SWAN cases; with and without output PCA; options 1 to 4 on the same reference. Length of the benchmark window: to discuss.

**Check before training: stationary cases vs travel time.**
- [ ] Compute the swell crossing time of the domain.
- [ ] Over about 1 hour: add lagged boundary inputs (t-1, t-2) or use instationary runs.
- [ ] Check storm-peak timing against the benchmark.

A stationary case assumes the waves cross the domain instantly. Long travel times shift and smooth storm peaks.

Metrics and thresholds are placeholders. Every option (1 to 4) is scored against the same benchmark.

## 4. Option 1 workflow, short overview (slide 4)

Spectral surrogate: train on selected SWAN cases, predict the full time series.

**Phase 1: build and validate the surrogate**
1. **Aligned input series:** spatially varying boundary spectra (same time step) + wind (u, v) + water level.
2. **Input features:** joint PCA of the boundary spectra (spectra input only); add wind and water level; scale. *To size: number of PCs.*
3. **MDA selection:** pick N representative time steps. *To size: N SWAN cases.*
4. **SWAN simulations:** full-boundary cases, relevant physics on; extract local spectra.
5. **Output PCA + GP / RBF training:** output PCA of local spectra (optional); fit GP / RBF; validate on independent SWAN cases. *Pass/fail: section 3.*

**Phase 2: prediction**
6. **Prediction:** fitted preprocessing + GP / RBF on every time step; inverse output PCA gives the local spectral series, then bulk parameters.

**Branches to decide**

| Branch | Step | A | B | Note |
|---|---|---|---|---|
| Input PCA? | 2 | Joint PCA over all boundary locations: keeps the boundary linked in time | PCA per boundary location: simpler, but more inputs for the GP / RBF | Not needed with bulk input (1.5, 2.5, 3, 4) |
| Which cases? | 3 | Observed co-occurrence (MDA on the hindcast): wind, waves and water level as they occur together. **Default** | Varied independently (LHS): also covers combinations not yet seen | |
| GP or RBF? | 5 | GP: gives an uncertainty that flags cases outside the training range | RBF: simpler and exact at the training cases, no uncertainty | |
| Output PCA? | 5 | With: about 1,260 correlated values per spectrum become a few coefficients; PCA on log or sqrt keeps energy positive | Without: one GP / RBF per spectral value | Test with and without |

Options 1.5 to 4: same flow. ML input: bulk per partition with the real spectra in SWAN (1.5, 2.5), one super-point spectrum (2) or bulk parameters with separate sea and swell runs (3, 4). Bulk input: no input PCA. Assumes stationary cases and a fixed model configuration (travel-time check: section 3).

## 5. Spectral surrogate workflow, detailed (slide 5)

**Phase 1: build and validate the surrogate**
- **Aligned input series:** each time step: spatially varying boundary spectra + wind (u, v) + water level.
- **Input features\*:** spectra input: joint PCA over all boundary locations. Bulk input (1.5, 2.5, 3, 4): no input PCA. Add wind and water level; scale.
- **MDA selection\*:** pick N time steps. Keep all boundary locations linked; wind and water level included.
- **SWAN simulations:** run selected full-boundary cases with relevant physics; extract local spectra.
- **Output PCA\* (optional):** compress SWAN spectra at locations of interest (about 1,260 values each) into spectral coefficients. Test with and without.
- **GP / RBF training\*:** input features → spectral coefficients (or spectral values); validate on independent SWAN cases.

**Phase 2: prediction, apply the fitted workflow**
- **Full input series:** historical boundary spectra (or bulk per partition) + wind (u, v) + water level.
- **Fitted preprocessing\*:** apply the saved input PCA (if used) and the same feature scaling.
- **GP / RBF prediction:** predict local spectral coefficients with the trained surrogate.
- **Inverse output PCA\*:** if output PCA was used: reconstruct spectra, reverse any output scaling, reshape to frequency × direction.
- **Local spectral series\*:** complete time series at locations of interest; derive bulk parameters if needed.

**\* Points**
- **Spatial coherence:** all boundary spectra within each candidate belong to one time step. Never mix locations from different times.
- **Which cases:** observed co-occurrence (MDA on the hindcast, default) or varied independently (LHS). MDA picks whole real time steps and never mixes variables (Camus 2011).
- **Group weighting:** only if spectra stay in the input: 20 spectral PCs outvote 1 water level in the MDA distance (MFA is one fix). With bulk input, each variable counts about equally.
- **PCA:** input PCA may miss rare conditions. Output PCA is optional; its reason is output complexity (about 1,260 correlated values per spectrum), not data size. Check extremes, peaks, secondary systems, energy, non-negativity and extrapolation.
- **Validation:** independent SWAN cases, then the full SWAN benchmark (section 3).
- **Assumptions:** fixed model configuration and stationary cases. Check the swell crossing time first (section 3).

## 6. Glossary (slides 6 and 7)

**Waves and SWAN**
- **Offshore / nearshore:** deep water, where wave datasets exist, versus the shallow site near the coast, where the waves are needed.
- **SWAN:** the numerical wave model that computes the offshore to nearshore transformation over a grid.
- **Ray modelling:** a quick method that follows wave rays. Suits exposed sites with no breaking and simple bathymetry.
- **Hindcast:** a past wave and wind time series made by a model, for example ARGOSS or Aktis.
- **2D spectrum:** how the wave energy is split over frequency and direction at one point and one time step. About 1,260 values in the workshop set-up (35 frequencies × 36 directions).
- **Bulk parameters:** a few numbers that summarize a spectrum: Hs (significant wave height), Tp (peak period), Dir (mean direction) and spreading.
- **Partition, wind sea, swell:** one wave system within a spectrum: the wind sea (still driven by the local wind) or one swell (from a distant storm). Bulk per partition: separate Hs, Tp, Dir for each system.
- **Separate sea and swell runs:** SWAN run once for wind sea and once for swell, results added. Misses breaking of the combined field, so Hs can be unrealistic.
- **Relevant physics:** the SWAN processes switched on, including wind input and depth-induced breaking (waves breaking because the water gets too shallow).
- **Water level:** tide plus surge. It changes the depth, so it changes the breaking.
- **Boundary:** the offshore edge of the SWAN grid, where the waves are fed in.

**Current approaches**
- **Full time series:** SWAN run for every hour of the hindcast. Most accurate, but weeks of run time.
- **Transformation matrix:** SWAN run for a grid of bulk-parameter combinations; every hour is then interpolated from the grid.
- **Hybrid SWAN + ML:** SWAN run for a selected set of cases only; ML learns offshore to nearshore from them and predicts the full series.
- **Surrogate:** the trained ML model that stands in for SWAN.

**Tool options**
- **Spatially varying boundary:** different spectra fed in at several boundary locations.
- **Super-point:** one boundary location whose spectrum or bulk parameters are used for the whole boundary.
- **Options 1 to 4:** combinations of what the ML sees (spectra or bulk parameters) and where (varying boundary or super-point).
- **Options 1.5 and 2.5:** SWAN gets the real spectra (1.5) or the super-point spectrum (2.5); the ML only sees bulk parameters per partition.
- **Fallback rule:** test option 1 first; if it fails, move to the next option.
- **Passes:** the criterion an option must meet on data, accuracy (ML vs the full SWAN benchmark) and cost. No numbers yet.

**Benchmark**
- **Benchmark (reference series):** a full SWAN time series that every option is scored against.
- **Time step:** one hour of the input series. All boundary spectra in a case come from the same time step.
- **SWAN case:** one selected time step run in SWAN to make training data.
- **Stationary case:** a SWAN run of one time step that assumes the waves cross the domain instantly.
- **Travel time (crossing time):** how long swell takes to cross the domain. Over about an hour, stationary cases shift and smooth storm peaks.
- **Instationary run:** a SWAN run that marches through time, so travel time is included (to discuss).
- **Fixed model configuration:** the same grid, bathymetry and settings for every SWAN case.
- **RMSE and bias:** root mean square error (typical size of the error) and the average offset between surrogate and SWAN (to discuss).
- **Workability:** the share of time the sea state allows the planned work (to discuss).

**ML workflow**
- **Aligned input series:** boundary spectra, wind (u, v) and water level lined up on the same time steps.
- **Input features:** the numbers the ML actually sees for each time step, after compression and scaling.
- **PCA (principal component analysis):** replaces many correlated values with a few combined ones (principal components, PCs) that keep most of the variation.
- **Input PCA (joint PCA):** one PCA over all boundary locations together, so the boundary stays linked in time. Not needed when the ML input is bulk parameters.
- **Output PCA:** PCA of the SWAN spectra at the locations of interest, giving a few spectral coefficients per spectrum. Optional: needed for output complexity, not data size. Inverse output PCA turns the coefficients back into a spectrum.
- **Scaling / group weighting:** putting all inputs on the same range, for example 0 to 1. Equal scaling is not equal influence: 20 spectral PCs outvote 1 water level.
- **MFA (multiple factor analysis):** a PCA that first evens out the weight of each group (spectra, wind, water level). Only matters if spectra stay in the input.
- **MDA (maximum dissimilarity algorithm):** picks N time steps from the hindcast that are as different from each other as possible, so the extremes are covered. Always whole real time steps.
- **Observed co-occurrence vs varied independently (LHS):** training cases taken from real hours (MDA on the hindcast) versus combinations built by varying each variable separately (Latin hypercube sampling).
- **GP (Gaussian process):** a regression method that interpolates between the training cases and also gives an uncertainty, which flags cases outside the training range.
- **RBF (radial basis functions):** a simpler interpolation method that is exact at the training cases but gives no uncertainty.
- **Independent SWAN cases:** SWAN cases kept out of training and used only to check the surrogate.
- **Fitted preprocessing:** the saved input PCA and scaling from training, applied unchanged to new data.
- **Locations of interest / local spectral series:** the nearshore points where results are needed, and the predicted spectra there for every time step. Bulk parameters are derived from them.

## 7. Open questions (not in the deck; from its change log)

1. Where do 1.5 and 2.5 go in the fallback order? The deck keeps numeric order; the 16 Oct meeting notes suggest starting with 1.5 as the cheapest test.
2. For 1.5 and 2.5, should the ML output be local spectra or bulk parameters? Panos: "to be checked".
3. Which metrics and thresholds define "passes"?
4. Is the Paradip full SWAN run instationary and long enough to serve as the benchmark?
5. Length of the benchmark window.

## 8. Repo facts to know before coding (checked 16 Oct 2026, commit 35d3077)

- Useful existing code: `bluemath_tk/datamining/mda.py`, `datamining/pca.py`, `datamining/lhs.py`, `interpolation/rbf.py`, `interpolation/gps.py`, `waves/superpoint.py`, `waves/binwaves.py`, `wrappers/swan/`.
- `bluemath_tk/waves/partitioning.py` is empty (0 lines). Options 1.5 and 2.5 need spectral partitioning (wind sea + swells); `wavespectra` (optional extra `waves`) is believed to have a partitioning routine (not checked here: verify first).
- No HyWaves code is in this repo.
- `pytz` is imported but not listed in `pyproject.toml` dependencies, so a fresh install fails on import until it is added.
- Tests run with `pytest`; lint with `ruff` (see `.github/workflows/`).
