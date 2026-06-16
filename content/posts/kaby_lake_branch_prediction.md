---
title: "Intel Kaby Lake: Branch Prediction"
date: 2026-06-06T17:50:18+01:00
draft: false
toc: false
images:
tags:
  - cyclops
  - microarchitecture
---

## Introduction

This is the first in what I hope will be a multi-part investigation into
branch predictors, starting with my laptop's Intel Kaby Lake CPU.
Initially I just want to get a general overview the predictor's behaviour
rather than answer specific questions about its structure (e.g. "How many
predictor tables are there?").

Intel's Kaby Lake microarchitecture uses the same branch predictor as Skylake.
Like many modern branch predictors it has a hybrid structure that likely
incorporates multiple predictor tables and a global history in a TAGE-like
design.

## Methods

All experiments in this article are conducted using my microarchitecture
research tool [Cyclops](https://github.com/mj-penney/cyclops).

To profile the branch predictor I designed a workload that loops over an array
of a fixed length containing a repeating pattern.
For each iteration a branch is taken (T) if the current value in the pattern is
"1" and not taken (NT) if it is "0".
I vary the length of the pattern and train the branch predictor on the repeated
pattern using warmup runs before measuring results.
For this investigation the patterns are unbiased towards T/NT.

The predictor tables in TAGE-like predictors are indexed in part by using the
memory address of the branch instruction, or program counter (PC).
My workload currently uses only one branch PC, so it will not stress the
predictor table capcities.
However, varying the pattern length will tell us about how effectively the
predictor is able to represent the history of branch prediction results.

## Profiling

Initially I wanted to do a coarse-grained sweep of pattern lengths to see at
which length the predictor starts to struggle, and how its accuracy degrades.

<img src="/img/branch_fig_1.png" alt="Branch Misprediction Curves" class="center" />

The branch predictor achieves effectively no mispredictions for short
patterns (<1k).
At lengths of ~2-3k the misprediction rate rises exponentially.
It then flattens out and converges to 50% for very long patterns.

The smooth transition from near-perfect prediction to 50% mispredictions is
consistent with a predictor that uses multiple history lengths.
A predictor indexed using a single fixed history length would be expected to
exhibit a much sharper transition.
While this does not uniquely identify TAGE, it is consistent with
reverse-engineering work which point to a TAGE-like design.

The convergence to 50% mispredictions for long patterns suggests the predictor
is unable to distinguish any structure and can only make uninformed guesses.

Instructions per cycle (IPC) starts at about 1.12 for short patterns where
there are no mispredictions.
IPC drops sharply at around the same point that the misprediciton rate starts
increasing, before also starting to level off and converges to ~0.44 for very
long patterns.

The peak IPC value of 1.12 while at ~0% mispredictions was lower than I
expected considering the instruction level parallelism of Kaby Lake allows
several instructions to retire per cycle.
I suspect the IPC is being capped by a dependency chain in the workload,
specifically on the `sum` and `i` variables, who's values depend on their
values at the previous loop iteration.

Having seen how the predictor behaves over a wide range of pattern lengths, I
repeated the experiment for more focused ranges.
The figure below shows the point at which the predictor starts making mistakes.

<img src="/img/branch_first_mispredictions.png" alt="Branch Misprediction Curves" class="center" />

The predictor achieves effectively perfect prediction for random, unbiased
patterns of length up to ~900.
Beyond that, the exponential increase in mispredictions begins.

The next figure shows the misprediction rate and IPC for pattern lengths 2-5k,
where the majority of the increase in misprediction rate occurs.

<img src="/img/branch_fig_2.png" alt="Branch Misprediction Curves" class="center" />

The thing that jumped out at me here was that IPC did not decrease
significantly until the misprediction rate was already at ~10% (pattern length 
~3750), remaining above 1.1 until this point.
For this workload the CPU's performance is bottlenecked by something other
than branch misprediction penalties until the misprediction rate exceeds 10%.
This is consistent with my earlier suggestion that something else is capping
the IPC at 1.12.

It is also at about this point that the misprediction rate stops exponentially
increasing and begins to converge towards 50%.

## Key Results

- The smooth, broad structure of the misprediction rate vs pattern length curve
  is consistent with the idea that the predictor has a multi-table,
  multi-history-length, TAGE-like structure.
- For a single branch address, the predictor is able to achieve effectively 0
  mispredictions for random unbiased patterns of length up to ~900 once
  trained.
- The asymptotic approach to 50% mispredictions suggests that prediction never
  becomes systematically worse than random guessing.

## Future Investigations

- Vary the number and distribution of branch PCs.
    - Can I use this to infer more specific details about the predictor's
      structure?
- Vary the bias of the pattern towards T/NT.
    - Does the direction of the bias affect predictor performance?
- Compare results for different microarchitectures.
