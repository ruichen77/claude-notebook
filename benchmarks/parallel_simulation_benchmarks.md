# Core Scaling Benchmark Results

> **See also:** [[Quantum Simulation Concepts]], [[Spectroscopy Prediction Workflow]], [[Chi Sweep Debugging Notes]]

**Server**: landsman2  
**Date**: February 2026  
**CPU**: 2× Intel Xeon E5-2690 v4 @ 2.60GHz (28 cores total, 14 per socket)  
**RAM**: 1 TB  

---

## Q-DTC-Q System

**Hilbert space dimension**: 432 (6 × 12 × 6)  
**Flux points**: 101  

| Cores | Wall Time | Per Point | Speedup | Efficiency |
|-------|-----------|-----------|---------|------------|
| 1 | 120.0s | 1.189s | 1.00x | 100.0% |
| 2 | 61.2s | 0.606s | 1.96x | 98.1% |
| 4 | 34.5s | 0.342s | 3.48x | 87.0% |
| 8 | 20.3s | 0.201s | 5.93x | 74.1% |
| 14 | 11.4s | 0.113s | 10.56x | 75.4% |
| 21 | 8.9s | 0.088s | 13.49x | 64.3% |
| **28** | **6.7s** | **0.066s** | **18.03x** | 64.4% |

**Recommendation**: Use **28 cores** for fastest results (~7 seconds).

---

## R-Q-DTC-Q-R System

**Hilbert space dimension**: 3888 (6 × 12 × 6 × 3 × 3)  
**Memory per worker**: ~2.9 GB  

### Benchmark Results (29 flux points)

| Cores | Wall Time | Per Point | Speedup vs 8 | Efficiency |
|-------|-----------|-----------|--------------|------------|
| 8 | 573.0s | 19.76s | 1.00x | 100.0% |
| 14 | 441.0s | 15.21s | 1.30x | 74.2% |
| **21** | **328.8s** | **11.34s** | **1.74x** | 66.4% |
| 28 | 346.8s | 11.96s | 1.65x | 47.2% |

### Extrapolated to 101 Flux Points

| Cores | Est. Wall Time | Est. Minutes |
|-------|----------------|--------------|
| 8 | 1996s | 33.3 min |
| 14 | 1536s | 25.6 min |
| **21** | **1145s** | **19.1 min** |
| 28 | 1208s | 20.1 min |

**Recommendation**: Use **21 cores** for fastest results (~19 minutes for 101 flux points).

⚠️ **Important**: 28 cores is SLOWER than 21 cores due to NUMA overhead (crossing socket boundary at 14 cores).

---

## Key Findings

### 1. NUMA Effects
- **Single socket (≤14 cores)**: Memory access stays local, good efficiency
- **Cross socket (>14 cores)**: Performance degrades for memory-intensive R-Q-DTC-Q-R
- **Q-DTC-Q**: Small enough to benefit from all 28 cores despite NUMA

### 2. Memory Bandwidth
- R-Q-DTC-Q-R uses ~2.9 GB per worker
- 21 workers × 2.9 GB = ~61 GB (fits comfortably in 1 TB RAM)
- Bottleneck is memory bandwidth, not capacity

### 3. Optimal Settings

| System | Recommended Cores | Estimated Time (101 pts) |
|--------|-------------------|--------------------------|
| Q-DTC-Q (432 dim) | 28 | ~7 seconds |
| R-Q-DTC-Q-R (3888 dim) | 21 | ~19 minutes |

### 4. Scaling Formula (approximate)

For estimating simulation time:
```
Q-DTC-Q:     time ≈ 0.066s × n_flux_points  (with 28 cores)
R-Q-DTC-Q-R: time ≈ 11.3s × n_flux_points   (with 21 cores)
```

---

## How to Use These Results

When running simulations, set the number of cores in your script:

```python
from multiprocessing import Pool

# For Q-DTC-Q
N_CORES = 28

# For R-Q-DTC-Q-R
N_CORES = 21

with Pool(N_CORES) as pool:
    results = pool.map(sim_at_flux, phi_values)
```

---

## Future Work

To improve R-Q-DTC-Q-R performance further, consider:
1. **NUMA-aware parallelization**: Pin workers to specific sockets
2. **Sparse matrix methods**: Exploit sparsity in Hamiltonian
3. **GPU acceleration**: Offload diagonalization to GPU
4. **Reduced basis methods**: Use fewer basis states where appropriate

---

## Flux Point Scaling (Benchmarked Feb 2026)

### Important: Parallelization Overhead

Time-per-point is NOT constant at low flux counts due to process pool overhead.

### Q-DTC-Q (28 cores)

| Flux Pts | Wall Time | Per Point |
|----------|-----------|-----------|
| 11 | 1.65s | 0.150s |
| 21 | 1.81s | 0.086s |
| 51 | 3.54s | 0.069s |
| 101 | 6.63s | 0.066s |
| 201 | 13.05s | 0.065s |

**Asymptotic time per point**: ~0.065s (reached at 50+ flux points)

### R-Q-DTC-Q-R (21 cores)

| Flux Pts | Wall Time | Per Point |
|----------|-----------|-----------|
| 11 | 155s | 14.1s |
| 21 | 189s | 9.0s |
| 29 | 328s | 11.3s |
| 51 | 513s | 10.1s |

**Asymptotic time per point**: ~10-11s (reached at 30+ flux points)

### Extrapolation Guidelines

- For **Q-DTC-Q**: Extrapolate from 50+ flux points for less than 10% error
- For **R-Q-DTC-Q-R**: Extrapolate from 30+ flux points for ~10-15% error
- At low flux counts (less than 20), overhead dominates - do not extrapolate

### Updated Time Estimates

For 101 flux points:

- Q-DTC-Q (28 cores): ~7 seconds (0.065s x 101 + overhead)
- R-Q-DTC-Q-R (21 cores): ~19 minutes (11s x 101 + overhead)
