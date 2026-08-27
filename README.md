# Estimating How Monopiles Affect Turbulence in Ocean Models

Summer internship project at the Bedford Institute of Oceanography (BIO), Offshore Wind Farm Group.
June 4 – August 26, 2026

**Author:** Gloria Wang
**Supervisors:** Dr. Yongsheng Wu, Dr. Yongxing Ma

## Overview

This project estimates how offshore wind turbine monopile foundations disturb surrounding water flow in Halifax Harbour, and quantifies that disturbance in a form a regional ocean model can use. Monopiles disturb flow and can alter water column stratification, affecting mixing, oxygen transport, and nutrient exchange. The work is split into two studies built around the same simulation setup: an 8 m cylinder in cross-flow at Re = 4×10⁶, water depth 40 m, inflow velocity 0.5 m/s.

- **Study 1: Flow Past Cylinder Validation** — does the simulation reproduce known drag, shedding, and separation behavior for flow past a cylinder?
- **Study 2: Turbulence Parameterization** — how much turbulence does a monopile create, and what are αᵏ and c₄, the two closure parameters that describe it?

## Simulation Setup

| Parameter | Value |
|---|---|
| Reynolds number | 4 × 10⁶ |
| Turbulence model | Spalart–Allmaras DDES / LES |
| Mesh | ~43M cells, decomposed across 256 processors |
| Kinematic viscosity | 1.0 × 10⁻⁶ m²/s |
| Cylinder diameter | 8 m |
| Water depth | 40 m |
| Inflow velocity | 0.5 m/s |
| Domain size | 600 m × 120 m × 40 m |
| Cylinder center | (40, 60, 0) |
| Control volume | 1,680,000 m³ |

**Tools:** OpenFOAM (pimpleFoam), ParaView 5.7.0 (connected via pvserver over MobaXterm/ThinLinc remote HPC access), Python 3.8 with NumPy and Matplotlib for FFT and plotting, pyvista for a standalone post-processing script.

## Equations

**Flow past cylinder (validation study)**

| Equation | Description |
|---|---|
| Cd = Fd / (0.5·ρ·U∞²·D) | Drag coefficient |
| Cpb = (Pb − P∞) / (0.5·ρ·U∞²) | Base pressure coefficient |
| St = f·D / U∞ | Strouhal number |
| Lr / D | Recirculation length (normalized) |
| Re = ρ·U∞·D / μ | Reynolds number |

**Turbulence parameterization**

| Equation | Description |
|---|---|
| kRes = ½(u′² + v′² + w′²) | Resolved turbulent kinetic energy |
| εRes = 2ν·s′ᵢⱼ·s′ᵢⱼ | Resolved dissipation rate |
| εLES = εRes + εSGS | Overall dissipation rate |
| Pwake = −u′ᵢu′ⱼ(∂ūᵢ/∂xⱼ) | Wake-generated TKE production |
| Sk,LES = ⟨Pwake⟩·Vc | Volume-averaged TKE source |
| Sᵖⁱˡᵉ_k = αᵏ · ½·Cd,eff·(D/Acell)·|uh|³ | Ocean-model TKE source term |
| Sᵖⁱˡᵉ_ε = c₄ · (ε/k) · Sᵖⁱˡᵉ_k | Ocean-model dissipation source term |

A standard k–ε ocean-model closure represents the unresolved monopile as a quadratic drag force; αᵏ and c₄ are the two parameters this study estimates from LES.

## Pipeline

### Validation study
Drag was extracted via pressure integration on the cylinder surface:
`ExtractSurface → GenerateSurfaceNormals → Calculator → IntegrateVariables`

### Turbulence parameterization study
1. Reconstruct the decomposed case (`reconstructPar`) before reading with pyvista, since the available VTK build lacks `SetCaseType` and can't read decomposed cases directly.
2. Extract the averaging window. Only two time windows exist in the data: t = 399–595 and t = 1049–1400. The averaging window used is **t = 1049–1080** (32 steps, one vortex shedding cycle), chosen because it falls inside the validated drag steady-state window (t = 1049–1148).
3. Resolved TKE and dissipation:
   - `AppendAttributes` + fluctuation `Calculator` (U − U_average) + `Gradient` + `TemporalStatistics` for resolved dissipation.
   - Chain `Calculator` filters sequentially, never branched, and always set a non-default "Result Array Name" to avoid silent overwriting.
4. Horizontal slice: Origin (40, 60, 0), Normal (0, 0, 1). Vertical slice: Origin (40, 60, 0), Normal (0, 1, 0).
5. Derive αᵏ (fraction of drag power becoming TKE) and c₄ (dissipation-equation closure parameter) from the drag power, resolved TKE, and dissipation results.

## Results

### Validation study

| Quantity | Value | Description |
|---|---|---|
| Cd | 0.404 | Drag coefficient |
| Cpb | −0.451 | Mean back-pressure coefficient |
| St | 0.505 | Strouhal number (shedding frequency) |
| Lr/D | 1.214 | Recirculation length |
| θsep | 117° | Mean separation angle |

Cd falls within the expected 0.2–0.5 range for post-critical flow (Re ≈ 4×10⁶); Cpb, Lr/D, and separation angle all sit within reported experimental ranges for this regime.

**Benchmarked against published literature:**

| Quantity | This Study | Miralles & Koobus | Yeon et al. | Roshko | Catalano et al. |
|---|---|---|---|---|---|
| Re | 4×10⁶ | 1×10⁶ | 7.6×10⁵ | 3.5–10×10⁶ | 1×10⁶ |
| Mean Cd | 0.404 | 0.289 | 0.23 | 0.70 | 0.31 |
| Mean Cpb | −0.451 | −0.25 | −0.24 | −0.86 | −0.32 |
| St | 0.505 | 0.50 | 0.46 | 0.27 | 0.35 |
| Sep. Angle | 117° | 128° | 135° | — | 110° |
| Lr/D | 1.214 | — | — | — | 1.04 |

### Turbulence parameterization study

| Quantity | Value | Description |
|---|---|---|
| kRes | 2.90×10⁻⁴ m²/s² | Resolved TKE (volume-averaged) |
| εLES | 1.66×10⁻⁶ m²/s³ | Overall dissipation rate |
| Sk,LES | 2.15×10⁻⁶ m²/s³ | Wake TKE production |
| Cd,eff | 0.404 | Effective drag coefficient |
| Pdrag | 4.83×10⁻⁶ m²/s³ | Drag power per unit mass |
| αᵏ | 0.445 | Fraction of drag power becoming TKE |
| c₄ | 1.478 | Dissipation-equation closure parameter |

c₄ (1.478) sits below the standard k–ε constant Cε2 (1.92).

## Conclusion

The validation study confirms the simulation reproduces published drag, pressure, shedding, and separation behavior for post-critical cylinder flow (Re = 4×10⁶). The LES-based analysis estimates αᵏ = 0.445 and c₄ = 1.478, indicating that wake TKE production corresponds to approximately 44.5% of the available drag power, below the common assumption of αᵏ = 1. Together, these give an LES-informed parameterization for implementation in a regional ocean model's k–ε closure: Sᵖⁱˡᵉ_k and Sᵖⁱˡᵉ_ε.

**Limitations:** SGS kinetic energy is currently neglected in k but included in ε. c₄ assumes local equilibrium in the near wake. Offline calibration against the model is still pending and can give more accurate results.

## Repository Structure

```
BIO-Summer2026/
├── README.md                 This file
├── presentation.html         Web version of the final presentation (open in a browser)
├── reports/                  Final written reports (PDF)
├── figures/                  Static figures (mesh, mid-plane slices)
├── visualizations/           Flow visualization GIFs (speed, viscosity, vorticity)
├── scripts/                  Post-processing / visualization scripts
│   └── archive/              Superseded script versions, kept for reference
└── web_assets/                Images used by presentation.html
```

## Code

- `scripts/vorticity_z.py` — vorticity contour visualization (z-component)
- `scripts/vorticity_z_v2.py` — updated version of the vorticity_z visualization
- `scripts/speed_2panel.py` — two-panel speed/velocity magnitude visualization
- `scripts/archive/` — earlier `.bak` versions of `vorticity_z_v2.py`, kept for reference

## Acknowledgements

- Dr. Marc Skinner — for a smooth hiring process that made this term possible
- Sharon Young — for HR support that made onboarding smooth
- Dr. Yongsheng Wu & Dr. Yongxing Ma — for supervision and guidance on OpenFOAM, ParaView, and turbulence-closure theory throughout the term
- Andy He — for always being willing to help
