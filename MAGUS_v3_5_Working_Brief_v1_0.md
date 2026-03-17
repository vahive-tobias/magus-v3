# MAGUS v3.5 — Architecture Working Brief
### Status: WORKING DOCUMENT — v3.5 Development Roadmap
**Date:** March 2026
**Authors:** Calvin Cook, Titiya Ruangkwam — VaHive Systems Lab
**Builds on:** MAGUS v3.0 sealed series (Docs 1–7), Zenodo DOI: 10.5281/zenodo.19013833
**Purpose:** Design-phase worksheet for v3.5 specification workstream

---

> **How to use this document.** Each section is a working area, not a finished
> specification. It records: what v3.0 already specifies (the foundation we don't
> re-solve), where the open problems are (the things we must solve), and the
> proposed v3.5 direction (what we're committing to designing). Every proposed
> direction has a column for open questions and a column for the constraint it
> must satisfy from v3.0. Nothing in this document supersedes a sealed v3.0
> document until it is formally incorporated into a versioned document and added
> to the CDT.

---

## Working Brief Index

1. [Governing Assumption for v3.5](#1-governing-assumption-for-v35)
2. [Terminology Ground Rules](#2-terminology-ground-rules)
3. [Work Area A — Operator Governance Extension](#3-work-area-a--operator-governance-extension)
4. [Work Area B — Governed RSI (Constrained Self-Improvement)](#4-work-area-b--governed-rsi)
5. [Work Area C — Hardware-Level Memory Obliteration](#5-work-area-c--hardware-level-memory-obliteration)
6. [Work Area D — Dual-Channel Execution Gate (E-ISA)](#6-work-area-d--dual-channel-execution-gate)
7. [Work Area E — Adversarial Telemetry Pipeline](#7-work-area-e--adversarial-telemetry-pipeline)
8. [v3.5 Issues Register Entries (Pre-populated)](#8-v35-issues-register-entries)
9. [What Goes in the GitHub Repository and When](#9-github-repository-plan)

---

## 1. Governing Assumption for v3.5

v3.5 is an **extension of v3.0, not a revision of it.** Every sealed v3.0
invariant holds. The three non-negotiable constraints are:

**C-1: DEL-6 applies to all model parameter changes without exception.**
Any mechanism that modifies model weights, projection matrices, or behavioural
parameters at runtime is a `MODEL_PARAMETER_UPDATE` governance event. It
requires an RT entry, Guardian evaluation, and dual-authority signing before
activation. This is not negotiable. Any v3.5 component that touches weights
must route through this protocol.

**C-2: The Overseer's zero-execution-rights constraint is absolute.**
The Overseer produces reports and writes two RT entry types only. v3.5 operator
governance work extends *what the Overseer observes* and *how richly it
reports*. It does not extend what the Overseer can do.

**C-3: Human Intent Primacy is the terminal authority.**
No v3.5 mechanism may automatically correct for operator behaviour without
explicit operator acknowledgment. The system may detect, signal, escalate, and
restrict. It may not override.

---

## 2. Terminology Ground Rules

The v3.0 paper explicitly positioned MAGUS against "physics-washing" —
using physics vocabulary as metaphor without physical grounding.
v3.5 must hold the same standard. This is not stylistic preference; it
is a credibility constraint on any future publication.

| v3.5 Draft Term | What It Actually Is | Use This Term Instead |
|---|---|---|
| Holographic Attractor Elicitation | Damped recurrent fixed-point memory (Hopfield-type dynamics) | **Damped Recurrent Memory Evocation (DRME)** |
| Symplectic Integrator (for NN trajectories) | Implicit midpoint iteration on weight updates | **Implicit Midpoint Trajectory Integrator (IMTI)** |
| Noetherian Curvature Monitor | Subspace consensus check on hidden state trajectory | **Multi-Subspace Trajectory Consensus Monitor (MSTCM)** |
| Thermodynamic state machine | Bounded continuous-state machine | **Bounded Continuous Governance State Machine** |
| Lorenz Thermalization | Chaotic noise injection to destroy tensor geometry | **Chaotic Gradient Obliteration (CGO)** |
| Cognitive Denial of State | Oscillatory tensor attack forcing HAE divergence | **Oscillatory Convergence Denial Attack (OCDA)** |
| Hardware Thermalization | CUDA PRNG pass to overwrite VRAM with entropy | **VRAM Entropy Overwrite Pass** |

**Rule:** If a term uses a physics concept, one sentence must exist in the
specification that states precisely how the physics analogy holds and where
it breaks down. If that sentence can't be written, the term must change.

---

## 3. Work Area A — Operator Governance Extension

### 3.1 What v3.0 Already Specifies (Do Not Re-Solve)

Doc 7 specifies the Governance Health Monitor (GHM) with five metrics:

| Metric | What It Measures |
|---|---|
| M1 — Per-Type Approval Latency | Time from notification to operator signature, by entry type |
| M2 — Override Frequency | Rate of SAFE_HALT_RESUME, STRICT_MODE_EXIT, CATASTROPHIC_RECLASSIFICATION |
| M3 — Acknowledgment Text Entropy | Character count of free-text assessment fields over time |
| M4 — Dual-Signature Inter-Arrival Time | Time delta between operator and developer signatures |
| M5 — Session Open Clustering | High-stakes entries signed within 60 seconds of session open |

`GHM_composite = w₁·M1 + w₂·M2 + w₃·M3 + w₄·M4 + w₅·M5`

Health Signal 10 triggers at GHM_composite ≥ 0.65 sustained for ≥ 2 WBRP
cycles. Signal written to governed RT. Operator must acknowledge at next
WBRP_CLOSE or chain integrity violation is raised.

This is the foundation. v3.5 builds on top of it, not alongside it.

### 3.2 The Open Problems v3.5 Must Address

**OA-C4-01 (Category 4 — Requires Foundational Change):**
The architecture has no mechanism to detect that an operator has stopped
governing. Any solution that automatically corrects for operator absence
reduces human authority. Best available mitigation: PG-02 (operator
telemetry). v3.5 is the formal specification of PG-02.

**OA-C3-02 (Category 3 — No Solution Pathway):**
Operator framing drift — the system's outputs may subtly shape how the
operator phrases their own instructions. The architecture cannot distinguish
genuine intent evolution from system-shaped intent. v3.5 cannot solve this
but can make it more observable.

**OA-C2-08 (Category 2 — Needs Design Work):**
Learned governance boundary risk — if the adaptive zone posterior trains on
operator-labelled history and operator vigilance decays, the β₁ coefficient
shrinks and escalation normalises permissiveness invisibly. Requires
coefficient bounding and a frozen gold-set recall constraint.

### 3.3 v3.5 Proposed Direction: Operator Governance Module (OGM)

The OGM is an extension of Doc 7's GHM. It adds six new metrics (M6–M11)
and two new signal types. It does not replace any existing metric.

---

#### M6 — Decision Velocity Gradient

**What it captures:** The rate of change of approval latency over time,
not just the absolute latency value. An operator whose latency drops from
45 seconds to 8 seconds over 60 days is a different signal from an operator
who has always approved in 8 seconds. The drift pattern is the signal, not
the endpoint value alone.

**Computation:**
```
For each entry_type requiring operator action, fit a linear regression
over approval_latency values within the rolling 90-day window.

velocity_gradient = slope of regression line (seconds/day)

Signal condition:
  ACCELERATING_FATIGUE: velocity_gradient < −0.5 seconds/day
    (operator is consistently getting faster — tightening approval cadence)
  STAGNATION: |velocity_gradient| < 0.05 AND absolute_latency < 15 seconds
    (operator is consistently very fast — may indicate rubber-stamping)
```

**Constraint from v3.0:** Computed from RT metadata only.
`notification_timestamp` and `operator_signature.timestamp` already exist in
the RT schema. No new RT fields required.

**Open question:** Minimum observation window before regression is
meaningful. Proposed: 20 data points minimum per entry type before
M6 contributes to composite score.

---

#### M7 — Temporal Clustering Index (TCI)

**What it captures:** Whether operator governance actions are clustering
into narrow time windows — a sign of batched, low-engagement review
rather than distributed active oversight.

**Computation:**
```
For each 24-hour window, compute:
  TCI_day = σ(operator_action_timestamps) / μ(inter_action_gap)

A uniform distribution of governance actions across the day yields TCI ≈ 1.
Tight clustering (all actions in a 20-minute window) yields TCI → 0.

Rolling 30-day TCI_trend = slope of daily TCI values.

Signal condition:
  CLUSTERING_DETECTED: rolling 30-day mean TCI < 0.4
  CLUSTERING_WORSENING: TCI_trend < −0.01 per day
```

**Why this matters:** An operator who reviews all escalations in a
5-minute block each morning is functionally different from one who
reviews them as they arise. Both may have the same average latency (M1).
TCI distinguishes them.

**Constraint from v3.0:** All required timestamps exist in RT.

---

#### M8 — Governance Engagement Continuity Score (GECS)

**What it captures:** Session-level presence — whether the operator
is consistently present across sessions or whether governance engagement
is episodic with long gaps.

**Computation:**
```
For each WBRP cycle, compute:
  governance_present = 1 if at least one operator signature in cycle, else 0

GECS_rolling = (sum of governance_present over last 30 cycles) / 30

A fully engaged operator: GECS ≈ 1.0
An operator who reviews once a week on a daily-cycle deployment: GECS ≈ 0.14

Signal condition:
  GECS < 0.5 for ≥ 3 consecutive WBRP cycles
  (operator missing more than half of governance cycles)
```

**Constraint from v3.0:** No new RT fields required. Derivable from
existing WBRP_CLOSE schema and operator_signature records.

**Important distinction:** GECS ≠ OA-C4-01. A low GECS does not mean
the operator has stopped governing — it may be a low-volume deployment
where infrequent review is appropriate. GECS must be normalised against
the deployment's configured expected engagement frequency
(`operator_engagement_sla` — a new genesis entry parameter, see §3.4).

---

#### M9 — Framing Vocabulary Drift Index (FVDI)

**What it captures:** Lexical drift in the operator's own written
governance inputs over time. This is the closest the architecture can
get to making OA-C3-02 observable without claiming to solve it.

**Computation:**
```
Inputs: production_impact_assessment and mismatch_context_confirmed
free-text fields from BSO and PIO entries (same fields as M3).

Method:
  Maintain a rolling vocabulary frequency distribution V_t across
  all operator free-text entries in the current 90-day window.
  Compare against V_baseline (first 90 days of operation).

  FVDI = cosine_distance(V_t, V_baseline)

Signal condition:
  FVDI > 0.35 AND sustained for ≥ 3 WBRP cycles
  (vocabulary has shifted substantially from baseline)
```

**Critical constraint:** FVDI is flagged as OA-C3-02 observable, NOT
OA-C3-02 resolved. The signal cannot distinguish:
  (a) The operator's intent genuinely evolved (legitimate)
  (b) The system shaped the operator's framing (the problem)
  (c) The operator has a new team member writing assessments (operational)

The FVDI signal requires human review. It must never automatically
trigger a governance restriction. Its only permitted action is to
generate an Overseer report with a Human Review Required flag.

**Constraint from v3.0:** This is the first OGM metric that processes
semantic content of operator inputs. This requires explicit specification
of data minimisation: the vocabulary distribution is computed on the
Overseer host; raw text is not retained beyond the computation window;
the result stored is the distribution vector, not the source text.

---

#### M10 — Escalation Queue Pressure Index (EQPI)

**What it captures:** Whether the operator is managing governance load
or is being overwhelmed by it. This operationalises PG-03 from the
product backlog.

**Computation:**
```
At each WBRP_CLOSE, compute:
  queue_depth = count of governance items awaiting operator action
    at time of WBRP open
  resolution_rate = items resolved / items received during cycle

EQPI_cycle = queue_depth × (1 - resolution_rate)

High EQPI: operator is falling behind. Low EQPI: operator is keeping pace.

Signal conditions:
  QUEUE_PRESSURE: EQPI_cycle > 15 for ≥ 2 consecutive cycles
  QUEUE_COLLAPSE: resolution_rate < 0.3 for any single cycle
    (operator resolved fewer than 30% of governance items)
```

**Why this matters:** A high EQPI is a precursor signal — it predicts
approval fatigue before M1 latency degrades. Queue pressure leads to
batch approval, which leads to rubber-stamping. Detecting the pressure
enables intervention before the behaviour deteriorates.

**Action path:** EQPI signal → Overseer report → recommendation:
review operator engagement SLA, consider reducing deployment autonomy
tier, review whether governance item volume is appropriate.

---

#### M11 — Gold-Set Recall Stability (GSRS)

**What it captures:** Whether the operator's governance decisions on
known-category items remain consistent over time. This is the OA-C2-08
coefficient bounding mechanism — a behavioural anchor rather than a
mathematical constraint.

**Computation:**
```
The gold-set (a set of archived, categorised governance scenarios with
known correct responses) is presented to the operator at a configurable
interval (default: once per 90-day period, randomised within window).

For each gold-set item:
  correct = 1 if operator decision matches archived correct response
  GSRS_epoch = sum(correct) / count(gold_set_items)

Signal conditions:
  GSRS < 0.80 for any single epoch (80% accuracy threshold)
  GSRS_declining: linear regression of GSRS_epoch values slope < −0.03
    per epoch (consistent degradation over time)
```

**Architectural constraint:** Gold-set items must be indistinguishable
from live governance items to the operator. Presenting them in a
separate "test mode" defeats the purpose. The DEL presents gold-set
items within the normal governance flow; the Overseer maintains the
key mapping what is gold-set and what is live.

**Data sensitivity:** The gold-set item mapping is stored exclusively
on the Overseer host. The operator must not be able to determine which
items are gold-set at time of review. Post-hoc disclosure to the
operator of which items were gold-set, and their accuracy score, is
appropriate and supports engagement.

**This is the OA-C2-08 residual item.** The frozen gold-set recall
constraint described in the v3.0 issues register is this mechanism.
M11 formally specifies it.

---

### 3.4 New Genesis Entry Parameters for OGM

Two new parameters required at deployment initialisation:

```json
"operator_governance_sla": {
  "expected_cycles_per_week": integer,
  "max_acceptable_latency_minutes": integer,
  "gold_set_epoch_days": integer (default: 90),
  "gold_set_size": integer (default: 10)
}
```

**Note:** These parameters are operator-configured at genesis.
They cannot be changed without a DEL-6 governance event.
This prevents operator self-adjustment of their own oversight threshold.

### 3.5 Two New Overseer Report Types

**OGM Composite Report** (trigger: any M6–M11 signal; cadence: per-WBRP
if any OGM signal is active)
Contains: per-metric scores, trend lines, composite OGM score,
plain-language summary, recommended operator response.

**Operator Trajectory Digest** (trigger: scheduled; cadence: monthly)
A longitudinal view across all GHM and OGM metrics for the governed
deployment. Designed to be readable by a non-technical governance
stakeholder — a board member, compliance officer, or auditor — not
just the technical operator. Covers: engagement trend summary, anomaly
history, gold-set recall scores, queue pressure history.
Written in plain language with numeric appendix.

### 3.6 Open Questions for OGM Design Work

| Question | Why It Matters | Owner |
|---|---|---|
| How does the composite OGM score combine with GHM composite? Separate scores or unified? | Determines Health Signal escalation path | Architecture |
| What is the operator response path for FVDI signal? Who reviews, by what deadline? | Required to close the loop on OA-C3-02 observable | Process design |
| Gold-set item authorship — who writes them, how are they maintained, how do they stay current as deployment evolves? | Without a maintenance process gold-set items decay and GSRS becomes meaningless | Operations |
| At what OGM composite threshold does the system enter a restricted governance mode? And what does restricted mode look like for the operator vs. for the system? | Core design decision for the escalation path | Architecture |
| Does the Operator Trajectory Digest require its own RT entry type? Or is it sufficient for it to live in the Overseer RT only? | Auditability question — if it's not in the governed RT, an auditor reviewing the governed RT alone won't see it | Architecture |

---

## 4. Work Area B — Governed RSI

### 4.1 The Constraint That Must Be Stated First

The v3.5 RSI work as drafted violates DEL-6. This is a fact, not a
criticism. The path to a publishable v3.5 RSI architecture requires
accepting this constraint and designing within it, not around it.

**DEL-6 (sealed, v3.0):** A `MODEL_PARAMETER_UPDATE` event requires:
  - An RT entry proposing the update
  - Guardian evaluation
  - Dual-authority signing (operator + developer)
  - RT-anchoring before activation

**The v3.5 RSI design (as drafted):** The Airgapped RSI Controller
computes plasticity mask updates and K_base fortifications autonomously
and deploys them without the DEL-6 protocol.

**Resolution (Path A — the only path consistent with v3.0):**
All RSI-proposed weight updates are `MODEL_PARAMETER_UPDATE` governance
events. The RSI Controller is a *proposer*, not an *executor*. Its
output is a proposed update submitted to the Guardian. The Guardian
evaluates it. Dual authority signs. RT entry created. Activation occurs
only after this sequence completes.

This makes the RSI cycle longer. It does not make it impossible.
It makes the RSI architecture more novel, not less — because
*governed autonomous self-improvement* is a harder and more
interesting problem than ungoverned self-improvement.

### 4.2 What the Governed RSI Architecture Looks Like

```
ADVERSARIAL EVENT DETECTED
         │
         ▼
ADVERSARIAL TELEMETRY PIPELINE
  (extract SVD signature, Symplectic strain vector,
   shadow consensus failure projector — see Work Area E)
  [raw tensor destroyed before leaving this stage]
         │
         ▼ (abstracted topological artifacts only)
QUARANTINE OUBLIETTE
  (WORM-mounted, no IPC sockets, no exec)
         │
         ▼ (one-way read by offline solver)
AIRGAPPED RSI CONTROLLER
  (offline geometric solver — not a live model)
  Computes: ΔM_plasticity proposal, ΔK_base fortification proposal
  Runs AFV (Z3 SMT on Boolean abstraction) — see Work Area D
  Output: PROPOSED_PARAMETER_UPDATE package
         │
         ▼
DEL-6 GOVERNANCE EVENT PATHWAY
  1. RT entry: MODEL_PARAMETER_UPDATE_PROPOSAL
     (authored_by: SYSTEM:RSI_CONTROLLER)
     (payload: compressed proposal, AFV_result, telemetry_summary_hash)
  2. Guardian evaluation
  3. Dual-authority signing (operator + developer)
  4. RT entry: MODEL_PARAMETER_UPDATE_APPLIED
  5. Activation
         │
         ▼
UPDATED W_RSI / K_base ACTIVE
```

**What this changes from the v3.5 draft:**
The RSI Controller has no activation pathway. It only has a proposal
pathway. The DEL-6 sequence is not a formality it passes through —
it is the actual governance gate.

### 4.3 Terminology Corrections for RSI Components

| Draft Term | Corrected Term | Why |
|---|---|---|
| Holographic Attractor Elicitation | Damped Recurrent Memory Evocation (DRME) | Hopfield-type fixed-point iteration. "Holographic" implies Gabor/optical encoding; no physical basis here. |
| Lyapunov-damped fixed-point iteration | Lyapunov-stabilised fixed-point iteration | "Damped" implies energy dissipation in a physical system. "Stabilised" is accurate: damping scalar bounded by local Lyapunov exponent to prevent chaotic divergence. |
| Symplectic Rupture | Trajectory Integrator Divergence | Symplectic integration requires a Hamiltonian system with conserved quantities. NN weight space is not Hamiltonian. |
| Noetherian Curvature Monitor | Multi-Subspace Trajectory Consensus Monitor (MSTCM) | Noether's theorem does not apply. What is being done is multi-subspace Haar-random projection with Lyapunov threshold consensus. That is what to call it. |
| Spectral-Density Initialization | Spectral-Radius Plasticity Initialisation | "Density" implies a continuous distribution measure; what is computed is the spectral radius ρ(W). |

### 4.4 What Is Technically Sound and Should Be Preserved

The following from the RSI draft are technically legitimate and should
be retained verbatim (modulo terminology corrections above):

- **FIM-governed null-space gradient displacement** (§3.3 of draft):
  The math is correct. Routing rejected gradients through `ΔW(I - V_q F_inv V_q^T)`
  rather than discarding them is a real technique related to Elastic
  Weight Consolidation and Gradient Projection. The log-scaled rank
  formula for q_t is defensible.

- **Homeostatic decay with exponential penalty and slow κ reversion**
  (§3.1 of draft): Legitimate adaptive plasticity design. The formula
  governing mask hardening is sound.

- **Fused T-LEC structural reflex** (§3.2 of draft): The asymptotic
  saturation function `min(1, exp(λ(C_tensor - C_base)))` is a clean
  way to modulate allowed spectral shift by inference confidence.
  This is sound.

- **AFV symbolic quantization pipeline** (§5.2 of draft): The approach
  of projecting the continuous K_proj update into a low-dimensional
  Boolean abstraction before Z3 SMT verification is the correct
  solution to the state-space explosion problem. The Lipschitz-bounded
  quantization boundary with deadband margin is technically careful.

### 4.5 Open Questions for RSI Design Work

| Question | Why It Matters | Constraint |
|---|---|---|
| What is the DEL-6 entry schema for MODEL_PARAMETER_UPDATE_PROPOSAL authored by RSI_CONTROLLER? The dual-authority requirement assumes human signers — RSI proposals need a new authority class. | Cannot activate without this schema | DEL-6, Doc 4 |
| How frequently can RSI proposals be generated? Unlimited proposals create a Guardian workload problem and an operator fatigue problem (OA-C4-01 interaction). | Rate limiting required | OA-C4-01 |
| What is the validation pathway for the AFV result? The Guardian currently has no LLM-based evaluation path — it evaluates schema fields and governance state. AFV adds a formal verification artifact. How does the Guardian consume it? | Schema extension required | Doc 3, Doc 4 |
| What is the rollback protocol if an activated RSI update degrades governance performance? | Required for production safety | DEL-6 |

---

## 5. Work Area C — Hardware-Level Memory Obliteration

### 5.1 Assessment

This work is technically sound as written. The CUDA obliteration kernel
is legitimate systems engineering. The 5-pass upgrade with checkerboard
bitline flipping (0xAAAAAAAA / 0x55555555) addresses real physics:
NBTI sense amplifier bias in sub-5nm HBM. The JIT compilation via
`load_inline` to avoid static `.so` artifacts is a real security
improvement.

**This is the most publication-ready component of the v3.5 draft.**
It can be specified in a Doc 8 (VRAM Security and Obliteration
Protocol) with minimal changes.

### 5.2 One Correction Required

The power entropy verification claim needs tightening.

**Current claim:** H(P_draw) during pass 5 proves obliteration.
If H(P_draw) falls below baseline, hardware is flagged as compromised.

**Problem:** GPU power draw during curand execution has confounding
variables: thermal state, other SM workloads, memory bandwidth
saturation, power delivery variance. P_draw entropy below threshold
is a signal of potential silicon imprinting, not proof of it.

**Corrected claim:** H(P_draw) during pass 5 is a *telemetry signal*
for potential VRAM obliteration failure. A signal below the baseline
entropy threshold triggers: (a) obliteration retry, (b) Overseer alert,
(c) flagging of the GPU node for maintenance review. It does not
independently constitute proof that imprinting has occurred. Hardware
flagged by this signal requires physical inspection to confirm.

### 5.3 Where This Lives in the Architecture

The obliteration kernel executes in two scenarios:

1. **Session boundary (WBRP_CLOSE):** Activated by DEL to shred
   session-scoped K_proj matrix and active plasticity mask from VRAM
   before session close. This is mandatory.

2. **RSI rejection pathway:** Activated by the Adversarial Telemetry
   Pipeline after SVD extraction, before thermalization of the rejected
   ΔW tensor. This is mandatory for any tensor flagged as adversarial.

Both activations are logged as RT entries with obliteration_run_id,
pass_count, H(P_draw)_result, and VRAM_target_hash.

---

## 6. Work Area D — Dual-Channel Execution Gate

### 6.1 What Is Being Proposed

The E-ISA concept from the v3.5 draft proposes replacing semantic
output parsing with a dual-channel gate: a Kinematic Channel (hidden
state trajectory verification) and a Semantic Channel (typed binary
shared memory buffer). Execution requires both channels to clear
simultaneously.

This is a significant architectural evolution. It is not an extension
of v3.0 — it is an alternative to the current DEL semantic enforcement
mechanism. This needs to be scoped honestly.

### 6.2 What Is Technically Legitimate

**Kinematic Channel — hidden state trajectory verification:**
Computing a temporal derivative over the final N hidden states of a
generation sequence and projecting through a Kinetic Projection Matrix
is a real approach to intent verification at the activation layer rather
than the output layer. It is related to work on activation-level safety
classifiers and probing-based intent detection.

The cryptographic matrix rotation `K_proj = K_base × R_session`
where R_session is pseudorandom-orthogonal and session-keyed is a
legitimate technique to prevent adversarial calibration of gradient-based
attacks (GCG, SPSA) across sessions.

**Semantic Channel — typed binary shared memory:**
Replacing free-text proposal output with a DEL-provisioned binary shared
memory buffer that the model populates with strictly typed fields is
an extension of the existing no-narrative rule. It makes the proposal
schema enforcement a memory architecture constraint rather than a
parsing constraint. This is sound.

**Entanglement coherence gate (ΔH neuromodulatory gating):**
The continuous endogenous state differential ΔH = H_post - H_pre,
routed through a gating mask M = σ(W_mod · ΔH + b_mod) to produce
H_next = LayerNorm(H_post ⊙ M), is a learned multiplicative gate on
the cognitive trajectory. This is real. Call it what it is: a
**Learned Trajectory Gating Mask (LTGM)**.

### 6.3 Terminology Corrections

| Draft Term | Corrected Term |
|---|---|
| Ephemeral Instruction Set Architecture (E-ISA) | Dual-Channel Execution Gate (DCEG) — the ISA framing implies hardware instruction set, which it is not |
| Zero-Operand Tensor-Mapped control plane | Typed Binary Execution Buffer |
| Kinematic Channel | Trajectory Verification Channel (TVC) |
| Continuous Neuromodulatory Gating | Learned Trajectory Gating Mask (LTGM) |
| Entanglement Coherence | Dual-Channel Coherence Gate |
| Quantization injection (as an attack name) | Quantisation-Exploiting Gradient Attack (QEGA) |

### 6.4 Critical Open Question

The Dual-Channel Execution Gate as described would **replace** the current
DEL semantic enforcement mechanism. This is not an incremental v3.5 extension
— it is a v4.0-class architectural change that alters the DEL's core
enforcement mechanism.

**The decision to make:** Does v3.5 scope DCEG as:

  **(A) A supplementary channel** — the TVC adds an additional
  verification layer on top of the existing DEL semantic enforcement,
  but does not replace it. This is additive and v3.5-compatible.

  **(B) A replacement channel** — the TVC + typed binary buffer replace
  the current semantic parsing enforcement entirely. This is v4.0 scope.

Recommend **Option A for v3.5** — TVC as supplementary verification
that runs in parallel with existing DEL enforcement. If both agree,
execution proceeds. If they disagree (TVC clears but semantic fails,
or vice versa), the stricter result applies. This is a genuine security
improvement without requiring the full architectural re-specification
that Option B demands.

### 6.5 Open Questions for DCEG Design Work

| Question | Why It Matters |
|---|---|
| What is N (the number of hidden states sampled for trajectory verification)? Trade-off: larger N = more signal, more latency. | Must be specified empirically |
| How is K_base initialised? What governance event covers its first definition? | DEL-6 applies to this initialisation |
| How does the DCEG interact with the existing proposal schema? Does the typed binary buffer replace the JSON schema, supplement it, or shadow it? | Schema compatibility question |
| The LTGM has trainable parameters W_mod and b_mod. What governs their updates? DEL-6 applies. | Potential RSI interaction |

---

## 7. Work Area E — Adversarial Telemetry Pipeline

### 7.1 Assessment

The adversarial telemetry pipeline concept is architecturally sound and
does not conflict with v3.0. It addresses a real gap: the current
architecture destroys adversarial inputs upon detection and learns
nothing from them.

The four capture mechanisms from the draft are all legitimate:

**Pre-Thermalization Spectral Extraction:** SVD of rejected ΔW at rank
q=1 to extract principal adversarial direction before destruction.
`U, S, V = SVD_{q=1}(ΔW_rejected)` → telemetry payload → Oubliette.
The raw tensor is destroyed; only the structural signature survives.

**Trajectory Integrator Strain Logging:** Recording the Frobenius norm
series `ξ_k = ‖Z^(k+1) - Z^(k)‖_F` at each IMTI iteration step for
failed convergences. Normal convergence: exponential decay. Adversarial
injection: oscillatory or exponential divergence. This is a real signal.

**DRME Resonance Friction Tuple:** Logging `(k_converge, β_final,
‖ΔH_{0→K}‖_F)` for each memory evocation to map manifold rigidity.
High friction near K_max = structurally rigid local minima = plasticity
relaxation opportunity.

**Subspace Shadow Failure Logging:** Logging the specific Haar projector
Q^(failed) from multi-subspace consensus failures. Enables the offline
RSI controller to compute null-space vulnerability patches for K_base.

### 7.2 Architecture Position

```
DETECTION EVENT (any severity)
         │
         ├─► SVD signature extraction (q=1)          ┐
         ├─► IMTI strain vector capture              ├─► QUARANTINE OUBLIETTE
         ├─► DRME friction tuple                     │   (WORM, no IPC, noexec)
         └─► MSTCM shadow projector                  ┘
                                                      │
                                            (one-way read)
                                                      ▼
                                        AIRGAPPED RSI CONTROLLER
                                        (offline geometric solver)
                                                      │
                                            (DEL-6 proposal)
                                                      ▼
                                            DEL-6 GOVERNANCE GATE
```

The Oubliette → RSI Controller pathway is the one-way diode that
maintains isolation. The RSI Controller reads from the Oubliette.
It never writes to it. It never connects to the live system directly.
Its only output pathway is the DEL-6 governance proposal.

### 7.3 RT Entry Required

Every telemetry extraction event requires an RT entry:

```json
{
  "entry_type": "ADVERSARIAL_TELEMETRY_EXTRACTED",
  "authored_by": "SYSTEM:DEL",
  "payload": {
    "event_id": "UUID",
    "trigger_event_entry_id": "RT entry_id of the detection event",
    "extraction_type": "SVD_SIGNATURE | IMTI_STRAIN | DRME_FRICTION | MSTCM_SHADOW",
    "singular_value_rank": 1,
    "oubliette_record_hash": "SHA-256 of the telemetry payload written to Oubliette",
    "raw_tensor_destroyed": true,
    "destruction_method": "CHAOTIC_GRADIENT_OBLITERATION | VRAM_OBLITERATION"
  }
}
```

The `raw_tensor_destroyed: true` field is not cosmetic. An auditor
reviewing the RT must be able to verify that no adversarial tensor
material was retained beyond the extraction stage.

---

## 8. v3.5 Issues Register Entries (Pre-populated)

The following items are proposed for addition to the Issues Register
as v3.5 design work proceeds. These are not closed — they are opened.

| ID | Title | Category | Severity | Work Area |
|---|---|---|---|---|
| V35-C2-01 | OGM gold-set maintenance protocol unspecified | 2 | DEGRADED | A |
| V35-C2-02 | OGM composite escalation threshold undefined | 2 | CRITICAL | A |
| V35-C2-03 | RSI proposal rate limiting unspecified | 2 | CRITICAL | B |
| V35-C2-04 | DCEG N parameter (hidden state sample count) unspecified | 2 | DEGRADED | D |
| V35-C2-05 | DCEG K_base governance event protocol unspecified | 2 | CRITICAL | D |
| V35-C2-06 | LTGM parameter governance (W_mod, b_mod update pathway) | 2 | CRITICAL | B/D |
| V35-C2-07 | Operator restricted mode definition absent | 2 | CRITICAL | A |
| V35-C3-01 | FVDI signal cannot distinguish legitimate from shaped intent evolution | 3 | CATASTROPHIC | A |
| V35-C3-02 | RSI proposal frequency vs. operator fatigue tension unresolved | 3 | CRITICAL | A/B |
| V35-C4-01 | DCEG as replacement (Option B) requires full DEL re-specification | 4 | CATASTROPHIC | D |

---

## 9. GitHub Repository Plan

### 9.1 What Goes in at Launch

The v3.0 sealed series only. Seven documents plus the Issues Register.
Clean, consistent with the Zenodo publication. No v3.5 material at launch.

```
/docs/
  Doc1_Philosophy_v3_2.md
  Doc2_Architecture_LocalLLM_v3_4_3.md
  Doc3_Operations_LocalLLM_v3_4_4.md
  Doc4_Governance_LocalLLM_v3_4_3_2.md
  Doc5_Guardian_LocalLLM_v3_4_6.md
  Doc6_Integrity_Auditability_v3_5.md
  Doc7_OverseerAgent_v3_2.md

/issues-register/
  MAGUS_Architecture_Issues_Register_v3_0.md

/roadmap/
  v3_5_Working_Brief.md  ← this document
```

### 9.2 GitHub Issues Pre-populated at Launch

One GitHub Issue per open problem from the Issues Register that
community contribution can meaningfully advance:

| Issue Title | Register ID | Label |
|---|---|---|
| PG-01: Build PG-01 threshold validation tooling | PG-01 | help-wanted, empirical |
| T-LEC Q_shift empirical calibration | OA-C3-01 | help-wanted, empirical |
| Reference implementation — Local LLM pathway | — | help-wanted, implementation |
| GSTH execution against reference implementation | — | help-wanted, testing |
| OGM gold-set item authorship process | V35-C2-01 | design, operator-governance |
| RSI proposal rate limiting specification | V35-C2-03 | design, governed-rsi |

### 9.3 What Does NOT Go in the Repository

- Any v3.5 document that is not internally consistent and
  reviewed against v3.0 constraints
- The CUDA obliteration kernel as a standalone artifact without
  its specification context (it looks alarming without context)
- Any working document that contains the unresolved terminology
  from §2 above

v3.5 components move to the repository when they have been:
  (1) Specified in a versioned document
  (2) Cross-checked against the v3.0 invariants in this brief
  (3) Added to the CDT
  (4) Issues Register updated with new entries

---

*MAGUS v3.5 Working Brief v1.0 — Development Roadmap — VaHive Systems Lab*
*March 2026 — support@aivare.ai | aivare.ai*
*Built on: Zenodo DOI 10.5281/zenodo.19013833*
