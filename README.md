# OpenMC STARFIRE Gap Streaming

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/alanhalle/openmc-starfire/blob/main/openmc_starfire.ipynb)
[![Paper DOI](https://img.shields.io/badge/paper-10.5281%2Fzenodo.22810354-blue)](https://doi.org/10.5281/zenodo.22810354)
[![Code DOI](https://img.shields.io/badge/code-10.5281%2Fzenodo.21418400-blue)](https://doi.org/10.5281/zenodo.21418400)

An open, reproducible test case for **deep-penetration weight-window variance reduction**, built on a shielding problem with real design provenance: 14-MeV neutron streaming through straight and stepped gaps between the outboard shield sectors of the STARFIRE conceptual tokamak.

The whole thing runs end-to-end on the Google Colab free tier.

**Paper:** [doi.org/10.5281/zenodo.22810354](https://doi.org/10.5281/zenodo.22810354)

## What this is

The quantities that set shielding design margins — the dose behind an unbroken shield, and the residual leakage past a gap that has been stepped to defeat line-of-sight — lie many decades below the source. Analog Monte Carlo resolves them worst. The established remedies, CADIS and FW-CADIS, need an external deterministic transport solve to build the adjoint source, which puts them outside a pure Monte Carlo workflow.

This repo shows that the mesh-based **MAGIC** method implemented natively in OpenMC converges such quantities with no deterministic solver, and quantifies what that buys. The physics is checked against a 1984 MORSE-CG study of the same configuration (see [Citation](#citation)), whose original inputs and code no longer survive.

OpenMC's fidelity for fusion streaming is already established elsewhere against dedicated experiments. The contribution here is complementary: an independent reproduction of a specific historical design study, and a deep-penetration test case that anyone can rerun for free.

## Repository contents

| File | Description |
|------|-------------|
| `openmc_starfire.ipynb` | Full Colab notebook — install, materials, geometry, weight-window generation, production runs, sensitivity studies |
| `data/results_full.json` | The 14-point parametric dataset (straight, stepped, solid), each row tagged `analog` or `ww` with its relative error |
| `data/results.json` | **Superseded.** The May 2026 pre-correction sweep (mono-energetic source, unmixed materials). Kept for provenance only — do not use |
| `CITATION.cff` | Citation metadata for the software archive |

## Model

**Geometry.** The toroidal shield is idealized to a rectilinear slab: 200 cm transverse (x), 100 cm high (y), both reflective; beam axis is z. Three-zone outboard bulk shield, 108 cm total — high-flux (HFS, 50 cm), medium-flux (MFS, 40 cm), low-flux (LFS, 18 cm). A source void precedes it and a detector void (z = 108–160) follows, both vacuum-terminated.

**Gap configurations.**

- *Straight slot* — runs un-offset through all three zones.
- *Single-step dog-leg* — straight through the inboard half, then jogged laterally by one step (~3 cm centre-to-centre), the two legs joined by a transverse connector so the void channel stays continuous. A thermal-expansion gap jogs; it does not become two holes.
- *Three-step staircase* — offset by one step at each zone interface, so no straight-line path traverses the shield.

**Materials** (homogenized, by volume fraction, from the original design):

- HFS — 5% Ti-6Al-4V + 65% TiH₂ + 15% B₄C + 15% H₂O
- MFS — 70% Fe-1422 + 15% B₄C + 15% H₂O
- LFS — 100% Fe-1422

Fe-1422 is the STARFIRE low-activation bulk-shield steel; its exact composition is not in the open literature, so it is approximated here by a 316-type austenitic stainless at 8.0 g/cm³. That over-represents nickel, but iron governs attenuation at these energies. This is the principal residual material uncertainty.

**Source.** A 35-group degraded spectrum (~80% of its weight in the 14-MeV group), emitted from a plane at the shield front into the forward hemisphere — a lightweight surrogate for the original two-step coupled pseudo-source.

**Nuclear data.** ENDF/B-VIII.0 processed with NJOY2016; B-10 from ENDF/B-VII.1 (the VIII.0 evaluation has an NJOY2016 processing incompatibility).

**Tally.** Track-length flux in the detector void, volume-averaged over 200 × 100 × 52 = 1.04 × 10⁶ cm³.

## Weight windows

Mesh-based weight windows are generated in-code by MAGIC, iteratively, with no external deterministic solve. Unbiasedness is established three ways: a like-for-like analog comparison, independent-seed replicates, and mesh- and iteration-count sensitivity studies — all agreeing within 0.7σ.

## Results

Weight-window-converged deep-penetration values (per source neutron):

| Case | Volume-averaged flux (n·cm⁻² per source n) | FSD | Wall time |
|------|--------------------------------------------|-----|-----------|
| Straight 1 cm | 2.50 × 10⁻⁹ | 0.052 | 66 min |
| Unbroken shield (solid) | 5.37 × 10⁻¹⁰ | 0.017 | 60 min |
| Three-step gap (1 cm, 5 cm offset) | 6.42 × 10⁻¹⁰ | 0.018 | 62 min |

**Efficiency.** Analog sampling resolves the unbroken shield only to FSD = 0.142 at 10⁷ histories; weight windows reach FSD = 0.017 at the wall time above. Since FSD ∝ N<sup>−1/2</sup>, matching that by analog sampling alone would take roughly **70×** more histories — about **50×** for the three-step case. That history-count ratio is the primary figure because it follows from Monte Carlo statistics alone, independent of hardware.

The measured wall-clock gain is more conservative: weight-window runs cost roughly 6–8× more per history here, for a true figure-of-merit gain of about **7–10×** on the Colab free tier specifically. Expect that to move with different hardware or a different weight-window implementation.

**Verification against the 1984 reference:**

| Quantity | 1984 MORSE-CG | Present (OpenMC) | Agreement |
|----------|---------------|------------------|-----------|
| Straight-slot transverse falloff | ~1 decade, edge to x = 10 cm | 1.08 decades | Quantitative |
| Unbroken-shield background | FSD > 3 — not converged | FSD 0.017 | Converged (units differ; a convergence comparison, not a magnitude match) |
| Three-step leakage vs. solid | "Within an order of magnitude" | 0.08 decade (ratio 1.19) | Quantitative |
| Single-step shadow depth | ~5 decades below straight peak | 2.85 decades | Qualitative |
| Single-step redirected peak | ~4× below straight peak | 214× | Qualitative |

The two single-step rows reproduce the *structure* of the original result — a shadow on the inner-leg centreline and a redirected peak at the outer leg — but not its magnitudes. The original modelled an idealized single step because the real sector joint is a castellated labyrinth for which no detailed geometry was ever specified, so the corner is under-determined in both calculations. The 1984 values also carry FSD 0.7–1.3. Discussion is in the paper.

## Running the notebook

1. Open `openmc_starfire.ipynb` in [Colab](https://colab.research.google.com) (or click the badge).
2. Mount Google Drive — the notebook caches compiled binaries and nuclear data there (~400 MB).
3. **Cell A** installs OpenMC (~30 s if cached, ~20 min cold).
4. **Cells B–D** verify, fetch nuclear data (one-time, ~15 min), and smoke-test.
5. Run the numbered sections in order. Section 13 generates weight windows and the production runs; 14 is the single-step dog-leg; 16 is the vacuum-boundary sensitivity check.

Weight-window generation and production runs take roughly an hour each. The notebook checkpoints to Drive, so a Colab reset does not cost you the run.

**Environment:** Colab free tier, Python 3.12, OpenMC 0.15.3, ENDF/B-VIII.0.

## A note on the surrogate

The notebook also contains Gaussian-process surrogate-fitting code over the parametric dataset. It is **not part of the paper** — it was cut during revision — and is kept here only because it runs and may be useful to someone. It is not maintained and its results are not verified against anything.

## Citation

**The paper** (cite this for the work itself):

Halley, A. M. (2026). *In-Code MAGIC Weight Windows for Deep-Penetration Gap Streaming: A Fully Reproducible Open-Source Benchmark Derived from a 1984 Fusion Shield Study.* Preprint.
DOI: [10.5281/zenodo.22810354](https://doi.org/10.5281/zenodo.22810354)

**This repository** (cite this for the code and data):

DOI: [10.5281/zenodo.21418400](https://doi.org/10.5281/zenodo.21418400)

**The original study being reproduced:**

Halley, A. M. & Miller, W. H. (1986). Neutron streaming through gaps in fusion reactor shielding. *Fusion Technology*, **10**, 424–430.
DOI: [10.13182/FST86-A24782](https://doi.org/10.13182/FST86-A24782)
Free ePrint: https://www.tandfonline.com/eprint/HPSVMVPVRBC5VBCTDTWU/full?target=10.13182/FST86-A24782

Halley, A. M. (1984). *Neutron Streaming Through Straight and Stepped Gaps in Fusion Reactor Shielding.* M.S. thesis, University of Missouri–Columbia.
DOI: [10.32469/10355/112299](https://doi.org/10.32469/10355/112299)

*Original calculations performed by Alan Halley using MORSE-CG under the supervision of William H. Miller. This repository reproduces them with OpenMC 0.15.3 — a different code, nuclear-data library, steel composition, source treatment, and dose-conversion basis, so "reproduction" here means the study's conclusions, not numerical replication.*

## Author

Alan Halley — [alanhalley.com](https://alanhalley.com)
