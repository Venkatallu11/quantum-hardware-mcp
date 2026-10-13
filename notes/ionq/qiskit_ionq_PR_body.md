# PR title

Fix JobEstimate.cost always returning None

# PR body (paste below the title)

## Summary

`JobEstimate.cost` is `None` after every successful `IonQClient.estimate_job()` call. The class reads the response key `estimated_cost`, but `GET /jobs/estimate` returns the value under `estimated_total_cost`. Because the attribute is populated with `dict.get()`, the mismatch fails silently — no exception, just `None`.

## Reproduction

```python
from qiskit_ionq import IonQProvider

provider = IonQProvider()
backend = provider.get_backend("qpu.forte-1")

est = backend.client.estimate_job(
    backend="qpu.forte-1",
    oneq_gates=285,
    twoq_gates=11,
    qubits=8,
    shots=1100,
)

print(est.cost)       # None
print(repr(est))      # JobEstimate(..., cost=None None, ...)
print(est.to_dict())  # {... 'estimated_total_cost': 2254.88, ...}
```

The value is in the payload the whole time, under a different key. Reading `estimated_total_cost` out of `to_dict()` gives correct quotes: a 54-circuit plan on `qpu.forte-1` returned $2,254.88, within $5 of the published rate card for the same gate and shot counts. So the API side looks right; this is purely the client-side key name.

## Why it matters

`estimate_job()` exists so cost can be checked *before* committing to a hardware submission. Returning `None` instead of the quote defeats that. And because it is a silent `None` rather than a raised error, a caller doing arithmetic on it gets a `TypeError` far from the real cause, while a caller formatting it into a log or a budget check just records `"None"` and moves on. On a job in the low thousands of dollars that is an easy mistake to make.

## Changes

- `JobEstimate.__init__` reads `estimated_total_cost`, falling back to `estimated_cost` so any deployment still emitting the older spelling keeps working.
- `__repr__` no longer renders `cost=2254.88 None` when the payload carries no `cost_unit`; the unit is omitted when absent.
- Adds `test/ionq_client/test_job_estimate.py` — as far as I can tell the first coverage for this class. Eight tests: key precedence, the fallback, the absent-key case, a representative full payload, and both repr paths.

## Verification

- `test/ionq_client/` — 12 passed.
- Full suite — 488 passed, 2 pre-existing failures in `test/ionq_gates/test_gates.py` (`test_qasm3_export`, `test_qasm_export_transpiled`), both `MissingOptionalLibraryError` from a missing optional dependency in my environment. Confirmed identical on a clean checkout of `a4a6da4` before these changes.
- **Five of the eight new tests fail without the source change**, so they pin the actual regression rather than just describing it.
- `black --check` clean; `pylint --rcfile=.pylintrc` rates both touched files 10.00/10.

## Note

One thing worth a maintainer's eye: I confirmed the client half directly against `ionq_client.py:511`, but my evidence for the server returning `estimated_total_cost` is observed API responses rather than the API spec. If the canonical field name is something else, the fallback ordering should change accordingly.

Found while budgeting a funded H4 quantum-chemistry run on Forte Enterprise 1 under the IonQ Research Credits Program. Reported at the suggestion of Vadim Karpusenko (IonQ DevRel).

🤖 Generated with [Claude Code](https://claude.com/claude-code)

https://claude.ai/code/session_01JX7Yp28x6ymu1vp57T9KuK
