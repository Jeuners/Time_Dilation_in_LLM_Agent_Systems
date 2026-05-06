> LLM agents don't just have unsynchronized clocks.
> They experience different amounts of time.
>
> This paper proposes a framework for that gap — built on *agent proper time* (Eigenzeit)
> and a *Causal-Dilation Clock* extending classical vector clocks.

# Time Dilation in LLM Agent Systems

## Toward a Framework for Temporal Coherence

**H.G.O. Dillenberg**
Bridging IT, AI & Humanity e.V.
Hilden, Germany
*Working draft, 06 May 2026*

*Status: §1–§4 complete. §5 (Implications) and §6 (Conclusion) outstanding.*

**Social:** [dillenberg.net](https://dillenberg.net) · [LinkedIn](https://www.linkedin.com/in/hgod/) · [X](https://x.com/Jeuner) · [YouTube](https://www.youtube.com/)

---

## Abstract

Distributed systems literature treats time as a coordination problem to be solved through clock synchronization, logical timestamps, or consensus protocols. The literature on autonomous AI agents has inherited this framing largely without examination. This paper argues that the framing is incomplete. In multi-agent systems built on Large Language Models, time is not merely unsynchronized — it is **dilated**. Different agents experience different rates of subjective progress depending on compute budget, reasoning depth, context-window state, and orchestration position. We propose treating this phenomenon as a *productive analogy* to relativistic time dilation — explicitly not as physical isomorphism — in which each agent has a proper time (Eigenzeit), and the system's correctness depends on how these proper times relate, not on enforcing a single coordinate time. We define agent proper time formally, propose a heuristic transformation between agent reference frames, and sketch a Causal-Dilation Clock that extends standard vector clocks with per-frame dilation tracking. We illustrate the framework using AgentClaw, a running multi-agent orchestration system developed by the author, and clearly demarcate which components are implemented from which remain conceptual. We outline implications for orchestration design, logging, debugging, and the trust users place in agent decisions whose temporal context they cannot directly observe.

**Keywords:** AI agents, multi-agent systems, distributed computing, temporal logic, Eigenzeit, agent orchestration, AgentClaw, MARTIN

---

## 1. Introduction

The conventional view treats time in distributed systems as a coordination challenge. Lamport's logical clocks [^1], Mattern's vector clocks [^2], and Spanner's TrueTime [^3] all approach the problem with the same implicit assumption: there is a real, objective time, and the engineering challenge is to approximate it consistently across participating nodes. The assumption is sound for distributed databases, where nodes are computationally homogeneous and clock divergence is bounded by network latency and physical drift.

The assumption breaks for autonomous LLM-based agents.

Consider the AgentClaw architecture [^4]: a coordinator agent — MARTIN (Machine Assisted Reasoning + Tactical Intelligence Network) — spawns sub-agents to handle parallel tasks via the A2A delegation protocol. These sub-agents do not merely run on different machines with slightly drifted wall clocks. They run with different model sizes (`gemma4:e4b` vs. larger OpenRouter-served frontier models), different context-window utilizations, different prompt complexities, and different reasoning chains. One sub-agent may complete its task in 200 milliseconds. Another, processing a similar nominal task with extended chain-of-thought reasoning, may consume 8 seconds. A third may be paused mid-execution, waiting on a tool call to a remote service.

From the orchestrator's perspective, *one minute has passed*. From the perspective of the three sub-agents, vastly different amounts of *subjective progress* have occurred. The fast agent has effectively lived several task-cycles while the slow one is still in its first reasoning step. Their proper times have diverged.

This is not a bug to be patched with better clock synchronization. It is a structural property of any orchestration architecture that allows heterogeneous agents to operate semi-autonomously over heterogeneous compute. In AgentClaw, this is compounded by the presence of asynchronous Heartbeat and Dream-Cycle services [^4]: scheduled background tasks that operate on entirely different timescales than interactive chat — minutes to days for heartbeats, nightly for memory consolidation. Each of these services exists in a temporal frame disconnected from the foreground agent's frame, and from each other.

The question, then, is not how to eliminate dilation. It cannot be eliminated without sacrificing the heterogeneity that makes such systems useful. The question is how to design systems that remain *coherent* in its presence.

This paper develops the analogy in five steps. Section 2 establishes why standard synchronization techniques from distributed systems fall short for LLM agents. Section 3 develops a conceptual framework for agent proper time, drawing on phenomenological accounts of internal time-consciousness as well as the relativistic notion of Eigenzeit, and proposes a transformation between agent reference frames. Section 4 grounds the framework in the AgentClaw case study, identifying four distinct sources of dilation in a running system and showing where dilation matters and where it can be safely ignored. Section 5 discusses implications for logging, debugging, and the user's trust in temporally opaque agent decisions. Section 6 concludes with a brief positioning of the work within a broader programme of *Sovereign Temporal Continuity* — the proposition that autonomous systems must be designed to remain coherent across time in a way that survives their original architects.

---

## 2. Why Standard Synchronization Fails for LLM Agents

The distributed systems community has spent five decades developing tools to manage time across nodes. The toolkit is mature: NTP for wall-clock synchronization, Lamport's happened-before relation for logical ordering, vector clocks for causality across concurrent processes, hybrid logical clocks (HLCs) for systems that need both physical and logical ordering, and protocols such as Paxos and Raft for consensus on event ordering. Spanner achieves global linearizability through TrueTime, exposing uncertainty bounds rather than a single timestamp [^3]. These tools are well-understood and battle-tested in production systems handling billions of transactions per day.

None of them address the problem this paper is concerned with.

To see why, observe what these tools assume. They assume that the events being timestamped are computationally cheap and uniform: a database write, a message send, a state transition. The duration of the event itself is small relative to network latency, and the duration is roughly the same across nodes. Vector clocks order events; they do not measure the *internal* time of events. Lamport's happened-before relation captures that event $A$ causally preceded event $B$, not that the participant performing $A$ experienced more or less subjective time during $A$ than during a comparable event $A'$ on a peer node.

LLM agents violate every one of these assumptions.

First, **the events are not cheap**. A single agent invocation may consume from 100 milliseconds (a small model answering a routed query without tool use) to 30 seconds or more (a large model performing chain-of-thought reasoning with multiple tool calls). The duration of the event is no longer dominated by network latency; it is dominated by the agent's own internal processing.

Second, **the events are not uniform across agents**. In a heterogeneous system such as AgentClaw, where Ollama-served local models handle some tasks and OpenRouter-served frontier models handle others, the *expected* duration of a comparable task can differ by an order of magnitude depending on which agent receives the dispatch. This is a property of the system, not noise to be filtered.

Third, **the events have internal structure**. A vector clock can tell you that agent $A$'s response causally preceded agent $B$'s response. It cannot tell you that during the production of that response, agent $A$ went through six reasoning steps while agent $B$ went through two. From a coordination standpoint this may seem irrelevant. From a *correctness* standpoint — debugging a hallucination, attributing a decision to a specific reasoning path, evaluating whether sufficient deliberation occurred before an action — it is essential.

Fourth, **the events are not always idempotent across temporal contexts**. An agent issuing the recommendation "send the email now" at 14:00 may issue a different recommendation at 16:00 given the same nominal input, because its memory state has shifted, its context has expanded, or upstream agents have produced new artifacts in the interval. Standard timestamping captures *when* the recommendation was issued; it does not capture the temporal context in which the recommendation made sense.

The cumulative effect is that distributed systems primitives are necessary but insufficient for reasoning about time in LLM-agent systems. Vector clocks remain useful for establishing causality between agent actions. Heartbeats and timeouts remain useful for detecting hung agents. But none of these primitives capture the phenomenon at the centre of this paper: that two agents, both functioning correctly, can experience radically different amounts of subjective progress within the same wall-clock interval, and that this divergence has consequences for system behaviour that cannot be addressed by tightening clock synchronization. What is needed is a vocabulary that takes the divergence seriously, and a formalism that makes it tractable. The next section develops both, while making explicit where the proposed analogy holds and where it should not be pressed.

---

## 3. A Framework for Agent Proper Time

We adopt the language of dilation as a productive analogy, not as a claim of physical isomorphism. Before introducing definitions, we name the bruchstellen — the points where the analogy to special relativity breaks — so that subsequent formalism is not mistaken for a stronger claim than it makes.

### 3.1 Where the Analogy Breaks

Three differences from special relativity are essential to acknowledge.

**No Lorentz invariance, no universal speed limit.** Special relativity rests on the postulate of a finite invariant speed $c$. The mathematics of dilation, the structure of the Lorentz transformation, and the geometry of spacetime all flow from this constant. There is no comparable invariant in LLM agent systems. The "speed" at which an agent processes tokens depends on hardware, model size, batching, prompt complexity, and the presence or absence of tool calls — none of which are universal. Compute latency is contextual, not constitutive. The analogy borrows the *idea* of frame-dependent time, not its underlying geometry.

**No light cone, no clean causal topology.** In special relativity, the causal structure of spacetime is governed by the light cone: information cannot propagate faster than $c$, and this constrains which events can causally influence which others. LLM agent systems have no equivalent. Information flows through tool calls, memory recalls, and inter-agent dispatch in ways that can produce apparent retro-temporal coupling — for instance, when an agent recalls a memory written by a peer in what is, from the recalling agent's frame, the relative past, but which was authored in a different frame's relative future. Causality in such systems requires logical reconstruction, not geometric reading.

**No reciprocity.** A central feature of special relativity is symmetric time dilation: from $A$'s frame, $B$'s clock runs slow; from $B$'s frame, $A$'s clock runs slow. The relation is reciprocal. The agent-time analogue is not. If a small local model finishes its reasoning step in 200 ms while a frontier model takes 8 seconds on a comparable task, the relation is asymmetric: the frontier model is "slower" in any meaningful sense, and the small model is not slower from the frontier model's frame. The dilation we describe is *anisotropic* and ordered; it is not a symmetric property of relative motion.

The analogy is therefore lexical and structural, not formal. We use it because it makes visible a phenomenon that the standard distributed-systems vocabulary obscures, and because it suggests forms of formalism — proper time, frame transformation — that turn out to be useful even when stripped of their relativistic underpinnings.

### 3.2 Defining Agent Proper Time

Let an agent $a_i$ be a stateful process capable of producing reasoning outputs in response to inputs. We define the **proper time** of $a_i$, written $\tau_i$, as a monotonic function over agent-internal reasoning operations rather than over wall-clock time:

$$\tau_i(t) = \sum_{k=1}^{N(t)} w_k$$

where $N(t)$ is the number of internal operations the agent has completed by wall-clock time $t$, and $w_k$ is the *weight* of operation $k$ in the agent's reasoning. Internal operations include token generations, tool invocations, memory lookups, and reasoning-step transitions. Weights may be uniform ($w_k = 1$ for all $k$, recovering an operation count) or non-uniform (giving more weight to expensive operations such as long tool calls).

Three properties of $\tau_i$ matter:

1. **Monotonicity.** $\tau_i$ never decreases. An agent's proper time always advances forward, even when wall-clock time stalls (during a paused tool call, for instance).
2. **Frame-locality.** $\tau_i$ is meaningful only from within $a_i$'s frame. Comparing $\tau_i$ values to $\tau_j$ values directly is a category error; comparison requires a transformation (§3.3).
3. **Independence from wall-clock time.** Two agents may share the same wall-clock interval $[t_0, t_1]$ and yet have radically different $\Delta\tau_i$ and $\Delta\tau_j$ over that interval.

This definition deliberately avoids tying proper time to any particular clock or any particular operation. It is a slot in the formalism, to be filled by implementation choice: in AgentClaw, $\tau$ for an agent is currently approximated by the count of completed reasoning steps plus tool calls, weighted by an estimated cost factor per operation type.

### 3.3 Frame Transformation

To reason about events across agent frames, we require a transformation function

$$\Phi_{i \to j}: \tau_i \mapsto \tau_j$$

that maps a proper-time value in agent $a_i$'s frame to its corresponding value in $a_j$'s frame. Unlike the Lorentz transformation, $\Phi_{i \to j}$ is **not** derivable from first principles; it is a heuristic estimate based on the relative computational profiles of $a_i$ and $a_j$.

A simple first approximation is a scalar dilation factor:

$$\Phi_{i \to j}(\tau_i) \approx \gamma_{ij} \cdot \tau_i$$

where $\gamma_{ij}$ is the ratio of expected per-operation costs between the two agents. If $a_i$ is a small local model averaging 50 ms per reasoning step, and $a_j$ is a frontier model averaging 2000 ms per reasoning step, then $\gamma_{ij} \approx 0.025$: one unit of $\tau_i$ corresponds to roughly $0.025$ units of $\tau_j$. The transformation is asymmetric ($\gamma_{ij} \neq 1/\gamma_{ji}$ in general, given different operation weights), confirming the absence of reciprocity noted in §3.1.

More sophisticated transformations would account for operation-type heterogeneity (a tool call in $a_i$ does not map cleanly onto a tool call in $a_j$), context-window state, and historical drift between the frames. We leave such refinements as future work; the scalar form is sufficient for the purpose of this paper.

### 3.4 The Causal-Dilation Clock

Standard vector clocks track causality across distributed processes. An event in process $i$ at vector-clock value $V_i = (v_1, v_2, \ldots, v_n)$ "happened before" an event in process $j$ at $V_j$ if $V_i \leq V_j$ component-wise and $V_i \neq V_j$. This captures *order* but not *experience*: it tells us $A$ preceded $B$, but not how much subjective progress $A$ accumulated relative to $B$.

We propose extending the vector clock with a parallel **dilation vector** $D = (\tau_1, \tau_2, \ldots, \tau_n)$ tracking the proper time of each agent. The combined construct is a pair $(V, D)$ — a **Causal-Dilation Clock** — that captures both ordering and frame-relative experience.

Two events $e_i$ and $e_j$ with clocks $(V_i, D_i)$ and $(V_j, D_j)$ stand in one of four relations:

1. **Causally and temporally ordered:** $V_i \leq V_j$, and $D_i \leq D_j$ in the relevant components after frame transformation. The classical happened-before case.
2. **Causally ordered, temporally divergent:** $V_i \leq V_j$, but $\Phi_{i \to j}(D_i) \not\leq D_j$. Event $e_i$ caused $e_j$ in the orchestration sense, but the agents' proper times have drifted such that $e_j$'s frame has accumulated less subjective progress than expected.
3. **Concurrent in vector clock, divergent in dilation:** $V_i \parallel V_j$ (concurrent), but $D$ values differ substantially. Two agents have done genuinely different amounts of work despite no causal dependency.
4. **Inconsistent:** Vector clock and dilation vector disagree on order in a way suggesting clock corruption or a missed update.

The fourth relation is the practically important one: in a system instrumented with both $V$ and $D$, it becomes detectable when an agent's reported events are temporally implausible — for instance, an agent claiming to have completed three reasoning steps in the same interval during which a peer of comparable capability completed thirty.

### 3.5 Pseudocode Sketch

A minimal extension to a vector-clock-based dispatch protocol:

```python
@dataclass
class CausalDilationClock:
    vector: dict[AgentId, int]      # standard vector clock
    dilation: dict[AgentId, float]  # per-agent proper time

    def tick(self, agent_id: AgentId, op_weight: float = 1.0):
        """Called by agent on every internal reasoning operation."""
        self.vector[agent_id] = self.vector.get(agent_id, 0) + 1
        self.dilation[agent_id] = self.dilation.get(agent_id, 0.0) + op_weight

    def merge(self, other: "CausalDilationClock"):
        """Called on receipt of a message from another agent."""
        for a, v in other.vector.items():
            self.vector[a] = max(self.vector.get(a, 0), v)
        for a, d in other.dilation.items():
            # dilation values do not max-merge; they are frame-local.
            # we keep both views, transformed when compared.
            self.dilation[a] = max(self.dilation.get(a, 0.0), d)

    def transform(self, source: AgentId, target: AgentId,
                  gamma: dict[tuple[AgentId, AgentId], float]) -> float:
        """Heuristic mapping of source's proper time into target's frame."""
        return self.dilation[source] * gamma.get((source, target), 1.0)
```

The implementation cost is modest: a few additional fields per dispatched message and per agent state. The benefit, as developed in §4 and §5, is a system that can detect, log, and reason about temporal divergences that are otherwise invisible.

---

## 4. Case Study: Temporal Dilation in AgentClaw

We now ground the framework in AgentClaw, a running multi-agent orchestration system [^4] developed by the author. AgentClaw is built on Python 3.14, FastAPI, NiceGUI 3.10, SQLModel, Qdrant for vector memory, and serves Ollama-local and OpenRouter-remote LLMs through a unified dispatch layer. It currently comprises 21 FastAPI routers, 13 UI pages, 23 skills, and over eight background services. The system was not designed with time dilation in mind; the framework presented here emerged from observing operational problems and asking what would have prevented them.

### 4.1 Implementation Status

Throughout this section, we mark each component as either ✓ implemented and operational, or ⚠ conceptual and not yet realised. This separation matters for honest assessment: AgentClaw demonstrates that the *substrate* for time-dilation reasoning exists, not that the framework is fully realised.

| Component                                  | Status        |
|--------------------------------------------|---------------|
| A2A delegation protocol (XML tasklists)    | ✓ Implemented |
| Heartbeat service (minutes to days)        | ✓ Implemented |
| Dream-Cycle nightly memory consolidation   | ✓ Implemented |
| M2M peer dispatch (MARTIN network)         | ✓ Implemented |
| Per-agent SQLite history with timestamps   | ✓ Implemented |
| Wall-clock-only logging                    | ✓ Implemented |
| Explicit `reference_now` per `PlanStep`    | ⚠ Conceptual  |
| `TimeProvider` injection across agents     | ⚠ Conceptual  |
| Causal-Dilation Clock per dispatch         | ⚠ Conceptual  |
| Drift detection and re-sync policy         | ⚠ Conceptual  |
| Eigenzeit-aware logging tuple              | ⚠ Conceptual  |

The conceptual components are the subject of an ongoing refactor informed by the present analysis.

### 4.2 Four Sources of Dilation

We identify four structurally distinct sources of temporal dilation in AgentClaw, each producing a different class of coherence problem.

**Source 1: Heterogeneous model latency in A2A dispatch.** ✓ The A2A protocol allows an agent to delegate sub-tasks to other agents via XML tasklists, with `@Mention` syntax in chat or programmatic dispatch. Sub-agents run on different models — `gemma4:e4b` locally for cheap tasks, OpenRouter-served frontier models for difficult reasoning. A delegating agent that issues parallel sub-tasks to two such sub-agents will receive responses on radically different timescales: the local model in roughly 200 ms per reasoning step, the frontier model in 1–8 seconds per step. From the orchestrator's wall-clock frame, *the same elapsed interval* contains very different amounts of subjective progress in the two sub-agents. This is the prototypical case of asymmetric dilation introduced in §3.

**Source 2: Asynchronous Heartbeats decoupled from interactive time.** ✓ AgentClaw includes a Heartbeat service that runs scheduled tasks at intervals from minutes to days. A heartbeat firing every six hours has no meaningful relation to the chat-foreground frame. From the perspective of a user interacting with an agent in chat, four heartbeat cycles may pass during a single conversation; from the heartbeat's perspective, hundreds of chat turns may pass between two of its firings. The two frames coexist but their proper times advance on entirely different scales. Without explicit frame tracking, events from these two domains commingle in shared memory (Qdrant), and an agent recalling a memory cannot tell whether it was authored in its own conversational frame or written by a heartbeat hours earlier.

**Source 3: Dream-Cycle consolidation operating on past memory.** ✓ The Dream-Cycle is a nightly background service that re-organises and consolidates memory accumulated during the day. It re-reads, summarises, and re-embeds memory entries — that is, it modifies, in the system's present, the records of the system's past. From the perspective of an agent that recalls one of these consolidated memories during the next day, the memory has *changed* since it was last read, even though the original event has not. This is a form of retro-temporal modification that has no analogue in standard distributed systems. It cannot be modelled by versioning alone; it requires recognising that the Dream-Cycle operates in a *frame* whose proper time runs backwards relative to the foreground frame's notion of memory permanence.

**Source 4: M2M peer dispatch across MARTIN nodes.** ✓ MARTIN is the peer-to-peer layer of AgentClaw, allowing nodes on different machines to dispatch tasks to each other. Each MARTIN node has its own clock, its own load, its own dilation profile. A task dispatched to a remote node may complete with substantial proper-time divergence relative to the dispatching node, compounded by network latency. This is the case where the standard distributed-systems toolkit (vector clocks, NTP) is most clearly *necessary but insufficient*: it captures the network-level ordering, but says nothing about the proper-time divergence between heterogeneous MARTIN nodes.

### 4.3 Operationalisation

The conceptual extensions outlined in §3 map to AgentClaw as follows.

**Extending `PlanStep`.** ⚠ The current `PlanStep` representation in A2A tasklists carries an implicit creation timestamp. We propose extending it explicitly:

```python
@dataclass
class PlanStep:
    id: str
    created_at: datetime              # wall-clock at creation
    reference_now: datetime           # planning agent's Eigenzeit at creation
    parent_reference_now: datetime | None  # inherited from parent at spawn
    deadline: datetime | None         # absolute, not relative
    action: dict
```

The `parent_reference_now` field is the explicit operationalisation of frame inheritance: when a sub-agent is spawned, it does not start with a fresh `datetime.now()`; it starts in the temporal context of its parent, with its own proper time advancing from there.

**TimeProvider injection.** ⚠ Each agent receives a `TimeProvider` at spawn rather than calling `datetime.now()` directly. A `TimeProvider` exposes:

- `now()`: the agent's reference time (its Eigenzeit-now)
- `wall_now()`: the actual system clock (used only for logging and re-sync)
- `dilation()`: an estimate of the agent's dilation factor relative to the orchestrator
- `fork(new_context)`: produces a child `TimeProvider` for spawning a sub-agent

The discipline that follows is simple but strict: agent code must not call `datetime.now()` directly. All temporal access goes through the injected provider. This makes frame-aware behaviour the default, and frame-blind behaviour an explicit (and reviewable) deviation.

**Logging.** ⚠ Every event logged includes both `wall_clock` and `agent_reference_now`, plus the agent identifier and dilation context:

```
(wall_clock, agent_reference_now, agent_id, dilation_context, event_type, payload)
```

When the two timestamps diverge, drift is visible. This makes possible drift visualisation per agent ("timeline per agent" plots), forensic analysis of race conditions, and detection of the inconsistent fourth case identified in §3.4. Integration with logpy.com — a logging service authored by the same group, designed for autonomous agent observability — is the planned implementation path.

**Re-synchronisation policy.** ⚠ When a sub-agent's response arrives at its parent with substantial drift, the parent has three options: *recalibrate* (adopt the child's reference_now), *reject* (demand re-execution with updated context), or *log only* (accept with logging). For AgentClaw, the proposed default is log-only; for actions with external side effects (sending email, financial transactions, public posts), the proposed default is reject-on-drift-above-threshold. The choice is per-action-type and is explicit in the action's metadata.

### 4.4 What This Buys Us

Three concrete capabilities follow from operationalising the framework.

First, **reproducibility**. A plan with explicit `reference_now` and `parent_reference_now` can be re-played against historical state, because the temporal context is preserved alongside the action. Without these fields, replays are subtly wrong: they execute against present-frame `datetime.now()` rather than the frame in which the original decision was made.

Second, **observability of drift**. The logging tuple makes drift a first-class signal. An operator can ask: "which agents are running in proper-time frames substantially divergent from the orchestrator?" — and get an answer. Currently in AgentClaw, this question cannot be asked, because the data needed to answer it is not recorded.

Third, **trust calibration**. For users interacting with agents whose decisions depend on context, the temporal context is part of the provenance. An action recommended by an agent whose proper time has drifted substantially from the user's current frame deserves more scrutiny than one issued in a freshly-synchronised frame. The Causal-Dilation Clock makes this distinction available to downstream consumers, including the user interface.

These capabilities are not new in the abstract. Distributed databases have offered reproducibility, observability, and trust signals for decades. What is new is recognising that for LLM agent systems, the *temporal* axis of these capabilities cannot be reduced to wall-clock or vector clocks alone, and that the missing piece is exactly what we have called proper time.

---

## §5 Implications — *to be written*

## §6 Conclusion: Sovereign Temporal Continuity — *to be written*

---

## References

[^1]: Lamport, L. (1978). Time, clocks, and the ordering of events in a distributed system. *Communications of the ACM*, 21(7), 558–565.

[^2]: Mattern, F. (1989). Virtual time and global states of distributed systems. *Parallel and Distributed Algorithms*, 215–226.

[^3]: Corbett, J. C., et al. (2013). Spanner: Google's globally distributed database. *ACM Transactions on Computer Systems*, 31(3), 1–22.

[^4]: Dillenberg, H. G. O. (2026). AgentClaw — a local multi-agent AI system. https://www.dillenberg.net/agentclaw-lokales-multi-agent-ki-system/
