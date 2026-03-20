---
name: submit-slurm-job
description: "Submit GPU compute jobs to the Slurm cluster. Handles sbatch script generation with correct partition, GPU type, python path, and node selection."
user_invocable: true
---

# Submit Slurm GPU Job

## Overview

Generate and submit sbatch scripts for GPU compute jobs on the cluster. Handles all cluster-specific details: partition, GPU types, node selection, python path.

## Configuration

Before using this skill, set the following in your project's `CLAUDE.md` or environment:

| Variable | Example | Description |
|----------|---------|-------------|
| `PYTHON_PATH` | `/path/to/miniconda3/envs/myenv/bin/python3` | Full path to Python interpreter |
| `PROJECT_DIR` | `/path/to/your/project` | Project root / default output directory |
| `PARTITION` | `home` | Slurm partition name |

## Workflow

### Step 1: Gather Parameters

Ask the user (with `AskUserQuestion`) what they want to run. Key parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `job_name` | (required) | Short job name for SBATCH |
| `gpu_type` | `A800` | GPU type (see table below) |
| `n_gpu` | `1` | Number of GPUs |
| `time` | `24:00:00` | Wall time limit |
| `mem` | `32G` | Memory |
| `cpus` | `4` | CPUs per task |
| `script` | (required) | Python script path |
| `args` | (required) | Script arguments |
| `output_dir` | `{PROJECT_DIR}` | Directory for log files |

### Step 2: Select GPU and Node

Available GPU types and their nodes:

| GPU Type | Node | GRES Syntax |
|----------|------|-------------|
| V100 | n002, n003 | `gpu:V100:1` |
| A100_80G | n001 | `gpu:A100_80G:1` |
| A800 | n004 | `gpu:A800:1` |
| A100_40G | n005 | `gpu:A100_40G:1` |
| NV5090 | n006 | `gpu:NV5090:1` |
| H200 | n007 | `gpu:H200:1` |

**CRITICAL**: GPU type MUST be explicit in `--gres` (e.g., `gpu:A800:1`, NOT `gpu:1`).

If the user wants multiple GPUs on the same node, use `gpu:TYPE:N` (e.g., `gpu:A800:2`).

If the user specifies a node directly (e.g., "submit on n004"), infer the GPU type from the table above.

### Step 3: Write and Submit sbatch Script

Generate the sbatch script following this template:

```bash
#!/bin/bash
#SBATCH --partition={PARTITION}
#SBATCH --cpus-per-task={cpus}
#SBATCH --mem={mem}
#SBATCH --gres=gpu:{gpu_type}:{n_gpu}
#SBATCH --nodes=1
#SBATCH --time={time}
#SBATCH --job-name={job_name}
#SBATCH -o {output_dir}/{job_name}_%j.out

echo The current job ID is $SLURM_JOB_ID
echo Running on $SLURM_JOB_NODELIST
echo CUDA devices: $CUDA_VISIBLE_DEVICES
echo ==== Job started at `date` ====
nvidia-smi
echo

{PYTHON_PATH} \
    {script} \
    {args}

echo
echo ==== Job finished at `date` ====
```

Key rules:
- **Partition**: Use the configured `{PARTITION}` (check your cluster's available partitions)
- **Python path**: Use full path `{PYTHON_PATH}` (do NOT use `conda activate`)
- **Output**: Use `-o` for single combined stdout+stderr file
- **No `--nodelist`** unless the user explicitly requests a specific node

Write the script to `{output_dir}/{job_name}.sh`, then submit with `sbatch`.

### Step 4: Report

After submission, report:
- Job ID(s) from sbatch output
- Log file location: `{output_dir}/{job_name}_{JOBID}.out`
- How to monitor: `squeue -u $USER`, `tail -f {log_file}`
- How to cancel: `scancel {JOBID}`

## Multiple Jobs

If the user wants to submit multiple jobs (e.g., different datasets on different GPUs):
1. Create separate `.sh` scripts for each job
2. Submit all with a loop: `for f in script1.sh script2.sh ...; do sbatch $f; done`
3. Report all job IDs

## Common Patterns

### Sequential tasks on one GPU
Put multiple python commands in a single script, separated by `echo` markers.

### Parallel tasks across GPUs
Create separate scripts, each requesting one GPU. Submit all scripts independently.

### Using `--nodelist` for specific nodes
Only add `#SBATCH --nodelist=n004` when the user explicitly wants a specific node. Otherwise let Slurm schedule based on GPU type availability.
