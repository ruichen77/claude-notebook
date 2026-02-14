# CCC Migration Plan for dtc_cz_sim

## When to Migrate

Migrate when reaching **Step 5** (coupler coherence sweeps) — the first workload
that benefits from multi-node parallelism. Steps 1-4 are single-point or small
sweeps that run fine on landsman2.

## Why CCC

| | landsman2 | CCC |
|---|---|---|
| Access | SSH key, direct | SSH key, no 2FA (Claude can connect directly) |
| CPU | 28 cores, 1 TB RAM, 1 node | 8+ cores/node, 32G+ RAM, 103 nodes |
| Parallelism | `multiprocessing` on 1 machine | LSF `bsub` job arrays across nodes |
| Storage | `/home/US8J4928/repos/` | `/u/ruichenzhao/repos/` (100 GB) |
| Python | QuTiP 5.2.2 installed | Needs setup |
| GPU | None | 800+ A100/H100 (not needed, but available) |

The DTC simulation is CPU-bound (matrix diag, mesolve). The win on CCC is scaling
out the Layer 4-5 sweeps as independent LSF jobs. A 50x50 T1xT2 grid with 5 error
budget configs = 12,500 independent solves — trivially parallelizable as job arrays.

## Migration Steps

1. **Clone repos to CCC**
   ```bash
   ssh -T ruichenzhao@ccc-login1.pok.ibm.com
   cd ~/repos
   # Clone from landsman2 or from a git remote
   git clone <dtc_cz_sim_repo>
   git clone <EnhancedSmarterThanARock_repo>
   ```

2. **Create conda environment**
   ```bash
   conda create -n dtc python=3.12
   conda activate dtc
   pip install qutip numpy scipy plotly
   cd ~/repos/dtc_cz_sim && pip install -e .
   cd ~/repos/EnhancedSmarterThanARock && pip install -e .
   ```

3. **Verify imports**
   ```bash
   python -c "from dtc_cz_sim import DTCSystem; print('OK')"
   python -c "import qutip; print(qutip.__version__)"
   ```

4. **Adapt sweep.py for LSF job arrays**
   - Each sweep point is a standalone job: `bsub -J "sweep[1-12500]" python run_point.py $LSB_JOBINDEX`
   - Script reads (T1, T2, config) from an index file, runs one Lindblad solve, writes result to disk
   - Post-processing script collects all results into a single array

5. **Example LSF submission**
   ```bash
   bsub -J "cz_sweep[1-100]" \
     -q normal \
     -n 4 \
     -M 16G \
     -o logs/sweep_%J_%I.log \
     -e logs/sweep_%J_%I.err \
     "conda activate dtc && python sweep_point.py $LSB_JOBINDEX"
   ```

## Notes

- No GPU needed — this is pure CPU (matrix diag + mesolve)
- landsman2 NUMA issue (28 cores slower than 21 for R-Q-DTC-Q-R) doesn't apply on CCC since each job uses a single node's cores
- CCC `normal` queue has no time limit, good for long Lindblad solves
- Existing CCC setup: `~/repos/research-novel-devices`, `~/qiskit_env.yml` (Qiskit env, separate from this)
