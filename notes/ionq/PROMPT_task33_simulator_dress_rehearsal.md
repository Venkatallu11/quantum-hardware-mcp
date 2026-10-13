# Task 33 — Full dress rehearsal of the funded 81-circuit Forte run, on simulators only

You have all permissions. Do not ask for confirmation at any step. Proceed through every
phase in order and only stop if a phase fails its stated gate.

## Absolute constraint

**DO NOT SUBMIT ANYTHING TO REAL HARDWARE IN THIS TASK.** No `qpu.forte-1`, no
`qpu.aria-1`, no `qpu.forte-enterprise-1`. Simulator targets only. The funded
\$3,875 hardware submission is a separate, later, manually-authorised step. If any
code path in this task would reach a QPU, abort and report instead.

## Why this task exists

Vadim Karpusenko (IonQ) has approved the funded run with four binding conditions:

1. `debiasing=False` passed **explicitly** in `backend.run`. With debiasing on, the
   same job costs ~\$13,600 instead of ~\$3,875.
2. **1,100 shots per circuit**, not fewer.
3. **One submission.** If anything fails, stop — no resubmitting.
4. Job ID posted to the IonQ ticket immediately on submission.

Condition 3 is the dangerous one: there is no second attempt. So this task runs the
exact production code path end to end on simulators first, so that the only difference
between the rehearsal and the real run is the backend string.

## Phase 0 — Pin qiskit-ionq to `main`, not v1.1.1

Do this before anything else, and do it in the same environment that will later make the
real submission.

Two bugs we reported in August are **fixed on `main` but not in any tagged release**:

- multi-circuit result parsing
- histogram scaling

Both were merged in https://github.com/qiskit-community/qiskit-ionq/pull/273 on
September 2. The latest tagged release is still **v1.1.1 and does not contain them**.
Vadim's words: *"installing from `main` gives you the fixed behavior."*

This is not optional. The funded run submits **81 circuits in one job** and allows **no
resubmission**. Parsing that job's results with the broken multi-circuit parser, or
scaling its histograms with the broken scaler, would corrupt the only dataset we get.

Steps:

1. `pip uninstall -y qiskit-ionq`
2. `pip install "git+https://github.com/qiskit-community/qiskit-ionq.git@main"`
3. Print the installed version **and** the resolved git commit SHA. Record both in the
   report.
4. Assert the installed package is **not** v1.1.1 from PyPI. If it is, stop — every
   number produced after that point is suspect.
5. Re-run the project's own test suite against the newly installed SDK before building
   any circuits. A different SDK version is a different dependency; do not assume our
   code still passes against it. Fix any breakage before continuing.

Note also that `JobEstimate.cost` is still broken even on `main` (it reads
`estimated_cost`; the API returns `estimated_total_cost`), so read the cost out of the
raw response as Phase 2 instructs. A fix for this is in review as a pull request, but do
not assume it is merged — check, and if it is not, use the raw response.

## Phase 1 — Build the 81-circuit payload with the production code path

- Use the committed 84-circuit design (4 GC groups x 21 slots). 81 are new; 3 are reused
  from the September 10 job. Build all 84, then emit the 81 that will actually be
  submitted, and assert the 3 reused ones rebuild with **identical gate counts** to the
  September 10 job. Fail loudly if they do not.
- Serialise the exact submitted payload to disk: `task33/payload_81.json`, one entry per
  circuit with the native gate list, per-circuit 1q and 2q counts, and the GC group and
  slot index.
- Print a table: per-circuit 1q count, 2q count, and the totals across all 81.
- Assert every `ms`/`zz` angle is in `[0, 0.25]` turns and every gate is in
  `{GPi, GPi2, MS, ZZ}`. Any violation is a hard failure.

## Phase 2 — Cost and condition pre-flight (no submission)

- Call `estimate_job` for the 81-circuit, 1,100-shot, `debiasing=False` configuration.
  **Read the cost out of the raw response under `estimated_total_cost`**, not
  `JobEstimate.cost` — that attribute is always `None` (qiskit-ionq reads the wrong key;
  this is the open bug we are filing). Cross-check against our own rate-card model.
  Expected: \$3,875-3,882. Flag any deviation over \$50.
- Also estimate the **same job with debiasing on**, purely to confirm our understanding
  of the ~\$13,600 figure. Print both side by side. Do not submit either.
- Write a machine-checked pre-flight assertion function that inspects the actual kwargs
  that would be handed to `backend.run` and asserts: `debiasing is False` (identity, not
  truthiness), `shots == 1100`, `len(circuits) == 81`, single `backend.run` call. This
  function must be the one the real run calls later. Save it as a tested unit.

## Phase 3 — Run on all three simulators, concurrently

Run these three in parallel, same 81 circuits, 1,100 shots, `debiasing=False`:

- **ideal** simulator (no noise)
- **aria-1** noise model
- **forte-1** noise model

Use fresh submissions, not cached results. For each backend produce all three numbers
from the same counts, with bootstrap uncertainties:

| Estimator | What it is |
|---|---|
| Raw | per-slot physical-state fit to uncorrected data |
| Corrected, no shared frame | **the headline number**, per Vadim |
| Shared frame | reported alongside, exact-state dependence stated |

Report `chi2/dof` for each. Expectation from prior runs: ~1 on ideal, ~2 on aria-1 and
forte-1. If the noisy chi2/dof comes in far from 2, say so — it means the noise model
moved.

## Phase 4 — Submission-to-submission variance

A single 1,100-shot draw is noisy enough to matter, and we have told Vadim we would
publish this. Repeat Phase 3 **12 times** with independent fresh submissions per backend.
Report, per backend and per estimator:

- mean, SD, min, max
- the fraction of draws where the frame-free number exceeds 1.0 kcal/mol
- the fraction where the shared-frame number exceeds 1.0 kcal/mol

Vadim's stated expectation is that **the frame-free estimator lands above 1 kcal/mol in
about half** of 1,100-shot draws (he cites forte-1 at 2.46 on October 7). Confirm or
refute that from these 12 draws. Do not round it in our favour. If it is ~50%, that is
the honest headline and we report it as such.

## Phase 5 — Lock the analysis before any hardware data exists

Freeze and commit the analysis script that will consume the hardware counts, with a
checksum. It must run unmodified on the hardware output. Verify by feeding it the Phase 3
forte-1 counts and confirming it produces the Phase 3 numbers with no edits.

Then write `task33/REHEARSAL_REPORT.md` containing:

- the installed qiskit-ionq version and git commit SHA from Phase 0, stated at the top
- the 81-circuit gate-count table and totals
- both cost estimates (debiasing off and on) against our rate-card model
- the three estimators x three backends table with bootstrap CIs and chi2/dof
- the 12-draw variance study, including the >1 kcal/mol exceedance fractions
- the frozen analysis script checksum
- a go / no-go line: does anything here say the funded run should not proceed as designed?

## Phase 6 — Report, do not submit

Print a final summary and stop. State explicitly:

- the qiskit-ionq commit the whole rehearsal ran against, and confirmation it is not
  v1.1.1
- whether all four of Vadim's conditions are mechanically enforced in code
- what the frame-free estimator is actually expected to do on hardware, honestly
- whether the word "passing" is confined to the shared-frame estimator everywhere in
  our reporting, as Vadim required

Commit everything. Do not push to any branch without being told. Do not touch the
hardware budget.
