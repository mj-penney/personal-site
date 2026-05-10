+++
date = '2026-05-03T17:50:37+01:00'
draft = false
title = 'Cyclops'
+++

## Project Overview

**Cyclops** is a framework for investigating CPU behaviour/architecture and
custom code performance.

It allows the user to create experiments using low-level resources like PMU
counters and `rdtscp`, with sensible implementations to improve avoid pitfalls.
**Cyclops** makes it easy to design reproducible and accurate experiments that
can be easily orchestrated from the command line and visualised to gain
insights.

## Features

- Batches with user-defined warmup runs, batch runs and aggregation
- Workload plugin system to easily write custom workloads with scriptable
  workload parameters (see `docs/workload.md`)
- Parameter sweeps to run multiple batches while varying one parameter and
  keeping the others fixed
- Metric groups including raw metrics like PMU counters (`perf_event_open()`)
  and `rdtscp`, and ratios like IPC (see `docs/metrics.md` for measurement
  methodology)
- Results are written to stdout & CSV files with metadata for reproducibility
- Python scripts for running and visualising some default experiments - see
  `experiments/`

## Build and Run

```bash
# clone repository
git clone https://mj-penney/cyclops.git cyclops

# enter repository
cd cyclops

# build
make

# run (generate help text)
./cyclops -h
```

## Example Usage

### Example 1

- Single batch
- 10 warmup runs
- 20 batch runs
- `BRANCH` workload with default params
- `IPC` metric group

```bash
./cyclops -u 10 -r 20 -w BRANCH -m IPC
```

### Example 2

- 3 warmup runs
- 5 batch runs
- `STRIDED_ARRAY` workload
- `L1D_READS` metric group
- 19 batches, sweeping the `array-elements` param from 10 to 100 in steps of 5

```bash
./cyclops -u 3 -r 5 -w STRIDED_ARRAY -m L1D_READS -s array-elements=10:100:5
```

## Experiments

The `cyclops` tool is designed to be highly scriptable, and make it easy to
design performance & microarchitecture experiments.

In `experiments/` there are example Python scripts for running experiments.
To run these experiments, you will first need to build the `cyclops` binary
(see above).

You will then need to create a Python virtual environment and install the
necessary packages:

```bash
cd experiments
python -m venv venv
pip install -r requirements.txt
```

Before running the example experiments, activate the virtual environment you
just created:

```bash
source venv/bin/activate
```

Then run the experiments:

```bash
python3 estimate_cache_size.py
python3 branch.py
```

This will generate the figures below (png files will be saved in
`experiments/`).

### L1 Cache and LLC Size Estimation

This experiment uses the `STRIDED_ARRAY` workload, sweeping through increasing
array sizes, to estimate L1D and LLC capacities from cache miss rates.

This experiment can be run with:

```bash
python3 estimate_cache_size.py
```

#### Results

![Alt txt](/img/estimate_cache_size.png)

As the array size increases, and exceeds the size of a cache, the cache can no
longer hold all the data.
Some will need to be fetched from other caches or DRAM, resulting in an
increase in the cache miss rate at this point.

Here we can see that there is a large jump in the L1D miss rate when the array
is between 2\*10^4 and 4\*10^4 Bytes, and a large jump in LLC miss rate
between 2\*10^6 and 4\*10^6.

These ranges align with the actual cache sizes for my CPU:

- **L1D:** 32KiB per physical core
- **L3:** 3MiB
