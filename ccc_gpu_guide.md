# CCC (Cognitive Computing Cluster) GPU Guide

## Connection

```bash
ssh ruichenzhao@ccc-login1.pok.ibm.com
```

- Login nodes: `ccc-login[1-4].pok.ibm.com` (new cluster, V100/A100/H100)
- Old cluster: `dccc-login[1-4].pok.ibm.com`
- Auth: SSH key only (registered via Compute@Research Portal)
- Requires IBM VPN if off-campus

### Claude Access

CCC uses SSH key auth (no 2FA), so Claude can connect directly:
```bash
ssh -T ruichenzhao@ccc-login1.pok.ibm.com "command"
```

---

## Available GPUs (as of Feb 2026)

| GPU Model | VRAM | Hosts | GPUs per Host | Total GPUs |
|-----------|------|-------|---------------|------------|
| NVIDIA A100 SXM4 | 40 GB | ~90 nodes | 8 | ~720 |
| NVIDIA A100 SXM4 | 80 GB | ~10 nodes | 8 | ~80 |
| NVIDIA H100 80GB | 80 GB | ~2 nodes (cccxc701, cccxc702+) | 8 | ~16 |

- 103 GPU hosts total, 800+ GPUs
- CUDA driver: 570.86.10 or 575.57.08
- NUMA: 2 sockets per host, GPUs 0-3 on NUMA 0, GPUs 4-7 on NUMA 1

---

## Job Scheduler: LSF (Platform LSF)

### Queues

| Queue | Status | Run Limit | Use Case |
|-------|--------|-----------|----------|
| `normal` | Open | No limit | Batch jobs (default) |
| `interactive` | Open | 360 min (6 hrs) | Interactive/debug sessions |
| `short` | Closed | - | - |
| `priority` | Closed | - | - |

- `normal`: Max 1250 jobs per user, no time limit
- `interactive`: 6 hour wall clock limit, for interactive use

---

## Submitting GPU Jobs

### Basic batch job (1 GPU)
```bash
bsub -gpu - myjob
```

### Specify resources
```bash
bsub -J my_job -M 8G -n 4 -gpu "num=2" python your_script.py
```

### Exclusive GPU (recommended for training)
```bash
bsub -gpu "num=1:mode=exclusive_process" python train.py
```

### Interactive GPU session
```bash
bsub -q interactive -Is -gpu "num=1:mode=exclusive_process" /bin/bash
```

### Full example with all options
```bash
bsub -J train_model \
  -q normal \
  -n 8 \
  -M 32G \
  -gpu "num=2:mode=exclusive_process" \
  -o output_%J.log \
  -e error_%J.log \
  python train.py
```

### Key `-gpu` Options

| Option | Description |
|--------|-------------|
| `num=N` | Request N GPUs |
| `mode=exclusive_process` | Exclusive GPU access (recommended) |
| `mps=yes` | Enable Multi-Process Service (for sharing GPUs) |
| `mps=yes,share` | Share MPS daemon across jobs (same user, same GPU reqs) |

### Key `bsub` Flags

| Flag | Description |
|------|-------------|
| `-gpu "num=N"` | Request N GPUs |
| `-n N` | Request N CPU cores |
| `-n N,M` | Request N to M CPU cores |
| `-M 8G` | Memory limit |
| `-R "rusage[mem=X]"` | Memory resource requirement |
| `-J name` | Job name |
| `-q queue` | Queue name |
| `-o file` | Stdout file (`%J` = job ID) |
| `-e file` | Stderr file |
| `-Is` | Interactive with pseudo-terminal |
| `-w "done(jobid)"` | Wait for dependency |

---

## Monitoring

### Check your jobs
```bash
bjobs                    # list your jobs
bjobs -l <jobid>         # detailed job info
bjobs -sum               # summary
```

### GPU cluster status
```bash
lsload -gpu              # host-level GPU summary (utilization %)
lsload -gpuload          # per-GPU details (temp, mem, utilization)
lshosts -gpu             # GPU topology (model, driver, NUMA)
```

### Queues
```bash
bqueues                  # all queues
bqueues -l <queue>       # detailed queue info
```

### Available hosts
```bash
bhosts                   # host status
```

---

## Storage

- **Home**: `/u/ruichenzhao/` (100 GB quota, currently ~5% used)
- **Check quota**: `gquota ~`
- **Project storage**: `/dccstor/` (request access if needed)

---

## Existing Setup on CCC

- **Repos**: `~/repos/research-novel-devices`, `~/repos/research-qubit-calculations`
- **Conda env**: `~/qiskit_env.yml` (Qiskit environment definition)
- Limited modules available (`module avail` shows mostly defaults)

---

## Tips

- Use `tmux` on compute nodes for long-running interactive sessions
- `interactive` queue has 6-hour limit; use `normal` for longer jobs
- Most hosts are idle most of the time - plenty of A100-40G availability
- For 80G VRAM needs, target `cccxc515`, `cccxc547`, `cccxc548`, `cccxc549` (A100-80G) or `cccxc701+` (H100)
- Submit jobs from login nodes, not compute nodes
