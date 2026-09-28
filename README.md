# Compute error budgets for a platform review

A harbor-format benchmark task, authored for the AfterQuery Experts programme as data
annotation for AI evaluation. Task slug `afterquery/reckon-error-budgets` — **Software / Site Reliability**.

An AI agent gets a container, this brief and nothing else. It works alone for
4 hours. Then a sealed verifier it never sees scores the result `0` or `1`.

## What it asks for

Error budgets for the platform review

The full brief is in [`instruction.md`](instruction.md).

| | |
| --- | --- |
| **Deliverable** | `/app/error_budget.csv` |
| **Agent budget** | 4 hours (14400s) |
| **Expert estimate** | 4 hours |
| **Machine** | 1 CPU, 2048 MB RAM, 0 GPU |
| **Verifier budget** | 300s |
| **Tags** | `slo`, `error-budget`, `observability`, `log-analysis`, `python` |

## Why it is hard

Working out each service's good-request share and error budget for a review is routine work for a site reliability engineer or a platform engineer on call for the observability stack: take the collector's request records, apply each service's objectives from the catalogue, and report by week. The data is synthetic and was made truth first. 207516 requests were generated service by service over a fortnight, each answered by one real service, and the 16 sealed figures are counted from which service really answered.

<details>
<summary>Full note</summary>

The collector's records were written afterwards, stream by stream. The figures are exact and nothing is computed at marking time. The team's note gives every rule the marking uses. Read literally it gets 2 of 16, because three streams plainly do not follow it: one reports latency in microseconds, one in seconds, and one writes the status as text such as "503 Service Unavailable". Each is obvious on sight, and each spoils that service's two weekly lines where only 1 line may be wrong. What decides the task is the relabelling. The collector's rules say which service each stream carries, and for 4 streams they are wrong: two pairs of rules were pasted the wrong way round when the collector was rebuilt. Nothing records it, and the ruled reading fails silently - every service still gets a believable good share and budget - at a cost of 8 lines. Nothing first-order shows it, by construction: each crossed pair is two services with the same latency objective and the same target, much the same traffic, and the same latency distribution, so volumes, percentiles and error rates are all unremarkable either way. What differs is the runtime, which the catalogue lists. A service on a garbage-collected runtime slows as its heap fills and snaps back when it is collected, so one request's latency foretells the next one's; on a native runtime it does not. The correlation between neighbouring ordinary requests' log latencies is at least 0.27 on every stream that really carries a collected service and at most 0.01 on every other, and four streams show the runtime of the other service in their pair. No weekly percentage ever looks at the order of requests, and the note never says what a runtime does to latency. tests/review_check.py measures all of this against the sealed figures and refuses to pass otherwise: of 16 readings the right one gets 16 and the best other 14, against a bar of 15.

</details>

## How it is verified

The verifier reads /app/error_budget.csv with the csv module and compares each service-week's good_pct within 0.002 and budget_used_pct within 0.5 with /tests/sealed.json; at most 1 of the 16 lines may be wrong. None of the agent's code is imported or run. The sealed figures exist only in the verifier image, /tests is mode 700 there, and the agent's container never mounts it.

<details>
<summary>Full note</summary>

The tolerance on good_pct is below one request in a week's traffic, which is fair because goodness is decided exactly: no latency lies within a tenth of its objective, since ordinary requests stop at nine tenths of it and slow ones begin at 1.15 times it. The two services of a crossed pair differ by more than the tolerances in every week, so the ruled reading gets none of their lines right by chance. Nothing in the agent's container attests any figure and nothing has to balance; a sheet made on the rules as written is as self-consistent as the right one. The reward is binary and is written to /logs/verifier/reward.txt on every path; an empty container scores 0 and the reference scores 1. The generator is seeded and shipped.

</details>

<details>
<summary>Reference approach (spoiler)</summary>

solution/budgets.py reads only /app and writes /app/error_budget.csv with the standard library. It groups the records by stream and undoes each stream's habit - microseconds or seconds to milliseconds by the size of the middle latency, a text status to its leading number. For each stream it takes the correlation between the log latencies of neighbouring ordinary requests and calls the stream collected or native; a stream that shows the other runtime from the service ruled onto it is given to the other such service with the same objectives. Good requests are those with a status below 500 and a latency no more than the objective; a week's good share and the budget used follow the note. In the task's own container it gets 16 of 16.

</details>

## Layout

```
instruction.md          the brief - the only thing the agent is told
task.toml               artifacts, timeouts, resources, metadata
environment/            Dockerfile and data for the agent's container
solution/solve.sh       reference solution; scores 1 by construction
tests/                  verifier image, grading logic and ground truth
```

## Running it

With [harbor](https://github.com/laude-institute/harbor) installed and Docker running:

```bash
uv tool install harbor

harbor run -p . -a oracle -e docker   # reference solution - must score 1
harbor run -p . -a nop    -e docker   # nothing happens    - must score 0
```

Those two runs are the calibration the bundle has to satisfy: the oracle run substitutes
`solution/solve.sh` for the agent and must score `1`, the nop run does nothing and must
score `0`. Anything in between means the verifier is measuring the wrong thing.

## Reading this

`tests/` and `solution/` hold the answers. To try the task honestly, read only
`instruction.md` and `environment/`.

---

Part of a set of 38 tasks indexed at
[afterquery-data-annotation](https://github.com/helexdev1004/afterquery-data-annotation).
