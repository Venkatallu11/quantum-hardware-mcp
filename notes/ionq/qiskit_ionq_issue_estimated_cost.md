# JobEstimate.cost is always None: reads `estimated_cost`, API returns `estimated_total_cost`

## Summary

`JobEstimate.cost` is always `None` for every successful call to `IonQClient.estimate_job()`.
The class reads the response key `estimated_cost`, but the `GET /jobs/estimate` endpoint
returns the cost under `estimated_total_cost`. Since the attribute is populated with
`dict.get()`, the mismatch fails silently and the user gets `None` rather than an error.

## Affected code

`qiskit_ionq/ionq_client.py`, in `JobEstimate.__init__` (line 511 on `main`):

```python
# Core numeric results
self.cost: float | None = data.get("estimated_cost")          # <-- always missing
self.cost_unit: str | None = data.get("cost_unit")
self.exec_time: float | None = data.get("estimated_execution_time")  # seconds
self.queue_time: float | None = data.get("current_predicted_queue_time")  # sec
```

`IonQClient.estimate_job()` (line 345) passes the decoded response body directly:

```python
url = self.make_path("jobs", "estimate")
...
return JobEstimate(res.json())
```

so the wrapper sees the raw API payload and the key simply is not there.

## Reproduction

```python
from qiskit_ionq import IonQProvider

provider = IonQProvider()                      # IONQ_API_KEY in env
backend  = provider.get_backend("qpu.forte-1")

est = backend.client.estimate_job(
    backend_name="qpu.forte-1",
    qubits=8,
    gate_counts={"1q": 285, "2q": 11},
    shots=1100,
)

print(est.cost)        # None
print(est.cost_unit)   # None
print(repr(est))       # JobEstimate(..., cost=None None, ...)

# the value is present in the payload, under a different key:
print(est.to_dict())   # {... 'estimated_total_cost': 2254.88, ...}
```

Observed: `est.cost is None`.
Expected: `est.cost == 2254.88` (the figure the API actually returned).

Reading `estimated_total_cost` out of `to_dict()` gives correct quotes. For a 54-circuit
plan on `qpu.forte-1` it returned \$2,254.88, which is within \$5 of the published
rate-card calculation for the same gate counts and shot count, so the API side looks right
and this is purely the client-side key name.

## Why it matters

`estimate_job()` exists so that cost can be checked *before* committing to a hardware
submission. Returning `None` instead of the quote defeats that, and because it is a silent
`None` rather than a raised error, a caller that does arithmetic on it gets a
`TypeError` far away from the real cause, while a caller that formats it into a log or a
budget check just records "None" and moves on. On a job in the low thousands of dollars
that is a meaningful footgun.

## Suggested fix

Read the documented key, while staying tolerant of the older name in case any deployment
still emits it:

```python
self.cost: float | None = data.get("estimated_total_cost", data.get("estimated_cost"))
```

Two related points the maintainers may want to fold in:

1. `cost_unit` also comes back as `None` in the payloads observed here. If the endpoint
   does not return a unit, the `__repr__` renders as `cost=2254.88 None`; defaulting it to
   `"USD"`, or omitting the unit from the repr when it is absent, would read better.
2. Nothing in the class validates that the cost was found. A one-line warning when the
   payload carries neither key would have surfaced this immediately instead of silently
   producing `None`.

A regression test asserting `JobEstimate({"estimated_total_cost": 1.23}).cost == 1.23`
would pin the behaviour.

## Environment

- `qiskit-ionq`: v1.1.1 (also present on `main` as of 2026-10-13)
- Backend: `qpu.forte-1`
- Endpoint: `GET /jobs/estimate`

Happy to open a PR with the fix and the test if that is useful.

---

Context: found while budgeting a funded H4 quantum-chemistry run on Forte Enterprise 1
under the IonQ Research Credits Program. Reported at the suggestion of Vadim Karpusenko
(IonQ DevRel), who is aware of this issue and the two earlier parsing bugs fixed in #273.
