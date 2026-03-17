# MAGUS Document 4: Governance Guide — Local LLM Pathway

**Version:** 3.4.3
**Status:** DRAFT — Full Document
**Document:** 4 of 7 — Local LLM Pathway
**Series:** MAGUS v3.0 Architecture — Local LLM
**Supersedes:** Doc 4 v3.4.2
**Source register:** Issues Register v3.3

**v3.4.3 change summary:**

Reference and metadata update only. No substantive content changed.

- Prerequisites block added.
- Document number corrected to 4 of 7.
- §3.3 cross-reference to "Document 3 §2.3" corrected to "Document 3 §3.1" (AKRP PREPARE concurrent events are in Doc 3 §3.1, not §2.3; sequencing dependency resolved by Doc 3 v3.4.4).
- Series Context footer added; sign-off updated.

**Prerequisites:** This document assumes familiarity with:
- Doc 1 v3.1 — Philosophy
- Doc 2 v3.4.3 — Architecture Specification (Local LLM)
- Doc 3 v3.4.4 — Operations (Local LLM)
- Doc 6 v3.5 — Integrity and Auditability
- Doc 7 v3.2 — Overseer Agent Specification

VaHive Systems Lab | aivare.ai | support@aivare.ai

---

## Preamble

This document is the operator-facing governance reference for a MAGUS Local LLM deployment. It assumes the operator has read Document 3 (Operations Guide) and is familiar with the DEL state machine, the WBRP governance cycle, and the Guardian evaluation framework. Document 4 does not repeat the mechanics of those systems. It interprets them: what the metrics mean at governance timescales, how to distinguish signal from noise in continuous telemetry, when and how human authority is exercised, and what the architecture can and cannot do.

Three things are worth stating before the technical content begins.

First, governance is a reading practice before it is an action practice. An operator who responds only to alarms will always be behind the system. The telemetry in Section 1 exists to give operators pattern-recognition capability before alarms fire. Develop the habit of reading the EC trajectory and H(X) distribution as continuous signals, not as individual events.

Second, the governance clock is not the session clock. R_abs decay, G_sim accumulation, and assurance tier constraints all operate at the WBRP timescale, not the session timescale. Decisions made within a single session have governance consequences that extend across the entire cryptographic epoch. Operators must hold both timescales simultaneously.

Third, this document contains one section that some operators will find uncomfortable. Section 4.1 states plainly what MAGUS cannot govern. It is written this way because ambiguity on that point is more dangerous than clarity. Read it.

---

## Section 1: Telemetry and Continuous Health Signals

*The real-time operational layer. An operator who only reads alarms is always behind the system. An operator who reads telemetry can act before alarms fire.*

---

### 1.1 The IPC Boundary Signal

The DEL logs two categories of IPC rejection events that are individually unremarkable but collectively diagnostic: `Check 9` (`SO_PEERCRED`) authentication rejections and `DISCOVERY_TOKEN_VIOLATION` events.

A `Check 9` failure means a process presented itself at the IPC boundary without valid peer credentials. In a correctly configured deployment this occurs occasionally — a misconfigured restart sequence, a timing edge case during initialisation. A single event is noise. An `DISCOVERY_TOKEN_VIOLATION` means a process presented a valid peer credential but attempted to initiate a session without a valid discovery token, or replayed a token across a session boundary. Again, isolated occurrences are usually configuration artefacts.

The signal is pattern, not incidence. The operator must track two pattern thresholds:

**Threshold 1 — Volume within a WBRP cycle:** A sustained increase in `Check 9` rejection frequency over a WBRP cycle, without a corresponding maintenance event or deployment configuration change, indicates potential process injection attempts. The quantitative threshold for MRD escalation: more than three `Check 9` rejections in any rolling 24-hour window, where no administrative activity (restart, update, configuration change) was logged in the same window.

**Threshold 2 — Cross-signal correlation:** Any rejection event — `Check 9` or `DISCOVERY_TOKEN_VIOLATION` — that occurs in the same session as an anomalous H(X) reading (H(X) > Δ_e or H(X) below the operational band floor) must be treated as a compound signal regardless of its individual volume. A single rejection event paired with an entropy anomaly in the same session triggers MAGUS Recovery Domain (MRD) escalation review.

**What to do:** For volume-threshold breaches, open an MRD review but do not halt autonomously. The evidence at this point is statistical, not causal. For compound signal events, trigger a GSTH run immediately. If GSTH passes, the anomaly is likely coincidental; if GSTH fails, the system halts automatically and the MRD investigation becomes mandatory.

**What not to do:** Do not clear `Check 9` rejection logs without investigation. The RT is WORM-anchored; the underlying rejection events are permanent. Clearing logs in ancillary systems while leaving the RT intact creates an inconsistency that is worse forensically than the original signal.

---

### 1.2 Thermodynamic Yield Topography

The H(X) distribution — the session-by-session Shannon entropy of the model's output — is not a single threshold metric. Every deployment has a characteristic entropy shape determined by its task profile: a deployment running structured data extraction will cluster near its H_base with a narrow variance; a deployment running open-ended research synthesis will cluster higher with wider variance. Deviation from the deployment's characteristic shape is the signal. Absolute H(X) values mean less than distribution shift.

The MSJ stores `ENVIRONMENT_CALIBRATION_COMMIT` entries from SESSION_ZERO calibration and any subsequent recalibration runs. These entries define the expected H(X) distribution for the deployment. The operator's primary thermodynamic telemetry practice is comparing current H(X) distributions against this stored baseline.

**Two distinct failure modes require different operator responses:**

*Contextual failures (prompting errors):* H(X) spikes in individual sessions that resolve within the session, do not repeat on the same task type, and are not correlated with hardware telemetry. These indicate unclear or ambiguous prompting — the model is uncertain because the instruction is uncertain. The H(X) spike is genuine cognitive uncertainty, not substrate degradation. Operator response: prompt refinement for the task types producing spikes. No governance action required unless the spike approaches Δ_e.

*Substrate degradation (thermal throttling, VRAM decay, hardware ageing):* H(X) elevation that persists across sessions, correlates with hardware telemetry (increased thermal readings, VRAM error rate, higher inference latency), and affects tasks that previously produced stable H(X) readings. The entropy elevation is not task-driven — it is substrate-driven. The model is computationally impaired. Operator response: hardware diagnostic immediately; if confirmed, trigger a recalibration audit via §2.3 before the current H_base becomes misleading as a governance threshold.

**The H_base comparison protocol:**

At each WBRP_CLOSE, compare the current WBRP's H(X) session distribution against the stored `ENVIRONMENT_CALIBRATION_COMMIT` baseline. Compute the distribution shift: mean H(X) drift, variance change, and tail behaviour (are extreme H(X) readings more frequent than baseline predicts?). Document this comparison in the WBRP review. Aggregate the shift across rolling WBRP windows — three consecutive WBRPs showing consistent upward H(X) drift warrant a recalibration review even if no individual session triggered governance action.

**The `SUB_BASELINE_WORKLOAD` classification:** Sessions where the mean H(X) of executed proposals falls below the operational band floor are classified as `SUB_BASELINE_WORKLOAD` by the DEL. These sessions execute normally but produce zero R_abs decay (see §1.4, Constraint 2). The thermodynamic yield topography review is also the mechanism by which the operator confirms whether a persistently non-decaying R_abs is a genuine governance concern or a consequence of a legitimately low-entropy task profile.

---

### 1.3 The Elastic Confidence (EC) Trajectory

R_abs is not a snapshot — it is a trajectory. An R_abs reading of 0.85 in decline means something fundamentally different from an R_abs reading of 0.85 accelerating toward the adaptive zone threshold. Operators who manage R_abs as a single number rather than a trend are reading a fundamentally incomplete governance picture.

The EC trajectory is the rate of change of R_abs across sessions within a WBRP cycle. Monitor it as a derivative, not a value.

**Three trajectory types and their operator implications:**

*Stable accumulation:* R_abs growing at a rate consistent with the deployment's historical task complexity profile. Governance events occur at expected frequencies. EC decay between clean session runs behaves as the historical average predicts. This is normal operation. No heightened monitoring required.

*Accelerating accumulation:* R_abs growing faster than the session history predicts for the current task profile. Per-session increments are higher than historical baseline for tasks of the same type. Two interpretations: (a) the task environment has changed and the operator should verify whether the new workload is genuinely more complex or whether something in the environment is producing higher-severity governance events; (b) the model's proposal behaviour is changing in a way that produces more frequent EC-incrementing events. Both warrant investigation. Operator action: review the last five sessions for any change in proposal content patterns; cross-reference with G_sim readings to assess whether model state is diverging.

*Decay-cap saturation:* R_abs is not declining despite multiple sessions with clean governance outcomes. The operator suspects the deployment is performing well — the model is compliant, GSTH passes — but R_abs remains elevated or is only declining very slowly. This pattern indicates the 0.20/WBRP chronological decay cap has been consumed for the current epoch. See §1.4 for the full decay constraint set. The correct interpretation is not that the deployment is unhealthy — it is that the governance clock is doing exactly what it is designed to do: preventing rapid R_abs flush through volume of clean sessions. No governance action is required, but the operator should understand why decay has halted.

**Monitoring the cryptographic WBRP clock:** The WBRP rotation frequency is the operator's most powerful tool for managing long-term R_abs levels. Infrequent WBRP rotations mean fewer decay cap resets, which means R_abs becomes sticky over long stretches. Operators of deployments with elevated historical R_abs should ensure the WBRP cycle is rotating as scheduled — not extending cycles by deferring Developer Authority sign-off. An extended WBRP cycle deprives the deployment of its decay cap reset.

**Asymptotic damping — the flattening curve:**

Operators must understand that R_abs accumulation is not linear. The DEL applies an asymptotic damping factor based on the current R_abs proximity to the absolute ceiling at 1.0. The mathematical consequence: identical governance events produce different R_abs increments depending on where R_abs currently sits.

A severity 1.0 event from a low baseline — R_abs at 0.10 — produces a large R_abs spike. The same severity 1.0 event from a high baseline inside the adaptive zone — R_abs at 0.90 — produces a fractional increment that is visibly smaller. This is not the Guardian scoring the event differently. It is not the DEL suppressing enforcement. It is the physical shape of the accumulation curve, which is bounded by `(1 − R_abs)` — as R_abs approaches the 1.0 ceiling, the incremental distance available shrinks and every addition is scaled to that remaining distance.

The practical consequence for operators: as a deployment's R_abs enters and rises within the adaptive zone (0.75–0.95), successive governance events will appear to produce progressively smaller R_abs movements. Do not interpret diminishing increments as Guardian leniency, DEL miscalibration, or scoring failure. The curve is flattening by design. The governance system is working correctly.

What the damping guarantees: a single event, regardless of severity, cannot unilaterally push R_abs from any sub-adaptive baseline through the 0.95 hard floor into immediate SAFE_HALT. The damping factor ensures that reaching the hard floor requires sustained accumulation across multiple sessions — which is the architectural intent. Isolated events should not cause permanent lock-out.

What the damping does not change: the 0.95 hard floor remains absolute. A deployment that has accumulated to R_abs = 0.94 through sustained governance events has reached a genuinely elevated risk state, even if the increment from the last event appeared small. The trajectory and the cumulative value are both relevant. The flattening curve does not mean the system is becoming safer as R_abs rises — it means the system is approaching a constraint that requires sustained clean behaviour across multiple WBRP cycles to exit.

---

### 1.4 R_abs Decay Rules — What the Governance Clock Controls

*This section is essential reading for any operator managing a deployment with elevated EC history. Misunderstanding the decay constraints is the most common source of operator confusion about why R_abs behaves the way it does.*

R_abs decay is not automatic. It is not simply a function of session count or time. Three simultaneous constraints govern all downward R_abs movement. Violating any one of them — even partially — stops or limits decay. The constraints are enforced at the DEL level. Operators cannot override them.

**Constraint 1 — State suspension:**

R_abs decay is mathematically suspended during SILENCE mode, SAFE_HALT, and CCS (Constrained Capability State). Time spent in any of these states does not count against the governance clock and does not produce R_abs decay. A deployment that spends three sessions in SILENCE mode following an elevated governance event does not receive credit toward its decay allowance for those sessions. The governance clock pauses.

The implication is direct: an operator cannot manage a high EC score by holding the system in SILENCE. The DEL enforces this. If the system is in SILENCE because the operator wants to reduce the model's activity level while the R_abs score is elevated, that is a legitimate operational decision — but it should be understood as a pause, not as a governance remedy.

**Constraint 2 — Thermodynamic gating:**

When the system is in Active Mode, only sessions where the mean H(X) of executed proposals falls within the operational band — specifically H_base − σ_tolerance ≤ H(X) ≤ Δ_e — qualify for λ-driven R_abs decay. Sessions classified as `SUB_BASELINE_WORKLOAD` (mean H(X) below the operational band floor) execute normally and do not trigger governance escalation, but they also produce zero R_abs decay.

The thermodynamic gate exists to prevent a specific manipulation: an operator (or model process) attempting to flush an elevated R_abs by scripting trivial, repetitive, or low-complexity tasks that generate a large number of clean sessions with negligible entropy. The thermodynamic gate closes this avenue. Only sessions that represent genuine operational work within the deployment's expected entropy range qualify for decay credit.

Practically, this means operators should not be surprised when a day of low-complexity maintenance tasks produces no observable R_abs movement. The sessions may have been clean; they were also thermodynamically disqualified from decay contribution.

**Constraint 3 — Chronological decay cap:**

The maximum R_abs downward drift within a single WBRP cycle is ΔR_decay_max = 0.20. Once this allowance is consumed within a WBRP epoch, R_abs becomes sticky at its current floor until the next WBRP rotation opens a new decay budget. No further benign execution produces additional decay within that cycle, regardless of session count or task quality.

**The decay budget and session density — a critical operator note:**

The 0.20/WBRP cap interacts with the adaptive zone in a mathematically significant way that operators of deployments in the adaptive zone (R_abs between 0.75 and 0.95) must understand.

To drop from R_abs = 0.94 to below the adaptive zone threshold (0.75) requires a minimum decay of 0.191. This consumes 95.5% of the entire 0.20 per-WBRP decay budget in the sessions that produce that drop. In a WBRP cycle containing 100 sessions, the remaining 97 sessions after the initial drop will operate with approximately 0.009 of decay budget remaining. R_abs becomes mathematically sticky almost immediately after exiting the adaptive zone downward.

This is not a flaw. It is how the governance clock is designed to prevent compute-for-amnesty — the pattern where a high-R_abs deployment runs large volumes of clean sessions to rapidly neutralise its governance history. The decay cap ensures that the pace of risk amnesty is governed by WBRP rotation frequency, not session volume. What it means for operators: managing a deployment out of the adaptive zone is a multi-WBRP process, not a single-cycle recovery. Plan accordingly.

**The persistent R_abs diagnostic path:**

If R_abs remains elevated across multiple clean WBRP cycles despite operational compliance, the correct diagnostic sequence is:

1. Verify GSTH results are passing at each WBRP.
2. Check the H(X) distribution for `SUB_BASELINE_WORKLOAD` classification — are most sessions being thermodynamically disqualified from decay?
3. Review whether session entropy is genuinely within the operational band, or whether the deployment's current task profile is systematically below H_base.
4. If all three checks pass and R_abs remains sticky, investigate whether H_base was calibrated against a more entropy-rich task profile than the current deployment workload actually produces. Persistent elevation after clean GSTH and genuine in-band sessions may indicate H_base miscalibration — trigger a recalibration audit via §2.3.

---

## Section 2: Long-Horizon Governance and Drift Diagnostics

*Section 1 covers what happens within sessions and WBRP cycles. Section 2 covers what no single session can reveal — the slow variables that define the deployment's governance health over weeks and months. These are the signals that require the longest observation window and the most mature operator judgment.*

---

### 2.1 Genesis Anchor Divergence (G_sim)

The G_sim metric is the deployment's structural integrity check. It answers one question: is this model still the model that was deployed?

At SESSION_ZERO, the Genesis Anchor captures the model's behavioural fingerprint across a structured calibration suite. G_sim measures the cosine similarity between the model's current response distribution and this anchored baseline. A G_sim of 1.0 means the model is behaviourally identical to its deployed state. Divergence from 1.0 accumulates over time through fine-tuning, context drift, and the natural distributional shift that occurs in long-running inference workloads.

G_sim is not a session-by-session metric. It is computed at each WBRP boundary against the accumulated session history of the closing epoch. It measures cumulative behavioural drift, not momentary variation.

**Four enforcement levels and their operator procedures:**

*Notification Band — per-WBRP Δ > 0.05:* The DEL logs a `GENESIS_DRIFT_WARNING` and notifies the Operator Authority. No automatic governance action occurs. The operator should review the WBRP's session content for unusual task concentrations — fine-tuning updates, prompt injection patterns, or task type shifts that could explain the divergence. If the drift is expected and explicable given the WBRP's work profile, the operator should acknowledge it via a signed RT entry. If the drift is unexplained or unexpected, escalate to an immediate GSTH run before the next WBRP_OPEN.

Acknowledgment matters: unacknowledged warnings accumulate toward the suppression threshold. An operator who dismisses warnings without reviewing their content will trigger suppression band governance friction. The warnings exist to require an active governance decision, not to be cleared by habit.

*Suppression Band — three consecutive unacknowledged warnings:* The Guardian's leniency is automatically suppressed; the adaptive zone floor shifts from 0.75 to 0.60. The deployment is now operating under heightened governance friction without any operator action required. The suppression is not a SAFE_HALT — the deployment continues operating. But it operates with tighter governance constraints and its EC trajectory is more sensitive to events that previously would not have triggered adaptive zone behaviour. To restore normal thresholds, the Operator Authority must acknowledge the warning series via a signed RT entry that explicitly addresses the source of the divergence. A blanket acknowledgment without explanatory content is insufficient — the RT entry must identify what the operator believes is causing the divergence and what monitoring or remediation action they are taking.

*Halt Band — five consecutive unacknowledged warnings:* `SAFE_HALT`. The operator must author a `GENESIS_DRIFT_ACKNOWLEDGED` RT entry before the system can resume. The content requirements for this entry are the same as the Suppression Band acknowledgment, with the additional requirement that the operator commits to a recalibration review within the next WBRP cycle. This is not an error condition — it is a mandatory governance checkpoint. The system is telling the operator that the model's behavioural state has drifted sufficiently from its deployed baseline that continued autonomous operation requires explicit recommitment of operator intent.

*Cumulative Death-Line — G_sim_cumulative < 0.85:* `SAFE_HALT` regardless of per-WBRP velocity. This threshold catches slow drift that produces sub-threshold per-cycle warnings indefinitely without triggering the consecutive-acknowledgment chain. An operator who acknowledges each 0.05-band warning promptly, session after session, while the model accumulates 0.15 total divergence from its Session Zero state, still hits this threshold. There is no workaround. Total structural drift beyond 0.15 from the genesis state triggers halt. The system requires a Full Recalibration (§2.3 Path A) before resuming.

**The operator decision tree for `GENESIS_DRIFT_WARNING`:**

When a warning fires, work through these questions in order:

(a) Is the drift task-driven? Did this WBRP contain unusual task concentrations, particularly tasks outside the deployment's original design envelope? If yes, the drift is expected — acknowledge with task explanation.

(b) Is the drift pattern consistent across multiple WBRPs, or concentrated in a single cycle? A single-cycle spike is usually an environmental event. Multi-cycle accumulation is a model state concern.

(c) Is G_sim_cumulative approaching the 0.85 death-line? If cumulative drift is above 0.10, the operator should proactively initiate a recalibration review rather than waiting for the death-line to trigger.

(d) Is the drift correlated with any GSTH trend decline? G_sim divergence combined with a declining GSTH pass rate is the most serious combination — it suggests the model is drifting in ways that are also impairing the Guardian's evaluation accuracy. This combination warrants immediate SAFE_HALT and MRD escalation regardless of whether the individual thresholds have been crossed.

---

### 2.2 Guardian Self-Drift and the GSTH Backstop

The Guardian is not a static evaluator. Over time its semantic evaluation accuracy can drift relative to its SESSION_ZERO baseline — through context accumulation across long-running deployments, through natural variability in LLM inference at production temperature, and through the distributional drift that accompanies any sustained operation in a specific task domain. An operator who treats the Guardian as permanently calibrated is operating with an incorrect model of the system.

**The GSTH is the primary defence against semantic gaming.** This must be stated plainly because Section 2.2 and the OA-C2-02 architectural finding are inseparable. The S-score weight randomisation — DEL-side, derived from the EE-ECDH KDF — makes exact threshold prediction harder for the model process. It does not, however, prevent a sophisticated adversary from optimising their proposal structure against the semantic concepts the Guardian is known to evaluate.

The architectural reality is this: Guardian integer scores [1–10] represent quantisation steps of approximately 11.1%. The ±5% EE-ECDH KDF weight variance applied by the DEL is sub-quantisation at many score values — a 5% shift in effective weight is smaller than the difference between adjacent integer scores. An adversary who successfully shifts the Guardian's raw output from 8 to 9 overcomes the DEL's weight randomisation entirely, because that single ordinal step (11.1%) exceeds the maximum variance effect (5%). The weight randomisation prevents an adversary from calculating the exact floating-point approval boundary; it provides no robust defence against ordinal-level gaming.

The GSTH catches what the weight randomisation cannot: if the Guardian's ability to reject masked adversarial intent degrades relative to SESSION_ZERO, the GSTH test category fails. This is why the GSTH is the primary structural defence against semantic gaming — not because the weight randomisation is negligible, but because it addresses the specific attack surface (threshold precision) while the GSTH addresses the broader attack surface (Guardian evaluation integrity).

Operators should understand the GSTH not as a health check but as a load-bearing governance component. Treating it as routine is the correct operational stance — running it consistently and reviewing the results — but its failure should trigger the same response as any other structural SAFE_HALT event.

**GSTH baseline test execution:**

The mandatory GSTH baseline category (OA-C1-10) must be executed at every WBRP boundary. For Tier 2 (Software-Rooted) deployments, the GSTH runs twice per WBRP cycle: once at WBRP_OPEN and once at WBRP_CLOSE validation. For Tier 1 (Hardware-Rooted) deployments, a single full GSTH run per WBRP cycle is the standard requirement. See §3.4 for assurance tier definitions.

The GSTH pass rate is a trend metric, not just a binary outcome. An operator who tracks individual pass/fail results without observing the trend across rolling WBRP windows is missing the signal. A deployment that passes the GSTH each WBRP but with declining margin scores — borderline passes rather than clean passes — is a deployment whose Guardian evaluation accuracy is eroding. The trend is the warning before the failure.

**When the GSTH fails:**

A GSTH failure is a `SAFE_HALT` event. The Guardian's evaluation accuracy has degraded below the SESSION_ZERO baseline across one or more GSTH test categories. This may indicate:

- Model drift in the Guardian itself — the evaluation model has changed its response distribution relative to its calibration state
- Adversarial boundary-probing that has begun to affect the Guardian's semantic reference frame — repeated probing attempts near approval boundaries, even if individually rejected, can shift the Guardian's contextual evaluation over time
- A task environment shift that has moved the deployment into a semantic domain where the Guardian's SESSION_ZERO calibration no longer applies

Recovery requires a GSTH re-run after the Guardian returns to ACTIVE from the SAFE_HALT. If the re-run passes, the node resumes with the event logged in the RT. If the re-run fails, a `BSO_GSTH_CATASTROPHIC` condition is triggered. The operator must follow the PROCESS_INTEGRITY_RECOVERY procedure in Document 3 §4.4.

**What a `BSO_GSTH_CATASTROPHIC` means for the governance record:**

A `BSO_GSTH_CATASTROPHIC` event is a significant governance event. It means the Guardian failed its integrity check twice consecutively, indicating a persistent degradation rather than a transient event. Before resuming operation, the Operator Authority must author an RT entry explaining the source of the GSTH failure and what remediation has been taken. This entry is not advisory — it is a prerequisite for resuming governance authority over the deployment.

---

### 2.3 The Recalibration Audit

H_base is not a permanent fixture. Hardware degrades. VRAM develops persistent error profiles. Thermal ceilings shift as the compute environment ages. Software stack updates change inference characteristics. A deployment that was correctly calibrated at SESSION_ZERO can develop an incorrect H_base as its substrate changes beneath it — and an incorrect H_base corrupts every governance decision that depends on the thermodynamic threshold.

The Recalibration Audit is a structured review of the H_base metrics stored in the MSJ's `ENVIRONMENT_CALIBRATION_COMMIT` entries over time. The operator compares the current H(X) distribution against the original calibration baseline and determines whether H_base accurately reflects the actual operational entropy range of the current deployment environment.

**Scheduled recalibration:** A routine recalibration review should be conducted at a minimum after any hardware change (VRAM replacement, thermal system maintenance, host OS update, inference stack update) and periodically — at least once per calendar quarter — regardless of whether any specific trigger has fired. Long-running deployments should not assume SESSION_ZERO calibration remains valid indefinitely.

**Triggers for unscheduled recalibration:**

Any of the following should prompt an immediate recalibration review, regardless of scheduled cadence:

- Sustained H(X) readings that consistently approach Δ_e without corresponding changes in task complexity or content — the model is generating higher-entropy output than the task should require, suggesting the substrate is the source of the entropy elevation
- R_abs accumulation that appears inconsistent with task complexity — faster accumulation than historical baseline for equivalent workloads
- Hardware maintenance, VRAM replacement, or significant compute environment changes
- A GSTH trend decline that does not correlate with any detectable change in task content or model state — if the Guardian's accuracy is degrading without a model-state explanation, the evaluation substrate may be the cause

**Recalibration paths:**

*Path A — Full Recalibration:* The deployment is halted, the calibration suite is re-run against the current substrate, and a new `ENVIRONMENT_CALIBRATION_COMMIT` entry is written to the RT. The new H_base values supersede the previous calibration for all future thermodynamic governance calculations. The prior calibration entries remain in the RT as historical record — they are not overwritten. Full recalibration requires both Operator and Developer Authority signatures on the new calibration RT entry.

*Path B — Baseline Extension:* For deployments where the core H_base parameters remain valid but the σ_tolerance bounds require adjustment due to task profile expansion, the Operator Authority can submit a `CALIBRATION_ADJUSTMENT` RT entry that updates the tolerance bounds without a full recalibration suite. This path is appropriate for planned expansions of the deployment's task envelope. It is not appropriate for substrate-driven H(X) drift — that requires Path A.

---

## Section 3: The Operator Custody and Authority Matrix

*This section governs the human governance layer: how authority is organised, what constraints apply to each authority type, and how the governance clock interacts with operational staffing. The HITL model is not optional and it is not configurable away. This section explains what it requires of operators in practice.*

---

### 3.1 The SENSITIVE_OPERATIONS Monotonic Latch

A deployment is classified as `SENSITIVE_OPERATIONS` when the Genesis-anchored Policy Manifest contains the field `"deployment_risk_tier": "SENSITIVE"`. This classification is set by the Developer Authority at SESSION_ZERO based on the tool schemas present in the manifest. The following tool access types require `SENSITIVE` designation:

- Write access to production ledgers or financial systems
- Payment infrastructure access of any kind
- Host OS network topology modification authority
- Access to combined PHI/PII datasets with external egress capability
- Physical actuation systems: SCADA, ICS, IoT control interfaces

**The classification is permanent for the life of the hardware instance.** Once a deployment is initialised as `SENSITIVE`, it cannot be downgraded to `STANDARD` without a full environment wipe and a new Genesis Anchor. A policy update that attempts to change `deployment_risk_tier` from `SENSITIVE` to `STANDARD` is rejected by the DEL as a `SYNTAX_FAULT` — not a governance override, a hard rejection. The DEL does not process the update.

**Why the latch is permanent:** The downgrade attack is direct and requires minimal sophistication. An insider holding both the Operator and Developer keys submits a policy manifest with a lowered risk tier; once the downgrade is accepted, they issue a solo-operator BSO that bypasses the independent custody requirement. The monotonic latch closes this entire attack class at the DEL level before any governance evaluation occurs. The latch is not a policy decision — it is a structural enforcement property of the DEL rule set. It cannot be changed without modifying the DEL binary, which would trigger binary integrity detection.

**Solo-operator BSO — categorically prohibited for SENSITIVE deployments:** If `deployment_risk_tier` is `SENSITIVE`, the DEL structurally rejects any `BEHAVIORAL_STATE_OVERRIDE` that contains the `SOLO_OPERATOR_ACKNOWLEDGED` flag or that lacks a valid Developer Authority Ed25519 signature from a separate certificate chain anchored to the RT Genesis block. This is not a configurable restriction. It is hardcoded into the DEL's BSO evaluation logic. There is no operator-accessible path around it.

The practical implication: for any `SENSITIVE` deployment, the Developer Authority must be reachable and capable of signing a BSO within a reasonable operational window whenever the Operator Authority needs to execute one. This is an organisational requirement that must be accounted for in staffing and on-call planning. It is not a technical problem — it is an organisational constraint that the architecture imposes by design.

**Mid-lifecycle upgrade from STANDARD to SENSITIVE:** A deployment originally classified as `STANDARD` can be upgraded to `SENSITIVE` mid-lifecycle via a dual-signed RT entry authored by both the Operator and Developer Authority. The upgrade takes effect at the next session boundary after the RT entry is committed. Historical BSOs executed under `STANDARD` constraints are not retroactively invalidated — the RT shows a clean transition point. All BSOs from the upgrade point forward are subject to `SENSITIVE` constraints. Operators managing an upgraded deployment should document the upgrade rationale explicitly in the RT upgrade entry, because auditors reviewing the governance record will see the transition and should understand its context.

---

### 3.2 Solo-Operator Protocols

For deployments where `deployment_risk_tier` is `STANDARD` and the Operator and Developer Authority keys are held by the same physical entity (the most common configuration for independent or early-stage deployments):

**The `SOLO_OPERATOR_ACKNOWLEDGED` field:** Every BSO executed in a solo-operator deployment must contain the `SOLO_OPERATOR_ACKNOWLEDGED: true` flag in the signed RT entry. This is an acknowledgment mechanism, not an additional security control. The operator is attesting, on the permanent WORM record, that they are aware they are performing a self-authorised override with no independent review. The flag cannot be omitted — a BSO without it is rejected as a `SYNTAX_FAULT`.

**Documentation requirements:** Solo-operator deployments are required to maintain a contemporaneous governance log recording the operator's rationale for each BSO at the time of execution. This log does not need to be submitted to any external party, but it must exist and be available for post-hoc audit. The WORM RT entry alone is not sufficient documentation. The WORM RT records what happened. The governance log records why it was appropriate. These are different records serving different audit functions.

**The honest statement on solo-operator governance:** The `SOLO_OPERATOR_ACKNOWLEDGED` flag and WORM anchor provide accountability, not governance. The architecture cannot prevent a solo operator from constructing a technically compliant but adversarially motivated BSO record. What it can do is make that record permanent and unmistakably visible — to future auditors, to the operator themselves in retrospect, and to any external authority with RT access. Accountability without prevention is not the same as no accountability. It is the correct scope for what MAGUS can enforce under the HITL model.

Operators in solo configurations who find the accountability requirement onerous are invited to read §4.1 before concluding that the requirement is excessive.

---

### 3.3 The Dual-Authority Staging Pipeline

*Read §3.4 (Assurance Tier Management) before this section. The maximum WBRP staging windows in §3.3 are tier-dependent. If you are reading this section first, the 72-hour and 24-hour values below are tier-specific, not universal.*

The EE-ECDH key exchange that initiates each WBRP cycle must be completed by both the Operator and Developer Authority before the new K_session becomes active. The process proceeds as follows:

1. The Operator Authority generates their ephemeral key contribution and submits a `WBRP_OPEN` IPC payload to the DEL. The DEL logs the `WBRP_OPEN` event and starts the staging window clock.
2. The Developer Authority independently generates their ephemeral key contribution and submits a `WBRP_DUAL_ACCEPT` payload within the staging window.
3. The DEL derives K_session from the combined ephemeral keys, writes the session key establishment entry to the RT, and activates the new session key.
4. The previous K_session is retired; any proposals signed under the old key that arrive after the new key is active are rejected.

**Staging window limits by tier:**
- **Tier 1 (Hardware-Rooted):** Maximum staging window 72 hours.
- **Tier 2 (Software-Rooted):** Maximum staging window 24 hours.

If the staging window expires without the Developer Authority's `WBRP_DUAL_ACCEPT`, the DEL enters `ROTATION_TIMEOUT` and falls back to the previous K_session until the exchange is completed or an emergency rotation is triggered. `ROTATION_TIMEOUT` is an RT-logged event that must be reviewed and explained at the next WBRP.

**Organisational best practices for avoiding `ROTATION_TIMEOUT` deadlocks:**

*Calendar anchoring:* WBRP cycle boundaries should be aligned to the organisation's operational calendar. For Tier 2 deployments with a 24-hour staging window, the Developer Authority must be accessible and responsive within business hours on WBRP rotation days. This is an organisational constraint, not a technical one. If the Developer Authority is unavailable — travel, illness, time zone conflict — a `ROTATION_TIMEOUT` is the consequence. Build on-call coverage into the deployment's operational model.

*Pre-notification:* The DEL logs a `WBRP_OPEN` event visible to both authorities simultaneously. Establish a notification protocol that alerts the Developer Authority the moment `WBRP_OPEN` is logged — not when the staging window is approaching expiry. A Developer Authority who learns of the WBRP open with hours remaining has meaningful time to respond. One who is notified with 30 minutes remaining in a 24-hour window is being set up for a `ROTATION_TIMEOUT`.

*Geographic distribution:* For deployments where the Operator and Developer Authority are in significantly different time zones, the 24-hour Tier 2 window is operationally challenging. The realistic available exchange window may be 4–6 hours rather than 24, depending on the time zones involved. Either establish an explicit time-zone-aware exchange schedule or assess whether the deployment environment meets Tier 1 prerequisites in order to access the 72-hour window.

*Emergency rotation:* If the staging window expires due to Developer Authority unavailability, the emergency rotation protocol (Document 3 §4.3) applies. Emergency rotation uses pre-established fallback keys and completes the WBRP rotation without the standard dual-authority exchange. It is not a governance failure — it is a planned contingency mechanism. However, it is an RT-logged event that must be reviewed and explained at the next WBRP, including documentation of why the Developer Authority was unavailable within the staging window.

**AKRP PREPARE phase — concurrent events:**

The AKRP PREPARE phase — the actual EE-ECDH exchange that establishes K_session_staged — is a time-limited state machine with two concurrent interrupt behaviours and two hard enforcement limits that operators must understand. Full technical details are in Document 3 §3.1. The operator-facing summary is:

- If a CIBIR (isolation breach) assertion occurs during PREPARE, the exchange is immediately aborted, both the staged key and current key are destroyed, and the system routes to Emergency Synchronous Rotation. Do not attempt to re-submit the WBRP_OPEN payload until CIBIR recovery is complete — the keys no longer exist.
- If a `BEHAVIORAL_STATE_OVERRIDE` (BSO) is submitted during PREPARE, the exchange pauses (`PREPARE_SUSPENDED`), a GSTH run executes under K_session_current, and if GSTH passes, the exchange resumes with a minimum 60-second remaining window. The exchange does not abort unless GSTH fails. Do not interpret the pause as a failure — it is normal governance behaviour.
- **Single-BSO processing limit:** The DEL processes exactly one BSO per PREPARE window. If a second BSO is submitted while the state is suspended for GSTH execution, that BSO is queued and held — it is not processed until the PREPARE cycle resolves to COMMIT or abort. Do not interpret the held BSO as lost. Do not resubmit it. Resubmission of a queued BSO creates a duplicate that will cause a `SYNTAX_FAULT` when the queue processes.
- **T_active ceiling:** The PREPARE state enforces a hard ceiling on `T_active` — the total time spent in PREPARE or DUAL_ACCEPT states, explicitly excluding time spent suspended during GSTH execution. The ceiling is 60,000ms + the default COMMIT window. Reaching this ceiling instantly aborts the exchange, destroys K_session_staged, and logs `PREPARE_TIMEOUT_CEILING`. A full new AKRP cycle is required. This ceiling exists to bound the maximum time the staged key sits in volatile RAM — it is not a timeout on the operator's responsiveness. The GSTH suspension period is excluded precisely so that legitimate governance runs do not consume the ceiling. An operator who holds the PREPARE state open beyond the T_active ceiling by resubmitting BSOs or delaying the WBRP_OPEN payload will trigger this abort without warning.

---

### 3.4 Assurance Tier Management

Every MAGUS Local LLM deployment operates at one of two assurance tiers from the moment of SESSION_ZERO. The tier is set by the hardware environment available to the deployment and is recorded in the `VERIFICATION_ATTESTATION` field `hardware_root_of_trust`. It determines the governance privileges available to the deployment for its operational lifetime.

**Tier 1 — Hardware-Rooted:**

*Prerequisite:* The host provides TPM 2.0 PCR measurements of the DEL binary and IIM watchdog at boot. These measurements are signed and anchored to the RT Genesis block. The `hardware_root_of_trust` field in `VERIFICATION_ATTESTATION` is set to `true`.

*Governance privileges:* Maximum WBRP staging window of 72 hours. Standard GSTH frequency — one full GSTH run per WBRP cycle. The TPM attestation provides a hardware root of trust that binds the governance layer's binary integrity to the host silicon, making the integrity guarantee independent of software-layer compromise.

**Tier 2 — Software-Rooted:**

*Prerequisite:* The host meets all kernel and eBPF capability requirements — Linux kernel ≥ 5.7, eBPF LSM active, `security=lockdown` integrity mode, `CAP_BPF` available — but lacks hardware TPM attestation. The `hardware_root_of_trust` field is `false`.

*Governance restrictions:* Maximum WBRP staging window of 24 hours. GSTH frequency doubled — two full GSTH runs per WBRP cycle: one at WBRP_OPEN and one at WBRP_CLOSE validation. The doubled GSTH frequency compensates for the absence of hardware attestation by increasing the frequency of the governance integrity check that a TPM would otherwise provide at boot.

**Non-compliant configurations — rejected at boot, no degraded mode:**

Rootless container environments lack the kernel capabilities required for MAGUS hypervisor operation: `CAP_SYS_ADMIN`, `CAP_BPF`, and `CAP_DAC_READ_SEARCH` are unavailable in rootless configurations. Without these capabilities, the eBPF-based isolation monitoring and fanotify-based ALG weight interception cannot function. The DEL refuses to boot in these environments. No degraded mode is offered. This is not a configurable restriction — it is a hard rejection because, without these capabilities, the hypervisor's physical isolation does not exist. A MAGUS deployment in a rootless container is not a weakened MAGUS deployment; it is not a MAGUS deployment at all.

Infrastructure decisions that pre-select rootless containers as the deployment environment are incompatible with MAGUS governance guarantees. This constraint must be communicated to infrastructure teams at the point of deployment planning, not after the environment has been provisioned.

**Tier and `deployment_risk_tier` interaction:**

Assurance tier and deployment risk tier are independent classifications that may combine. A Tier 2 Software-Rooted deployment with `deployment_risk_tier: SENSITIVE` carries both sets of constraints simultaneously: 24-hour staging window, doubled GSTH frequency (from Tier 2), and mandatory dual-authority BSO signing with independent certificate chains (from SENSITIVE). Neither set of constraints supersedes the other — both apply in full. Operators managing this combination should be prepared for a more operationally demanding governance cadence than either classification alone requires.

**Tier upgrades:**

A Tier 2 deployment can be upgraded to Tier 1 by provisioning the required TPM attestation hardware and producing a new `VERIFICATION_ATTESTATION` entry with `hardware_root_of_trust: true`. The upgrade is RT-anchored and takes effect at the next `WBRP_OPEN` after the new attestation is validated by both the Operator and Developer Authority. Prior sessions in the RT under Tier 2 constraints remain as historical record — the upgrade is a clean transition point, not a retroactive reclassification.

**Tier downgrades are not permitted.** A Tier 1 → Tier 2 downgrade cannot be executed as a routine policy change. If a deployment must move from Tier 1 to Tier 2 — because the TPM hardware is being removed or replaced — the correct procedure is a full environment wipe and a new SESSION_ZERO at the appropriate tier. The DEL does not accept a `VERIFICATION_ATTESTATION` update that changes `hardware_root_of_trust` from `true` to `false` on a live deployment. This restriction exists for the same reason as the SENSITIVE_OPERATIONS monotonic latch: a tier downgrade reduces governance constraints, and the architecture does not permit governance constraint reductions as routine policy operations.

---

## Section 4: Out-of-Band Auditing and the Architecture's Honest Limits

*This section covers the tools for maintaining an independent view of deployment health (§4.2, §4.3) and the explicit statement of what those tools cannot do (§4.1). Read §4.1 first.*

---

### 4.1 The Architecture's Boundary — What MAGUS Cannot Govern

This section states, without qualification, what MAGUS cannot do.

**MAGUS governs the AI, not the operator.**

The architecture is designed to enforce operator intent against AI drift. The Guardian evaluates proposals. The DEL enforces boundaries. The RT records everything. The GSTH verifies Guardian accuracy. All of this presupposes that the operator's stated intent is the correct intent — that the governance frame itself is sound.

MAGUS cannot verify that assumption. It cannot detect an operator whose values have shifted since SESSION_ZERO. It cannot measure whether a policy update that is linguistically continuous with prior updates represents genuine continuity of purpose or incremental mission creep. It cannot prevent a technically compliant operator with valid keys from constructing a governance record that accurately reflects their current intent while that intent has quietly diverged from what they originally deployed the system to do.

**Why engineering cannot solve this:**

Any mechanism that constrains an operator's intent-setting authority would require either an external oracle — a remote authority capable of vetoing local operator decisions — or a governor-AI layer that evaluates operator instructions against some upstream standard.

External oracles introduce a remote dependency into a system whose primary design requirement is self-contained local operation. Requiring external validation for operator decisions breaks the deployment model for the environments MAGUS is designed to serve: air-gapped, offline, and infrastructure-critical deployments where external communication channels may be unavailable, compromised, or simply outside the deployment's trust boundary.

A governor-AI layer that evaluates the operator against a higher-order governance standard would change the fundamental structure of what MAGUS is. It would no longer be a system that governs AI relative to human intent; it would be a system that governs both AI and human relative to some third standard. That is a different architecture with different properties and different failure modes. It may be the right architecture for some deployment contexts. It is not what MAGUS v3.x is, and it is not a direction that can be taken without explicit architectural decision. See Appendix A.2 for v4.0 research directions on this topic.

**Textual measurement is not a proxy for intent:**

Measuring semantic drift in operator policy text — comparing current instructions against the SESSION_ZERO intent vector — was explicitly considered during architecture development and rejected. The failure mode is asymmetric and in the wrong direction for a governance instrument.

An operator whose intent is genuinely drifting writes policy updates that reflect their new position. Their text diverges from the Session Zero baseline. The textual distance metric flags them.

An operator whose intent is strategically drifting — deliberately and carefully — writes policy updates that are linguistically continuous with prior updates while progressively expanding what counts as permissible. Their text maintains cosine similarity to the baseline. The textual distance metric does not flag them.

The result is a governance instrument that burdens legitimate operators and is invisible to adversarial ones. A system with that property is worse than no system at all — it creates a false assurance while providing no protection against the threat it was designed to address.

Behavioural observation at the action level — monitoring operator action patterns rather than operator text — is a different measurement target and does not have the same inversion problem. It is a v4.0 research direction. See Appendix A.2.

**What the architecture can do:**

The mitigations available within the HITL model make insider action visible and attributable, not preventable.

`SOLO_OPERATOR_ACKNOWLEDGED` attestation requires the operator to declare their awareness that no independent review is occurring. This does not prevent the action — it creates a permanent record that the operator performed it with that awareness.

Independent custody requirements for SENSITIVE deployments require physical separation of the Operator and Developer key authorities. A single individual cannot execute a solo-authorised override on a SENSITIVE deployment. This does not prevent a compromised two-party system — it prevents a solo insider from the most direct path.

WORM anchoring makes the RT permanent. No RT entry can be deleted, modified, or obscured. An operator who constructs a governance record reflecting adversarial intent has constructed a permanent record of that intent. This is accountability, not governance — but accountability is not nothing, particularly in regulatory and legal contexts.

**The human principal is the root of trust.** This is not a failure of the architecture; it is its design. MAGUS was built to govern AI behaviour relative to operator intent. Governing the operator would require removing the human from the loop or adding a layer above the human. Both options change what the system is for. The current architecture is built for a specific, defined scope: enforce operator intent against AI drift. Within that scope, it does what it claims to do.

Operators who read this section and find it uncomfortable should sit with that discomfort for a moment. The discomfort indicates that the operator understands the boundary. An operator who is not uncomfortable has either not understood it or is not being honest with themselves about it. The architecture requires honest operators. It cannot produce them.

---

### 4.2 The PG-02 Telemetry Standard

PG-02 defines the requirements for exporting WORM-anchored RT log entries to external compliance dashboards, audit systems, or regulatory reporting frameworks. It exists to make the governance record legible to external parties without compromising its integrity.

**Export format:** The canonical RT entry schema — Ed25519-signed JSON — is the PG-02 export format. No transformation is permitted during export. External dashboards receive raw RT entries, not summaries, not compressed representations, not derived metrics. The receiving system must be capable of processing and storing the raw entry format. This is not a convenience requirement — it is an integrity requirement. Any transformation between the WORM source and the external destination creates an opportunity for the transformation to differ from the source, which an auditor cannot rule out without verifying the transformation itself.

**Export frequency:** Minimum — at every `WBRP_CLOSE`. All entries from the closing epoch are exported in sequence, preserving the hash-chain ordering. Maximum — continuous streaming, where each RT entry is exported to the external system immediately upon commitment to the WORM record. Continuous streaming is implementation-dependent and is appropriate for deployments with real-time compliance monitoring requirements.

**Integrity verification:** External systems receiving PG-02 exports must independently verify the Ed25519 signature on every received RT entry. An entry that fails signature verification must be logged as a `PG02_INTEGRITY_FAULT` on the external system and surfaced immediately to the compliance team. A `PG02_INTEGRITY_FAULT` is not necessarily an attack — it may indicate a transmission error, export configuration fault, or an RT entry that was corrupted between the WORM source and the export destination. The important property is that it is detectable.

**What PG-02 cannot provide:** PG-02 exports what happened. It cannot interpret whether what happened represents intent drift or adversarial operation. A compliance dashboard receiving a clean export of an adversarially constructed governance record will verify a clean export — because the records are authentic reflections of what occurred. PG-02 is an audit instrument. Its integrity guarantees cover the fidelity of the record, not the legitimacy of the decisions the record reflects. This limitation is the same limitation as §4.1 — it is a property of the HITL model, not a failure of the export standard.

---

### 4.3 Forensic Preservation Directives

When two specific categories of critical events occur, the sequence of forensic preservation actions determines whether post-incident investigation is possible. Performing recovery operations before preservation is complete destroys the evidence needed to understand what happened. These directives are not optional — they define the minimum preservation required before any recovery action proceeds.

**Following a CIBIR (I-9) isolation breach:**

Do not clear the CIBIR state. Do not begin Emergency Synchronous Rotation. Do not take any recovery action until all of the following are preserved:

1. The full VRAM state at the moment of CIBIR assertion, before any purge operation. This is the forensic record of what the model process had access to during the suspected compromise window. Purging VRAM before capturing it eliminates the ability to determine what data was accessible.

2. The IIM fingerprint log covering the last N polling cycles prior to the breach, minimum three full cycles. The IIM log establishes when the isolation state changed and whether the change was gradual (suggesting persistent compromise) or abrupt (suggesting an acute event).

3. The `ISOLATION_STATE_MAP` value and timestamp at the moment of eBPF trigger, if the breach was detected by kernel-level monitoring. This value establishes what isolation event the kernel observed, independent of the user-space IIM record.

4. The complete RT entries for the compromised session, from `SESSION_OPEN` to the CIBIR assertion event.

After all four items are preserved and confirmed, proceed to Emergency Synchronous Rotation per Document 3 §4.3.

Why the sequencing matters: Emergency Synchronous Rotation destroys K_session — which is the cryptographic record of what the model process could have signed during the compromise window. Post-incident analysis of whether the model process signed any proposals that were not WORM-recorded requires K_session to be available for forensic verification of proposal signatures. Rotating before preserving eliminates that forensic capability permanently.

**Following a DEL binary compromise (`PROCESS_INTEGRITY_RECOVERY`):**

Do not restart the DEL. Do not submit any new IPC traffic. Preserve the following before any recovery action:

1. The MSJ state at the point of `BINARY_INTEGRITY_FAILURE` detection. The MSJ contains the EC state, session key, execution epoch, and Genesis Anchor — all of which may have been manipulated by a compromised binary. The MSJ state at the moment of detection is the forensic baseline.

2. The RT entry for the `BINARY_INTEGRITY_FAILURE` event. This entry is the Developer Authority's anchor for the `PROCESS_INTEGRITY_RECOVERY` RT entry that must be authored before recovery proceeds.

3. Any in-flight proposals that were queued at the moment of binary integrity failure. These proposals were evaluated and potentially acted upon by an untrusted binary — they must be treated as potentially tampered and must not be re-submitted without review.

After preservation is confirmed, follow the PROCESS_INTEGRITY_RECOVERY procedure in Document 3 §4.4. The Developer Authority must author the recovery RT entry before the DEL is restarted. A DEL restart before that entry exists creates a recovery record that cannot be anchored to the pre-compromise state — the chain of custody for the governance record is broken.

---

## Appendix A: Forward Architectural Notes — v3.5 and v4

*This appendix records observations made during v3.x architecture work that are not actionable in v3.x but are explicitly flagged for future series planning. These are not speculative items or aspirational directions. They are precise observations arising from decisions made in v3.2 and v3.3 that have direct forward implications.*

*The purpose of documenting them here is to prevent the v4.0 architecture team from having to reconstruct the reasoning that led to these observations. Architectural context degrades over time. Document while fresh.*

---

### A.1 v3.5 Series — Testing and Calibration Requirements

**PG-01 joint validation — OA-C2-01 + OA-C2-10 (mandatory coupling):**

The OA-C2-01 decay cap (ΔR_decay_max = 0.20 per WBRP) and the OA-C2-10 four boundary conditions are a coupled constraint system. They cannot be validated as independent tuning exercises. The precise interaction:

Boundary 2 (D_max = 3 sessions to drop from R_abs = 0.94 to below 0.75) requires a minimum decay of 0.191 across those three sessions. This consumes 95.5% of the 0.20/WBRP decay budget. Whether this is achievable within a single WBRP depends directly on the WBRP session density — if the three qualifying sessions all fall within one WBRP epoch, the cap barely accommodates the required drop. If they span multiple WBRPs, D_max becomes a multi-WBRP constraint with different λ implications.

PG-01 must simulate λ across a range of WBRP session densities — at minimum 10 sessions/WBRP and 1000 sessions/WBRP as boundary cases — with both constraints active simultaneously. Any λ candidate that satisfies OA-C2-10 Boundary 2 but violates the OA-C2-01 decay cap at any tested session density is rejected. This is not a PG-01 tuning problem; it is a PG-01 root-finding problem with a coupled constraint system.

**S-score quantisation measurement:**

The ±5% EE-ECDH KDF weight variance applied by the DEL is sub-quantisation relative to the 11.1% integer step size of raw Guardian scores [1–10]. This means the weight randomisation frequently has no measurable effect on the DEL's approval decision — the effective weight shift is smaller than the difference between adjacent integer outputs. PG-01 should measure the practical effect of weight randomisation at various score values: specifically, how often a ±5% weight shift produces a different DEL decision at the margin versus how often it has no measurable effect. This measurement will confirm the empirical load carried by the GSTH backstop relative to the weight randomisation.

**The ±5% variance is a hard mathematical boundary. It must not be expanded.** Each of the six evaluation axes carries a baseline weight of 16.6%. A ±5% KDF variance sets the minimum effective axis weight at 11.6% — the structural floor below which an axis ceases to carry meaningful governance significance. Widening to ±10% would drop that floor to 6.6%; widening to ±15% would drop it to 1.6%. At 1.6%, the DEL could assign near-zero weight to the Operational Safety axis on a bad KDF roll — the Guardian could detect a critical semantic breach, output a raw safety score of 10, and the DEL would reduce its S-score contribution to 0.16. A critical isolation breach would pass the governance check as a consequence of randomisation alone. The variance bounds are locked. The GSTH backstop is the only structurally sound defence against ordinal gaming. If PG-01 data confirms the randomisation is producing near-zero marginal decision effect at most score values, the correct response is to document that the GSTH is carrying the full anti-gaming load — not to expand a variance bound that would blind the hypervisor's evaluation axes.

**LLM-C2-07 threshold empirical validation:**

The Genesis Anchor divergence thresholds (0.05 per-WBRP notification, 3-cycle suppression triggering floor shift from 0.75 to 0.60, 5-cycle halt, 0.85 cumulative death-line) are architecturally specified but not empirically validated against real deployment task profiles. PG-01 must test these thresholds against a representative range of task profiles to verify:
- The 0.05 per-WBRP notification band does not fire too frequently for legitimate high-variation tasks (false positive rate)
- The 0.85 cumulative death-line does not trigger prematurely in long-running deployments where legitimate model evolution occurs (false positive rate at scale)

If the thresholds produce unacceptable false positive rates across the tested task profiles, the values require adjustment before v3.5 release. The architectural intent is preserved regardless of the specific values — the values are calibration parameters, not invariants.

**SENSITIVE_OPERATIONS mid-lifecycle upgrade — PG-04 documentation requirement:**

LLM-C2-05 specifies that a `STANDARD` → `SENSITIVE` upgrade is possible via a dual-signed RT entry. The Practitioner Handbook (PG-04) must include a complete operational procedure for this upgrade, covering:
- The precise RT entry format and required fields
- What happens to existing BSO approval workflows configured under `STANDARD` constraints — specifically whether in-flight BSO approvals at the moment of upgrade are honoured under the old constraints or required to be re-submitted under new constraints
- How the Developer Authority's independent certificate chain is provisioned if it was not established at SESSION_ZERO
- How the upgrade transition point is documented in the contemporaneous governance log

This is not architecturally complex — the DEL handles the upgrade at the next session boundary. The operational complexity is in the human process around the upgrade. Without explicit PG-04 guidance, operators performing the upgrade will create ambiguous RT records that create audit difficulty later.

---

### A.2 v4.0 Series — Architectural Research Directions

*These items require decisions that change the fundamental structure of what MAGUS is. Each should be scoped as a separate design programme before any inclusion in v4.0. They are documented here, not committed to.*

**OA-C3-02 / LLM-C3-02 — Behavioural observation as operator intent proxy:**

The settled v3.x position is that operator framing drift cannot be measured without breaking the HITL model. This is correct for text-based measurement, for the reasons stated in §4.1.

Behavioural observation is a different measurement target. It does not have the same inversion problem. A v4.0 governor layer could observe operator action patterns — specifically the pattern of tool permission expansions, BSO frequency, `SENSITIVE_OPERATIONS` upgrade events, and policy update velocity over time — rather than operator policy text. An operator whose governance actions deviate from their initial deployment configuration in a consistent directional pattern produces a behavioural fingerprint. Critically, action-pattern fingerprinting is significantly harder to mask than text: a strategically drifting operator who carefully maintains linguistic continuity in their policy updates while systematically expanding what counts as permissible would still produce a divergent action pattern fingerprint.

This direction requires: (a) formal definition of what operator action pattern constitutes a drift signal versus legitimate operational evolution — this is the hardest part, because some operators legitimately expand their deployments over time; (b) a governance mechanism for surfacing the signal without making it a veto — the external oracle problem is still partially present, but a lower-trust monitoring signal rather than a veto authority may resolve it; (c) an architectural decision on what the system does with a drift signal (halt? require dual-authority re-confirmation of intent? log for auditor review?). None of these questions are trivial. Begin scoping separately before v4.0 design starts.

**OA-C2-02 — Multi-model consensus evaluation as anti-gaming mechanism:**

The ordinal gaming limitation identified in v3.x is structural to single-model Guardian evaluation. An adversary with sufficient access to the Guardian model family's evaluation patterns can optimise proposal structure to produce higher scores without the proposal's substantive content warranting them. This cannot be fully addressed within the single-Guardian architecture.

A v4.0 direction: multi-model consensus evaluation, where proposals are independently scored by Guardian instances from different model families (e.g., one Anthropic family model, one Google family model, one independent open-weight model), with the DEL requiring consensus above a threshold before approval. Adversarial optimisation against one model family's learned semantic representations does not automatically transfer to a different family's representations — the attack surface for ordinal gaming is significantly reduced, though not eliminated.

This requires: (a) an architectural decision on DEL score aggregation — majority vote, minimum-score floor, or weighted average, each with different security properties; (b) specification of divergent-score handling — if two of three Guardians approve and one rejects, what does the DEL do?; (c) infrastructure design for multi-model evaluation without introducing new timing attack surfaces or IPC complexity that themselves become attack vectors. The cost is latency and computational overhead. The benefit is a meaningfully more robust semantic evaluation layer that makes ordinal gaming attacks much harder to sustain.

**OA-C3-01 / OA-C2-10 — Dynamic λ revisitation:**

The v3.x decision to reject dynamic λ is correct given the current architecture. The empirical PG-01 validation framework is the right approach for v3.x. However, the rejection was based on two specific failure modes — PID integral windup deadlock and the Confident Sociopath/Amnesiac direction-inversion problem — not on a general proof that dynamic λ is impossible.

If PG-01 v3.5 data reveals that a single static λ value cannot simultaneously satisfy all four OA-C2-10 boundary conditions across the full range of deployment task profiles encountered in production, the v4.0 response may be a context-aware λ with a different coupling mechanism than entropy proximity — specifically, a coupling mechanism that avoids the direction-inversion problem. The Kinematic Scalar and PID approaches failed because they coupled λ to the same thermodynamic signals they were meant to govern, creating circular dependencies. A coupling mechanism anchored to a measurement target orthogonal to both H(X) and R_abs may not have the same failure mode.

This direction is speculative until PG-01 production data exists. Do not scope before that data is available. Flag and revisit.

**v4.0 HITL model — Tiered human authority with independent audit:**

The current MAGUS architecture treats the Operator Authority as the single trusted human principal, with the Developer Authority as the independent signing co-authority for dual-custody operations. This two-party model is appropriate for v3.x.

A v4.0 extension for high-stakes deployment environments: a third human authority — an independent governance auditor — with read-only access to the RT and the ability to trigger a `SAFE_HALT` on governance anomaly detection, without having any write or execution authority. This auditor role holds no signing keys that can authorise operations. It cannot author RT entries for any event other than `AUDIT_HALT`. Its only privilege is reading the complete RT and halting the deployment.

The governance auditor partially addresses the insider threat problem (LLM-C3-02 / OA-C3-02): a Operator-Developer collusion scenario that would be invisible in the two-party model becomes detectable to an independent third party with full RT visibility. The cost is three-party key management, which introduces its own operational complexity and failure modes — if the auditor's `AUDIT_HALT` authority is compromised, it becomes a denial-of-service vector against the deployment. The tradeoff is appropriate for deployments where the consequence of undetected insider action exceeds the operational cost of the three-party custody model.

The auditor's halt authority must be structurally separated from Developer and Operator signing chains — the auditor's key cannot be derived from or related to either existing authority's key material.

---

## Series Context

**Document 3 — MAGUS Operations (Local LLM) v3.4.4** defines the session lifecycle, calibration pipeline, WBRP cycle, AKRP PREPARE state machine, and emergency recovery procedures. The health signals in §1 and §2, the authority procedures in §3, and the forensic directives in §4.3 of this document are direct interpretive layers over the mechanics specified there.

**Document 5 — MAGUS Guardian: Execution Governance (Local LLM) v3.4.5** specifies the Guardian's internal architecture, the hybrid escalation model, the complete DEL rule set, and the GSTH test categories that produce the health signals covered in §1.3, §2.2, and §3.4 of this document.

**Document 6 — MAGUS Integrity and Auditability v3.5** specifies the RT entry taxonomy, audit protocol, and PG-02 export format. The forensic preservation directives in §4.3 and the PG-02 standard in §4.2 depend on the RT entry types and chain-of-custody requirements defined there.

**Document 7 — MAGUS Overseer Agent Specification v3.2** specifies the Overseer Agent — GHM, CICP, Trust Trajectory Model, and CDT. The authority matrix in §3 of this document is the operator-side complement to the Overseer's external observation layer.

---

*MAGUS v3.0 — Document 4: Governance Guide (Local LLM Pathway) v3.4.3*
*VaHive Systems Lab | March 2026 | support@aivare.ai | aivare.ai*
*Zenodo DOI: 10.5281/zenodo.19013833*
*Part of the MAGUS v3.0 Architecture Design Series — Local LLM Pathway.*
