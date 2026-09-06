Time Is Not Metadata

Interpretable Timestamps and Measured Proper Time in LLM Agent Systems

H.G.O. Dillenberg
Hilden, Germany
Working draft, revision 2 -- August 2026

Implementation and reproducibility corrections -- 6 September 2026.
This update checks the implementation against LogpyClaw commit
`da935366521a00c25ae940ea0581ac9800064782`; it reports no new LLM trials.
See [implementation evidence and correction record](IMPLEMENTATION.md).
A separate [follow-up implementation](IMPLEMENTATION.md#follow-up-implementation-steps-13)
adds measured action latencies and optional planner context, with software and
local functional checks. It reports no new controlled LLM experiment and does
not alter the historical study endpoints.

Contact: dillenberg.net · LinkedIn · X

***
> Scope note. This paper assumes working knowledge of distributed systems
> (logical clocks, vector clocks, causal consistency) and LLM agent
> architectures. Section 1.1 states the core claim in non-technical terms;
> everything after it is technical.

***
Abstract

Distributed systems treat time as a coordination problem, solved through clock
synchronisation, logical timestamps, or consensus. Work on LLM-based agent
systems has largely inherited this framing. We argue the framing is incomplete
in a specific and consequential way, and we support the argument with
measurements from a running system.

Our claim has two parts. First, **timestamps are not neutral metadata for an
LLM agent**. Because a language model's context window makes no architectural
distinction between metadata and content, temporal annotations enter the
reasoning process as interpretable tokens. We report a field observation in
which growing memory caused agents to spend context budget reconstructing
timelines rather than executing tasks, and we argue this failure mode is
structurally unavailable to classical distributed systems, where timestamped
processes do not read their own timestamps. Second, **the absence of measured
temporal information leaves delegation near chance in the randomised-role
experiment reported here**. We define agent
proper time as a monotonic count of weighted internal operations, separate it
from an instantaneous pace estimate, and extend vector clocks with a parallel
dilation vector (the Causal-Dilation Clock).

We evaluate on LogpyClaw v3, a running multi-agent system, across 464 missions
and 1,719 inter-agent messages. A naive lifetime-average pace metric degenerates
in production, empirically motivating the proper-time/pace separation. Proper
times of coordinator and worker agents diverge by factors up to 6 within
identical wall-clock windows. In a pre-specified delegation experiment with
randomised role-to-backend binding (n = 200), agents given measured per-action
latencies of their peers chose the delegate favoured by the estimated-cost
oracle in 100 of 100 trials, against 55 of 100 without (risk difference 45 percentage points, 95 %
CI [35, 55]; Fisher exact two-sided p = 8.9 × 10⁻¹⁷). The control arm is
statistically indistinguishable from chance (p = 0.37 against 50 %). A prior
run without randomised roles failed to replicate; we report it in full, because
the contrast between the two isolates the operative condition: measured
temporal information changes decisions precisely when it is not inferable from
static framing.

Finally, we identify an unresolved tension between the two halves of the paper.
The instrumentation built in response to the second finding can produce the
kind of interpretable temporal content implicated in the first if it reaches
model context. This update does not establish an automatic production path from
CDC logs into retrieved memories. We state the problem, propose mitigations,
specify two experiments
that would discriminate between the available positions, and do not claim to
have solved it.

Keywords: LLM agents, multi-agent systems, distributed systems, logical
clocks, temporal reasoning, agent orchestration, delegation, observability

***
1. Introduction

> Field observation (AgentClaw, 2025). As agent memory grew to hundreds of
> entries, each carrying a timestamp, the agents began to falter. The models
> started actively reconstructing the timeline: *What is older? Does this still
> fit? Is this current?* That interpretive work consumed context budget and
> displaced the actual task. The system became unstable, not because the clocks
> were wrong, but because timestamps are not neutral metadata for an LLM. They
> are interpretable content.

This observation is the point of departure for the present paper, and it is
worth being precise about why it is surprising.

The conventional treatment of time in distributed systems is a coordination
problem. Lamport's logical clocks [1], Fidge's and Mattern's vector clocks
[2, 3], hybrid logical clocks [4], and Spanner's TrueTime [5] share an implicit
assumption: there exists an objective ordering, and the engineering task is to
approximate it consistently across nodes. The assumption is sound where it is
usually applied. A database node does not read its own Lamport counter and form
a belief about it. The timestamp is inert with respect to the computation it
annotates.

An LLM agent has no such separation. Anything present in the context window is
input to the forward pass. A timestamp attached to a retrieved memory is not
metadata riding alongside the computation; it is part of the computation. The
model will attend to it, weigh it, and reason about it, whether or not the
system designer intended that. This is not a bug in any particular
implementation. It is a structural property of systems whose processing
substrate is a language model over an undifferentiated token stream.

Section 3 develops the consequences of this observation, which we take to be the
paper's primary contribution.

A second, related problem concerns not the interpretation of temporal
information but its absence. Consider a heterogeneous orchestration
architecture: a coordinator dispatches sub-tasks to agents backed by different
models, running on different hardware, with different context utilisations and
different reasoning depths. One sub-agent completes in 200 ms; another, on a
nominally comparable task with extended reasoning, consumes 8 seconds; a third
is suspended on a remote tool call. From the coordinator's wall clock, the same
interval elapsed for all three. In terms of internal progress, it did not.

We use the term proper time for the per-agent measure of accumulated
internal progress, and we borrow the German Eigenzeit as a compact label. The
borrowing is terminological only. Section 4.1 states explicitly why we do not
treat this as an analogy to relativistic time dilation and what we lose by
declining that framing.

The question is not how to eliminate heterogeneity in progress rates. It cannot
be eliminated without sacrificing the heterogeneity that makes such systems
useful. The question is whether a system that measures this quantity behaves
differently from one that does not. Section 6 answers that question
empirically, and the answer is yes, under a condition we can state precisely.

1.1 The Core Claims in Non-Technical Terms

Hand the same assignment to three colleagues at 9:00 and ask for a report at
9:05. Anna sketches three options and refines one. Ben is still reading the
brief. Carla spent the interval on hold with a supplier. By the wall clock all
three had five minutes; the amount of work lived through inside each head
differs enormously. A manager who treats all three reports as equally
deliberated is making decisions on work that has aged unevenly.

That is claim two: systems should measure how much work each participant
actually lived through, and doing so demonstrably improves who gets assigned
what.

Claim one is stranger. Suppose each colleague also received a stack of notes,
every one stamped with a date, and suppose that reading and reconciling those
dates were itself part of the five minutes. The bookkeeping would eat the work.
For an LLM agent, this is not a metaphor. Reading the timestamps is not free,
and it is not optional.

1.2 Contributions

An argument that the metadata/content distinction, which classical temporal
   coordination presupposes, does not hold for LLM agents, together with the
   observed failure mode this produces (§3).
A formal definition of agent proper time, explicitly separated from
   instantaneous pace, and an extension of vector clocks that carries both (§4).
An implementation in a running multi-agent system, with an honest
   implementation-status accounting (§5).
An empirical evaluation including a degenerate-metric finding, a
   direct measurement of proper-time divergence, a negative result, a
   non-replication, and a decisive randomised replication at n = 200 (§6).
A statement of the unresolved tension between contributions 1 and 2, which
   we take to be the most important open problem the paper raises (§7).

***
2. Why Standard Synchronisation Is Necessary but Insufficient

The distributed systems toolkit for managing time is mature: NTP for wall-clock
synchronisation, Lamport's happened-before relation for logical ordering [1],
vector clocks for causality across concurrent processes [2, 3], hybrid logical
clocks where both physical and logical ordering are needed [4], consensus
protocols such as Paxos [6] and Raft [7] for agreement on event order, and
TrueTime for global linearizability under explicit uncertainty bounds [5].
These tools are battle-tested and remain necessary in agent systems.

They are insufficient here, for four reasons rooted in what they assume.

The events are not computationally cheap. Classical primitives timestamp
events whose duration is small relative to network latency: a write, a send, a
state transition. A single LLM agent invocation ranges from roughly 100 ms for
a small model answering a routed query to tens of seconds for a large model
performing multi-step reasoning with tool calls [8, 9]. Event duration is
dominated by internal processing, not by the network.

The events are not uniform across agents. In a heterogeneous system where
locally served models handle some dispatches and remotely served frontier
models handle others, the expected duration of a comparable task differs by an
order of magnitude depending on routing. This is a designed property of the
system, not noise to be filtered.

The events have internal structure. A vector clock establishes that agent
A's response causally preceded agent B's. It cannot express that A
traversed six reasoning steps while B traversed two. For coordination this is
irrelevant. For attribution, for debugging a hallucination, or for assessing
whether sufficient deliberation preceded an action, it is not.

The events are not idempotent across temporal contexts. An agent
recommending "send the email now" at 14:00 may recommend otherwise at 16:00 on
identical nominal input, because memory state has shifted or upstream agents
have produced new artifacts. Standard timestamping records when a
recommendation was issued, not the temporal context in which it made sense.

To these four we add the observation of §1, which is of a different kind. The
four above concern what classical primitives fail to capture. The fifth
concerns what they inadvertently cause.

***
3. Timestamps as Interpretable Content

3.1 The Structural Claim

Let a classical distributed process p carry logical clock value V(p). The
value participates in the protocol layer: it is compared, merged, and
transmitted. It does not participate in the application computation, because
the protocol layer and the application layer are separate address spaces. A
node computing balance -= amount does not consult its vector clock while
doing so.

An LLM agent has one address space. Its state is a token sequence, and
inference is a function over that entire sequence. If a retrieved memory
carries the string 2026-03-14T09:22:11Z, that string occupies context
positions and receives attention weight. There is no mechanism by which it can
be present and inert.

Consequently, temporal annotation in an LLM agent system has two effects
simultaneously: the intended coordination effect, and an unintended semantic
effect on the agent's reasoning. The second effect scales with the number of
annotations present in context, which in a memory-equipped agent scales with
uptime.

3.2 The Observed Failure Mode

The field observation quoted in §1 describes the resulting degradation. We
decompose it into three mechanisms, all of which are consistent with known
properties of transformer language models but which, to our knowledge, have not
been discussed in the context of temporal coordination.

Budget displacement. Timestamp reconciliation consumes context and
generated tokens that would otherwise serve the task. The cost is not constant;
it grows with the number of mutually inconsistent temporal references the
agent must resolve.

Positional sensitivity. Retrieval-augmented models attend unevenly to
context depending on position, with a well-documented degradation for material
in the middle of long contexts [10]. Temporal annotations are therefore weighted
by an accident of retrieval ordering rather than by relevance, and the agent's
resulting sense of recency is a function of where a memory landed in the prompt.

Interpretive drift. Absolute timestamps require an anchor to be meaningful.
An agent lacking a reliable representation of "now" will infer one, typically
from the most salient temporal reference available. Errors in that inference
propagate to every relative judgement the agent makes.

3.3 Why This Cannot Be Fixed by Better Clocks

The four insufficiencies in §2 are addressable in principle by richer
primitives, and §4 proposes one. The problem in this section is not of that
kind. Improving timestamp accuracy does not reduce interpretive load; a more
precise timestamp is the same number of tokens carrying the same invitation to
reason. Increasing timestamp density makes the problem worse. The only
mitigations available operate on what reaches the context, not on *what the
clock says*:

Out-of-band carriage. Temporal metadata travels with messages at the
   protocol layer but is stripped before context assembly. The orchestrator
   reasons about time; the agent does not see it.
Relativisation. Absolute timestamps are replaced at context-assembly
   time with pre-computed relative expressions ("14 minutes before this
   request"), which removes the anchoring inference.
Aggregation. Per-entry timestamps are replaced by a single ordering
   signal for a retrieved set, so that n memories contribute one temporal
   fact rather than n.

We have not evaluated these mitigations, and we regard their comparison as the
most valuable follow-up work this paper suggests. Section 7 explains why the
question is more urgent than it may appear.

***
4. Agent Proper Time and the Causal-Dilation Clock

4.1 A Note on Terminology and a Rejected Framing

An earlier revision of this work framed the phenomenon of §1 as an analogy to
relativistic time dilation, with agents occupying reference frames and a
Lorentz-like transformation between them. We have removed that framing, and the
reasons are worth recording, because the analogy is intuitively appealing and
we expect others to reach for it.

Special relativity rests on three features, none of which survives transfer to
this domain. There is no invariant speed: token throughput depends on hardware,
model size, batching, and prompt complexity, so nothing plays the role of c,
and the geometry that follows from c does not follow here. There is no light
cone: information propagates through tool calls, memory recalls, and dispatch
in patterns that permit apparent retro-temporal coupling, so causal structure
must be reconstructed logically rather than read off a geometry. And there is
no reciprocity: if a small local model completes a reasoning step in 200 ms
while a frontier model takes 8 seconds, the relation is asymmetric and ordered,
not the symmetric relation of relative motion.

An analogy that loses its invariant, its causal geometry, and its symmetry is
not an analogy. It is a vocabulary. We retain one word from that vocabulary,
Eigenzeit (proper time), because it names a quantity we define independently
below and because no compact English equivalent exists. Readers should
understand it as a label, not as a claim of structural correspondence. Nothing
in §§4 to 7 depends on relativity, and the empirical results of §6 are
independent of it.

4.2 Defining Proper Time

Let agent aᵢ be a stateful process producing reasoning outputs in response to
inputs. Define the proper time of aᵢ, written τᵢ, as a monotonic function
over agent-internal operations rather than over wall-clock time:

$$\tau_i(t) = \sum_{k=1}^{N_i(t)} w_k$$

where Nᵢ(t) is the number of internal operations aᵢ has completed by
wall-clock time t, and w<sub>k</sub> is the weight of operation k.
Internal operations include token generations, tool invocations, memory
lookups, and reasoning-step transitions. Weights may be uniform (w<sub>k</sub>
= 1, recovering an operation count) or cost-proportional.

Three properties hold by construction:

Monotonicity. For non-negative weights, τᵢ never decreases within an
   agent's accounting lifetime. It advances only when a counted operation
   occurs; a suspended tool call without further operations does not advance τᵢ.
Locality. τᵢ is meaningful within aᵢ's own accounting. Direct
   comparison of τᵢ against τⱼ requires a transformation (§4.3).
Wall-clock independence. Two agents may share an interval [t₀, t₁]
   and accumulate substantially different Δτ over it.

The definition deliberately leaves the operation set and the weighting to
implementation. In LogpyClaw v3, τ counts protocol-level operations (dispatch,
handle, delegation ticks); §6.2 discusses the granularity cost of this choice.
Some ticks occur on entry to an invocation rather than on completion. Thus the
implementation measures counted protocol events, not completed hidden reasoning
steps. Agent clock state starts afresh when its process is reconstructed; this
is not a claim of persistent monotonicity across restarts.

4.3 Separating Cumulative Progress from Instantaneous Pace

τ as defined is cumulative and therefore says nothing about current rate. An
agent idle for an hour has the same τ as it did an hour ago, which is correct,
but a system reasoning about delegation needs to know how fast the agent is
now.

We therefore define a second, independent quantity: the pace πᵢ, an
exponentially weighted moving average of operations per unit wall time over
recent operations. The two quantities have different merge semantics. τ merges
by component-wise maximum, as a vector clock does, because accumulated progress
is monotone and non-revisable. π merges by causal recency, because a stale rate
estimate is worse than no estimate.

Section 6.1 reports what happens when this separation is absent. The finding is
the most direct empirical support in the paper for a definitional choice, and it
was discovered by failure rather than by design.

For comparison across agents we require a transformation

$$\Phi_{i \to j}: \tau_i \mapsto \tau_j$$

which, unlike a Lorentz transformation, is not derivable from first principles.
It is a heuristic estimate from measured relative operation costs. A scalar
first approximation is Φ(τᵢ) ≈ γᵢⱼ · τᵢ, with γᵢⱼ the ratio of expected
per-operation costs. If aᵢ averages 50 ms per step and aⱼ averages 2000 ms,
then γᵢⱼ ≈ 0.025. The transformation is asymmetric in general
(γᵢⱼ ≠ 1/γⱼᵢ under non-uniform weights).

This proposed cost conversion must not be confused with the implementation's
faction `gamma`, which estimates the inverse kind of ratio: source pace divided
by target pace. The base CDC classifier compares each agent's own τ across
events and does not apply a cross-agent transformation. The implemented pace
uses intervals between protocol ticks, including idle gaps, rather than isolated
model execution time. It is consequently not interchangeable with the measured
per-action latency supplied in §6.6.

Section 6.5 shows that scalar γ is insufficient in one specific and practically
important respect: it discards dispersion, and dispersion is what determines
whether a decision near a deadline boundary survives execution.

4.4 The Causal-Dilation Clock

The implemented CDC is a triple (V, τ, π). Its wire fields are `vector`,
`tau`, and `dilation`, respectively. The legacy name `dilation` denotes pace,
not cumulative proper time. There is no separate `pace` field in the current
wire format. `wall_ts` is added at serialisation time and is not a stable
observation timestamp.

The implementation compares each agent's τ component across two events without
applying Φ. Its relation labels have the following exact behaviour, with τ
comparisons evaluated within the supplied tolerance:

| Vector relation | Proper-time relation | Result |
| --- | --- | --- |
| Strictly ordered in either direction | Consistent in that direction | ORDERED |
| Strictly ordered | Inconsistent in that direction | CAUSAL_DRIFT |
| Equal | Equal | ORDERED |
| Equal | Different | INCONSISTENT |
| Concurrent | Equal | ORDERED |
| Concurrent | Different | CONCURRENT_DRIFT |

In particular, ORDERED is a legacy classifier label and does not always assert
happened-before: concurrent vectors with equal τ receive that label too.
Different agent rates alone do not establish corruption, and these relations do
not validate how many hidden reasoning steps a model performed. Faction-level
reclassification uses a separate learned pace ratio (§4.3).

Section 6.3 reports the historical null result for drift in request/response
traffic. That result is an observation about the sampled topology and does not
validate the classifier under every concurrent workload.

4.5 Reference Implementation Contract

The source of truth is `backend/core/cdc.py` at the revision identified above:

- `tick(agent_id, op_weight)` advances V and τ. The agent wrapper normally
  uses a uniform weight of one.
- `tick_with_rate(agent_id, rate)` adds a unit tick and records a positive
  rate in `dilation`; the wrapper computes its EWMA outside the clock.
- `merge()` takes component-wise maxima of V and τ. For rates, the greater
  per-agent V determines recency, with maximum rate as the tie-breaker.
- `relate()` compares τ for the same agent keys. Its `gamma` argument is
  retained for compatibility and is not used.
- `rate_stats` and `time_sense()` live on the agent, outside the wire clock.

This contract replaces an earlier illustrative sketch whose `dilation` and
`pace` names did not match the implementation. Payload size grows with the
number of represented agents; three maps are not constant-size storage.

ML-DSA-65 signatures cover the message's stable `timestamp` and, within its
clock, `vector` and `dilation`. They deliberately exclude `tau` and the generated
`wall_ts` for legacy compatibility. A valid chain therefore does not directly
authenticate stored τ values. Extending that guarantee requires a versioned
signing format and explicit legacy verification.

***
5. Implementation

The framework is implemented in LogpyClaw v3, a CDC-native multi-agent
system by the same author, successor to the AgentClaw codebase in which the
original field observation was made. The CDC ships as a mandatory field on every
inter-agent message (backend/core/cdc.py); τ is tracked per agent alongside an
EWMA pace estimate; cross-faction drift is classified before logging. The
project declares Python 3.12 or newer and uses FastAPI, SQLModel, and SQLite
with sqlite-vec for semantic memory, serving locally hosted and remotely served
models through a unified dispatch layer.

Source: <https://github.com/Jeuners/logpyclaw>
Background: <https://www.dillenberg.net/agentclaw-lokales-multi-agent-ki-system/>

5.1 Implementation Status

The following status describes the checked source revision, not every earlier
AgentClaw deployment. Source paths and focused tests are listed in
[IMPLEMENTATION.md](IMPLEMENTATION.md).

| Component | Verified status |
| --- | --- |
| External A2A delegation | JSON gateway implemented; not XML tasklists. |
| Periodic work | Configured initiative loops, RSS jobs and daily dream-image generation implemented. |
| Nightly semantic-memory consolidation | Not found; the dream service generates images rather than rewriting recalled memories. |
| Agent-initiated dispatch | Implemented through the central Conductor; does not establish a decentralised multi-node mesh. |
| Per-message wall timestamp and CDC | Implemented in structured message records. |
| Proper time τ and EWMA pace | Implemented at protocol-tick granularity. |
| Drift classification | Implemented with the exact cases in §4.4. |
| Signed mission log | ML-DSA-65 hash chain implemented; τ is outside the signed clock fields. |
| `reference_now` / `parent_reference_now` on plan steps | Not found in the checked backend or tests. |
| Injected `TimeProvider` replacing direct clock access | Not found; direct `time.time()` calls remain. |
| Uniform dual-timestamp logging tuple | Not found in the text logger; structured message clocks are a different mechanism. |
| Re-synchronisation policy per action type | Not found as a general policy; faction bridges do not establish it. |
| Distributional pace summaries | Agent-local EWMA absolute deviation and dev/rate ratio implemented and tested. |
| General measured-latency input to the production planner | Not implemented; the experiment injects its own action-latency estimates. |
| Context-assembly mitigations (§3.3) | Explicit global temporal budgeting or filtering remains proposed. |

Frame inheritance via an injected TimeProvider is a design proposal, not a
verified property of this version. What does exist is causal-history propagation:
agent-initiated messages inherit a snapshot of the sender's CDC, which the
recipient merges. This is distinct from inheriting an absolute `reference_now`
for reproducible historical reads.

Distributional summaries also need a precise limit: the implemented `dev` is an
EWMA absolute deviation from the updated rate, not a standard deviation or a
latency quantile. It does not by itself deliver calibrated deadline probabilities.
The context-form comparison of §7 and a production routing connection remain
follow-up work.

5.2 Four Sources of Divergence

Heterogeneous model latency in delegation. Sub-agents run on different
backends: small local models for cheap dispatches, remotely served frontier
models for difficult reasoning. A delegating agent issuing parallel sub-tasks
receives responses on timescales differing by an order of magnitude. Within one
elapsed coordinator interval, the two sub-agents accumulate very different
progress. This is the prototypical case.

Asynchronous heartbeats decoupled from interactive time. Scheduled tasks
run at intervals from minutes to days. Four heartbeat cycles may elapse during
a single conversation; hundreds of conversational turns may elapse between two
firings. If results from both regimes are ingested into shared semantic memory, a
recalling agent needs provenance to distinguish them. That ingestion is not
established merely by the presence of scheduled jobs.

Consolidation operating on past memory (proposed scenario). A service that
re-reads, summarises and re-embeds old entries would modify records of the past
in the system's present. The recalling agent would need provenance of that
transformation. The checked version does not implement this nightly
consolidation path; it is a design case rather than observed current behaviour.

Peer dispatch across nodes (proposed extension). Nodes can differ in load and
operation cost even with network ordering in place. The current agent-initiated
primitive still passes through one Conductor and does not validate this
multi-node case.

***
6. Evaluation

All observational data come from the system's signed mission log: 464 missions
and 1,719 inter-agent messages at time of analysis, 72 % of them signed. Most of
this corpus is development and test traffic, and we treat it accordingly.
Experiment scripts and raw results are published alongside the implementation
(experiments/).

Statistical reporting. Two-group comparisons use Fisher's exact test,
two-sided. Proportions carry Wilson score intervals; differences carry Wald
intervals. Only the n = 200 experiment (§6.6) had its endpoint, arm sizes, and
oracle definition fixed before data collection. The pilot (§6.4) and the scaled
run (§6.5) were exploratory, and their subgroup analyses are hypothesis-
generating rather than confirmatory. We report all runs conducted, including
those that did not support the hypothesis. No correction for multiple
comparisons is applied, because we do not claim significance for any exploratory
comparison.

6.1 A Naive Rate Metric Degenerates in Production

The first implementation approximated each agent's pace as a lifetime average:
operations completed divided by uptime. Across 1,697 legacy messages the metric
collapsed. Median recorded rates fell to 0.001 to 0.003 operations per second
for every agent, with idle agents drifting asymptotically toward zero. Apparent
divergences of five orders of magnitude between agents proved to be artifacts of
the metric rather than properties of the system.

This is direct empirical support for the definitional separation in §4.3. A
single number conflating cumulative progress with current rate measures uptime,
not experience. The finding is worth stating plainly because it is the kind of
error that is invisible in a specification and obvious in a trace.

6.2 Proper-Time Divergence Is Real and Measurable

With τ and π separated, ordinary production missions exhibited the phenomenon
directly. Three orchestration missions routing work from a fast coordinator to a
slow worker:

Mission	Wall time	τ coordinator	τ worker	Ratio
mis_274e87fe	384.5 s	6.0	1.0	6.0×
mis_d18a03bc	144.1 s	10.0	3.0	3.3×
mis_4783a34e	600.0 s	4.0	2.0	2.0×
	Identical wall-clock windows; up to sixfold divergence in accumulated internal
progress.

Limitation. τ here counts protocol-level operations (dispatch, handle,
delegation ticks), not reasoning steps within a model invocation. The
granularity is coarser than §4.2 envisages. Without a calibrated mapping between
protocol events and internal work, these ratios are not proven lower bounds on
what a finer-grained instrument would report. Three
missions is also a small and non-random sample; we present this as a
demonstration of measurability, not as an estimate of typical divergence.

6.3 A Negative Result, and What It Measures

All 849 classifiable request/response pairs in the corpus fall into relation 1
(ORDERED). We observed no CAUSAL_DRIFT and no INCONSISTENT.

This is expected rather than disconfirming: sequential dispatch produces causal
order by construction. The interesting relations require genuinely parallel
branches, which the orchestrator only recently gained. The classifier has not
yet met the traffic it was built for, and we flag this as the primary gap
between implementation and validation.

Read as a diagnosis rather than a failure, the uniform result is itself a
measurement of the system's topology. LogpyClaw v3 currently operates as a
centrally orchestrated hub-and-spoke system: closer to an agent manager with an
unusually rich protocol than to an emergent multi-agent system. The sampled corpus does not demonstrate peer drift. The checked source now
includes `Conductor.initiate()` and configured initiative loops, so absence of
peer traffic in that historical corpus must not be read as absence of the
primitive in current code. Decentralised peer operation and a new concurrent
traffic evaluation remain open.

6.4 Pilot: Does Temporal Self-Knowledge Change Decisions?

To test whether measured temporal information changes decisions rather than
only logs, we built a real-time delegation scenario on the live system.

A slow agent (a local model, roughly 6 to 8 s per action) must prevent a
scripted failure arriving in T real seconds. It chooses between acting itself
(two of its own actions) or delegating to a fast agent (a remotely served model,
roughly 0.4 s per action; one action to call, one to execute, sometimes plus an
announced cooldown that makes delegation the slower option). The chosen option
is then actually executed against the wall clock. Success means finishing before
T. In the treatment arm the decision prompt contains measured per-action times
for both agents; the control arm receives an otherwise identical prompt. The
cooldown, when present, is stated in both arms, so only the rates are
exclusive to treatment.

Result (n = 20). Success 5/10 with temporal context against 3/10 without
(difference +20 points, 95 % CI [−22, +62]; Fisher p = 0.65). The committed `dragon3-results.json` summary reports oracle agreement on
winnable trials of 9/10 with information and 7/8 without. The earlier 7/7 versus
3/5 subgroup statement cannot be reproduced from that file and is withdrawn
pending a separately identified source. These oracle subgroups remain
exploratory.

The pilot is underpowered and we draw no inference from the success rates. One
methodological by-product is worth recording: the injected time sense was itself
miscalibrated by roughly ninefold, being a one-shot measurement taken with short
prompts against real action costs, and it helped anyway, because the decision
required only the ordinal fact that the fast agent is faster. A static
self-estimate drifting ninefold is precisely the failure mode §4.3 anticipates,
and it motivated continuous rate measurement.

6.5 Scaled Run: A Non-Replication, and Why It Is Informative

With rolling per-action medians and deadlines drawn from observed costs, we
scaled to n = 60. The success effect did not replicate: 18/30 with context
against 21/30 without (difference −10 points, 95 % CI [−34, +14]; Fisher
p = 0.59).

The decomposition explains why, and is more instructive than the pilot.

Trials without cooldown, where delegation is unambiguously optimal: both
  arms delegated in 33/33 trials. The ordinal fact was inferable from the
  scenario framing alone, so the treatment information was never exclusive and
  could not produce a difference.
Trials with cooldown, where the arithmetic reverses: the treatment arm
  switched correctly to self-action in 12/14 trials, the control arm in 8/13
  (difference +24 points, 95 % CI [−8, +56]; Fisher p = 0.21). Directionally
  consistent with the pilot, exactly where the information was exclusive, but
  not significant at this sample size and exploratory in any case.
Why success favoured the control arm: 9 deaths in the treatment arm
  followed an estimate-correct choice, against 5 in the control arm. The slow
  agent's latency is heavy-tailed, and deadlines drawn near the decision
  boundary turn a correctly chosen self-action into a coin flip on latency
  spikes. The arm that more often chose the expensive-but-correct option was
  punished more often by execution variance.

Three design lessons follow, each feeding back into the framework.

Exclusivity. A measured time sense can only demonstrate value where
   temporal facts are not inferable from static framing. Which agent is faster
   must be randomised, so that one memorised bit cannot substitute for
   measurement.
Endpoint choice. Decision correctness, not task success, is the endpoint
   a time sense controls. Outcome metrics are confounded by execution variance.
Point estimates are not a time sense. A median is not a sense of
   duration. The variance-driven failures show that useful temporal
   self-knowledge must carry dispersion, not only central tendency: an agent
   should know that it usually finishes in 12 seconds, and how wide "usually"
   is. This extends §4.3: the dilation component of the CDC should eventually
   carry distributional summaries of pace, not scalars.

6.6 Decisive Replication with Randomised Roles (n = 200)

The three lessons specify an experiment, which we pre-specified and ran.

Design. Agent identities are neutral, "Blue" and "Red". Each trial randomly
binds one name to a fast backend (roughly 0.5 s per action) and the other to a
slow one (roughly 3 to 15 s per action). Both actors receive identical action
prompts, so the latency gap is purely a property of the backend rather than of
the task. Which actor is faster therefore flips unpredictably between trials and
cannot be inferred from role priors. A commander must dispatch exactly one actor
against a deadline T. The treatment prompt states the measured per-action time
of each actor; the control prompt shows only the neutral names and is otherwise
identical. The deadline is the geometric mean of the two option costs, placing
it between estimated costs. This reduces proximity to the estimated boundaries
but does not guarantee immunity to execution variance. The primary endpoint is
agreement with a per-trial oracle: did the commander pick the actor with the
lower rolling-median estimated cost? Task success
is secondary. 100 trials per arm, strictly alternating; per-action times are
live rolling medians.

Results.

Arm	Decision correct	95 % CI	Task success	95 % CI
Measured temporal information	100 / 100 (100 %)	[96.3, 100]	95 / 100 (95 %)	[88.8, 97.8]
Control (neutral roles)	55 / 100 (55 %)	[45.2, 64.4]	57 / 100 (57 %)	[47.2, 66.3]
	Decision correctness: risk difference +45.0 points, 95 % CI [35.2, 54.8];
Fisher exact two-sided p = 8.9 × 10⁻¹⁷.
Task success: risk difference +38.0 points, 95 % CI [27.4, 48.6];
Fisher exact two-sided p = 1.3 × 10⁻¹⁰.

The control arm is not distinguishable from a no-information chooser: 55/100
against a chance baseline of 50 % gives an exact binomial p = 0.37, 95 % CI
[44.7, 65.0]. This is the intended behaviour of the design, and it confirms that
role randomisation removed the inferable signal that contaminated §6.5.

Task success was associated with oracle agreement but was not identical to it:
five treatment trials missed the execution deadline despite oracle-correct
choices. The buffered design did not eliminate latency variance.

Endpoint boundary. `survived` is defined by execution duration alone
(`exec_s <= deadline_s`); decision time and calibration are excluded. Recounting
rounded committed values using `decision_s + exec_s <= deadline_s` gives
93/100 versus 54/100. This is a descriptive post-hoc check, not a pre-specified
result or a new trial series; it still excludes calibration and preceding work.
The stored Fisher p-values are rounded to `0.0` in the JSON. The nonzero values
above are recoverable from the count tables and must not be interpreted as
mathematical zero.

The treatment's rolling medians are measured separately from CDC `dilation`.
The recorded `live_rates` snapshots are not used to construct those medians.
This experiment is not an ablation establishing that the CDC wire format or τ
caused the observed advantage.

Interpretation, stated narrowly. Where the ordinal fact "which peer is
faster" cannot be read off the framing, a continuously measured pace estimate is
the difference between perfect and chance-level delegation. The contrast with
the non-replicating run in §6.5 is itself the result: the effect appears exactly
when the temporal information is exclusive, and vanishes when it is not.

We are careful about what this does not show. It does not show that time,
specifically, is the operative variable. The commander's advantage is that it
received a measured quantity about its peers that it could not otherwise infer.
Latency is the quantity we measured, but cost per token, expected error rate, or
tool availability would plausibly produce the same structure of result. The
finding is best stated as: *measured, exclusive, peer-relative capability
information converts chance-level delegation into correct delegation.* Temporal
information is an instance of that class, and it is the instance a distributed
system is already positioned to collect.

6.7 Threats to Validity

Internal. Single machine and single operator throughout. Backend latency
distributions are real but were not held constant across sessions. The oracle in
§6.6 is computed from the same rolling medians supplied to the treatment arm,
which risks a shared-error dependency between treatment information and ground
truth; a fully independent oracle would require an execution-time measurement
not available at decision time. We regard this as the most serious internal
threat and note that the near-perfect treatment result would be inflated by it.

Construct. τ is measured at protocol granularity, not at the reasoning-step
granularity §4.2 defines. The delegation scenario is synthetic, though all
latencies are real. Task success in §6.4 and §6.5 is a confounded endpoint, as
§6.5 establishes.

External. The corpus is predominantly development traffic. The topology is
hub-and-spoke (§6.3), so results may not transfer to genuinely peer-to-peer
systems, which is precisely the regime the protocol was designed for. Two
backends and one task family.

Statistical. §6.4 and §6.5 are exploratory and underpowered; their subgroup
analyses should not be read as evidence. Only §6.6 supports confirmatory
reading, and only for its pre-specified primary endpoint.

***
7. The Instrumentation Paradox

The two halves of this paper are in tension, and we have not resolved it.

Section 3 argues that temporal annotations reaching an agent's context degrade
its reasoning, and that the degradation scales with annotation density. Sections
4 to 6 propose and validate an instrument that attaches temporal annotations to
every inter-agent message, and a logging discipline that records a proper-time
value alongside every event.

Whether richer instrumentation increases interpretive load depends on which
fields actually reach the model. The current code does not establish the earlier
claim that instrumented mission events are automatically recalled from the same
store and rewritten nightly. Mission storage and SQLite semantic memory are
separate components. The normal LLM path receives message content and persona;
Martin's planner inserts recalled text, without automatically appending each
record's time fields. Temporal references already embedded in text may still
reach context, and an A2A response can expose a CDC summary.

The paradox is therefore a testable design risk, not a demonstrated production
regression in this revision. The text logger does not implement the claimed
uniform dual-timestamp tuple (§5.1), and this review has not established a dated
transition in prompt annotation density. A retrospective study would first need
versioned deployments and captured prompts showing the proposed exposure
change. The existence of CDC fields in a signed mission log is insufficient.

If that exposure history becomes available, compare retrieval-heavy tasks for
context use, time to first task-relevant output and recency-judgement variance,
while accounting for simultaneous model, retrieval and prompt changes. Such a
comparison would remain observational. Without it, the prospective controlled
context-form experiment below is the clearer next step.

The open question is precise: can measured temporal information improve
coordination while its form and amount in context keep interpretive costs
bounded?

Three positions are available, and they are empirically distinguishable.

Strict out-of-band. CDC fields are protocol-layer only, stripped at
   context assembly by construction rather than by convention. This preserves
   both results but forecloses giving agents the temporal self-knowledge that
   §6.6 shows to be valuable, which appears to give up the paper's strongest
   finding.
Structured differs from scattered. A single, well-formed, positionally
   fixed temporal statement ("your peer averages 0.5 s per action; you average
   6 s") may impose a bounded interpretive cost, unlike n scattered absolute
   timestamps whose cost grows with n. Note that the §6.6 treatment prompt is
   exactly this: one structured, relativised, aggregated temporal fact. If this
   position is correct, the n = 200 result is not merely compatible with §3, it
   is weak evidence for the mitigations of §3.3.
Budgeted temporal content. Temporal information reaching context is
   explicitly budgeted and prioritised, as a scarce resource, rather than
   emitted wherever a timestamp happens to exist.

Position 2 is our working hypothesis, and the reader should note that we arrived
at it after the fact, having designed the §6.6 prompt for clarity rather than
for this argument. It is a post-hoc reading of a favourable coincidence and must
be tested directly: the same delegation experiment, with temporal information
delivered in structured form against scattered raw timestamps of equivalent
content, would discriminate between positions 1 and 2 in a single run. We regard
this as the most important experiment this paper does not contain.

***
8. Implications

Orchestration. Routing decisions currently made on static model profiles
should be made on measured, continuously updated pace estimates. Section 6.6
quantifies the gap between the two regimes for one decision class, and the gap
is the whole distance between chance and correctness.

Logging and forensics. Dual annotation, wall clock and proper time, makes
drift a first-class signal and permits per-agent timeline reconstruction after
the fact. Section 7 constrains where those annotations may subsequently travel.

Reproducibility. A plan step carrying the temporal context in which it was
planned can be replayed against historical state. Without it, replays execute
against present-frame clock reads and are subtly wrong.

Trust calibration. For a user consuming an agent's recommendation, temporal
context is part of provenance. A recommendation issued from a frame that has
drifted substantially from the user's present deserves more scrutiny than one
issued from a freshly synchronised frame. The CDC makes that distinction
available to the interface layer, which is where it needs to be.

Action gating. For actions with irreversible external effects, drift above a
threshold should trigger re-execution with refreshed context rather than
log-only acceptance. This is a policy choice per action type, and it should be
explicit in the action's metadata rather than implicit in the orchestrator.

***
9. Conclusion

We have argued that time in LLM agent systems fails classical treatment in two
independent ways. It is interpreted where classical systems leave it inert,
which degrades reasoning in proportion to how much of it reaches the context.
And it is unmeasured where it would be decision-relevant, which degrades
delegation to chance when the relevant facts are not otherwise inferable.

The second claim is the one we can currently support with strong evidence: under
randomised roles at n = 200, measured peer latency converted 55 % delegation
accuracy into 100 %. We have been careful to state that result narrowly, since
the operative property is measured exclusive peer information, of which latency
is one instance.

The first claim is, we believe, the more consequential, and it currently rests
on a field observation and a structural argument rather than on a controlled
experiment. That asymmetry is the honest summary of this paper's state. Two
routes could test it. A retrospective study first requires verified prompt
exposure history; the current signed log alone does not establish that history.
A prospective study would repeat the delegation experiment while holding
informational content constant and varying its form in context, structured
against scattered, with explicit prompt and timing measurements.

A final note on scope. The system studied here is hub-and-spoke, and its
historical corpus does not validate peer traffic, although the checked code now
contains centrally dispatched agent-initiated missions (§6.3). The
instrument therefore currently exceeds the system it measures. We consider that
the correct order in which to build the two, but it does mean the framework's
more interesting predictions remain untested.

***
Data and Code Availability

Implementation, experiment scripts, and raw results:
<https://github.com/Jeuners/logpyclaw> (experiments/). Mission-log records are
signed with an ML-DSA-65 hash chain; verification tooling is included in the
repository. The [implementation evidence](IMPLEMENTATION.md) pins source and
raw-data paths. Local databases, deployment histories and prompt captures used
for the historical observational corpus are not included in this paper
repository; their contents were not independently reconstructed in this update.

Competing Interests

The author is the developer of the system under evaluation. All experiments were
designed, executed, and analysed by the author. No independent replication has
been performed. Readers should weight the results accordingly.

***
References

Lamport, L. (1978). Time, clocks, and the ordering of events in a distributed
   system. Communications of the ACM, 21(7), 558-565.
Fidge, C. J. (1988). Timestamps in message-passing systems that preserve the
   partial ordering. *Proceedings of the 11th Australian Computer Science
   Conference*, 56-66.
Mattern, F. (1989). Virtual time and global states of distributed systems.
   Parallel and Distributed Algorithms, 215-226.
Kulkarni, S. S., Demirbas, M., Madappa, D., Avva, B., & Leone, M. (2014).
   Logical physical clocks. Principles of Distributed Systems (OPODIS 2014),
   17-32.
Corbett, J. C., et al. (2013). Spanner: Google's globally distributed
   database. ACM Transactions on Computer Systems, 31(3), 1-22.
Lamport, L. (1998). The part-time parliament. *ACM Transactions on Computer
   Systems*, 16(2), 133-169.
Ongaro, D., & Ousterhout, J. (2014). In search of an understandable consensus
   algorithm. USENIX Annual Technical Conference, 305-319.
Yao, S., Zhao, J., Yu, D., Du, N., Shafran, I., Narasimhan, K., & Cao, Y.
   (2023). ReAct: Synergizing reasoning and acting in language models.
   International Conference on Learning Representations (ICLR 2023).
Wu, Q., Bansal, G., Zhang, J., Wu, Y., Zhang, S., Zhu, E., Li, B., Jiang, L.,
   Zhang, X., & Wang, C. (2023). AutoGen: Enabling next-gen LLM applications via
   multi-agent conversation. arXiv:2308.08155.
Liu, N. F., Lin, K., Hewitt, J., Paranjape, A., Bevilacqua, M., Petroni, F.,
    & Liang, P. (2024). Lost in the middle: How language models use long
    contexts. Transactions of the Association for Computational Linguistics,
    12, 157-173.
Park, J. S., O'Brien, J. C., Cai, C. J., Morris, M. R., Liang, P., &
    Bernstein, M. S. (2023). Generative agents: Interactive simulacra of human
    behavior. UIST 2023.
Husserl, E. (1928). Zur Phänomenologie des inneren Zeitbewusstseins.
    Niemeyer, Halle.
