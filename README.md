# OpenMC STARFIRE Gap Streaming

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/alanhalle/openmc-starfire/blob/main/openmc_starfire.ipynb)

Reproduce **Halley & Miller (1986)**, *Neutron Streaming Through Gaps in Fusion Reactor Shielding*, using [OpenMC](https://openmc.org) 0.15.3 on Google Colab free tier, then fit a Gaussian Process surrogate on the parametric dataset.

## Background

The 1986 paper (original calculations by Alan Halley using MORSE-CG) characterized neutron streaming through straight and stepped gaps in the STARFIRE tokamak shield. This repo reproduces those calculations with a modern Monte Carlo code and adds a machine-learning surrogate.

## Repository Contents

| File | Description |
|------|-------------|
| `openmc_starfire.ipynb` | Full Colab notebook — install, geometry, runs, GP surrogate |
| `data/results.json` | All 14 OpenMC results (straight + stepped + solid baseline) |

## Physics Setup

**Shield geometry** (108 cm total, z-axis):
- HFS: 50 cm TiH₂ (3.75 g/cm³)
- MFS: 40 cm B₄C (2.52 g/cm³, natural boron)
- LFS: 18 cm Fe-1422 (316SS placeholder, 8.0 g/cm³)

**Source**: 14.1 MeV mono-energetic neutrons, forward-directed plane source

**Nuclear data**: ENDF/B-VIII.0 processed with NJOY2016; B-10 from ENDF/B-VII.1 (VIII.0 has a NJOY2016 incompatibility)

**Tally**: volume-integrated flux in 200×100×22 cm detector void (divide by 440,000 cm³ for average flux)

## Results

### Straight slots

| Gap (cm) | Vol-int flux (n·cm/source-n) | Rel. err |
|----------|------------------------------|----------|
| 1  | 1.1123e-1 | 0.5% |
| 2  | 2.2145e-1 | 0.3% |
| 5  | 5.5724e-1 | 0.2% |
| 10 | 1.1243e+0 | 0.1% |

Flux scales linearly with gap width (pure geometric streaming through void) ✓

### Stepped slots (3-segment, each layer shifted by `offset`)

| Width (cm) | Offset (cm) | Vol-int flux | Rel. err |
|------------|-------------|--------------|----------|
| 1 | 2.5 | 3.6202e-3 | 4.1% |
| 1 | 5.0 | 3.5729e-3 | 4.7% |
| 1 | 10.0 | 3.2396e-3 | 3.9% |
| 2 | 2.5 | 9.2993e-3 | 2.9% |
| 2 | 5.0 | 7.3288e-3 | 3.7% |
| 2 | 10.0 | 6.3946e-3 | 3.8% |
| 5 | 2.5 | 2.9945e-1 | 0.6% | ← overlap regime |
| 5 | 5.0 | 4.1717e-2 | 1.3% |
| 5 | 10.0 | 2.0238e-2 | 1.8% |

The w=5cm/off=2.5cm point is in the **overlap regime** (offset < half-width → near-direct streaming).

### Solid shield baseline

| Case | Vol-int flux | Rel. err |
|------|--------------|----------|
| No gap | 9.4997e-4 | 8.6% |

Streaming enhancement (1 cm straight / solid): ≥117×. True ratio >>117 — solid baseline dominated by fast-neutron bypass; variance reduction (CADIS) needed for accuracy.

## GP Surrogate

Scikit-learn `GaussianProcessRegressor` in log₁₀(flux) space.

**Features**: `gap_type` (0=straight, 1=stepped), `log₁₀(width)`, `step_offset`, `overlap_ratio`

**Key design choices**:
- `log₁₀(width)` instead of raw width: straight-slot flux ∝ width exactly, so the law is linear in log-log space — the GP interpolates/extrapolates correctly
- `alpha=1e-8` diagonal jitter replaces `WhiteKernel` (avoids `ConvergenceWarning` on near-noise-free MC data)
- Kernel: `ConstantKernel × RBF`, 10 optimizer restarts

**13-fold LOO cross-validation results**:
- Mean |error|: 0.28 decades
- Max |error|: 0.61 decades (w=5cm/off=2.5cm — overlap boundary)
- Within ±0.5 dec: 12/13
- Within ±1.0 dec: 13/13

## Running the Notebook

1. Open `openmc_starfire.ipynb` in [Google Colab](https://colab.research.google.com) (or click the badge above)
2. Mount Google Drive — the notebook caches compiled binaries and nuclear data there (~400 MB total)
3. Run **Cell A** (installs OpenMC; ~30 sec if cached, ~20 min first time)
4. Run **Cell B** (verify), **Cell C** (nuclear data, one-time ~15 min), **Cell D** (smoke test)
5. Run Sections 1–9 in order

**Environment**: Colab free tier, Python 3.12.13, OpenMC 0.15.3, ENDF/B-VIII.0

## Citation

Halley, C.A. & Miller, L.G. (1986). Neutron streaming through gaps in fusion reactor shielding. *Fusion Technology*, 10(3), 878–883.  
DOI: [10.13182/FST86-A24782](https://doi.org/10.13182/FST86-A24782)  
Free ePrint: https://www.tandfonline.com/eprint/HPSVMVPVRBC5VBCTDTWU/full?target=10.13182/FST86-A24782

*Original calculations performed by Alan Halley using MORSE-CG. This repository reproduces those results with OpenMC 0.15.3.*

## Author

Alan Halley — [alanhalley.com](https://alanhalley.com)
