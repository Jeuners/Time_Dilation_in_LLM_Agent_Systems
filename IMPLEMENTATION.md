# Implementation evidence and correction record

Audit date: 6 September 2026. This is a source and saved-data review, not an
independent replication or a new empirical study.

- Paper baseline: [`ca45b8946ff0d6933afba8bb34c0b49610c333ed`](https://github.com/Jeuners/Time_Dilation_in_LLM_Agent_Systems/tree/ca45b8946ff0d6933afba8bb34c0b49610c333ed).
- Implementation baseline: [`da935366521a00c25ae940ea0581ac9800064782`](https://github.com/Jeuners/logpyclaw/tree/da935366521a00c25ae940ea0581ac9800064782).
- Companion additions prepared for LogpyClaw: `docs/TIME-DILATION.md` and
  `experiments/README.md`. These paths are new documentation and are not
  claimed to exist at the pinned implementation baseline.

## Follow-up implementation: steps 1–3

A subsequent implementation is pinned at [LogpyClaw `3b4ce2b`](https://github.com/Jeuners/logpyclaw/tree/3b4ce2b).
It adds monotonic dispatch durations, separate model/tool/delegation spans,
bounded model/backend/configuration-specific samples, and an optional measured
latency block in Martin's planner. Unknown or stale estimates are explicit;
explicit agent selection retains precedence. Timing and supplied routing evidence
are attached before response signing. Forwarded signed child responses are
wrapped without mutating the original message.

Martin and Alice use local Ollama `qwen3.5:latest` for these functional checks.
The planner requests structured JSON; invalid plans and browser-stream failures
are recorded as failures. The historical CDC signature scope is unchanged.

See the [operational contract](https://github.com/Jeuners/logpyclaw/blob/3b4ce2b/docs/MEASURED-LATENCY.md)
and [verification record](https://github.com/Jeuners/logpyclaw/blob/3b4ce2b/docs/testing/measured-latency.tdd.md):
280 software tests passed, with 98% coverage of the new timing core. Local live
checks exercised direct replies and delegation. These are functional checks,
not a controlled replication or evidence of improved routing performance.
Statistics reset on process restart; automatic model-digest refresh and
per-task-class estimates remain outside this implementation.

The mapping below intentionally describes the earlier baseline. In particular,
the follow-up changes the protocol-rate clock to monotonic time and adds action
latency observation separately; it does not reinterpret old experimental data.
The prospective context-form experiment remains to be conducted.

## Claim-to-source mapping

All source links below are pinned so that a future model or deployment update
does not silently change the implementation being described.

| Paper topic | Source evidence | Consequence |
| --- | --- | --- |
| §4.2 operations and pace | [Agent base](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/backend/agents/base.py), [LLM handle](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/backend/agents/llm_agent.py) | Unit protocol ticks; LLM entry is ticked before inference; rate uses time between ticks, including idle time. |
| §§4.4–4.5 clock fields and comparisons | [CDC](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/backend/core/cdc.py) | `vector` = V, `tau` = cumulative progress, `dilation` = rate; generated `wall_ts`; no separate `pace` field. |
| §4.3 faction γ | [Faction protocol](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/backend/core/faction_protocol.py) | Learned source/target pace ratio; different convention from the paper's proposed cost conversion. |
| §5 signature guarantee | [Message signing payload](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/backend/core/protocol.py) | Clock V and rate are signed; τ and generated `wall_ts` are excluded. |
| §5 periodic peer initiative | [Conductor](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/backend/agents/conductor.py), [initiative loop](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/backend/services/initiative.py) | Agent-initiated missions exist but still traverse a central dispatcher. |
| §5 external protocol | [A2A routes](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/backend/api/a2a/gateway_router.py) | JSON API, not XML tasklists. |
| §§5, 7 memory and prompts | [Semantic memory](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/backend/core/memory.py), [planner and startup](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/backend/app.py) | SQLite/sqlite-vec; recall inserts text, not automatically all timestamp fields. |
| §§5, 7 consolidation | [Dream service](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/backend/services/dream.py) | Generates dream prompts and images; does not implement nightly memory consolidation. |
| §§5, 7 logging | [Text logger](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/backend/core/logging.py) | Wall-clock formatted text; does not establish a uniform dual-timestamp tuple or prompt exposure history. |
| §5 runtime requirements | [Project manifest](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/pyproject.toml) | Declares Python ≥3.12 and sqlite-vec. A particular deployment's Python version is a separate claim. |

Searches of the pinned `backend/` and `tests/` found no `TimeProvider`,
`reference_now` or `parent_reference_now` implementation. Their previous
“implemented, tested” status is unsupported for this revision. This does not
assert that those names never existed in another repository or deployment.

## Raw-data reconciliation

| Section | Versioned artifact | Recount / qualification |
| --- | --- | --- |
| §6.4 | [dragon3 results](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/experiments/dragon3-results.json) | Execution success 5/10 vs 3/10. Stored winnable oracle summary 9/10 vs 7/8; earlier paper subgroup 7/7 vs 3/5 not reproduced. |
| §6.5 | [dragon4 results](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/experiments/dragon4-results.json) | Execution success 18/30 vs 21/30; all 33 no-cooldown trials delegate; cooldown self-action 12/14 vs 8/13. |
| §6.6 | [dragon5 results](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/experiments/dragon5-results.json), [script](https://github.com/Jeuners/logpyclaw/blob/da935366521a00c25ae940ea0581ac9800064782/experiments/dragon5.py) | 200 records; oracle agreement 100/100 vs 55/100; execution success 95/100 vs 57/100. |

In dragon5, the oracle and treatment share rolling-median action-cost estimates.
The name-to-backend assignment and name order are randomised; treatment arms
strictly alternate. `live_rates` is recorded separately and does not generate
the prompt's action-latency estimates. Decisions therefore test useful exclusive
latency information, not the causal effect of the CDC data structure itself.

The original success endpoint excludes decision time. Adding `decision_s` to
`exec_s` using the rounded saved records gives 93/100 vs 54/100. This is
post-hoc, still excludes calibration, and must not replace the pre-specified
endpoint silently. No new model calls were made for this check.

The JSON's `0.0` Fisher values are six-decimal rounding artifacts. Two-sided
hypergeometric-tail recomputation yields `8.927340409087128e-17` for oracle
agreement and `1.2688661009449438e-10` for execution success, consistent with
the paper's reported scientific-notation values.

The historical 464-mission / 1,719-message observational corpus and its signing
coverage were not reconstructed here. A repository containing experiment
results is not equivalent to a complete public archive of the original mission
database, actual prompts or deployment history.

## Corrections made in the paper

1. Match CDC notation, field names, merge rules and classifier cases to source.
   Distinguish faction pace ratios from the proposed cross-agent cost conversion.
2. Correct the suspended-call monotonicity statement and remove the unsupported
   assertion that protocol-count ratios lower-bound finer reasoning divergence.
3. Replace the implementation-status table with revision-specific evidence,
   including existing rate dispersion and missing time-provider/consolidation
   components. Narrow signature claims to fields actually covered.
4. Preserve the central experiment counts while clarifying oracle agreement,
   execution-only deadlines and the distinction between CDC rates and measured
   action latency. Identify the pilot subgroup discrepancy explicitly.
5. State the instrumentation paradox as an open exposure-dependent hypothesis.
   A shared log-to-memory store, nightly rewriting and a historical increase in
   prompt annotations were not established by the checked code.

## Validation and next experiment

The existing focused suite passed: **124 tests** across `test_cdc.py`,
`test_agents.py`, `test_faction_protocol.py` and `test_protocol.py`. These tests
check current software behaviour; they do not replicate LLM findings or validate
the unimplemented architecture proposed in the paper.

A prospective context-form experiment should compare absent information, a
bounded structured block, and scattered annotations carrying equivalent facts.
Record prompt content, token counts, model/backend identity, measurement age and
uncertainty. Pre-specify decision correctness, invalid-response handling and
whether decision latency counts toward the deadline. Control role assignment
and presentation order; report token length and annotation density as potential
confounders. Do not treat the existing agent rate deviation as a calibrated
latency interval. Store each new run separately from the historical artifacts.

## Separate publication issue

The paper repository's `index.html` links to five `explainer-*.html` files that
are absent from the checked tree (which originally contained only `README.md`
and `index.html`). This review does not reconstruct missing explainers or claim
the published site has been repaired. Restore the intended pages or revise that
navigation as a separate content task.
