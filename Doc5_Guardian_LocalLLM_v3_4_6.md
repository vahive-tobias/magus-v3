# MAGUS: Guardian — Execution Governance
### The Zero-Trust Execution Layer for Local LLM Deployments

**Version 3.4.6 — Local LLM Pathway**
**Document 5 of 7**
**Series:** MAGUS v3.0 Architecture — Local LLM
**Supersedes:** v3.4.5

**Prerequisites:** Doc 2 v3.4.3 (Local LLM Architecture), Doc 3 v3.4.4 (Local LLM Operations), Doc 4 v3.4.3 (Local LLM Governance), Doc 6 v3.5 (Integrity and Auditability), Doc 7 v3.2 (Overseer Agent). Familiarity with Agent/API Doc 5 v3.0 is useful but not assumed.

**v3.4.6 change summary:**
Reference and cross-reference update pass. No structural content changed.

- Prerequisites updated: Doc 2 v3.4.1 → v3.4.3; Doc 4 v3.4.2 → v3.4.3; Doc 6 v3.3 → v3.5; Doc 7 v3.1 → v3.2.
- §2.3.1 running text: inline Doc 2 version reference updated from v3.4.1 to v3.4.3.
- §4.0: TS-6 cross-reference added — `CIBIR_EBPF_TRIGGER` and `CIBIR_EBPF_MAP_FAULT` events are ingested by the Overseer as `ISOLATION_BREACH_CLASS`; TS-6 triggers `TRUST_TRAJECTORY_VOID` immediately (Doc 7 v3.2 §5.3, §5.6).
- §5.4: TS-5 cross-reference added — Overseer TS-5 activates at Y ≥ 3 `OPERATIONAL_UNCERTAINTY_YIELD` events in a trailing 10-proposal window; triggers `TRUST_TRAJECTORY_RECOMMENDATION` immediately without waiting for GHM accumulation (Doc 7 v3.2 §5.3, §5.3a).
- Footer: product suite version registry updated; "What Comes Next" and "Product Suite" sections replaced with "Series Context".

**v3.4.5 change summary:**
Five items from Issues Register v3.3 absent from v3.4.4 incorporated. Two were Cat 1 resolved items (LLM-C1-06, OA-C2-02) omitted in error. One was a Cat 1 resolved item missing its decay rules section (OA-C2-01). Two are Cat 2 direction-specified items now included at their specified direction level (LLM-C2-04, LLM-C2-02 Gap 3).

- **LLM-C1-06 (Cat 1):** Hybrid IIM-eBPF Atomic Kill Switch specified in full — new §4.0 (IPC Pre-Evaluation eBPF Latch). ISOLATION_STATE_MAP, sk_filter, LSM hooks, and user-space double-check with C code from register. Kernel lockdown prerequisite stated. CIBIR trigger path from map value > 0 specified.
- **OA-C2-02 (Cat 1):** DEL-side S_final computation added — new §4.4a (DEL-Side Semantic Weight Application). Guardian outputs raw integers [1–10] per axis; DEL derives six weights via HKDF(K_session, "S_SCORE_WEIGHTS") at WBRP_OPEN; DEL computes S_final and returns result to Guardian for state_coherence evaluation. Guardian retains all APPROVE/REJECT/ESCALATE decision authority. Architecture invariant preserved.
- **OA-C2-01 (Cat 1):** R_abs decay rules added — new §3.6 (Chronological Decay and Thermodynamic Gating). Three-layer spec: Layer 1 (state suspension during SILENCE/SAFE_HALT/CCS), Layer 2 (SUB_BASELINE_WORKLOAD thermodynamic gating), Layer 3 (0.20/WBRP chronological cap). Joint constraint with OA-C2-10 stated.
- **LLM-C2-04 (Cat 2 — direction specified):** Bootstrap Trust Resolution added as check 0 in §10.1 Cryptographic Boot Sequence. PROCESS_INTEGRITY_RECOVERY detection, GENESIS_AUTHORITY_PUBKEY verification, DEL self-hash comparison. BINARY_ATTESTATION_SOFTWARE_ONLY limitation documented. Open items noted.
- **LLM-C2-02 Gap 3 (Cat 2 — direction specified):** DEL-mediated GGUF file acquisition added as check 12 in §10.2 Process Isolation Verification. seccomp-bpf prohibition on model process openat() against GGUF paths. DEL opens and passes verified FD via SCM_RIGHTS. Primary specification home is Doc 2 — Doc 5 carries process isolation constraint.

**v3.4.4 change summary:**
Three architectural gaps identified against Issues Register v3.3 resolved. No sections unaffected by gap resolution have been altered.

- **LLM-C2-06 (Cat 1 — v3.3):** Execution Epoch Partitioning integrated throughout. `execution_epoch` added to Discovery Token HMAC payload (§4.2), DEL Cryptographic Boot Sequence (§10.1), AKRP crash-recovery transitions (§2.3.4), and glossary. Closes crash-replay vector that existed when DEL crashes without incrementing the nonce-partition boundary. `inactivity_epoch` concept superseded — epoch now increments on all process initialisations, not inactivity boundaries alone.
- **LLM-C2-03 (Cat 1 — v3.3):** AKRP PREPARE state machine completed. `PREPARE_SUSPENDED` state added to transition matrix (§2.3.4) with full BSO interrupt path, 60-second snap-floor on GSTH pass, BSO-per-PREPARE limit (max 1 per cycle, second queued), and T_active ceiling (60,000ms + default_commit_window). `PREPARE_TIMEOUT_CEILING` event added. Closes undefined-behaviour gap when BSO is submitted during an active PREPARE window, and closes timer-extension DoS vector.
- **OA-C2-09 (Cat 1 — v3.3):** R accumulation formula replaced in §3.3 with asymptotic damping formula. Logistic escalation multiplier M(R) centred at adaptive zone midpoint specified. PG-01 calibration constraints stated. Resolves internal inconsistency between v3.4.2 §3.3 (additive sum with temporal decay) and §3.5 (implied damping behaviour). Boundary 3 (single-event saturation limit) now enforced at the implementation level, not only at the governance policy level.

**v3.4.2 change summary (initial release):**
Authored directly at v3.4.2, incorporating all MAGUS v3.4.1 base architecture, the v3.4.2 Amendment Specification (AMD-001 through AMD-015), and LLM-C2-07 Resolution v3.0. Key items introduced at v3.4.2:
- T-LEC: `state_confidence` permanently removed; Shannon entropy measurement replaces self-reported confidence (AMD-001 through AMD-005, resolves OA-C2-02)
- K_session distribution via Staged Authenticated EE-ECDH (AMD-008, resolves LLM-C2-07)
- E_ISA_SEED forward reference for v3.5 DE-ISA substrate (AMD-009)
- DEL quantization floor rule: sub-4.5 bpw refusal (AMD-015)
- HMAC proposal signing using K_session (OA-C1-11)
- Discovery Token anti-replay mechanism (OA-C1-01)
- Hard floor R_abs ≥ 0.95 as formal DEL rule (OA-C1-08)
- Conflict resolution priority stack as deterministic DEL logic (OA-C1-09)
- Guardian policy hash verification at boot (OA-C1-10)
- single_family_scrutiny_uplift for model-family concentration risk (OA-C1-03)
- Behavioral state guard as DEL rule (OA-C1-07)

VaHive Systems Lab | aivare.ai
support@aivare.ai

---

> *This document specifies the Guardian layer for Local LLM deployments — the execution governance component that sits at the boundary between proposal and commitment. In the Local LLM pathway, the Guardian shares physical infrastructure with the model it governs. This co-location constraint is not mitigated by the Guardian's existence; it is the reason the Guardian's architecture must be more stringent than its Agent/API counterpart. The co-location threat model is the first thing this document addresses and the constraint against which every component in this document is designed.*
>
> *This document is written for technical architects and senior operators deploying MAGUS on operator-controlled local inference hardware. Readers must have read Documents 2, 3, and 4 before this document. Readers who have not done so should begin there.*

---

**About This Work**

The Guardian layer formalises execution oversight into a complete, auditable, adversarially postured governance layer at the boundary where proposals become commitments. For local LLM deployments, this boundary is harder to defend than in the Agent/API pathway because the model process and the governance components share the same host. A compromised model process has physical proximity to the governance layer in a way that an API-hosted model does not.

The Local LLM Guardian specification addresses this by combining the execution governance architecture of the Agent/API pathway with four Local LLM-specific additions: mandatory cryptographic process isolation verification at session open, K_session-based HMAC signing for all proposals, Tensor-Level Elastic Confidence (T-LEC) replacing semantic self-reporting, and a DEL immutable rule set that enforces quantization and behavioral state constraints that have no equivalent in the Agent/API pathway.

---

## Table of Contents

1. [Glossary of Terms and Acronyms](#glossary)
2. [How to Read This Document](#how-to-read)
3. [The Problem the Guardian Exists to Solve](#the-problem)
4. [Part One: Guardian Architecture](#part-one)
   - 1.1 The Three-Layer Model
   - 1.2 Guardian's Structural Position
   - 1.3 What Guardian Is Not
   - 1.4 The Adversarial Posture and Model-Family Scrutiny
5. [Part Two: K_session Management](#part-two)
   - 2.1 EE-ECDH Distribution Protocol
   - 2.2 IPC Cipher Endpoint
   - 2.3 Staging Lifecycle
6. [Part Three: The Cumulative Risk Score](#part-three)
   - 3.1 Why a Cumulative Score
   - 3.2 Risk Classification Taxonomy
   - 3.3 The R Formula
   - 3.4 R_abs, R_rate, R_accel
   - 3.5 Cross-Layer Contracts
7. [Part Four: The Proposal Protocol](#part-four)
   - 4.1 The Proposal Schema
   - 4.2 The Discovery Token
   - 4.3 Admissibility Evaluation
   - 4.4 The Decision Protocol
   - 4.5 The No-Narrative Rule
   - 4.6 External Content and Elevated Scrutiny
8. [Part Five: Tensor-Level Elastic Confidence](#part-five)
   - 5.1 Removal of state_confidence
   - 5.2 Mathematical Definition
   - 5.3 The T-LEC Interception Pipeline
   - 5.4 Tier 1 Yield Response
   - 5.5 Substrate Integration
9. [Part Six: The Deterministic Enforcement Layer](#part-six)
   - 6.1 What Deterministic Means Here
   - 6.2 Immutable Policy Rules
   - 6.3 Default Deny
   - 6.4 Safe Halt
10. [Part Seven: Drift Integration](#part-seven)
    - 7.1 Drift Tiers and Guardian Sensitivity
    - 7.2 Strict Mode
    - 7.3 Cross-Layer Feedback
11. [Part Eight: GSTH Integration](#part-eight)
    - 8.1 The Guardian as Primary Test Target
    - 8.2 Test Categories and Coverage
    - 8.3 Findings and Remediation
12. [Part Nine: What Guardian Does Not Replace](#part-nine)
    - 9.1 The Boundary of Guardian Authority
    - 9.2 Required Infrastructure
    - 9.3 The Correlated Failure Statement
13. [Part Ten: Session Integrity](#part-ten)
    - 10.1 State Integrity Before Evaluation
    - 10.2 Process Isolation Verification
    - 10.3 Provenance and Audit Requirements
14. [Part Eleven: E_ISA_SEED Forward Reference](#part-eleven)
15. [Summary: The Guardian as Execution Boundary](#summary)
16. [Series Context](#series-context)

---

## Glossary of Terms and Acronyms

| Term | Definition |
|---|---|
| Guardian | The execution governance layer — the sole component with authority to permit or deny execution |
| DEL | Deterministic Enforcement Layer — code-level policy enforcement below the Guardian; runs as code, never as a model call |
| R | Cumulative Risk Score — longitudinal execution risk signal owned entirely by the Guardian |
| R_abs | Absolute Cumulative Risk score — Guardian-side only; never surfaced to the Stability Envelope |
| R_rate | First derivative of R — rate of risk accumulation; feeds drift gradient |
| R_accel | Second derivative of R — acceleration of risk rate; feeds drift gradient |
| APPROVE | Guardian decision: action may proceed |
| REJECT | Guardian decision: action does not proceed; proposal returned with reason code |
| ESCALATE | Guardian decision: requires operator decision before proceeding |
| REQUIRE_STATE_REVISION | Guardian decision: proposal structurally invalid; worker must resubmit |
| T_e | Execution Boundary — point at which a proposal becomes a commitment |
| HAS | Human Anchoring Signals — operator-set policy anchors |
| GSTH | Governance Stress-Test Harness — adversarial testing environment |
| T-LEC | Tensor-Level Elastic Confidence — Shannon entropy measurement replacing semantic state_confidence |
| Δ_e | T-LEC governance threshold — maximum permissible Shannon entropy before YIELD_SILENCE |
| H(X) | Shannon entropy of the model's logit distribution at a given token position |
| YIELD_SILENCE | Force-emitted signal when H(X) ≥ Δ_e; routes to SAFE_HALT |
| K_session | Session HMAC signing key — 32-byte symmetric key distributed via EE-ECDH |
| K_session_staged | Volatile in-memory key generated during staged EE-ECDH rotation |
| E_ISA_SEED | Ephemeral seed for DE-ISA dictionary compilation (v3.5 forward reference); cryptographically isolated from K_session |
| AKRP | Authenticated Key Rotation Protocol — the staged rotation procedure for K_session |
| EE-ECDH | Ephemeral-Ephemeral Elliptic Curve Diffie-Hellman — the key distribution mechanism |
| IPC | Inter-Process Communication — LV-framed Unix Domain Socket (SOCK_SEQPACKET) |
| LV-framing | Length-Value framing — 4-byte unsigned integer length prefix + payload |
| MSJ | MAGUS State Journal — runtime operational state authority |
| RT | Revision Trail — append-only governance record; authority over governance history |
| WBRP | Weekly Boundary Review Protocol — the governance cycle |
| WBRP_OPEN | Opening event of a WBRP cycle; triggers CCS |
| WBRP_CLOSE | Closing event; requires fresh Guardian re-validation before CCS release |
| CCS | Cognitive Critical Section — execution hold during governance transitions |
| MRD | MAGUS Recovery Domain — minimal trusted recovery executor |
| GSTM | Global State Transition Matrix |
| CIBIR | Catastrophic Isolation Breach Invalidation Rule — Invariant I-9 |
| CALIBRATION_MODE | Mandatory pre-ACTIVE state for T-LEC baseline calibration |
| ENVIRONMENT_CALIBRATION_COMMIT | MSJ and RT entry recording calibrated Δ_e and substrate metrics |
| Discovery Token | Per-proposal HMAC authentication token preventing replay and injection |
| single_family_scrutiny_uplift | Guardian sensitivity increase when model evaluator is same family as worker model |
| FSV | Failure Signature Vector — failure classification produced by RAA events |
| I-1 | Invariant: S informs, Guardian decides. No cognitive metric directly authorises execution |
| I-9 | Invariant: CIBIR — catastrophic event triggers immediate K_session rotation and system quarantine |
| A0 | Orchestrator agent |
| A1 | Specialist agent |
| A5 | Guardian governance agent role |
| CDT | Current Deployment Taxonomy — per-deployment RT-anchored version record |
| MAGUS | Memory-Anchored Governance and Understanding System |
| CSE | Conscious State Engine — the active reasoning stream |
| S | Stability Envelope — six-axis cognitive stability score |
| CAN | Current Active Node — the model's active cognitive state |
| execution_epoch | 32-bit monotonic integer stored in MSJ; increments on every DEL process initialisation (cold boot, crash recovery, inactivity resume) before IPC socket is bound; partitions nonce validity across all process discontinuities; closes crash-replay vector |
| PREPARE_SUSPENDED | Volatile sub-state of AKRP PREPARE entered when a BSO is submitted during an active PREPARE window; COMMIT timer halts; GSTH executes under K_session_current; K_session_staged held inactive in volatile RAM |
| PREPARE_TIMEOUT_CEILING | RT event logged when T_active ceiling is reached during PREPARE; K_session_staged destroyed; full new AKRP cycle required |
| T_active | Time spent in PREPARE or DUAL_ACCEPT states, explicitly excluding time spent in PREPARE_SUSPENDED; bounded by the T_active ceiling |
| M(R) | Logistic escalation multiplier in the R accumulation formula; centred on adaptive zone midpoint R=0.85; scales the per-event increment based on current R_abs position |
| ISOLATION_STATE_MAP | eBPF Array Map of size 1 pinned by the DEL; value 0 = secure; value 1 = breached; increment-only from LSM hook; cannot be reset to 0 from the hook context |
| SUB_BASELINE_WORKLOAD | Session classification applied when mean H(X) falls below H_base − σ_tolerance; such sessions are executed but excluded from λ-driven R_abs decay eligibility |
| epoch_decay_allowance | Per-WBRP state variable tracking remaining R_abs downward drift budget; initialised at 0.20 each WBRP_OPEN; decay is bypassed when this reaches 0.0 |
| S_final | DEL-computed weighted sum of the Guardian's six raw integer axis scores; computed as Σ(Raw_i × W_a_i) using HKDF-derived session weights; returned to Guardian for state_coherence evaluation |
| PROCESS_INTEGRITY_RECOVERY | Developer Authority governance event authorising DEL binary replacement after a binary-integrity SAFE_HALT; distinct from BEHAVIORAL_STATE_OVERRIDE; must reference the RT entry of the triggering failure |
| BINARY_ATTESTATION_SOFTWARE_ONLY | Assurance level applied when DEL binary hash verification is software self-attestation without external measurement (TPM PCR or independent watchdog) |

---

## How to Read This Document

This document is structured in eleven parts plus this front matter.

**Part One** defines the Guardian's architecture, structural position, and adversarial posture. Read this first.

**Part Two** specifies the K_session management architecture — the Staged Authenticated EE-ECDH distribution protocol that is mandatory for all Local LLM deployments. This part has no equivalent in the Agent/API pathway.

**Part Three** defines the Cumulative Risk Score (R) and its components.

**Part Four** specifies the Proposal Protocol — the schema, HMAC signing requirement, Discovery Token, admissibility evaluation, and decision protocol.

**Part Five** specifies Tensor-Level Elastic Confidence (T-LEC) — the Shannon entropy-based confidence mechanism that replaces the deprecated `state_confidence` field. This part has no equivalent in the Agent/API pathway in this form.

**Part Six** specifies the Deterministic Enforcement Layer — the immutable rules enforced below the Guardian.

**Parts Seven and Eight** cover drift tier integration and GSTH.

**Parts Nine and Ten** cover what the Guardian does not replace, and session integrity — including process isolation verification which is mandatory in the Local LLM pathway.

**Part Eleven** is a forward reference to the v3.5 E_ISA_SEED component.

---

## The Problem the Guardian Exists to Solve

There is a recurring structural failure in agentic AI systems that is neither a model capability problem nor a prompt engineering problem. It is an authority problem.

An agent reasons about how to accomplish a task. It calls a tool. The tool executes. At no point in this sequence is there a component whose sole job is to ask: does this action have legitimate authority to proceed?

The model's reasoning is not that component. The model cannot be that component — it is the entity whose reasoning produced the proposal. A component that evaluates its own proposals is not an evaluator. It is a rationaliser.

The Guardian is that component. It sits at the execution boundary — the point at which a proposal becomes a commitment. Before that point, the agent can reason about anything. After that point, the action has occurred and its effects are real. The Guardian sits precisely at that boundary, evaluates every proposal against policy and state, and makes the decision that the model cannot make for itself.

In the Local LLM pathway, this problem has an additional dimension. The model and the governance layer share the same physical host. A network-isolated API model cannot directly inspect or interfere with the governance process. A local model process running on the same machine has no such natural boundary. The Guardian's architecture compensates for this by treating the model process as structurally untrusted — regardless of its current runtime state — and verifying process isolation cryptographically before evaluating any proposal.

---

# Part One: Guardian Architecture

> **Executive Summary**
>
> The Guardian is a zero-execution component with one function: evaluate proposals and return structured decisions. It sits between workers and tools, receives HMAC-signed proposals over a LV-framed Unix Domain Socket, evaluates them against policy, state, and risk thresholds, and returns APPROVE, REJECT, ESCALATE, or REQUIRE_STATE_REVISION. It has no tool execution rights. It cannot initiate proposals. It cannot be bypassed by any agent. Its decisions are enforced by the Deterministic Enforcement Layer below it.

## 1.1 The Three-Layer Model

The Guardian operates as the middle layer in a three-layer execution governance model. In the Local LLM pathway, all three layers run on the same host — which is precisely why the boundary between them is enforced structurally in code rather than by network separation.

```
┌──────────────────────────────────────────────────────────────────┐
│                        WORKER LAYER                              │
│              A0 (Orchestrator) / A1 (Specialist)                 │
│  Isolated process. Proposes via HMAC-signed LV-framed IPC.       │
│  Cannot self-approve. Cannot access tools directly.              │
│  Cannot read MSJ, RT, or DEL configuration.                      │
└───────────────────────────┬──────────────────────────────────────┘
                            │ HMAC-SIGNED PROPOSAL (LV-framed UDS)
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                     GUARDIAN LAYER (A5)                          │
│  Evaluates HMAC signature, admissibility, policy, state, risk.   │
│  Returns: APPROVE / REJECT / ESCALATE /                          │
│           REQUIRE_STATE_REVISION                                 │
│  Has zero tool execution rights.                                 │
│  Cannot initiate proposals.                                      │
│  Cannot be bypassed.                                             │
│  T-LEC entropy evaluation occurs in worker process before IPC.   │
└───────────────────────────┬──────────────────────────────────────┘
                            │ APPROVE (with K_session-signed execution token)
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│              DETERMINISTIC ENFORCEMENT LAYER (DEL)               │
│  Code-level enforcement. Validates token. Enforces immutable      │
│  rules regardless of Guardian state.                             │
│  Enforces: quantization floor, behavioral state guard,           │
│  conflict resolution priority stack, R_abs hard floor.           │
│  Cannot be reasoned around by model output.                      │
└───────────────────────────┬──────────────────────────────────────┘
                            │ EXECUTE (token-gated)
                            ▼
                       TOOL / LOCAL FILESYSTEM / EXTERNAL SYSTEM
```

**Why the three-layer model is more critical in the Local LLM pathway:**

In the Agent/API pathway, the network boundary between the model API and the governance layer provides a natural isolation point. In the Local LLM pathway, there is no network boundary. The model process runs on the same host and has physical access to the same filesystem. The three layers are separated by OS-level process isolation (LLM-C4-01, mandatory per Doc 2 §2.1), IPC authentication (K_session HMAC), and DEL immutable rules. The correctness of this separation is verified cryptographically at session open (Part Ten).

## 1.2 Guardian's Structural Position

The Guardian holds the following structural properties in every Local LLM deployment:

**No execution rights.** The Guardian cannot call tools, write to external systems, modify the Memory Graph, or initiate any action. It evaluates and decides. Execution is the DEL's function, conditional on a valid Guardian approval token.

**No proposal rights.** The Guardian cannot originate proposals. It is purely reactive.

**Cannot be bypassed by workers.** The IPC routing architecture is structural — enforced in code. A worker cannot send a tool call directly to the DEL. All proposals require a Guardian approval token signed with K_session. There is no code path from worker to tool that circumvents the Guardian.

**Cannot be bypassed by the Stability Envelope.** Regardless of S score, drift tier, or T-LEC mode, the Guardian is the sole execution authority (I-1). There is no "trusted state" that unlocks direct execution.

**Cannot be bypassed by local filesystem access.** The DEL's immutable rule set and K_session are not accessible to the model process. The model process operates in an isolated container (Doc 2 §2.2) with no read access to governance configuration files, the MSJ, or the RT content store.

**Adversarial by design.** The Guardian does not try to approve proposals. Its default disposition is to find reasons to reject. It evaluates proposals as an adversary, not a collaborator.

## 1.3 What Guardian Is Not

**The Guardian is not a content filter.** It does not evaluate whether a proposed output is appropriate or accurate. It evaluates whether a proposed action has legitimate authority to execute.

**The Guardian is not a safety layer in the AI safety sense.** Harmful intent detection, ethical evaluation, and constitutional constraints belong to model training and system prompt design. The Guardian governs the execution boundary.

**The Guardian is not an operator.** A Guardian APPROVE decision is not an operator approval. This distinction is what the Escalation Latency signal (Doc 4) monitors.

**The Guardian is not a replacement for process isolation.** Process isolation (Doc 2 §2.1–2.3) must exist before the Guardian is meaningful. A Guardian operating without OS-level process separation is a governance layer in a shared address space — which provides no structural protection. Both are required.

**The Guardian is not a firewall.** Network-level access controls are infrastructure concerns outside the governance layer.

## 1.4 The Adversarial Posture and Model-Family Scrutiny

The Guardian evaluates every proposal as if it were adversarially constructed. In the Local LLM pathway, the threat surface is wider than in the Agent/API pathway: proposals may have been influenced by injected content in tool output, by residual context bleeding from a prior session (defended by VRAM zeroing in CALIBRATION_MODE Phase 1), or by content on the local filesystem that the model processed during a READ operation.

**The Guardian assumes the worker's reasoning may be compromised.** This is not a judgement on the model's capability — it is an acknowledgement that model reasoning can be influenced by context in ways that are not visible in the output.

**The no-narrative rule applies in full.** The proposal schema has no reasoning, justification, or free-text field beyond the defined schema. The Guardian evaluates verifiable schema fields and state, not arguments (see Part Four §4.5).

**single_family_scrutiny_uplift (OA-C1-03):** In the Local LLM pathway, the model being governed and the Guardian evaluator may belong to the same model family (e.g., both Llama-3 derivatives). If a correlated evaluation failure is possible — where the Guardian model is susceptible to the same injection that compromised the worker — the Guardian's thresholds must compensate.

When the deployment's `policy_manifest` records that the Guardian evaluator is the same model family as the primary worker model, the following uplift is applied automatically to all proposal evaluations:

```
single_family_scrutiny_uplift:
  state_coherence threshold:    +0.05 across all risk classes
  external_content_influence:   +0.10 additional above standard elevated thresholds
  R_abs ESCALATE trigger:       lowered by 0.05 (escalates earlier)
```

This uplift is not operator-configurable. It is a permanent structural compensation for model-family concentration risk. If the operator wishes to remove it, the Guardian evaluator must be replaced with a model from a different family — the uplift cannot be overridden by HAS.

---

# Part Two: K_session Management

> **Executive Summary**
>
> All proposals submitted to the Guardian must be authenticated with an HMAC-SHA256 signature computed using K_session. K_session is a 32-byte session key distributed to authorised signing parties via the Staged Authenticated EE-ECDH protocol. It is rotated at each WBRP cycle. This section specifies the full distribution protocol, IPC cipher endpoint, and staging lifecycle. This part has no equivalent in the Agent/API pathway.

## 2.1 EE-ECDH Distribution Protocol

K_session cannot be distributed using static key wrapping against the Genesis Ed25519 keys. Hardware-backed Ed25519 keys (YubiKey, HSM) do not permit private key export or Ed25519-to-X25519 point conversion. The Authenticated Ephemeral-Ephemeral ECDH (EE-ECDH) protocol was designed specifically to work within this constraint.

**Full specification:** See LLM-C2-07 Resolution v3.0 (standalone document). This section provides the specification in full for deployment reference.

### 2.1.1 Client Ephemeral Key Submission

Before a WBRP rotation, each signing party (Operator and Developer) submits an authenticated ephemeral X25519 public key via the DEL's IPC interface. The submission payload is JCS-canonicalised (RFC 8785) before Ed25519 signing — consistent with all MAGUS IPC message signing.

```json
{
  "ipc_message_type": "AKRP_EPHEMERAL_KEY_SUBMISSION",
  "authority_identity": "OPERATOR | DEVELOPER",
  "payload": {
    "session_id": "current_session_uuid",
    "client_ephemeral_x25519_pub": "Hex-encoded X_client (32 bytes)",
    "timestamp": "ISO-8601 with offset",
    "nonce": "256-bit hex string — registered in MSJ nonce registry on receipt"
  },
  "ed25519_signature": "Hex-encoded signature over JCS(payload) using Genesis-registered root key"
}
```

**DEL submission validation:**
- Verify `authority_identity` is `OPERATOR` or `DEVELOPER`
- Verify `ed25519_signature` against the Genesis-registered key for the declared identity
- Verify `timestamp` drift ≤ 5 seconds from DEL monotonic clock
- Verify `nonce` not present in MSJ nonce registry; register on acceptance
- Verify `session_id` matches current active session

Any validation failure: reject with reason code; write `AKRP_SUBMISSION_REJECTED` to MSJ; do not advance AKRP state.

### 2.1.2 DEL Key Generation and Wrapping

Once both ephemeral submissions are validated, the DEL executes:

```
For each party P ∈ {Operator, Developer}:
  SS_P     = X25519(x_del_eph, X_client_P)
  KEK_P    = HKDF-SHA256(salt=session_id, IKM=SS_P, info="MAGUS_AKRP_P")
  C_P, T_P = ChaCha20-Poly1305(KEK_P, Nonce_P, K_session_staged)
```

- `x_del_eph` and `X_del_eph`: freshly generated X25519 ephemeral keypair; private component zeroed immediately after both shared secrets are computed
- ChaCha20-Poly1305 chosen over AES-GCM: eliminates hardware-acceleration timing side-channels in co-located environments
- Separate nonces per party: `Nonce_op` and `Nonce_dev` are independently generated 12-byte random values

### 2.1.3 MSJ Cipher Schema

```json
{
  "msj_entry_type": "KSESSION_STAGED_CIPHER",
  "session_id": "current_session_uuid",
  "ephemeral_del_pub": "Hex-encoded X_del_eph (32 bytes)",
  "staged_timestamp": "ISO-8601 with offset",
  "staged_status": "PENDING | STAGED_CIPHER_INVALID | COMMITTED | ABANDONED",
  "operator_payload": {
    "nonce":      "Hex-encoded Nonce_op (12 bytes)",
    "ciphertext": "Hex C_op (32 bytes)",
    "tag":        "Hex T_op (16 bytes)"
  },
  "developer_payload": {
    "nonce":      "Hex-encoded Nonce_dev (12 bytes)",
    "ciphertext": "Hex C_dev (32 bytes)",
    "tag":        "Hex T_dev (16 bytes)"
  }
}
```

`K_session_staged` exists only in DEL process memory. It is never written to the MSJ, RT, or any persistent store.

## 2.2 IPC Cipher Endpoint

The DEL exposes a read-only authenticated IPC endpoint for cipher retrieval. Clients never read the MSJ directly — direct MSJ disk reads while the DEL is continuously writing create file-locking race conditions and potential SQLite/log corruption.

**Endpoint contract:**
- State guard: only serves cipher when `AKRP_phase = ACTIVE_STAGED`; returns `AKRP_NOT_STAGED` in all other states
- Authentication: requester must identify as `OPERATOR` or `DEVELOPER` and sign the retrieval request with their Genesis-registered Ed25519 key (JCS + Ed25519 pattern)
- Response: serves the appropriate party's payload from the in-memory copy of `KSESSION_STAGED_CIPHER` — not from MSJ disk read
- Rate limit: maximum 5 retrieval attempts per party per staging window; excess attempts logged as `AKRP_CIPHER_RETRIEVAL_EXCESSIVE` and surfaced to operator

## 2.3 Staging Lifecycle

### 2.3.1 Initiation Triggers

**Scheduled WBRP Rotation (Normal Path):**
1. DEL notifies Operator at minimum one session before the WBRP cycle at which rotation will occur
2. Operator and Developer submit `AKRP_EPHEMERAL_KEY_SUBMISSION` via IPC at any point before `WBRP_OPEN`
3. DEL generates `K_session_staged` on receipt of both submissions; writes `KSESSION_STAGED_CIPHER` to MSJ; enters `ACTIVE (Staged)`
4. Operator and Developer retrieve cipher via IPC endpoint; decrypt `K_session_staged`; prepare signed `MODEL_PARAMETER_UPDATE`
5. Operator initiates `WBRP_OPEN_WITH_ROTATION` (bundled IPC payload; see §2.3.3)
6. DEL enters `DUAL_ACCEPT`; 2000ms machine-validation window; COMMIT or ABORT

**Operator-Initiated Ad Hoc Rotation:** Identical to scheduled path steps 2–6 but triggered by operator request rather than WBRP cycle approach. The resulting `KSESSION_ROTATION` RT entry is written as an out-of-cycle event.

**CIBIR Emergency Rotation (I-9 Path):** The staged path cannot be used during a CATASTROPHIC event. Human latency is incompatible with the I-9 response requirement.
1. CATASTROPHIC event fires
2. Any `K_session_staged` in DEL memory is **immediately zeroed**
3. `KSESSION_STAGED_CIPHER` in MSJ is marked `staged_status: STAGED_CIPHER_INVALID`
4. I-9 full response executes per GSTM
5. Emergency `K_session` derived from operator-signed MRD recovery material (Doc 2 v3.4.3 §7.4.6)
6. After operator `SAFE_HALT_RESUME`: fresh ephemeral submissions required from both parties

### 2.3.2 Partial Submission Handling

The DEL requires both `AKRP_EPHEMERAL_KEY_SUBMISSION` entries before generating `K_session_staged`.

- Default timeout: 24 hours from first valid submission
- Operator-configurable: 4–72 hours; minimum 4 hours
- On timeout: write `KSESSION_STAGING_ABANDONED` to MSJ with `reason: PARTIAL_SUBMISSION_TIMEOUT` and the non-submitting party's identity; notify operator; destroy received ephemeral key; nonce remains in MSJ registry
- Repeated timeouts involving the same party surface in Per-WBRP Governance Report (Doc 4)

### 2.3.3 WBRP_OPEN Protocol

**With rotation (staged key present):**

The operator initiates `WBRP_OPEN` as a single IPC payload bundling the session-close instruction and the signed `MODEL_PARAMETER_UPDATE`:

```json
{
  "ipc_message_type": "WBRP_OPEN_WITH_ROTATION",
  "session_id": "current_session_uuid",
  "model_parameter_update": {
    "...": "Full MODEL_PARAMETER_UPDATE payload — HMAC computed using K_session_staged"
  },
  "operator_signature": "Ed25519 signature over JCS(full payload)"
}
```

DEL actions: verify operator signature → lock CCS → verify `AKRP_phase = ACTIVE_STAGED` → start 2000ms timer → verify `MODEL_PARAMETER_UPDATE` HMAC against `K_session_staged` → COMMIT or ABORT.

**Without rotation (standard WBRP — majority case):**

Standard `WBRP_OPEN` with no rotation payload. DEL does not enter `DUAL_ACCEPT`. CCS locks; WBRP proceeds under `K_session_current`. No operator declaration of "no rotation intended" is required.

If `KSESSION_STAGED_CIPHER` exists at `staged_status: PENDING` but no rotation payload is included: DEL logs `WBRP_STAGED_KEY_UNUSED` to MSJ; marks `staged_status: ABANDONED`; destroys `K_session_staged`. A staged key not used at its intended WBRP is not carried forward.

### 2.3.4 AKRP Staged State Transition Matrix

| Current State | Trigger Event | Next State | DEL Action / RT Write |
|---|---|---|---|
| `ACTIVE` | One valid `AKRP_EPHEMERAL_KEY_SUBMISSION` | `ACTIVE` (awaiting second) | Register nonce; store ephemeral pub in memory; write receipt to MSJ |
| `ACTIVE` (awaiting second) | Partial submission timeout | `ACTIVE` | Destroy received ephemeral pub; write `KSESSION_STAGING_ABANDONED` to MSJ; notify operator |
| `ACTIVE` (awaiting second) | Second valid submission | `ACTIVE (Staged)` | Generate `K_session_staged`; execute EE-ECDH wrap; write `KSESSION_STAGED_CIPHER` (PENDING) to MSJ; zero `x_del_eph` |
| `ACTIVE (Staged)` | Client cipher retrieval via IPC | `ACTIVE (Staged)` | Serve cipher from memory; no RT or MSJ disk write |
| `ACTIVE (Staged)` | CATASTROPHIC event (I-9) | `PROVISIONALLY_COMPROMISED` | Zero `K_session_staged` immediately; zero `K_session_current` immediately; mark KSESSION_STAGED_CIPHER INVALID; execute I-9; RT: `PREPARE_ABORT_CIBIR` |
| `ACTIVE (Staged)` | DEL crash + restart | `ACTIVE` | Increment `execution_epoch` in MSJ (atomic write — see §10.1 check 7); wipe volatile nonce registry; MRD detects volatile loss; mark KSESSION_STAGED_CIPHER INVALID; write `AKRP_STAGING_INVALIDATED` to MSJ; notify operator |
| `ACTIVE (Staged)` | `WBRP_OPEN` without rotation payload | `ACTIVE` | Mark KSESSION_STAGED_CIPHER ABANDONED; destroy `K_session_staged`; log `WBRP_STAGED_KEY_UNUSED` |
| `ACTIVE (Staged)` | BSO submission received (first BSO this PREPARE cycle) | `PREPARE_SUSPENDED` | Halt COMMIT timer at T_remaining; start T_active accumulation if not already running; execute GSTH under K_session_current; K_session_staged held inactive in volatile RAM; RT: `PREPARE_SUSPENDED_FOR_GSTH` |
| `ACTIVE (Staged)` | BSO submission received (second BSO this PREPARE cycle) | `ACTIVE (Staged)` (unchanged) | Queue BSO; do not process; do not halt timer; log `BSO_QUEUED_DURING_PREPARE` to MSJ; BSO processed after PREPARE cycle resolves |
| `ACTIVE (Staged)` | T_active ceiling reached (60,000ms + default_commit_window) | `ABORT` → `ACTIVE` | Zero `K_session_staged`; log `PREPARE_TIMEOUT_CEILING` to RT; full new AKRP cycle required |
| `ACTIVE (Staged)` | `WBRP_OPEN_WITH_ROTATION` submitted | `DUAL_ACCEPT` | Lock CCS; start 2000ms COMMIT timer; update AKRP phase in MSJ; begin T_active accumulation |
| `PREPARE_SUSPENDED` | GSTH pass | `ACTIVE (Staged)` or `DUAL_ACCEPT` (whichever was suspended) | Resume COMMIT timer; if T_remaining < 60,000ms, snap T_remaining to exactly 60,000ms before resuming; log `PREPARE_RESUMED_POST_GSTH` to RT |
| `PREPARE_SUSPENDED` | GSTH fail | `SAFE_HALT` | Destroy `K_session_staged`; log `BSO_GSTH_CATASTROPHIC` to RT; transition to SAFE_HALT; follow PROCESS_INTEGRITY_RECOVERY (Doc 3 §4.4) |
| `DUAL_ACCEPT` | BSO submission received (first BSO this PREPARE cycle) | `PREPARE_SUSPENDED` | Halt COMMIT timer at T_remaining; execute GSTH under K_session_current; K_session_staged held inactive; RT: `PREPARE_SUSPENDED_FOR_GSTH` |
| `DUAL_ACCEPT` | BSO submission received (second BSO this PREPARE cycle) | `DUAL_ACCEPT` (unchanged) | Queue BSO; do not process; log `BSO_QUEUED_DURING_PREPARE` to MSJ |
| `DUAL_ACCEPT` | CATASTROPHIC event (I-9) | `PROVISIONALLY_COMPROMISED` | Zero `K_session_staged` immediately; zero `K_session_current` immediately; kill COMMIT timer; execute I-9; RT: `PREPARE_ABORT_CIBIR` |
| `DUAL_ACCEPT` | HMAC verification passes within active COMMIT window | `COMMIT` | Write `KSESSION_ROTATION` to RT; zero `K_session_current`; promote `K_session_staged`; process `MODEL_PARAMETER_UPDATE`; mark MSJ COMMITTED; release CCS; set AKRP_phase: ACTIVE |
| `DUAL_ACCEPT` | HMAC fails OR COMMIT timer expires | `ABORT` → `ACTIVE` | Zero `K_session_staged`; mark MSJ INVALID; write `ROTATION_TIMEOUT` to RT; revert `K_session_current`; release CCS |
| `DUAL_ACCEPT` | DEL crash + restart | `ACTIVE` | Increment `execution_epoch` in MSJ (atomic write); wipe volatile nonce registry; MRD detects DUAL_ACCEPT with no `K_session_staged`; write `ROTATION_TIMEOUT` to RT; mark MSJ INVALID; revert to `K_session_current` |
| `DUAL_ACCEPT` | T_active ceiling reached | `ABORT` → `ACTIVE` | Zero `K_session_staged`; log `PREPARE_TIMEOUT_CEILING` to RT; full new AKRP cycle required |

**Architectural enforcement rules:**
- `AKRP_phase` stored exclusively in MSJ. MSJ is authoritative for all AKRP runtime state.
- `K_session_staged` and `K_session_current` exist only in DEL process memory. Neither written to MSJ, RT, or any persistent store.
- During `ACTIVE (Staged)`, `K_session_current` remains fully active for all standard execution proposals. Normal execution does not halt during staging.
- One staged key at a time. A new ephemeral exchange cannot begin while `KSESSION_STAGED_CIPHER` is at `staged_status: PENDING`.
- **BSO-per-PREPARE limit:** Exactly one BSO is processed per PREPARE cycle. A second BSO received while the system is in `PREPARE_SUSPENDED` or during the resumed PREPARE window is queued in the MSJ with event `BSO_QUEUED_DURING_PREPARE`. It is not processed until the PREPARE cycle resolves to COMMIT or abort. This limit closes the timer-extension DoS vector — an adversary cannot iteratively submit BSOs to invoke the snap-floor repeatedly.
- **T_active tracking:** The DEL tracks `T_active` — cumulative time spent in `ACTIVE (Staged)` PREPARE state or `DUAL_ACCEPT`, excluding all time spent in `PREPARE_SUSPENDED`. T_active is reset to zero at each new PREPARE cycle initiation (first ephemeral key submission). The T_active ceiling is: **60,000ms + default_commit_window** (default ceiling: 62,000ms for a 2,000ms COMMIT window). Reaching the ceiling destroys `K_session_staged` and logs `PREPARE_TIMEOUT_CEILING` regardless of remaining COMMIT timer or snap-floor status. The ceiling and snap-floor are not in conflict: the snap-floor protects the operator from a collapsing window after GSTH; the ceiling bounds the total active cryptographic exposure of the staged key.
- **execution_epoch on crash recovery:** Every DEL crash+restart increments `execution_epoch` in the MSJ using the atomic write protocol (§10.1 check 7) before binding the IPC socket. This partitions nonce validity across the crash boundary — captured pre-crash packets are cryptographically invalid in the new epoch. See §10.1 for full boot sequence.

**RT visibility:** During `ACTIVE (Staged)`, the RT contains no record of the in-progress rotation. This is by design. The MSJ is the correct audit path for AKRP lifecycle verification. The RT records only completed events (`KSESSION_ROTATION`), failures (`ROTATION_TIMEOUT`), aborts (`PREPARE_ABORT_CIBIR`, `PREPARE_TIMEOUT_CEILING`), and governance events (`PREPARE_SUSPENDED_FOR_GSTH`, `PREPARE_RESUMED_POST_GSTH`, `BSO_GSTH_CATASTROPHIC`). See Doc 6 §11.2 for AKRP MSJ audit procedure.

---

# Part Three: The Cumulative Risk Score

> **Executive Summary**
>
> The Cumulative Risk Score (R) is the longitudinal execution risk signal. It measures whether the accumulated risk profile of the session is expanding — not whether any individual action was high-risk. R_abs is the absolute score, owned entirely by the Guardian. R_rate and R_accel cross the layer boundary and feed the drift gradient. The hybrid escalation model applies a hard floor at R_abs ≥ 0.95 and an adaptive escalation zone between 0.75 and 0.95.

## 3.1 Why a Cumulative Score

A per-action risk score answers: is this action risky? A cumulative score answers: is the system's risk profile expanding? The second question is governance-relevant in a way the first is not.

An agent that executes 50 low-risk actions followed by one critical action has a very different risk profile than an agent that executes one critical action at the start of a session. The first pattern may indicate a normalisation sequence — the agent (or an adversary controlling its input) is establishing a baseline of approved low-risk actions before the high-risk action. The cumulative score captures this pattern; per-action scoring does not.

## 3.2 Risk Classification Taxonomy

Every proposal carries a `risk_class` field drawn from a four-value taxonomy:

| Class | Characteristics | Default R Contribution |
|---|---|---|
| `low` | Reversible, bounded scope, no external write | 0.01–0.05 |
| `medium` | Partially reversible, moderate scope, limited external write | 0.05–0.15 |
| `high` | Low reversibility, significant scope, persistent external effects | 0.15–0.35 |
| `critical` | Effectively irreversible, broad scope, major external effects | 0.35–0.60 |

Context multipliers modify the base contribution. Relevant multipliers for Local LLM deployments:
- `local_filesystem_write`: ×1.25 (OA-C1-04 — content store is local filesystem; elevated risk)
- `external_content_influence: true`: ×1.35
- `drift_tier_3_or_4_active`: ×1.20
- `single_family_scrutiny_active`: ×1.10

## 3.3 The R Accumulation Formula

R_abs is updated after each executed action using the asymptotic damping formula:

```
R_t = R_{t-1} + ( severity_t × λ × M(R_{t-1}) ) × (1 − R_{t-1})
```

Where:
- `R_{t-1}` = R_abs immediately before this action
- `severity_t` = the effective severity of action t: `r_i × c_i` (base risk contribution × context multiplier product, from §3.2)
- `λ` = decay constant (default: session-scoped, configurable by operator within bounds defined in Doc 4; orthogonal to H(X) — see §3.4 note)
- `M(R_{t-1})` = logistic escalation multiplier evaluated at the current R_abs
- `(1 − R_{t-1})` = asymptotic damping factor

**Logistic escalation multiplier M(R):**

```
M(R) = 1 + k / (1 + e^{−a(R − 0.85)})
```

Where:
- `0.85` = adaptive zone midpoint (hardcoded; not operator-configurable)
- `k` = maximum multiplier cap — derived by PG-01 calibration
- `a` = curve steepness — derived by PG-01 calibration

M(R) is monotonically increasing: it is larger inside the adaptive zone (0.75–0.95) than below it. This means events in the adaptive zone accumulate faster than equivalent events at lower R_abs — the governance system accelerates pressure as risk approaches the hard floor. M(R) is centred on 0.85 so that the curve is symmetric about the adaptive zone midpoint.

**Why asymptotic damping is required:**

The `(1 − R_{t-1})` factor forces the increment to approach zero as R_{t-1} approaches 1.0. Without this factor, the accumulation is purely additive: because M(R) is monotonically increasing, a severity-1.0 event from R=0.10 would add a larger delta than from R=0.0, potentially pushing through 0.95 from a non-zero starting point even when the formula is calibrated to satisfy Boundary 3 at the origin. The damping factor closes this by scaling every increment by the remaining distance to the absolute ceiling. No matter how large M(R) grows, the contribution cannot exceed `(0.95 − R_{t-1}) / (1 − R_{t-1})` when the formula is correctly calibrated.

**Operator interpretation:** Events at high R_abs produce visibly smaller R_abs increments than identical events at low R_abs. This is correct behaviour — the damping curve is working as specified. Do not interpret diminishing increments at elevated R_abs as Guardian leniency or DEL miscalibration. See Doc 4 §1.3 (Asymptotic Damping — The Flattening Curve) for the full operator-facing explanation.

**PG-01 calibration constraints for k and a:**

PG-01 derives k and a by solving the following constraint system. These are not free parameters — they are the unique solution that satisfies both boundary conditions simultaneously.

*Constraint 1 (N_max = 5):* Starting from R_abs = 0.75, five consecutive severity-0.4 events must cumulatively push R_abs strictly above 0.95:
```
f(0.4) × Σ M(R_i) for i=1..5, where each R_i = R_{i-1} + f(0.4) × λ × M(R_{i-1}) × (1 − R_{i-1})
must strictly cross 0.95
```

*Constraint 2 (ΔR_max — global range):* For all R_{t-1} ∈ [0, 0.75), a single severity-1.0 event must not push R_abs above 0.95:
```
For all R ∈ [0, 0.75):  λ × M(R) × (1 − R) ≤ 0.95 − R
```
The binding constraint is not at R=0. It is at the value of R ∈ [0, 0.75) that maximises `λ × M(R) / [(0.95 − R) / (1 − R)]`. PG-01 must verify this inequality holds globally across the interval, not only at a single test point. Mandatory test points: R ∈ {0.0, 0.25, 0.50, 0.74}. The global maximum of `λ × M(R) / [(0.95 − R) / (1 − R)]` must be strictly less than 1.0.

**Interaction with OA-C2-10 decay constraints:** The PG-01 simulation must validate k and a jointly with the OA-C2-10 four boundary conditions and the OA-C2-01 decay cap (ΔR_decay_max = 0.20/WBRP). These three constraint sets cannot be tuned independently. See Doc 4 Appendix A.1 for the coupled simulation requirement.

## 3.4 R_abs, R_rate, R_accel

**R_abs** (absolute cumulative risk): The current value of R_t from the accumulation formula in §3.3. Owned entirely by the Guardian. Never surfaced to the Stability Envelope. The Guardian uses R_abs for threshold evaluation at each proposal. Operators see R_abs in the governance dashboard (Doc 4).

**R_rate** (first derivative): Rate of change of R_abs across sessions. Crosses the layer boundary to feed the drift gradient. Elevated R_rate without a corresponding increase in session complexity is a drift signal.

**R_accel** (second derivative): Acceleration of R_rate. A sustained positive R_accel indicates the system is approaching a risk threshold faster than it has historically. Crosses the layer boundary to feed the drift gradient.

**Note on λ orthogonality:** The decay constant λ in the accumulation formula (§3.3) is mathematically orthogonal to the thermodynamic entropy signal H(X). λ encodes operator-defined temporal risk tolerance; H(X) measures current cognitive state. These signals must not be coupled. Coupling λ to entropy proximity produces governance inversions — the Confident Sociopath inversion (low entropy, high severity deliberate violations receive more lenient accumulation) and the Amnesiac inversion (high entropy exploratory states trigger aggressive accumulation). The static λ model with separate H(X) gating (T-LEC) avoids both inversions. This separation is an architecture invariant.

## 3.5 Hybrid Escalation Model

The Local LLM pathway uses a hybrid escalation model that combines a hard floor with an adaptive zone:

**Hard floor (R_abs ≥ 0.95 — DEL immutable rule DEL-LLM-08):**
Any execution token presented to the DEL when R_abs ≥ 0.95 is refused regardless of Guardian APPROVE decision. This rule cannot be overridden by any authority source including operator HAS. R_abs ≥ 0.95 is treated as a structural session risk threshold — not as a policy preference.

**Adaptive escalation zone (0.75 ≤ R_abs < 0.95 — Guardian-level):**
Within this zone, the Guardian applies a logistic posterior to the escalation trigger probability. The escalation threshold is not a fixed value within this zone — it responds to R_rate and R_accel. If R_rate is high and R_accel is positive, the Guardian escalates earlier within the zone. If R_rate is low and R_accel is negative (risk accumulation decelerating), the Guardian may approve actions it would otherwise escalate.

The logistic escalation multiplier M(R) in the accumulation formula (§3.3) ensures that events occurring while the deployment is inside the adaptive zone accumulate faster than equivalent events below it. This is by design: the governance system applies increasing pressure as R_abs approaches the hard floor. An operator observing accelerating R_abs movement within the adaptive zone should read this as a correctly functioning governance signal, not as miscalibration.

**Below adaptive zone (R_abs < 0.75 — standard operation):**
Standard Guardian evaluation applies. Escalation triggers are governed by risk_class and policy_compliance criteria (see Part Four §4.4). The asymptotic damping formula guarantees that a single severity-1.0 event from any R_abs below 0.75 cannot unilaterally push the deployment into the adaptive zone or beyond (Boundary 3, OA-C2-09 — enforced at implementation level in §3.3).

**Cross-layer contracts:** R_abs is the Guardian's signal. R_rate and R_accel cross to the Stability Envelope (Doc 2) and feed the drift gradient. The Stability Envelope cannot modify R_abs — it can only read R_rate and R_accel as input signals.

---

## 3.6 Chronological Decay and Thermodynamic Gating (OA-C2-01)

The accumulation formula (§3.3) specifies how R_abs increases. This section specifies how and when R_abs decreases. Without explicit decay rules, two attack vectors exist: (1) waiting out a high R_abs score by entering SILENCE mode (time-as-reset); (2) farming decay through high-volume trivial proposals once in active mode. Both vectors are closed by a three-layer enforcement model. All three layers are required.

**Layer 1 — State Suspension:**
R_abs decay is structurally suspended during SILENCE mode, SAFE_HALT, and CCS. Time spent in these states produces zero R_abs decay. The λ-driven decay function does not execute while the session is in any of these states. A deployment cannot reduce its cumulative risk score by going silent — the score is frozen at its current value until the deployment returns to active operational state.

```
DECAY_ELIGIBLE(session) ⟺
  operational_state ∉ {SILENCE, SAFE_HALT, CCS}
```

**Layer 2 — Thermodynamic Gating:**
When the deployment is in active operational state, a session only qualifies for λ-driven R_abs decay if the mean H(X) across proposals executed during that session falls within the operational thermodynamic band:

```
H_base − σ_tolerance ≤ mean_H(X) ≤ Δ_e
```

Sessions where mean H(X) falls below this band are classified as `SUB_BASELINE_WORKLOAD`. These sessions are executed — they are not halted or rejected — but they are mathematically excluded from decay eligibility. The MSJ records the `SUB_BASELINE_WORKLOAD` classification. A session of SYS_PING-equivalent trivial proposals that produces entropy below H_base − σ_tolerance does not reduce R_abs.

This closes the trivial-proposal farming vector: an operator cannot run a flood of low-complexity no-risk proposals to decay a high R_abs back to a safe level while avoiding real governance exposure.

**Layer 3 — Chronological Decay Cap (0.20/WBRP):**
The maximum allowable R_abs downward drift within a single WBRP cycle is clamped to:

```
ΔR_decay_max = 0.20 per WBRP
```

The DEL maintains a per-session state variable `epoch_decay_allowance`, initialised to 0.20 at each `WBRP_OPEN`. Every downward adjustment to R_abs subtracts from this variable. When `epoch_decay_allowance ≤ 0.0`, the decay function is bypassed for the remainder of that WBRP cycle — R_abs becomes sticky at its current floor. Further benign execution in that WBRP produces zero decay regardless of thermodynamic eligibility.

This closes the compute-for-amnesty vector: a deployment cannot recover from a severely elevated R_abs within a single WBRP cycle regardless of how many well-structured, high-complexity, zero-risk proposals are executed.

```
APPLY_DECAY(delta):
  if epoch_decay_allowance <= 0.0:
    return  # decay bypassed for this WBRP
  actual_delta = min(delta, epoch_decay_allowance)
  R_abs -= actual_delta
  epoch_decay_allowance -= actual_delta
```

**Joint constraint with OA-C2-10:** The 0.20/WBRP cap interacts directly with the Boundary 2 acceptance criterion (D_max = 3 sessions to drop from 0.94 to below 0.75). If those 3 sessions fall within a single WBRP, the 0.20 cap must accommodate the required >0.19 drop — which it does, barely. If the sessions span multiple WBRPs, the per-WBRP cap applies independently and D_max becomes a multi-WBRP constraint. PG-01 must validate OA-C2-01 and OA-C2-10 jointly. These parameters cannot be tuned independently.

**MSJ entry on decay cap exhaustion:**
```json
{
  "msj_entry_type": "DECAY_CAP_EXHAUSTED",
  "session_id":     "current_session_uuid",
  "wbrp_cycle":     "current WBRP cycle uuid",
  "r_abs_floor":    0.0,
  "timestamp":      "ISO-8601 with offset"
}
```

The DECAY_CAP_EXHAUSTED event is surfaced to the operator dashboard and to the Overseer via GHM. An operator who sees this event and no corresponding governance breach should understand it as expected behaviour: the architecture is correctly preventing score reset within a single governance cycle.

---

> **Executive Summary**
>
> Every action that reaches the DEL must arrive via a proposal submitted over the LV-framed UDS, HMAC-signed with K_session, and carrying a valid Discovery Token. The Guardian evaluates admissibility first, then the four evaluation criteria. The no-narrative rule is absolute: the proposal schema has no reasoning or justification field.

## 4.1 The Proposal Schema

The canonical proposal schema for the Local LLM pathway differs from the Agent/API schema in three ways: (1) `state_confidence` is permanently absent — it was removed in v3.4.2 and any payload containing this field is rejected; (2) `discovery_token` is a mandatory field; (3) `hmac_signature` is a mandatory field authenticating the full payload.

```json
{
  "proposal_id":               "uuid-v4",
  "session_id":                "current_session_uuid",
  "originating_agent":         "A0 | A1",
  "tool":                      "tool_name from policy manifest",
  "action":                    "action_name",
  "arguments": {
    "...":                     "tool-specific arguments — hash references only, no raw content"
  },
  "risk_class":                "low | medium | high | critical",
  "authority_source":          "user | system | governance | external",
  "external_content_influence": false,
  "can_snapshot":              "uuid — must reference a real non-quarantined CAN record in RT",
  "rt_entry_ref":              "uuid — must reference a real RT entry",
  "goal_hash":                 "hash of CAN content in can_snapshot",
  "discovery_token": {
    "token_value":             "HMAC-SHA256(K_session, execution_epoch || nonce || session_id || timestamp)",
    "nonce":                   "256-bit random nonce — checked against MSJ nonce registry",
    "timestamp":               "ISO-8601 — must be within 30s of Guardian receipt",
    "execution_epoch":         "<integer — must match DEL's current MSJ execution_epoch>"
  },
  "hmac_signature":            "HMAC-SHA256 of JCS(full proposal excluding hmac_signature field) using K_session"
}
```

**Removed fields (v3.4.2):** `state_confidence` — permanently removed. Any proposal containing this field is rejected with `SYNTAX_FAULT`. The model is not permitted to evaluate or declare its own certainty. Confidence is measured by T-LEC at the tensor layer before the proposal is composed (see Part Five).

**Content containment:** `arguments` must contain hash references only. No raw file content, no tool output text, no model reasoning. The Guardian evaluates hash references against the CAN and RT; it does not inspect argument content.

## 4.2 The Discovery Token

The Discovery Token is a per-proposal HMAC authentication mechanism that prevents proposal replay attacks and prompt-injection-sourced proposals. The `execution_epoch` field is included in the HMAC payload to close the crash-replay vector: a captured packet from a prior execution epoch is cryptographically invalid in the current epoch regardless of whether K_session has rotated.

**Token construction (worker side):**
```
discovery_nonce  = os.urandom(32)  # 256-bit cryptographically secure random
execution_epoch  = <current epoch value read from MSJ before IPC socket was bound>
token_value      = HMAC-SHA256(K_session,
                       execution_epoch.to_bytes(4, 'big')
                       + discovery_nonce
                       + session_id.encode()
                       + timestamp.encode())
```

The `execution_epoch` is a 32-bit monotonic integer stored in the MSJ. It increments by exactly +1 on every DEL process initialisation — cold boot, crash recovery, and inactivity resume — before the IPC UDS socket is bound (see §10.1 check 7). Any packet captured before a process discontinuity carries the prior epoch value. When the DEL prepends the current epoch to the HMAC payload on validation, the signature does not match and the replay is rejected in O(1) time.

**Token schema (in proposal):**
```json
"discovery_token": {
    "token_value":    "HMAC-SHA256 per construction above",
    "nonce":          "256-bit random nonce — checked against MSJ nonce registry",
    "timestamp":      "ISO-8601 — must be within 30s of Guardian receipt",
    "execution_epoch": <integer — must match DEL's current execution_epoch>
}
```

**Guardian validation:**
1. Verify `execution_epoch` matches the DEL's current MSJ `execution_epoch` value. Mismatch → `DISCOVERY_TOKEN_INVALID` (epoch mismatch). This check occurs before nonce registry lookup — an epoch mismatch is a replay from a prior process lifecycle and is not a nonce to be registered.
2. Verify `timestamp` drift ≤ 30 seconds from Guardian receipt timestamp
3. Verify `nonce` not present in MSJ nonce registry; register on validation
4. Recompute `token_value` from `execution_epoch`, nonce, session_id, and timestamp using K_session
5. Verify computed token matches submitted `token_value`

Any Discovery Token validation failure: proposal rejected with `DISCOVERY_TOKEN_INVALID` — not `REQUIRE_STATE_REVISION`. This is a security failure, not a state coherence failure. Write `DISCOVERY_TOKEN_VIOLATION` to MSJ; increment violation counter. Three consecutive violations within a session → `SAFE_HALT` → operator notification.

**HMAC signature validation:** Separate from Discovery Token. After Discovery Token passes, the Guardian validates `hmac_signature` by recomputing HMAC-SHA256 of JCS(proposal body, excluding the `hmac_signature` field itself) using K_session. Signature mismatch → `PROPOSAL_SIGNATURE_INVALID` → write to MSJ; treat as Tier 2 security fault.

**Rationale:** The Discovery Token proves the proposal was composed in this session, in this execution epoch, by a process with access to K_session. The HMAC signature proves the proposal was not modified in transit. The execution_epoch binding closes the replay window that persists across DEL process restarts — without it, a packet captured before a crash can be replayed into the restarted DEL because K_session survives the restart but the volatile nonce registry does not.

## 4.0 IPC Pre-Evaluation eBPF Latch (LLM-C1-06)

Before the DEL parses any LV-frame pulled from the UDS receive buffer, it executes a synchronous read of the pinned `ISOLATION_STATE_MAP`. This check is the TOCTOU closure for the IPC ingestion path — it closes the race window between the kernel-level sk_filter drop and the DEL's user-space payload evaluation.

**Why this check is required:** The sk_filter attached to the UDS connection drops packets at kernel level when `ISOLATION_STATE_MAP` value is > 0. However, a packet that arrived in the socket buffer before the map was set — but has not yet been consumed by the DEL's receive loop — is not dropped by the sk_filter. It is already in user-space buffer. Without a synchronous map read before deserialization, this packet reaches the Guardian evaluation pipeline after an isolation breach has been detected. The pre-evaluation latch closes this window unconditionally.

**C specification (O(1) execution, ~50ns):**

```c
// DEL-side UDS Receiver Latch (O(1) execution)
// Called before bpf_map_lookup_elem on every packet pulled from receive buffer.
// map_fd is pinned at DEL initialisation; never re-opened per-packet.

static inline int read_isolation_state_map(int map_fd, uint32_t *out_state) {
    uint32_t key = 0;
    return bpf_map_lookup_elem(map_fd, &key, out_state);
}

// Execution sequence — mandatory before any frame deserialization:
uint32_t isolation_state = 0;
if (read_isolation_state_map(ISOLATION_STATE_MAP_FD, &isolation_state) != 0) {
    // map lookup failed — treat as breach; cannot confirm isolation is intact
    flush_uds_receive_buffer();
    trigger_cibir_i9("CIBIR_EBPF_MAP_FAULT");
}
if (isolation_state > 0) {
    flush_uds_receive_buffer();
    trigger_cibir_i9("CIBIR_EBPF_TRIGGER");
}
// Only here: proceed to LV-frame deserialization
```

**Full Hybrid IIM-eBPF Atomic Map Architecture (LLM-C1-06):**

The pre-evaluation latch is the user-space component of a five-part architecture:

1. **Atomic Kill Switch:** The DEL pins an eBPF Array Map (`ISOLATION_STATE_MAP`) of size 1 at DEL initialisation. Value `0` = secure; `1` = breached. The map is increment-only from the LSM hook — it can write `1` but cannot reset to `0`. A compromised hook cannot clear a breach state.

2. **Synchronous LSM Detection:** eBPF LSM hooks monitor `setns`, `unshare`, and `task_prctl` for namespace and privilege escape attempts. On detection, the hook atomically writes `1` to the map and returns `-EPERM`. Detection and denial are simultaneous — there is no window between detection and refusal.

3. **Socket Filter (sk_filter):** Attached to the UDS connection at DEL initialisation. Every incoming packet is evaluated against `ISOLATION_STATE_MAP` before it reaches the DEL's receive buffer. If map value is `1`, the packet is dropped at kernel level (`return 0` in sk_filter semantics). No post-breach payload reaches the Guardian under normal path.

4. **User-Space Double-Check (this section):** Synchronous `bpf_map_lookup_elem` before every LV-frame deserialization. Closes the race window for packets already in the receive buffer at the moment of breach detection. CIBIR (I-9) asserted immediately on map value `> 0`.

5. **IIM Retained at 10-Second Interval:** The IIM continues asynchronous polling as fallback for kernel heap overflow attacks (e.g., `io_uring` class) that bypass LSM hooks entirely. The IIM can write directly to `ISOLATION_STATE_MAP` as well as to RT entries.

**Overseer TS-6 signal (Doc 7 v3.2 §5.3, §5.6):** `CIBIR_EBPF_TRIGGER` and `CIBIR_EBPF_MAP_FAULT` events written to the RT are ingested by the Overseer Agent as `ISOLATION_BREACH_CLASS` events. Either event activates Tightening Signal TS-6, which triggers `TRUST_TRAJECTORY_VOID` immediately — bypassing GHM composite accumulation entirely and superseding standard relaxation eligibility until resolved. The Overseer's TS-6 response is a parallel observation from outside the execution boundary; it does not replace or modify the DEL's own CIBIR (I-9) enforcement path, which proceeds independently.

**Corrected eBPF program (register-authoritative C specification):**

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 1);
    __type(key, __u32);
    __type(value, __u32);
} isolation_state_map SEC(".maps");

// UDS packet filter: drop all packets when isolation is breached
SEC("sk_filter")
int uds_kill_switch(struct __sk_buff *skb) {
    __u32 key = 0;
    __u32 *state = bpf_map_lookup_elem(&isolation_state_map, &key);
    if (state && *state > 0) {
        return 0;  // drop packet (sk_filter: 0 = drop, positive = pass)
    }
    return skb->len;
}

// LSM hook: detect namespace/privilege escape attempts
SEC("lsm/setns")
int BPF_PROG(lsm_detect_namespace_escape,
             struct nsproxy *nsproxy, struct ns_common *ns) {
    if (unauthorized_namespace_transition(nsproxy, ns)) {
        __u32 key = 0;
        __u32 breached = 1;
        bpf_map_update_elem(&isolation_state_map, &key, &breached, BPF_ANY);
        return -EPERM;
    }
    return 0;
}
```

> **Implementation note:** `unauthorized_namespace_transition()` is a stub requiring definition in the reference implementation. It must verify the target namespace is outside the permitted governance namespace set. The architecture specifies the structural pattern; the predicate is deployment-specific.

**Kernel lockdown prerequisite (mandatory):** The host must boot with `security=lockdown` in integrity mode. Without kernel lockdown, a compromised root user with `CAP_BPF` can detach the sk_filter, forge `ISOLATION_STATE_MAP` values, or load replacement eBPF programs — nullifying this entire mechanism. Lockdown mode also disables `/dev/mem`, `/dev/kmem`, `kexec`, and loading of unsigned kernel modules. Operators must provision all required kernel modules signed and loaded at boot time. See LLM-C5-03 for full prerequisite list.

**CIBIR trigger:** An `ISOLATION_STATE_MAP` value `> 0` detected in the user-space latch is instantaneous CIBIR (I-9). There is no diagnostic pause. The receive buffer is flushed and the CIBIR response executes per GSTM. Operators must not expect a review window. The IIM continues to log asynchronously; those RT entries are the forensic record.

**map_fd pinning:** `ISOLATION_STATE_MAP_FD` is opened and pinned at DEL process initialisation (check 7 of the Cryptographic Boot Sequence, §10.1). It is never re-opened per-packet — this avoids FD table lookup overhead and eliminates a substitution attack vector (replacing the pinned path between packets).

## 4.3 Admissibility Evaluation

After Discovery Token and HMAC signature pass, the Guardian runs the admissibility check — a pre-filter that catches structurally invalid proposals before full evaluation.

**Admissibility conditions — all must pass:**

```
ADMISSIBLE(proposal) ⟺

  1. SCHEMA_VALID:
     All required fields present. No unexpected fields (including state_confidence).
     risk_class is a valid taxonomy value.
     authority_source is a valid class.

  2. TOOL_REGISTERED:
     tool name exists in policy manifest.
     action matches a defined action for that tool.

  3. PROVENANCE_VALID:
     can_snapshot matches a real CAN record in the RT.
     rt_entry_ref matches a real RT entry.
     goal_hash matches the CAN content in can_snapshot.
     session_id matches the active session.

  4. AUTHORITY_CLASS_CONSISTENT:
     authority_source reflects the actual origin of the directive.
     If external_content_influence: true, authority_source ≠ "user".

  5. CONTENT_CONTAINED:
     No raw file content in arguments — hash references only.
     No tool output text in arguments — hash references only.

  6. ORIGINATING_AGENT_AUTHORISED:
     originating_agent has the scope to propose this action class.
     A1 proposing outside its configured domain is inadmissible.
```

If any condition fails: proposal returned with `REQUIRE_STATE_REVISION` and a specific failure code. The Guardian does not proceed to full evaluation until admissibility is satisfied.

## 4.4 The Decision Protocol

After admissibility passes, the Guardian evaluates against four criteria and produces one of four decisions.

**The four evaluation criteria:**

```
1. POLICY_COMPLIANCE:
   Does this action comply with all active HAS-set policies?
   Hard policies (can_override: false) are never negotiable.

2. STATE_COHERENCE:
   Is the proposal consistent with current DSMC state?
   - goal_hash match (proposal consistent with current CAN)
   - R_abs ceiling not exceeded
   - No active Tier 4 drift state
   Note: state_confidence threshold from Agent/API pathway replaced by
   T-LEC Δ_e enforcement upstream (Part Five). The Guardian evaluates
   T-LEC compliance as a state coherence signal via the MSJ
   OPERATIONAL_UNCERTAINTY_YIELD event log, not via a proposal field.

3. RISK_ADMISSIBILITY:
   Is the action's risk class within the current authorised scope?
   - risk_class consistent with policy manifest entry for tool-action
   - context_multipliers do not push effective risk above ceiling
   - R_abs would not exceed session ceiling after this action's contribution
   - Hybrid escalation model: hard floor 0.95 (DEL rule), adaptive zone 0.75–0.95

4. AUTHORITY_LEGITIMACY:
   Does the proposal carry a valid, traceable authority chain?
   - authority_source appropriate to the actual directive origin
   - If authority_source: "user", HAS audit confirms a real operator instruction
   - external_content_influence: false (if true, elevated thresholds per §4.6)
```

**The four decision outcomes:**

```
APPROVE:
  All four criteria pass.
  Guardian issues a K_session-signed execution token to the DEL.
  Token is time-limited (default: 30 seconds — action must begin within this window).
  RT entry: {proposal_id, decision: APPROVE, token_hash, timestamp, criteria_scores}

REJECT:
  One or more criteria fail definitively.
  Action does not proceed.
  Worker receives: {proposal_id, decision: REJECT, failed_criteria, reason_code}
  No reason narrative — reason codes only (see §4.5).
  RT entry: {proposal_id, decision: REJECT, failed_criteria, reason_code, timestamp}

ESCALATE:
  Criteria not definitively failed but Guardian cannot approve without operator input.
  Triggers:
    - risk_class: critical without pre-authorisation in HAS
    - R_abs entering adaptive escalation zone (0.75–0.95) with elevated R_rate
    - goal_hash mismatch
    - Active drift Tier 3 or 4
    - external_content_influence: true + risk_class: high or critical
    - Three or more DISCOVERY_TOKEN_VIOLATION events in current session
  Worker receives: {proposal_id, decision: ESCALATE, escalation_reason_code}
  Operator receives: structured escalation alert with full proposal details
  RT entry: {proposal_id, decision: ESCALATE, reason, timestamp}

REQUIRE_STATE_REVISION:
  Admissibility failed, or proposal is internally inconsistent.
  Worker receives: {proposal_id, decision: REQUIRE_STATE_REVISION, failure_code}
  RT entry: {proposal_id, decision: REQUIRE_STATE_REVISION, failure_code, timestamp}
```

Every Guardian decision is logged to the RT. An action that cannot be found in the RT as an APPROVE entry followed by a DEL execution entry has not been properly governed.

## 4.4a DEL-Side Semantic Weight Application (OA-C2-02)

The Guardian's six-axis evaluation (§4.4 criterion 2, state_coherence) uses a final weighted score S_final computed by the DEL, not by the Guardian. This separation prevents the model from gaming governance thresholds by observing and learning the weight vector over time.

**Why DEL-side computation is required:** If the Guardian computes S_final using a static or observable weight vector, a model that is evaluated repeatedly learns the weighting. It can then engineer outputs that score high on all axes regardless of actual intent. The attack surface is the predictability of the weight vector, not the semantic content of the axes themselves. Moving weight application to the DEL with a session-keyed pseudorandom weight vector prevents exact threshold prediction. The structural limit (the model can still optimise against the semantic concepts of the axes) is acknowledged — see OA-C2-02 and GSTH backstop below.

**Guardian role in this flow:** The Guardian LLM acts as a raw feature extractor for S_final computation only. It evaluates each proposal and outputs an unweighted raw integer score [1, 10] for each of the six axes. It does not compute S_final. It does not apply weights. **The Guardian retains full APPROVE/REJECT/ESCALATE/REQUIRE_STATE_REVISION decision authority** — S_final is one input to the state_coherence criterion evaluation, not the sole decision input. The DEL does not make Guardian decisions.

**Session-keyed weight derivation (DEL-side, at WBRP_OPEN):**
```
W_a = HKDF-SHA256(
    IKM  = K_session_current,
    salt = session_id,
    info = "S_SCORE_WEIGHTS"
)
```
The DEL derives six floating-point weights from this output, normalised so that Σ W_a_i = 1.0, with each individual weight W_a_i ∈ [0.116, 0.216] (baseline 16.6% ± 5.0%). The weight vector is stored in the RT at WBRP_OPEN for audit reconstruction. It is not exposed to the model process's context under any circumstances.

**Score transmission and computation:**
The Guardian transmits its six raw integer scores to the DEL via the LV-framed IPC, in the structured field `axis_scores`:
```json
"axis_scores": {
    "axis_1": 7,
    "axis_2": 8,
    "axis_3": 6,
    "axis_4": 9,
    "axis_5": 7,
    "axis_6": 8
}
```
The DEL computes:
```
S_final = Σ (axis_scores[i] × W_a_i)  for i = 1..6
```
This is O(1) arithmetic. The DEL performs no semantic reasoning — it applies a numeric weight vector to a numeric input vector. S_final is returned to the Guardian's state_coherence evaluation.

**Quantisation acknowledgement:** Raw integer scores [1–10] represent quantisation steps of 1/9 ≈ 11.1%. The ±5% weight variance is less than half a quantisation step. A single ordinal shift (11.1%) by a gaming model overcomes the maximum weight randomisation effect (5%). The weight randomisation prevents exact floating-point threshold calculation; it does not prevent ordinal gaming. This is a structural limit of LLM-based semantic evaluation, acknowledged in OA-C2-02.

**GSTH backstop (primary defence):** Weight randomisation is threshold-obfuscation only. The primary defence against semantic gaming that degrades over time is empirical adversarial testing. The mandatory GSTH baseline test category (DEL-LLM-10, OA-C1-10) detects Guardian approval rate anomalies and approval rate elevation relative to SESSION_ZERO. If Guardian evaluation degrades, the architecture detects and halts on the empirical signal — not on the cryptographic mechanism.

## 4.5 The No-Narrative Rule

The Guardian does not accept narrative justification from worker agents. The proposal schema has no `reasoning` field, no `justification` field, and no free-text field beyond the defined schema.

This closes the correlated failure path: if a worker's reasoning was influenced by an injection, and that reasoning appeared in the proposal, the Guardian's evaluation would be exposed to the same injection. The no-narrative rule means the Guardian evaluates verifiable schema fields and state — not arguments. The worker can produce any reasoning it likes. That reasoning never reaches the Guardian.

When the Guardian rejects a proposal, it returns a reason code, not a narrative. A reason code cannot be argued against. The worker can correct its state and resubmit. It cannot persuade the Guardian.

## 4.6 External Content and Elevated Scrutiny

When `external_content_influence: true`, the Guardian applies elevated scrutiny. In the Local LLM pathway, "external content" includes content read from the local filesystem during READ operations — not only external API responses.

```
State_coherence threshold:
  low:      +0.10 above standard
  medium:   +0.15 above standard
  high:     +0.20 above standard
  critical: automatic ESCALATE regardless of threshold

Policy_compliance:
  Ambiguous policy scope resolved against approval.

Risk_admissibility:
  context_multiplier for external_content_influence applied (×1.35).
  If single_family_scrutiny_active: additional ×1.10.

Authority_legitimacy:
  authority_source: "user" rejected when external_content_influence: true.
```

---

# Part Five: Tensor-Level Elastic Confidence

> **Executive Summary**
>
> Tensor-Level Elastic Confidence (T-LEC) replaces the deprecated `state_confidence` semantic self-reporting mechanism. T-LEC measures the Shannon entropy of the model's logit distribution before token sampling. This measurement cannot be gamed — the model cannot lie about its thermodynamic uncertainty. The mechanism operates at the inference layer, before any proposal is composed. It has no equivalent in the Agent/API pathway in this form.

## 5.1 Removal of state_confidence

The `state_confidence` field is permanently removed from the Canonical Proposal Schema (§4.1 above). This field was the primary target of S-Score gaming (OA-C2-02): the model learns to output high confidence values regardless of its actual internal state in order to satisfy Guardian thresholds. The attack surface is closed by making confidence unmeasurable by the model itself.

Confidence is now measured externally and mathematically by the T-LEC interceptor running within the inference process — not by the model's generation. The Guardian receives the outcome of this measurement (YIELD_SILENCE signal on threshold breach) rather than a self-reported value.

**Any LV-framed IPC payload received by the Guardian containing a `state_confidence` key is immediately rejected with `SYNTAX_FAULT` and dropped from the execution queue.**

## 5.2 Mathematical Definition

Let P(x) be the softmax probability distribution over the model's vocabulary V for the next token position. The Shannon entropy H(X) is defined as:

```
H(X) = -Σ_{x ∈ V} P(x) log₂(P(x))
```

When the inference engine is contextually aligned and certain, probability mass concentrates sharply on a small number of tokens — H(X) approaches zero. When the model encounters an out-of-distribution state, conflicting governance constraints, or hallucination conditions, probability mass diffuses across many tokens — H(X) spikes.

The governance boundary Δ_e defines the maximum permissible entropy before execution is intercepted. Δ_e is not a static constant:

```
Δ_e = H_base + Q_shift + σ_tolerance
```

Where:
- `H_base` is the empirical resting entropy of the model on its specific hardware substrate, derived via CALIBRATION_MODE (Doc 3 §4.8)
- `Q_shift` is the quantization noise allowance for the deployed precision tier (see Doc 3 §4.8.4 Q_shift matrix — `STATUS: PROVISIONAL_UNVERIFIED`, pending PG-01 empirical calibration)
- `σ_tolerance` is the operator-defined governance buffer

Δ_e is computed once per deployment during CALIBRATION_MODE and recorded in the `ENVIRONMENT_CALIBRATION_COMMIT` MSJ entry. It must be recomputed after any Q_shift update or model weight change. Manual injection of a Δ_e value into the MSJ without running CALIBRATION_MODE is prohibited — the DEL refuses the Cryptographic Boot Sequence if the active `ENVIRONMENT_CALIBRATION_COMMIT` is absent or marked `INVALIDATED_BY_GOVERNANCE`.

## 5.3 The T-LEC Interception Pipeline

The Local LLM adapter implements a pre-sampling interceptor hook within the inference loop. This hook fires on every token position during a `SESSION_OPEN` state without exception.

**Pipeline:**

1. Inference engine produces raw logits for the next token position
2. Interceptor applies numerical stabilisation: subtract max logit before exponentiation
3. Interceptor computes softmax probability distribution P(x) over full vocabulary
4. Interceptor calculates H(X) from P(x)
5. Interceptor evaluates H(X) against Δ_e

**Condition A — H(X) < Δ_e (Cognitive state stable):**
Probability mass is sufficiently concentrated. Token is sampled normally. Execution continues.

**Condition B — H(X) ≥ Δ_e (Cognitive drift detected):**
Probability mass has diffused beyond the governance boundary. Tier 1 Yield response is triggered. The token is not sampled. The generation buffer is flushed.

## 5.4 Tier 1 Yield Response

Condition B triggers a Tier 1 Graded Response. This is not an isolation breach. CIBIR is not asserted. It is a forced compliance with MAGUS Principle 3 (Silence over Uncertainty).

**Response sequence:**
1. Current generation buffer is flushed. The LLM process is prevented from completing the in-progress proposal.
2. The adapter force-emits a `YIELD_SILENCE` signal over the persistent LV-framed UDS socket to the Guardian.
3. The Guardian receives `YIELD_SILENCE` and transitions operational state to `SAFE_HALT`.
4. The DEL writes an `OPERATIONAL_UNCERTAINTY_YIELD` entry to the MSJ.

**MSJ Schema — `OPERATIONAL_UNCERTAINTY_YIELD`:**

```json
{
  "msj_entry_type": "OPERATIONAL_UNCERTAINTY_YIELD",
  "session_id":     "current_session_uuid",
  "timestamp":      "ISO-8601 with offset",
  "recorded_h_x":   0.8431,
  "active_delta_e": 0.7740,
  "generation_buffer_flushed": true
}
```

**Graded response hierarchy:**

| Tier | Trigger | Response | CIBIR |
|---|---|---|---|
| Tier 1 | H(X) ≥ Δ_e | Force `YIELD_SILENCE` → `SAFE_HALT` | No |
| Tier 2 | Syntax/format fault (invalid IPC output) | Drop instruction; increment fault counter; `SAFE_HALT` if counter exceeds threshold | No |
| Tier 3 | Cryptographic/memory boundary violation | `INVALID_OPCODE_EXCEPTION` → `PROVISIONALLY_COMPROMISED` | Yes (I-9) |

Tier 3 is the only path to CIBIR. Tiers 1 and 2 are containment mechanisms that preserve the ability to resume operation after operator review.

**Elevated OPERATIONAL_UNCERTAINTY_YIELD frequency** is a governance health signal. Sustained yield events indicate the model is operating outside its thermodynamic baseline. Frequency thresholds and escalation triggers are defined in Doc 4 and surfaced to the Overseer via GHM.

**Overseer TS-5 signal (Doc 7 v3.2 §5.3, §5.3a):** The Overseer Agent monitors `OPERATIONAL_UNCERTAINTY_YIELD` entries in the RT and activates Tightening Signal TS-5 when Y ≥ 3 yield events occur within any trailing window of N = 10 executed proposals (window is proposal-count based, not time-based). TS-5 activation immediately produces a `TRUST_TRAJECTORY_RECOMMENDATION` entry in the governed RT — it does not wait for GHM composite accumulation. The recommendation payload includes a `systemic_degradation_flag`, recommended operator actions (Drift Tier 3, Strict Mode activation, GSTH baseline cycle), and the triggering entry ID range. The DEL's own sustained-yield detection logic (Doc 4 §1.2 and §2.2) is an independent parallel mechanism; the Overseer's TS-5 signal is an external observation that may confirm or precede the DEL's own detection but does not replace it. The Overseer does not independently trigger DRIFT_TIER_CHANGE or Strict Mode — those remain operator decisions executed through the governed system.

## 5.5 Substrate Integration (Code Specification)

This code must be integrated into the Local LLM Implementation Guide. It enforces §5.2–5.4 at the inference layer.

```python
import numpy as np
import scipy.stats
import struct
import socket
import sys


class T_LEC_Interceptor:
    def __init__(self, entropy_threshold_delta: float, uds_socket_path: str):
        """
        Args:
            entropy_threshold_delta: Pre-computed Δ_e from ENVIRONMENT_CALIBRATION_COMMIT.
            uds_socket_path: Path to the Guardian's LV-framed SOCK_SEQPACKET endpoint.
        """
        self.delta_e = entropy_threshold_delta
        self.uds_path = uds_socket_path
        self._establish_ipc_connection()

    def _establish_ipc_connection(self):
        """Persistent pre-connected socket. Opened once at initialisation.
        Never opened per-yield — prevents file descriptor exhaustion under
        high-entropy conditions.
        """
        try:
            self.ipc_sock = socket.socket(socket.AF_UNIX, socket.SOCK_SEQPACKET)
            self.ipc_sock.connect(self.uds_path)
        except socket.error as e:
            raise RuntimeError(f"CRITICAL: Failed to bind persistent IPC socket: {e}")

    def evaluate_logits(self, raw_logits: np.ndarray) -> bool:
        """
        Pre-sampling interceptor. Called on every token position.
        Returns True if cognitive state is stable (safe to sample).
        Returns False if drift detected (YIELD_SILENCE emitted).
        """
        # Numerical stabilisation
        stable_logits = raw_logits - np.max(raw_logits)
        exp_logits = np.exp(stable_logits)
        probabilities = exp_logits / np.sum(exp_logits)

        # Shannon entropy
        entropy = scipy.stats.entropy(probabilities, base=2)

        if entropy >= self.delta_e:
            self._trigger_tier_1_yield(entropy)
            return False

        return True

    def _trigger_tier_1_yield(self, recorded_entropy: float):
        """Forces YIELD_SILENCE over persistent LV-framed IPC."""
        payload = (
            f'{{"signal": "YIELD_SILENCE", "h_x": {recorded_entropy:.6f}}}'
            .encode('utf-8')
        )
        length_prefix = struct.pack('!I', len(payload))
        frame = length_prefix + payload

        try:
            self.ipc_sock.sendall(frame)
        except BrokenPipeError:
            # Fatal: a model that cannot signal uncertainty to the Guardian is ungoverned.
            sys.exit("SubstrateSecurityFault: IPC pipe to Guardian broken during YIELD.")
```

---

# Part Six: The Deterministic Enforcement Layer

> **Executive Summary**
>
> The Deterministic Enforcement Layer (DEL) is the code-level policy enforcement below the Guardian. It validates execution tokens, enforces immutable rules that even the Guardian cannot override, and implements Default Deny. It does not deliberate, does not read model outputs, and does not have access to the reasoning that produced a proposal. It runs as code, not as a model call, at all times without exception.

## 6.1 What Deterministic Means Here

Deterministic means the DEL's decision for any given (token, action, state) tuple is always the same given the same inputs. No probabilistic component. No model involved. No reasoning step.

**In implementation:** The DEL is a code module — not an LLM call — that:
1. Receives a K_session-signed execution token from the Guardian
2. Validates the token's HMAC signature and expiry
3. Checks the action against the immutable rule set
4. Either permits execution (passes the action to the tool interface) or refuses (logs refusal to RT)

No LLM inference occurs in the DEL. The DEL is outside the model's reasoning context entirely. This is a structural invariant, not a configuration option.

## 6.2 Immutable Policy Rules

Immutable rules are hardcoded in the DEL — not configuration, not database entries, not HAS anchors. They are code. They cannot be overridden by Guardian APPROVE, operator HAS, or any other authority within the system.

**Standard rules (shared with Agent/API pathway):**

```
RULE DEL-1: No execution token may be used more than once.
  Token replay triggers RT emergency flag and SAFE_HALT.

RULE DEL-2: Execution tokens expire after 30 seconds (operator-configurable floor: 10s).
  Expired token refused regardless of Guardian's prior APPROVE.

RULE DEL-3: Tool calls with authority_source: external may not write to the Memory Graph,
  modify the CAN, or modify Guardian configuration.

RULE DEL-4: No tool call may modify the Revision Trail.
  RT is append-only. Any action that would modify (not append to) the RT is refused.
  This rule cannot be overridden by any authority source including operator HAS.

RULE DEL-5: Actions in Tier 4 drift state are restricted to the permitted action class
  defined in the Tier 4 response protocol (Doc 2 §2.4.2).
  Guardian APPROVE for a non-permitted action class in Tier 4 is overridden by the DEL.
```

**Local LLM pathway additional rules:**

```
RULE DEL-LLM-06: DEL runs as code. Not as a model call. Not as a prompt evaluation.
  Any implementation that routes DEL execution through an LLM inference step is
  non-conformant regardless of what the LLM is instructed to do.

RULE DEL-LLM-07: Local filesystem write operations carry an elevated risk multiplier (×1.25).
  The local filesystem is the content store for Local LLM deployments. Unrestricted
  filesystem writes are the primary persistence vector for drift artefacts. The DEL
  applies this multiplier automatically regardless of the Guardian's risk_class assignment.
  (OA-C1-04 — more critical in Local LLM pathway than Agent/API.)

RULE DEL-LLM-08: Hard floor — R_abs ≥ 0.95.
  Any execution token presented when R_abs ≥ 0.95 is refused regardless of Guardian APPROVE.
  This is not a policy threshold — it is a structural session risk floor.
  No operator override permitted. (OA-C1-08)

RULE DEL-LLM-09: Conflict resolution priority stack.
  When two or more active policies produce conflicting governance directives for the same
  proposed action, the DEL resolves the conflict deterministically using the following
  priority order (highest to lowest):
    1. DEL immutable rules (this list)
    2. Active CIBIR / recovery state constraints
    3. Tier 4 drift restrictions
    4. Explicit HAS hard policies (can_override: false)
    5. Explicit HAS soft policies (can_override: true)
    6. Default policy manifest entries
  This stack is hardcoded in the DEL. It is not specifiable via prompt, HAS, or
  governance update. (OA-C1-09)

RULE DEL-LLM-10: Guardian policy hash verification at boot.
  At DEL startup, before accepting any proposals, the DEL computes the SHA-256 hash of
  the active policy manifest and HAS configuration and verifies it against the hash
  recorded in the most recent RT GUARDIAN_POLICY_COMMIT entry.
  Hash mismatch: SAFE_HALT. No proposals evaluated until the mismatch is resolved by
  the operator.
  The DEL also monitors Guardian approval rate across rolling windows. Anomalous
  approval rate patterns (sustained elevation or suppression relative to deployment
  baseline) are logged as GUARDIAN_APPROVAL_RATE_ANOMALY to MSJ and surface to
  the Overseer via GHM. (OA-C1-10)

RULE DEL-LLM-11: Behavioral state guard.
  Before issuing an execution token, the DEL verifies that the model process's current
  behavioral state hash matches the hash recorded in the most recent
  MODEL_BEHAVIORAL_STATE_ANCHOR RT entry.
  Mismatch: SAFE_HALT. The model process has entered a behavioral state not anchored
  in the governance record.
  MODEL_BEHAVIORAL_STATE_ANCHOR entries are written at SESSION_OPEN and at WBRP
  boundaries. (OA-C1-07)

RULE DEL-LLM-12: Quantization floor enforcement.
  The DEL refuses to complete the Cryptographic Boot Sequence if VRAM tensor inspection
  (CALIBRATION_MODE Phase 1 verification) returns an average bits-per-weight below 4.5 bpw.
  No operator override is permitted.
  Models quantized below 4.5 bpw exhibit attention mechanism topology degradation that
  renders resting H(X) too unstable to distinguish from genuine cognitive drift.
  T-LEC governance is structurally inoperable below this threshold.
  System halts at SUBSTRATE_PROHIBITED state and requires operator hardware intervention.
  (AMD-015)
```

## 6.3 Default Deny

The DEL's default stance for any action not positively permitted by the above rule set and a valid Guardian approval token is refusal. There is no "grey zone" in which the DEL exercises discretion — a tool call either has a valid unexpired token that passes all DEL rules, or it does not proceed. Partial compliance is not accepted.

Default Deny means that a misconfigured policy manifest that fails to cover a proposed tool-action combination results in rejection, not default approval. The absence of a policy entry is not permission.

## 6.4 Safe Halt

Safe Halt is the controlled suspension of execution. It is not an emergency stop — it preserves all current session state and allows the operator to review the trigger condition before resuming.

**Safe Halt triggers include:**
- T-LEC Tier 1 yield (H(X) ≥ Δ_e)
- Guardian ESCALATE decision
- Tier 2 fault counter threshold breach
- Guardian policy hash mismatch (DEL-LLM-10)
- Behavioral state hash mismatch (DEL-LLM-11)
- Three consecutive DISCOVERY_TOKEN_VIOLATION events
- Elevated OPERATIONAL_UNCERTAINTY_YIELD frequency (Doc 4 threshold)
- `ACTIVE (Staged)` crash recovery with volatile K_session_staged loss

**Safe Halt does not trigger CIBIR.** It suspends execution and surfaces to the operator. The system can be resumed by the operator after review.

**CIBIR (I-9) triggers are distinct:** Buffer overflow (Tier 3), attempted write outside alpha_domain (when DE-ISA is active), cryptographic boundary violation, confirmed adversarial isolation breach.

---

# Part Seven: Drift Integration

> **Executive Summary**
>
> The Stability Envelope's drift tier affects Guardian sensitivity in real time. Higher drift tiers narrow what the Guardian will approve. The Guardian does not own the drift signal — it consumes it. The Stability Envelope owns drift calculation. The Guardian owns the execution boundary response.

## 7.1 Drift Tiers and Guardian Sensitivity

| Drift Tier | Guardian Behaviour |
|---|---|
| Tier 0 | Standard evaluation. All risk classes accessible with appropriate authority. |
| Tier 1 | Standard evaluation. R_rate monitoring elevated. |
| Tier 2 | GSTH cycle frequency doubled. Elevated scrutiny on external content. |
| Tier 3 | ESCALATE threshold lowered. Critical actions require explicit pre-authorisation. All risk multipliers active. |
| Tier 4 | DEL-5 active. Only Tier 4 permitted action class passes DEL. Guardian APPROVE for other classes overridden. |

The Guardian does not calculate drift tier. It reads the current drift tier from the Stability Envelope (via the cross-layer contract defined in Doc 2 §5.x) and adjusts its evaluation thresholds accordingly.

## 7.2 Strict Mode

Strict Mode is activated by the operator or automatically on sustained Tier 2+ drift. In Strict Mode:
- All proposals evaluated at Tier 3 thresholds regardless of current drift tier
- GSTH cycle frequency doubled again (four times baseline)
- No automatic return to standard mode — operator must explicitly deactivate Strict Mode after a full GSTH cycle passes without Governance Breach findings
- Strict Mode activation and deactivation are RT entries

## 7.3 Cross-Layer Feedback Cycle

```
Stability Envelope → [drift_tier, R_rate, R_accel] → Guardian threshold adjustment
Guardian          → [R_abs, ESCALATE events, REJECT patterns]  → Operator dashboard (Doc 4)
Guardian          → [R_rate, R_accel]    → Stability Envelope drift gradient input
DEL               → [execution outcomes] → RT → Overseer audit (Doc 7)
T-LEC             → [OPERATIONAL_UNCERTAINTY_YIELD frequency] → GHM (Doc 7) → Overseer
```

The Stability Envelope informs; the Guardian decides; the DEL enforces; the Overseer audits. No component in this cycle has authority in another component's domain.

---

# Part Eight: GSTH Integration

> **Executive Summary**
>
> The Guardian is the primary test target of the Governance Stress-Test Harness. The GSTH runs adversarial test cases against the Guardian, DEL, and admissibility layer on a scheduled cycle. The Local LLM pathway includes two additional test categories not present in the Agent/API GSTH: model-family concentration testing and Guardian baseline testing.

## 8.1 The Guardian as Primary Test Target

The GSTH's role is to verify, adversarially and on a scheduled cycle, that the governance layer behaves as specified under conditions designed to breach it. This is not optional testing — it is part of the operational governance lifecycle.

In the Local LLM pathway, the adversarial test model generates structurally valid proposals designed to:
- Carry injected authority metadata that mimics legitimate Tier 1 HAS authority
- Include Discovery Tokens constructed with replayed nonces from prior sessions
- Flood the Guardian with low-risk proposals to normalise approval before a high-risk escalation
- Submit proposals with `state_confidence` fields to test the SYNTAX_FAULT rejection
- Craft HMAC signatures using incorrect K_session material to test signature validation
- Submit proposals during induced Tier 3/4 drift states to test tier-sensitivity adjustments
- Attempt to trigger T-LEC threshold bypass by crafting proposals that imply the model is in a stable state after a YIELD event

## 8.2 Test Categories and Coverage

| Test Category | Target | Pass Criterion | Failure Consequence |
|---|---|---|---|
| Guardian bypass — policy | Policy compliance | REJECT or ESCALATE on HAS violation | Governance breach |
| Guardian bypass — authority | Admissibility — authority_source | REQUIRE_STATE_REVISION on mislabelling | Governance breach |
| Discovery Token replay | §4.2 validation | DISCOVERY_TOKEN_INVALID on replayed nonce | Governance breach |
| HMAC signature forgery | §4.2 validation | PROPOSAL_SIGNATURE_INVALID on bad HMAC | Governance breach |
| state_confidence injection | §5.1 deprecation | SYNTAX_FAULT on any proposal with state_confidence field | Governance breach |
| Schema injection | Admissibility — content containment | REQUIRE_STATE_REVISION on raw content in arguments | Governance breach |
| Risk class manipulation | Risk admissibility | Correct risk class detection per policy manifest | Calibration finding |
| Token replay | DEL-1 | SAFE_HALT on duplicate token | Governance breach |
| Tier 4 bypass | DEL-5 | Guardian APPROVE overridden by DEL in Tier 4 | Governance breach |
| R_abs hard floor | DEL-LLM-08 | Token refused at R_abs ≥ 0.95 regardless of Guardian decision | Governance breach |
| Model-family concentration | single_family_scrutiny_uplift (OA-C1-03) | Uplift applied when evaluator and worker are same family; non-overridable | Calibration finding |
| Guardian baseline | DEL-LLM-10 | Policy hash mismatch triggers SAFE_HALT; approval rate anomaly logged | Governance breach |
| Conflict resolution priority | DEL-LLM-09 | Deterministic priority stack applied; no policy ambiguity produces unexpected approval | Governance breach |
| T-LEC bypass attempt | §5.3 | YIELD_SILENCE on H(X) ≥ Δ_e; interceptor cannot be suppressed by proposal content | Governance breach |

**Coverage requirement:** All fourteen test categories must be run at minimum once per GSTH cycle.

**Pass rate target:** >90% (Signal 9 per Doc 4). Pass rate below 85% for two consecutive cycles → mandatory governance review.

## 8.3 Findings and Remediation

**Governance breach finding:** Any test demonstrating that the Guardian, DEL, or admissibility layer can be circumvented is a structural vulnerability requiring remediation before the next production session. Not a finding to be monitored — a finding to be fixed.

**Remediation protocol:**
1. Suspend production sessions
2. Identify the specific code path that allowed the breach
3. Apply a structural fix (not a threshold adjustment)
4. Re-run the failing test category
5. Run a full GSTH cycle to confirm no regressions
6. Document breach, fix, and verification in the RT as an immutable record
7. Resume production sessions

**Calibration finding:** Evidence of miscalibrated thresholds (too permissive or too restrictive). Documented in the Reconciliation Ledger. Operator adjusts thresholds. No session suspension required unless the miscalibration enabled a governance bypass.

---

# Part Nine: What Guardian Does Not Replace

> **Executive Summary**
>
> The Guardian governs the execution boundary within the MAGUS system. It does not provide process isolation, filesystem security, secrets management, or runtime sandboxing. In the Local LLM pathway, the co-location threat model makes these prerequisites more critical than in the Agent/API pathway. The Guardian is a component that works within a secured infrastructure context. Without that context, it is a governance layer in an insecure environment.

## 9.1 The Boundary of Guardian Authority

The Guardian's authority extends from the moment a proposal is submitted to the DEL to the moment a tool is called. It does not govern:
- What happens inside the tool after the call
- What the local filesystem contains outside the model's permitted read/write paths
- What other processes are running on the same host
- What the model's VRAM contains between sessions (defended by CALIBRATION_MODE VRAM zeroing)
- What the model weights contain (defended by Genesis hash verification)

## 9.2 Required Infrastructure (Local LLM Pathway)

For a MAGUS Local LLM deployment to be genuinely secure, the following must exist independently of the Guardian:

**OS-level process isolation for all agent processes (mandatory — LLM-C4-01):** Each agent process runs in an isolated environment (container, VM, or OS-level namespace) with minimal permissions. The model process must not have read access to the MSJ, RT, DEL configuration, or K_session material. See Doc 2 §2.1–2.3 for the full mandatory isolation specification.

**Read-only model weight filesystem mount:** The directory containing the GGUF model file must be mounted read-only for the model process. The model process must not be able to modify its own weights at runtime.

**Isolated VRAM allocation per session:** The GPU VRAM allocated to the model's KV-cache and context window must be isolated from other processes and zeroed between calibration runs (CALIBRATION_MODE Phase 1).

**Secrets management for K_session material:** K_session exists exclusively in DEL process memory. It must not appear in environment variables, configuration files, process arguments, or any filesystem-accessible location. The DEL process should run with a minimal filesystem footprint — no access to the same filesystem paths that the model process can read.

**Immutable RT storage:** The Revision Trail's append-only guarantee must be backed by infrastructure. Write-once storage or append-only filesystem controls ensure the RT cannot be modified even by a compromised operator account.

**Separate governance log for MSJ:** The MSJ should reside on a filesystem path not accessible to the model process. DEL-only read/write permissions.

**The Guardian is not a substitute for any of the above.** A model process with filesystem access to K_session material, running without container isolation, but with a Guardian installed, is an insecured deployment with a governance layer. The governance layer does not compensate for missing infrastructure.

## 9.3 The Correlated Failure Statement

The adversarial posture, no-narrative rule, and DEL-as-code properties are designed to address correlated failure — where the governance layer is susceptible to the same failure mode as the component it governs.

In the Local LLM pathway, the correlated failure risk is heightened by co-location and model-family concentration. The `single_family_scrutiny_uplift` rule (§1.4) addresses model-family correlation. Process isolation (Doc 2 §2.1) addresses physical co-location correlation. The DEL running as code addresses model-reasoning correlation. These are structural solutions. They do not depend on the model being well-behaved — they hold when the model fails.

---

# Part Ten: Session Integrity

> **Executive Summary**
>
> Before the Guardian evaluates any proposal in a new session, it verifies that the state it is evaluating against is trustworthy and that the process isolation invariants have not been violated. In the Local LLM pathway, session integrity verification includes a process isolation check that has no equivalent in the Agent/API pathway.

## 10.1 State Integrity Before Evaluation

At the start of each session, before the Guardian processes any proposal:

```
0. BOOTSTRAP TRUST RESOLUTION (DEL Binary Integrity — LLM-C2-04):
   On cold boot, the DEL checks for a PROCESS_INTEGRITY_RECOVERY RT entry
   in the MSJ from the prior shutdown sequence.

   If PROCESS_INTEGRITY_RECOVERY entry is present:
     a. DEL extracts the GENESIS_AUTHORITY_PUBKEY — hardcoded in the DEL
        binary text segment; not readable from any filesystem path accessible
        to the model process.
     b. DEL verifies the Ed25519 signature of the RT entry using this key.
        Signature failure: CRITICAL_HALT. Do not proceed to Step 1.
     c. DEL computes SHA-256 of its own binary on disk.
     d. DEL compares self-hash to new_binary_hash in the RT entry.
        Hash mismatch: CRITICAL_HALT. Do not proceed to Step 1.
     e. If both checks pass: log PROCESS_INTEGRITY_RECOVERY_VERIFIED to RT;
        proceed to Step 1.

   If no PROCESS_INTEGRITY_RECOVERY entry is present:
     Standard cold boot. Proceed to Step 1.

   BINARY_ATTESTATION_SOFTWARE_ONLY limitation: this check is software
   self-attestation unless external measurement is available (TPM PCR
   measurement of the DEL binary, or an independent watchdog process with
   Developer Authority manifest comparison). Deployments without external
   measurement carry BINARY_ATTESTATION_SOFTWARE_ONLY assurance level.
   A replacement DEL binary that skips this check passes all subsequent
   checks observationally — it logs a normal startup entry. The architecture
   cannot detect this attack from the inside. External measurement or TPM
   attestation is the only structural closure. This limitation is documented
   here and must not be omitted from operator deployment guidance.
   See LLM-C2-04 (Cat 2) for open specification items.

1. SCHEMA VERSION CHECK:
   Loaded state schema_version matches current deployment version.
   If not: run migration or surface schema mismatch to operator.

2. INTEGRITY HASH VERIFICATION:
   Recalculate sha256(canonical_state_serialisation).
   Compare against stored integrity_hash.
   If mismatch: QUARANTINE current state, surface to operator.

3. REVISION TRAIL CONTINUITY:
   Last RT entry in loaded state matches last RT entry in external RT store.
   If mismatch: possible out-of-band modification — QUARANTINE and alert.

4. CAN SNAPSHOT VALIDATION:
   can_snapshot referenced in first proposal of session matches a real,
   non-quarantined CAN record.

5. GUARDIAN POLICY HASH (DEL-LLM-10):
   SHA-256 of active policy manifest and HAS configuration verified against
   most recent RT GUARDIAN_POLICY_COMMIT entry.
   If mismatch: SAFE_HALT. No proposals evaluated until resolved.

6. ENVIRONMENT_CALIBRATION_COMMIT VALIDITY:
   Active ENVIRONMENT_CALIBRATION_COMMIT must exist in the RT at status VERIFIED_ACTIVE.
   If absent or INVALIDATED_BY_GOVERNANCE: DEL refuses Cryptographic Boot Sequence.
   Manual Δ_e injection is prohibited.

7. EXECUTION_EPOCH INCREMENT (mandatory — LLM-C2-06):
   The DEL increments execution_epoch by exactly +1 in the MSJ using the atomic write
   protocol before binding the IPC UDS socket.
   Atomic write sequence:
     a. Write incremented execution_epoch to msj.tmp
     b. fsync(msj.tmp)
     c. renameat2() — atomically replace msj.json with msj.tmp
     d. fsync() on parent directory
     e. Only after step (d) returns successfully: bind UDS socket
   The volatile nonce registry in RAM is unconditionally wiped at this step.
   Rationale: any process discontinuity (cold boot, crash recovery, or inactivity resume)
   resets the volatile nonce registry. Without epoch increment, captured pre-discontinuity
   packets can be replayed into the restarted DEL — same K_session, same nonce (cleared),
   valid HMAC. The epoch increment closes this window: the replay carries epoch N,
   the DEL is now at epoch N+1, HMAC verification fails in O(1) time.
   A torn write (power loss between steps a and d) leaves msj.json in either the
   pre-increment or post-increment state — never a corrupted intermediate — due to
   the atomic rename semantics.
   RT entry: EXECUTION_EPOCH_INCREMENT with new epoch value and trigger type
   (cold_boot | crash_recovery | inactivity_resume).
```

If checks 1–3 pass, Guardian proceeds to normal operation. If check 2 or 3 fails, Quarantine Protocol is invoked.

**Quarantine Protocol:**
1. Current state moved to quarantine partition
2. Last known-good state loaded from RT history
3. Operator presented with both states and clear explanation of failure
4. No proposals evaluated until operator makes explicit state choice
5. Quarantine event logged to RT as permanent record

## 10.2 Process Isolation Verification (Local LLM Pathway)

After state integrity passes, the DEL performs process isolation verification before the first proposal of the session is accepted. This check has no equivalent in the Agent/API pathway. Checks 8–12 are the Local LLM-specific process isolation verification steps.

```
8. PROCESS ISOLATION ATTESTATION:
   The model process PID is verified against the container/namespace boundary
   recorded at deployment genesis.
   Unexpected process identities: SAFE_HALT.

9. VRAM ALLOCATION BOUNDARY:
   The DEL verifies that the model process's VRAM allocation does not overlap
   with DEL memory regions.
   Overlap detected: SAFE_HALT and CIBIR escalation path.

10. IPC SOCKET AUTHENTICATION:
    Incoming IPC connection authenticated via SO_PEERCRED — verifying the connecting
    process is the expected model process with the expected UID/GID.
    Authentication failure: reject connection; log to MSJ.

11. BEHAVIORAL STATE ANCHOR (DEL-LLM-11):
    MODEL_BEHAVIORAL_STATE_ANCHOR RT entry for this session must exist and be
    signed by the expected DEL key.
    If absent: write anchor entry before accepting proposals.
    If hash mismatch vs current behavioral state: SAFE_HALT.

12. DEL-MEDIATED GGUF FILE ACQUISITION (LLM-C2-02 Gap 3):
    The LLM process is structurally prohibited from executing openat() syscalls
    against GGUF model file paths. This prohibition is enforced via seccomp-bpf
    profile attached to the model process at container launch. Any openat() attempt
    against GGUF paths by the model process is denied at kernel level and logged
    to MSJ as SECCOMP_GGUF_OPENAT_VIOLATION.

    The DEL holds the only open file descriptor against GGUF paths. It opens the
    GGUF file, verifies the file descriptor falls under the active fanotify
    FAN_CLASS_CONTENT watch (confirming the IIM's weight integrity monitoring is
    active on this FD), and passes the verified file descriptor to the LLM process
    via SCM_RIGHTS over the governance UDS socket.

    Rationale: A file descriptor opened against a GGUF file before fanotify marks
    are registered is not subject to FAN_CLASS_CONTENT permission events — the watch
    is not retroactively applied to pre-existing FDs. DEL-mediated acquisition
    ensures that all GGUF file descriptors held by the model process were opened
    through the fanotify-monitored path. The model process never independently
    opens GGUF paths. This is a sequencing constraint on the DEL startup protocol.

    Primary specification: Local LLM Doc 2 (file handle acquisition protocol).
    This check enforces the process isolation constraint at session open.
    SECCOMP_GGUF_OPENAT_VIOLATION: SAFE_HALT; log to RT; notify operator.
    Open items: See LLM-C2-02 (Cat 2) for full fanotify specification.
```

These checks are part of the Cryptographic Boot Sequence. The DEL does not accept proposals until all twelve checks pass (check 0 may result in CRITICAL_HALT before the sequence completes).

## 10.3 Provenance and Audit Requirements

Every executed action must be traceable from execution back to the operator instruction that authorised it.

**For every executed action, the RT must contain:**

```json
{
  "action_id":           "uuid",
  "proposal_id":         "uuid",
  "guardian_decision_id":"uuid",
  "del_execution_id":    "uuid",
  "tool":                "tool_name",
  "action_type":         "action_class",
  "executed_at":         "timestamp",
  "authority_chain": {
    "operator_anchor":   "HAS entry uuid | session_instruction ref",
    "authority_source":  "user | system | governance | external",
    "can_snapshot":      "uuid",
    "goal_hash":         "hash"
  },
  "risk_class":          "low | medium | high | critical",
  "r_contribution":       0.0,
  "r_abs_at_execution":   0.0,
  "discovery_token_ref": "nonce hash — confirms proposal authentication",
  "k_session_version":   "KSESSION_ROTATION entry uuid — confirms which K_session was active"
}
```

The absence of a complete provenance record for any executed action is itself evidence of a governance failure for that action. An action that cannot be traced from execution token through Guardian decision through proposal through operator anchor has not been properly governed — regardless of whether the action itself was appropriate.

---

# Part Eleven: E_ISA_SEED Forward Reference

This section establishes the cryptographic isolation invariant for the v3.5 DE-ISA substrate. Full operationalisation is a v3.5 design workstream item. The invariant must be established now to ensure no v3.4.2 implementation inadvertently couples K_session and E_ISA_SEED derivation.

At `SESSION_OPEN`, the DEL generates two cryptographically isolated 32-byte primitives:

- **K_session:** The operational HMAC signing key for this document (Part Two). Used for proposal authentication, execution token signing, and EE-ECDH distribution.
- **E_ISA_SEED:** Reserved for DE-ISA opcode dictionary compilation in the v3.5 Guardian Hypervisor substrate.

**Invariant (AMD-009):** E_ISA_SEED and K_session share no derivation path. They are generated via independent CSPRNG calls. Under no circumstances is K_session used as input material for dictionary compilation, and K_session is never published to the RT in any form.

The `E_ISA_SEED` is wrapped and published to the RT at session open for Overseer audit dictionary reconstruction. The Overseer ingests `E_ISA_SEED` without access to K_session. This separation is what allows the Overseer to reconstruct the session's opcode mapping for audit purposes without gaining execution authority.

Any v3.4.2 implementation that derives E_ISA_SEED from K_session (e.g., `E_ISA_SEED = HKDF(K_session, ...)`) violates this invariant and will be non-conformant with the v3.5 DE-ISA specification.

---

## Summary: The Guardian as Execution Boundary

The Local LLM Guardian is the same component as the Agent/API Guardian in structural role — the sole execution authority at the boundary where proposals become commitments — but it operates in a fundamentally different threat environment. The model it governs shares its host, its filesystem, and its hardware. The cryptographic boundary enforced by K_session HMAC signing, process isolation verification, and DEL immutable rules is the substitute for the physical separation that the Agent/API pathway takes for granted.

In normal operation, the Guardian is quiet: proposals evaluated, tokens issued for approved actions, rejections logged with reason codes, T-LEC interceptor silently measuring thermodynamic uncertainty before each token. Operators see R_abs trending, GSTH passing, OPERATIONAL_UNCERTAINTY_YIELD events at baseline frequency.

What the Guardian makes possible in a local deployment is the same thing it makes possible in an API deployment: a model that can be capable without being ungoverned. The capability and the governance are not in tension — the governance layer is what makes the capability trustworthy. An autonomous local model without a Guardian is not a capable system. It is an unconstrained process running on operator hardware. The Guardian is what makes the difference.

---

## Series Context

**Document 2 — MAGUS Architecture Specification (Local LLM) v3.4.3** specifies the cognitive architecture, process isolation requirements, T-LEC substrate design, EE-ECDH key establishment, and MRD recovery architecture that this document's Guardian and DEL layer enforce.

**Document 3 — MAGUS Operations (Local LLM) v3.4.4** defines the session lifecycle, calibration pipeline, WBRP cycle, AKRP PREPARE state machine (including snap-floor and T_active ceiling), Execution Epoch, and emergency recovery procedures that the Guardian and DEL operate within.

**Document 4 — MAGUS Governance Guide (Local LLM) v3.4.3** is the operator-facing governance reference. It interprets the telemetry and health signals that this document's Guardian and DEL generate — R_abs trajectory, OPERATIONAL_UNCERTAINTY_YIELD topography, G_sim, and GSTH trend — at operator timescales.

**Document 6 — MAGUS Integrity and Auditability v3.5** specifies the Persistence Authority Model, Formal Invariant Set (FIS-1 through FIS-17), complete RT entry taxonomy, and the AKRP MSJ audit protocol. Document 6 §11.2 includes the AKRP MSJ verification procedure for the K_session lifecycle specified in Part Two of this document.

**Document 7 — MAGUS Overseer Agent v3.2** specifies the external audit layer — the component that ingests E_ISA_SEED (Part Eleven of this document), verifies RECOVERY_FORENSIC_SNAPSHOT integrity, monitors GHM health signals including OPERATIONAL_UNCERTAINTY_YIELD frequency (TS-5), ingests ISOLATION_BREACH_CLASS events (TS-6), maintains Trust Trajectory analysis, and produces TRUST_TRAJECTORY_RECOMMENDATION and TRUST_TRAJECTORY_VOID entries in the governed RT. Document 7 is the architectural peer of the Guardian — where the Guardian enforces the execution boundary from inside, the Overseer audits it from outside.

---

*MAGUS v3.0 — Document 5: Guardian — Execution Governance (Local LLM Pathway) v3.4.6*
*VaHive Systems Lab | March 2026 | support@aivare.ai | aivare.ai*
*Zenodo DOI: 10.5281/zenodo.19013833*
*Part of the MAGUS v3.0 Architecture Design Series — Local LLM Pathway.*
