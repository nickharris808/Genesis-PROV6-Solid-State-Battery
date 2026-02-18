# Genesis PROV 6: Solver Architecture Overview

**Classification:** NON-CONFIDENTIAL
**Note:** This document describes the solver architecture conceptually. Solver source code is not included in this repository.

---

## Overview

The Genesis solid-state battery analysis employs a coupled multi-physics simulation framework consisting of four primary solver components. These solvers exchange field data at each time step to capture the interdependent physics of dendrite growth, ion transport, mechanical deformation, and thermal behavior in gyroid-structured LLZO electrolytes.

---

## 1. Phase-Field Dendrite Solver (Allen-Cahn)

### Purpose
Simulates lithium dendrite nucleation, growth, branching, and interaction with the gyroid scaffold geometry.

### Formulation
The Allen-Cahn phase-field equation governs the evolution of the order parameter phi (0 = electrolyte, 1 = lithium metal):

    d(phi)/dt = -L * (dF/dphi)

where L is the kinetic coefficient and F is the total free energy functional:

    F = integral[ f_chem(phi) + (kappa/2)|grad(phi)|^2 + f_elastic(phi, epsilon) ] dV

The free energy includes:
- **Chemical free energy** f_chem: Double-well potential driving phase separation
- **Interfacial energy**: Gradient penalty term with anisotropic kappa (crystallographic orientation dependence)
- **Elastic strain energy** f_elastic: Coupled from the mechanical solver, capturing stress effects on dendrite nucleation

### Coupling
- Receives: Stress field from mechanical solver, electrochemical potential from P2D solver
- Provides: Dendrite geometry (phase-field isosurface) to mechanical solver and P2D

### Key Parameters
- Grid resolution: Up to 1024 x 1024 (2D) or 128^3 (3D)
- Time stepping: Explicit Euler with adaptive dt
- Gyroid constraint: Phase-field masked by gyroid level-set function

---

## 2. P2D Electrochemistry Solver (Newman/Doyle/Fuller)

### Purpose
Models full-cell electrochemistry including charge transfer, solid-state diffusion, and electrolyte transport to predict cycle life and impedance evolution.

### Formulation
The Pseudo-Two-Dimensional (P2D) model solves coupled PDEs across the cell sandwich:

**Solid-phase potential (Ohm's law):**

    sigma_eff * d2(phi_s)/dx2 = j_n * a_s

**Electrolyte-phase potential (modified Ohm's law):**

    kappa_eff * d2(phi_e)/dx2 + kappa_D * d2(ln c_e)/dx2 = -j_n * a_s

**Electrolyte concentration (mass conservation):**

    epsilon * d(c_e)/dt = D_eff * d2(c_e)/dx2 + (1-t+)/F * j_n * a_s

**Solid-state diffusion (Fick's law in spherical particles):**

    d(c_s)/dt = D_s/r^2 * d/dr(r^2 * d(c_s)/dr)

**Charge transfer (Butler-Volmer):**

    j_n = i_0 * [exp(alpha_a*F*eta/RT) - exp(-alpha_c*F*eta/RT)]

### Monroe-Newman Criterion
The P2D solver evaluates the mechanical stress at the lithium metal interface and compares G_eff/G_Li against the Monroe-Newman threshold of 2.0x. For the gyroid architecture, this ratio is 8.9x.

### SEI Growth Model

    R_SEI(n) = R_SEI_0 + R_rate * sqrt(n)

where n is cycle number and R_rate = 0.5 (assumed parameter).

### Coupling
- Receives: Dendrite geometry from phase-field solver
- Provides: Electrochemical potential field to phase-field, current density to thermal model

---

## 3. Born Solvation Model (Quantum Sieve)

### Purpose
Calculates ion transport barriers through sub-nanometer gyroid channel constrictions where solvation shell stripping dominates selectivity.

### Formulation
The Born solvation energy for transferring an ion from bulk to a confined pore:

    Delta_G = (z^2 * e^2) / (8 * pi * epsilon_0 * r_ion) * (1/epsilon_pore - 1/epsilon_bulk)

where:
- z = ion charge
- e = elementary charge
- r_ion = ionic radius
- epsilon_pore = dielectric constant in confined pore (reduced from bulk)
- epsilon_bulk = bulk electrolyte dielectric constant

### Calibration
The Born model is calibrated against GROMACS umbrella sampling potential of mean force (PMF) calculations for Li+ and K+. The calibrated model is then used to predict Na+ barriers (where direct GROMACS PMF returned NaN due to convergence issues).

### Coupling
- Receives: Pore geometry from TPMS generator
- Provides: Ion flux selectivity coefficients as boundary conditions for P2D solver

---

## 4. Mechanical Solver (Biharmonic Plate Theory)

### Purpose
Computes mechanical response of the gyroid scaffold to dendrite tip loading, determining the suppression factor.

### Formulation
The biharmonic equation for thin plate deflection:

    D * nabla^4(w) = q(x,y)

where:
- D = E*h^3 / (12*(1-nu^2)) is the flexural rigidity
- w is the plate deflection
- q is the distributed load (dendrite tip pressure)

Boundary conditions: Clamped (fixed displacement and slope at edges).

### Suppression Factor
The suppression factor is the ratio of baseline deflection (uniform stiffness) to Genesis deflection (gyroid gradient stiffness):

    SF = w_baseline / w_genesis = 31334.63 / 4137.84 = 7.57x

### Sensitivity Analysis
- With lithium plasticity: 5.30x
- With 10 um glass manufacturing tolerance: 4.17x
- With 5 um premium glass tolerance: 5.45x

### Coupling
- Receives: Dendrite geometry from phase-field solver
- Provides: Stress field to phase-field (elastic energy contribution)

---

## 5. GROMACS Molecular Dynamics

### Purpose
Provides atomistic-level ionic conductivity prediction through mean square displacement (MSD) analysis of Li+ trajectories in the LLZO supercell.

### Configuration
- System: 2x2x2 supercell of cubic LLZO (Ia-3d)
- Atoms: 1536 total, 448 Li+ ions
- Box size: 9.4193 nm^3
- Temperature: 300 K (NVT ensemble)
- Trajectory: 15236 ps (approximately 15 ns usable data)
- Force field: LJ parameters adapted from Buckingham potentials (Adams & Rao 2012)

### Analysis Pipeline
1. Extract Li+ MSD from GROMACS trajectory (.xvg)
2. Auto-detect optimal fitting window (minimize R-squared while maintaining sufficient duration)
3. Fit MSD = 6*D*t to obtain diffusion coefficient D
4. Apply Nernst-Einstein relation: sigma = (n * q^2 * D) / (k_B * T * V)

### Results
- Optimal window: [1524, 4571] ps
- D = 3.803e-14 m^2/s (+/- 1.404e-15 m^2/s)
- sigma = 0.112 mS/cm
- R-squared = 0.999

---

## Coupling Architecture Summary

```
Phase-Field  <----->  Mechanical Solver
    |                       |
    v                       v
P2D Solver  <----->  Born Solvation
    |
    v
GROMACS MD (provides conductivity input to P2D)
```

Data exchange:
1. Phase-field --> Mechanical: Dendrite geometry (isosurface)
2. Mechanical --> Phase-field: Stress field (elastic energy)
3. Phase-field --> P2D: Phase distribution (electrode geometry)
4. P2D --> Phase-field: Electrochemical potential (nucleation driving force)
5. Born model --> P2D: Ion selectivity (flux boundary conditions)
6. GROMACS --> P2D: Ionic conductivity (transport property input)

---

*This document describes the solver architecture conceptually for reproducibility and transparency purposes. Solver source code is maintained separately and is not included in this public repository.*
