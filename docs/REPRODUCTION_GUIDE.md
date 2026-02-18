# Genesis PROV 6: Reproduction Guide

**Classification:** NON-CONFIDENTIAL
**Note:** This guide describes how to reproduce the key results conceptually. Full solver source code is not included in this public repository.

---

## Overview

This guide outlines the methodology for independently reproducing the key claims in the Genesis solid-state battery white paper. All results are computational and were originally produced on consumer hardware (Apple M3 Max, also verified on A100 GPU for GROMACS).

---

## 1. Verification Script (Included)

The simplest verification is to run the included claim verification script:

```bash
cd verification/
python verify_claims.py
```

This checks all key numerical claims against canonical reference values and performs independent calculations where possible (e.g., Born solvation energy).

**Requirements:** Python 3.8+, no external dependencies (uses only `json`, `math`, `os`, `sys`).

---

## 2. Reproducing Key Results

### 2.1 Monroe-Newman Criterion (Pillar 1)

**What to verify:** G_eff/G_Li > 2.0x for the gyroid scaffold at 30.6% porosity.

**Method:**
1. Obtain LLZO elastic properties from literature:
   - Bulk LLZO shear modulus: ~61 GPa (cubic phase)
   - Lithium metal shear modulus: ~4.2 GPa
2. Apply Hashin-Shtrikman bounds for a porous material at 30.6% porosity
3. Calculate effective shear modulus of the gyroid scaffold
4. Compute ratio: G_eff / G_Li

**Expected result:** 8.9x (well above 2.0x criterion)

**Key reference:** Monroe, C., & Newman, J. (2005). "The impact of elastic deformation on deposition kinetics at lithium/polymer interfaces." *J. Electrochem. Soc.*, 152(2), A396.

### 2.2 Biharmonic Suppression Factor (Pillar 1)

**What to verify:** Deflection ratio > 7x between uniform and gyroid-stiffened plates.

**Method:**
1. Set up biharmonic plate equation: D * nabla^4(w) = q
2. Solve for uniform stiffness (control) with clamped boundary conditions
3. Solve for gradient stiffness (gyroid) with same loading and boundaries
4. Compute ratio: w_control / w_gyroid

**Expected result:** 7.57x (clamped plate, ideal geometry)

**Sensitivity checks:**
- With lithium plasticity correction: 5.30x
- With 10 um manufacturing tolerance: 4.17x
- With 5 um premium glass tolerance: 5.45x

**Software:** Any PDE solver capable of biharmonic equations (FEniCS, custom finite difference, MATLAB PDE Toolbox).

### 2.3 Ionic Conductivity (Nernst-Einstein from MD)

**What to verify:** sigma = 0.112 mS/cm from Li+ diffusion in LLZO.

**Method:**
1. Set up LLZO supercell (2x2x2 cubic, Ia-3d) with 448 Li+ ions
2. Run NVT molecular dynamics at 300 K for at least 15 ns
3. Extract Li+ mean square displacement (MSD)
4. Identify optimal linear fitting window (maximize R-squared)
5. Fit MSD = 6*D*t to obtain diffusion coefficient D
6. Apply Nernst-Einstein: sigma = (n * q^2 * D) / (k_B * T * V)

**Expected result:** D ~ 3.8e-14 m^2/s, sigma ~ 0.112 mS/cm

**Key caveats:**
- Force field parameters significantly affect the result (use Adams & Rao 2012 Buckingham potentials or equivalent)
- Trajectory must be long enough for diffusive regime (check MSD linearity)
- Do NOT fit beyond the actual trajectory length

**Software:** GROMACS 2023+ recommended. Verified on both A100 GPU and Apple M3 Max.

### 2.4 Born Solvation Selectivity (Pillar 2)

**What to verify:** Li+ transport barrier at 0.7 nm pore constriction.

**Method:**
1. Calculate Born solvation energy:
   Delta_G = (z^2 * e^2) / (8 * pi * eps0 * r_ion) * (1/eps_pore - 1/eps_bulk)
2. Use dielectric constant of confined water in sub-nm pore (eps_pore ~ 20)
3. Compare Li+, Na+, K+ barriers

**For higher fidelity:**
1. Set up GROMACS umbrella sampling with Li+ pulled through a cylindrical pore
2. Compute potential of mean force (PMF) via WHAM
3. Extract barrier height from PMF profile

**Expected result:** Li+ barrier ~ 7.1 kJ/mol, K+ ~ 7.7 kJ/mol

**Note:** Direct PMF for Na+ may fail to converge. Calibrate Born model against Li+ and K+ PMF results, then predict Na+.

### 2.5 Gyroid Geometry and Tortuosity (Pillar 3)

**What to verify:** Tortuosity = 1.18 at 30.6% porosity for Schoen Gyroid.

**Method:**
1. Generate Gyroid implicit surface: sin(x)*cos(y) + sin(y)*cos(z) + sin(z)*cos(x) = t
2. Set level-set parameter t = 0.6 for 30.6% porosity
3. Voxelize on 120^3 grid
4. Compute tortuosity via shortest-path algorithm (MCP geometric)
5. Average over 50+ random start-end pairs

**Expected result:** tau = 1.18 +/- 0.04, connectivity = 100%

**Software:** scikit-image (marching cubes for surface, MCP for tortuosity), NumPy, SciPy.

### 2.6 Cycle Life (P2D Model)

**What to verify:** > 70% retention at 1000 cycles.

**Method:**
1. Implement P2D (Newman) model with Butler-Volmer kinetics
2. Use LLZO transport properties (sigma = 0.112 mS/cm corrected, or 0.477 mS/cm legacy)
3. Include SEI growth model: R_SEI(n) = R_0 + 0.5 * sqrt(n)
4. Cycle the model at 1C rate for 1000 cycles
5. Track capacity at each cycle

**Expected result:** ~71.9% retention at 1000 cycles (using legacy conductivity)

**Caveat:** Using corrected conductivity (0.112 mS/cm) would increase impedance and reduce retention below 71.9%.

**Software:** PyBaMM (open source P2D solver) or custom implementation.

---

## 3. Hardware Requirements

All simulations were verified on consumer hardware:

| Simulation | Hardware | Performance |
|---|---|---|
| GROMACS MD (1536 atoms) | Apple M3 Max | 56.7 ns/day |
| Phase-field (1024x1024) | Apple M3 Max GPU | 2.2 GVoxelSteps/s |
| CalculiX FEM (1331 nodes) | Apple M3 Max | < 1 minute |
| Biharmonic solver | Apple M3 Max | < 1 second |
| TPMS generation (120^3) | Apple M3 Max | < 10 seconds |
| P2D cycling (1000 cycles) | Apple M3 Max | < 5 minutes |

No specialized HPC resources are required.

---

## 4. Key Dependencies

For a full reproduction:

```
# Core scientific computing
numpy >= 1.24
scipy >= 1.10
matplotlib >= 3.7

# Molecular dynamics
gromacs >= 2023  (for ionic conductivity)

# Surface generation
scikit-image >= 0.21  (for marching cubes)

# Testing
pytest >= 7.0

# Optional (for P2D)
pybamm >= 23.1  (or custom implementation)
```

---

## 5. Literature for Independent Validation

To validate Genesis results against published data:

| Property | Published Value | Reference |
|---|---|---|
| Cubic LLZO conductivity | 0.1-1.0 mS/cm | Multiple (see README references) |
| LLZO activation energy | 0.25-0.35 eV | Rangasamy et al. 2012 |
| LLZO lattice parameter | 12.94-13.13 A | Murugan et al. 2007 |
| Monroe-Newman criterion | G > 2*G_Li | Monroe & Newman 2005 |
| LLZO shear modulus | ~61 GPa | Various nanoindentation studies |
| Li metal shear modulus | ~4.2 GPa | Standard materials data |
| LLZO fracture toughness | ~1.0 MPa*sqrt(m) | Wolfenstine et al. 2018 |

---

*This guide is provided for transparency and reproducibility. Independent reproduction of these results by qualified researchers is encouraged. Please cite this work if you build upon these methods.*
