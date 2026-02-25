# adiabatic_cz Package — Rotating-Basis CZ Gate Simulator

## Motivation

Existing codebases (dtc_cz_sim on RockBottom, duffing_cz on qiskit-dynamics) evolve the **full Hilbert space** (225-432 dim) with the Hamiltonian interpolated in the lab frame. We showed this has fundamental interpolation accuracy limits (~2.8% H error that doesn't decrease with cache density).

Seth Merkel's `cauer_package` solves this by working in the **instantaneous eigenbasis**: pre-compute eigenvalues E(phi) and non-adiabatic coupling on a dense flux grid, cubic-spline interpolate those smooth functions, then evolve only 20-50 states via H_eff(t) = diag(E(phi(t))) + (dphi/dt)*M(phi(t)).

## Key Design Decisions

1. **Charge-basis transmon Hamiltonian** (NOT Duffing oscillator)
   - Exact H = 4Ec(n-ng)^2 - Ej*cos(phi) diagonalized in charge basis
   - More accurate than Duffing quartic approximation, especially for DTC (low Ej/Ec)
   - Takes simDict input directly (same as RockBottom) — no parameter conversion
   - Each transmon diagonalized individually, truncated to `cutoff` energy levels
   - Coupled system built in truncated eigenbasis (~432 dim)

2. **GPU-compatible (CCC A100/H100)**
   - All matrix construction and diagonalization in JAX (`jnp.linalg.eigh`)
   - Eigensolver sweep: `jax.vmap` over 500 flux points — batched GPU diag
   - Propagation: 30x30 LMDE — too small for GPU, but GPU won't hurt
   - Same code runs on CPU (landsman3) and GPU (CCC) via `jax.config`

3. **Eigenbasis cached separately**: compute_eigenbasis() is expensive (~10-30s) but only depends on the system (not the pulse). Cache as .npz and reuse across pulse sweeps.

4. **Two propagators sharing the same eigenbasis**:
   - LMDE: H_eff = diag(E) + dphi/dt*M (with non-adiabatic coupling)
   - Trotter: diag(exp(-iE*dt)) stepping (without M, pure adiabatic)
   - Same splines, same projections — only difference is M term

5. **JAX autodiff for pulse derivatives**: `jax.grad` gives exact dphi/dt with zero hand-derivation.

## Repo & Deployment

- **Local (dev)**: `~/repos/dtc_adiabatic_sim/`
- **Git remote**: `git@github.ibm.com:Ruichen-Zhao/dtc_adiabatic_sim.git`
- **CCC**: `/u/ruichenzhao/repos/dtc_adiabatic_sim/` (duffing_cz conda env, A100 GPUs)
- **landsman3**: `.../dtc_adiabatic_sim/` (qiskit_dyn env, CPU fallback)
- **Workflow**: develop locally -> git push -> git pull on remote -> run
- **Dependency**: `interpax` (JAX cubic splines, pip install)

## Package Structure

```
dtc_adiabatic_sim/
├── pyproject.toml
├── adiabatic_cz/
│   ├── __init__.py          # Lazy imports (same pattern as duffing_cz)
│   ├── jax_setup.py         # COPY from duffing_cz
│   ├── math_utils.py        # Subset: hc, freq, cosm, sinm (drop duffing/exchange)
│   ├── system.py            # NEW: charge-basis transmon + from_simdict()
│   ├── pulse.py             # ADAPT: add JAX autodiff derivatives, PulseConfig
│   ├── eigensolver.py       # ★ NEW CORE: phase-locked sweep + coupling matrix + splines
│   ├── propagator.py        # ★ NEW CORE: LMDE and Trotter propagators
│   ├── metrics.py           # COPY from duffing_cz
│   ├── simulation.py        # NEW: high-level API (run_cz, run_comparison, run_dynamics)
│   ├── plotting.py          # ADAPT: eigenbasis diagnostics plots
│   └── io.py                # COPY from duffing_cz
└── examples/
    ├── run_basic_cz.py
    ├── compare_lmde_trotter.py
    └── eigenbasis_diagnostics.py
```

## File-by-File Design

### 1. `system.py` — NEW (charge-basis transmon)

Replaces DuffingSystem entirely. Uses exact transmon Hamiltonian in charge basis.

```python
@dataclass
class ChargeTransmonParams:
    """Single transmon defined by Ej, Ec, ng."""
    Ej_MHz: float      # Josephson energy
    Ec_MHz: float      # Charging energy
    ng: float = 0.0    # Charge offset
    nmax: int = 25     # Charge basis truncation (2*nmax+1 states)
    cutoff: int = 6    # Energy levels to keep after diag

class ChargeTransmon:
    """Single transmon in charge basis."""
    def __init__(self, params):
        # Build H = 4Ec(n-ng)^2 - Ej/2*(|n+1><n| + h.c.)
        # Diagonalize, truncate to cutoff levels
        # Store: eigenvalues, eigenvectors, a (lowering op in eigenbasis)

    @classmethod
    def from_f01_anharm(cls, f01_GHz, anharm_GHz, ng=0, nmax=25, cutoff=6):
        """Create from f01 and anharmonicity (simDict format).
        Ec = |anharm| (GHz -> MHz), Ej = (f01 + Ec)^2 / (8*Ec)
        """

@dataclass
class DTCParams:
    """Double Transmon Coupler: two islands + shared junction."""
    Ej_L_MHz: float    # Left island Ej
    Ec_L_MHz: float    # Left island Ec
    Ej_R_MHz: float    # Right island Ej
    Ec_R_MHz: float    # Right island Ec
    rm: float          # Junction ratio (shared/individual)
    ng_L: float = 0.0
    ng_R: float = 0.0
    nmax: int = 25
    cutoff: int = 12   # DTC levels to keep

class DTCSystem:
    """Full Q-DTC-Q system built from charge-basis transmons."""
    def __init__(self, q0, dtc, q1, g_q0_dtcL, g_q1_dtcR):
        # q0, q1: ChargeTransmon instances
        # dtc: DTCParams
        # Build operators in truncated eigenbasis
        # Total dim = q0.cutoff * q1.cutoff * dtc.cutoff

    @classmethod
    def from_simdict(cls, simDict, rm=0.25, nmax=25):
        """Build from RockBottom simDict format."""

    def build_H0(self, phi_ext=None):
        """Full Hamiltonian at given external flux.
        Includes: individual transmon terms + exchange couplings
        + shared junction cosm/sinm term (flux-dependent).
        """

    def get_idle_dressed_states(self):
        """Dressed states at idle flux (for comp state identification)."""

    def compute_zz(self, Hds):
        """ZZ from dressed diagonal."""
```

**DTC internal structure**: The two coupler islands each have their own charge-basis Hamiltonian. The shared junction couples them via -rm*Ej*cos(phi_L - phi_R - phi_ext). The individual island Hamiltonians are diagonalized and truncated first, then the coupled system (including shared junction + qubit couplings) is built in the truncated eigenbasis.

**Why charge basis matters**: The DTC coupler has anharm = 80 MHz -> Ec ~ 80 MHz. With f01 = 3.8 GHz, Ej/Ec ~ 180. Even at this ratio, the Duffing 4th-order approximation has ~0.5% error in the 3rd level energy, which accumulates through the simulation.

### 2. `pulse.py` — ADAPT (add derivatives)

Copy ramen/square_ramen from duffing_cz. Add:

```python
@dataclass
class PulseConfig:
    gate_time_us: float
    amplitude_rad: float
    pulse_type: str = 'ramen'
    ramen_params: RamenParams = field(default_factory=RamenParams)
    ramen_width: float = None  # for square_ramen

def pulse_with_deriv(t, config):
    """Return (phi(t), dphi/dt) using JAX autodiff.
    The existing ramen/square_ramen are pure jnp, so jax.grad works.
    """
```

### 3. `eigensolver.py` — ★ NEW CORE

#### EigenbasisData dataclass
```python
@dataclass
class EigenbasisData:
    phi_grid: jnp.ndarray        # (n_phi,) flux in radians
    eigenvalues: jnp.ndarray     # (n_phi, n_eigs) angular freq (rad*MHz)
    eigenvectors: jnp.ndarray    # (n_phi, full_dim, n_eigs) phase-locked
    coupling_matrix: jnp.ndarray # (n_phi, n_eigs, n_eigs) non-adiabatic M
    comp_indices_eig: jnp.ndarray # (4,) comp state indices in eigenbasis at idle
    idle_idx: int                # index into phi_grid at idle flux
    n_eigs: int
    full_dim: int
```

#### compute_eigenbasis() — GPU-accelerated
```python
def compute_eigenbasis(system, n_phi=501, n_eigs=30, phi_range=None):
    """Phase-locked eigenvector sweep + non-adiabatic coupling.

    GPU strategy: jax.vmap(jnp.linalg.eigh) for batched diagonalization,
    then sequential phase-locking pass (inherently sequential).

    Algorithm:
    1. Build H(phi) for all flux points [vmap on GPU]
    2. Diagonalize all H(phi) [vmap on GPU — 500x batched eigh]
    3. Phase-lock eigenvectors [sequential, on CPU]
    4. Compute non-adiabatic coupling M [vectorized finite difference]
    5. Identify comp states at idle via max bare-state overlap
    """
```

#### EigenbasisInterpolator — cubic splines via interpax
```python
class EigenbasisInterpolator:
    """JAX-compatible cubic spline interpolation of E(phi) and M(phi)."""
    def __init__(self, eig_data):
        # interpax.CubicSpline for E(phi) and M(phi)
    def eigenvalues(self, phi) -> jnp.ndarray:  # (n_eigs,)
    def coupling_matrix(self, phi) -> jnp.ndarray:  # (n_eigs, n_eigs)
    def zz(self, phi) -> float
    def H_eff(self, phi, dphi_dt) -> jnp.ndarray:
        """diag(E(phi)) + dphi_dt * M(phi)"""
```

### 4. `propagator.py` — ★ NEW CORE

#### LMDE propagator (Seth's approach)
```python
def propagate_lmde(interp, pulse_config, phi_ext_0,
                   n_times=200, solver_method='jax_expm', max_dt=0.0005):
    """Generator: G(t) = -i*H_eff_rf(t)
    where H_eff_rf = diag(E(phi(t)) - E_idle) + (dphi/dt)*M(phi(t))
    Rotating frame at idle removes fast diagonal oscillations.
    solve_lmde evolves U(t): dU/dt = G(t)*U, U(0) = I.
    """
```

#### Trotter propagator (adiabatic-only baseline)
```python
def propagate_trotter(interp, pulse_config, phi_ext_0,
                      n_steps=2000, n_times=200):
    """Pure adiabatic: U_step = diag(exp(-i*E(phi(t))*dt))
    No non-adiabatic coupling M — isolates that contribution.
    Uses same cubic spline eigenvalues as LMDE.
    """
```

#### Result dataclass
```python
@dataclass
class PropagationResult:
    U_eig: np.ndarray       # (n_eigs, n_eigs) unitary in eigenbasis
    U_comp: np.ndarray      # (4, 4) projected gate unitary
    tlist: np.ndarray        # (n_times,) time points
    psi_traj: np.ndarray    # (n_times, n_eigs, 4) trajectories (optional)
    method: str             # 'lmde' or 'trotter'
```

### 5. `simulation.py` — high-level API

```python
def run_cz(system, pulse_config, method='lmde', eig_data=None, **kwargs):
    """One-stop: compute eigenbasis (or reuse) -> propagate -> metrics."""

def run_comparison(system, pulse_config, **kwargs):
    """LMDE vs Trotter side-by-side, shared eigenbasis."""

def run_dynamics(system, pulse_config, method='lmde', n_times=500, **kwargs):
    """Time-resolved populations, leakage, conditional phase."""
```

### 6. Copied verbatim from duffing_cz
- `jax_setup.py`
- `metrics.py`
- `io.py` (adapted for new result types)
- `__init__.py` (lazy imports, updated module list)

### 7. `math_utils.py` — subset
Keep: `hc`, `freq`, `cosm`, `sinm`
Drop: `duffing`, `duffing_from_transmon`, `exchange` (replaced by charge-basis construction)

### 8. `plotting.py` — adapted
Reuse duffing_cz foundations. Add eigenbasis-specific:
- `plot_eigenbasis_spectrum()` — E(phi) + ZZ shared-x stacked
- `plot_coupling_matrix()` — heatmap of |M_ij(phi)|
- `plot_comparison()` — LMDE vs Trotter side-by-side
- `plot_spline_quality()` — eigensolver diagnostics

## Data Flow

```
simDict
  |
  v  from_simdict()
DTCSystem (charge-basis transmons, truncated eigenbasis)
  |
  v  compute_eigenbasis()  [vmap eigh on GPU, ~500 diags of ~432x432]
EigenbasisData (cached as .npz, reusable across pulses)
  |
  v  EigenbasisInterpolator()
Cubic splines for E(phi), M(phi), ZZ(phi)
  |
  |-- propagate_lmde()     [H_eff = diag(E) + dphi/dt*M, ~30x30 ODE]
  |-- propagate_trotter()  [diag(exp(-iE*dt)), no M term]
  |
  v  U_comp = U_eig[comp_idx, comp_idx]
4x4 gate unitary
  |
  v  compute_all_metrics()
GateMetrics (fidelity, leakage, conditional phase)
```

## GPU Strategy

| Phase | Dimension | GPU approach |
|-------|-----------|-------------|
| H construction | 500 x 432x432 | `jax.vmap(build_H0)` |
| Eigensolve | 500 x 432x432 | `jax.vmap(jnp.linalg.eigh)` — **main GPU win** |
| Phase locking | sequential | CPU (inherently sequential) |
| Spline fitting | 500 points | CPU (interpax, one-time) |
| LMDE propagation | 30x30 ODE | JAX ODE solver (works on GPU, not bottleneck) |
| Trotter propagation | 30-dim vectors | `jax.lax.scan` (works on GPU) |

The big GPU win is batched diagonalization: 500 independent eigh(432x432) in one GPU kernel. On A100 this should be ~100x faster than sequential CPU eigh.

## CCC Deployment

```bash
# Conda env (reuse duffing_cz env)
conda activate duffing_cz
pip install interpax

# Submit eigensolver job (GPU)
bsub -J adiab_eig -gpu "num=1:mode=exclusive_process" -n 4 -M 16G \
  python -u examples/run_basic_cz.py > output.log 2>&1

# Or interactive
bsub -q interactive -Is -gpu "num=1:mode=exclusive_process" /bin/bash
```

## Verification Plan

1. **Eigensolver quality**: E(phi) from eigensolver vs direct eigh(H(phi)) at test points
2. **Charge vs Duffing**: Compare eigenvalues from charge-basis vs duffing_cz at idle
3. **LMDE vs duffing_cz**: Same pulse -> fidelity/leakage should agree to ~1e-3
4. **Trotter convergence**: increasing n_steps should converge toward LMDE
5. **GPU vs CPU**: bit-identical results (f64), timing comparison

## Implementation Order

1. Scaffold: pyproject.toml, __init__.py, copy verbatim files
2. system.py: ChargeTransmon, DTCSystem, from_simdict()
3. pulse.py: copy + add pulse_with_deriv(), PulseConfig
4. eigensolver.py: compute_eigenbasis() + EigenbasisInterpolator
5. propagator.py: propagate_lmde() then propagate_trotter()
6. simulation.py: run_cz(), run_comparison(), run_dynamics()
7. plotting.py: adapt + eigenbasis diagnostics
8. examples/run_basic_cz.py: first end-to-end test
9. Deploy to CCC, verify GPU acceleration

## Dependencies

```toml
[project]
dependencies = [
    "numpy", "scipy", "jax[cpu]",
    "qiskit-dynamics", "interpax", "plotly",
]
```
For GPU: `jax[cuda12]` instead of `jax[cpu]`.
