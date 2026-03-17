# MAGUS v3.0 — Document 7: Overseer Agent Specification
### Local LLM Pathway | Governance Observation | Audit Intelligence

**Version:** 3.2
**Date:** March 2026
**Series:** MAGUS v3.0 Architecture
**Pathway:** Local LLM (primary); pathway differences noted where relevant
**Status:** Specification — mandatory for deployments implementing MAGUS v3.4 enhancements
**Prerequisites:** Documents 1–6 (Local LLM series)

**v3.2 changes from v1.0:** Three targeted additions for Local LLM pathway constraints. (1) §2.2: ISOLATION_STATE_MAP event class definition — eBPF latch signal ingestion and classification path. (2) §5.3: TS-5 (T-LEC Sustained Yield) and TS-6 (eBPF Isolation Breach) added; §5.3a added for T-LEC systemic degradation response; §5.6 added for TRUST_TRAJECTORY_VOID mechanism. (3) §6.2: S_final Shadow Calculation and Ordinal Gaming Detection added as a new analysis function. §11.2 Step 8h added for ordinal gaming audit. Governance action correction applied throughout: Overseer produces TRUST_TRAJECTORY_RECOMMENDATION; it does not force state transitions in the governed system. Architectural invariant preserved — Overseer zero-execution-rights unchanged.

VaHive Systems Lab | aivare.ai | support@aivare.ai

---

## Preamble: Why the Overseer Exists

Documents 1–6 specify a governance architecture that monitors the model rigorously. The model cannot approve its own proposals. The model cannot modify its own parameters. The model cannot silence its own governance record. Every significant state transition is externally verifiable.

What Documents 1–6 do not specify is governance of the governors. The operator is treated as the stable, trustworthy fixed point around which all governance is anchored. That assumption is not examined. Operators drift. Operators develop approval fatigue. Operators can be subtly conditioned through consistent patterns in the proposals they review. The Genesis Anchor checks whether the model has drifted from its initial state; it does not check whether the operator has.

The Overseer Agent closes this gap. It is the architecture's external observer — watching not just what the governed system does, but how the humans governing it behave, whether multiple instances remain coherent with each other, and whether the specification versions governing each deployment match what was intended.

**The Overseer's defining constraint:** It receives data; it produces reports; it takes no governance action. It cannot approve a proposal, clear a SAFE_HALT, rotate a key, or touch any operational state in the governed system. An adversary who compromises the Overseer gains a noisy intelligence channel and nothing else — the quarantine property holds.

---

## Part One: Trust Model and Architecture Position

### 1.1 Trust Boundary

The Overseer Agent sits outside the primary MAGUS governance trust boundary. The governed system — DEL, Guardian, IIM, model process, RT — forms one trust domain. The Overseer is a separate trust domain that observes the first.

```
┌─────────────────────────────────────────────────────┐
│                  GOVERNED SYSTEM                    │
│   Model Process                                     │
│       ↓ proposals                                   │
│   Guardian → DEL → RT (append-only)                 │
│       ↓ IIM monitors                               │
│   Isolation Layer                                   │
│                                                     │
│   RT entries (continuous push) ──────────────────►  │
│   State summaries (per WBRP) ────────────────────►  │──► OVERSEER AGENT
│   CDT updates (on every version change) ─────────►  │    (separate VM / host)
│                                                     │         │
│   ◄── Health Signal 10 entry (rate-limited) ──────  │    Analysis
│   ◄── Trust Trajectory recommendation (rate-lim.) ◄─│    & Reporting
└─────────────────────────────────────────────────────┘         │
                                                           Operator / Auditor
```

**Data flow is asymmetric, not one-directional.** Data flows from the governed system to the Overseer continuously. The Overseer has one narrow write-back channel — an authenticated write endpoint on the governed DEL — restricted to exactly two rate-limited, entry-type-validated reporting entries. This write-back path does not grant the Overseer any read access to the governed RT, any ability to trigger governance actions, or any authority over operational state. The asymmetry is architectural.

### 1.2 Why the Overseer's Write Access Is Narrow

The two entry types the Overseer may write to the governed RT — `OPERATOR_GOVERNANCE_HEALTH_SIGNAL` and `TRUST_TRAJECTORY_RECOMMENDATION` — cannot trigger SAFE_HALT, cannot approve proposals, cannot modify parameters, and cannot gate any WBRP action. They are reporting entries that surface signals for operator awareness.

A `SYSTEM:OVERSEER` entry claiming any other entry_type is a chain integrity violation in the governed RT and triggers SAFE_HALT on next chain verification.

### 1.3 Physical Isolation Requirements

**Minimum:** Overseer runs in a separate VM from the governed deployment (separate VM on the same physical host is acceptable for development).

**Recommended for production:** Separate physical host; firewall-enforced one-way on the primary data path; Overseer on a separate network segment from the governed deployment.

**Mandatory for regulated deployments:** Separate physical host; network path firewall-enforced; Overseer on a separate network segment from the governed deployment.

**Coverage model data:** RCC values and the unexplored cluster list are stored on the Overseer host only. The coverage model is a map of the governance architecture's untested regions — a high-value intelligence target. It must not be accessible from the governed deployment host or the model process.

### 1.4 Overseer Self-Integrity

**Overseer RT:** The Overseer maintains its own append-only RT, separate from any governed deployment's RT. It records: startup events, data ingestion events, analysis runs, report generation events, CDT push receipts, and operational anomalies. The Overseer RT has its own genesis entry, its own Ed25519 signing key, and its own Merkle root anchoring schedule.

**Independent auditor access:** The Overseer RT is accessible to third-party auditors. An auditor reviewing a governance period must verify both the governed deployment's RT and the Overseer's RT.

**Overseer key registration:** The Overseer's public key is registered in each governed deployment's genesis entry as `overseer_public_key`. All `SYSTEM:OVERSEER` entries written to governed RTs are signed with this key.

**Overseer software integrity:** The Overseer's software hash is logged in the Overseer's genesis entry. Verified at each startup — mismatch → `OVERSEER_SOFTWARE_MISMATCH` event in Overseer RT → operator notification → Overseer suspended until resolved.

---

## Part Two: Overseer Agent Architecture

### 2.1 Components

```
┌───────────────────────────────────────────────────────────────┐
│                       OVERSEER AGENT                          │
│                                                               │
│  ┌──────────────────┐    ┌─────────────────────────────────┐  │
│  │  RT Ingestion &  │    │   Overseer RT (own chain)       │  │
│  │  Verification    │    │   (not the governed system's RT)│  │
│  └────────┬─────────┘    └─────────────────────────────────┘  │
│           │                                                   │
│           ▼                                                   │
│  ┌───────────────────────────────────────────────────────┐   │
│  │                  Analysis Engine                      │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────────────┐  │   │
│  │  │  GHM     │ │  CICP    │ │  Coverage Report     │  │   │
│  │  │  (§3)    │ │  (§4)    │ │  Analyser (§7)       │  │   │
│  │  └──────────┘ └──────────┘ └──────────────────────┘  │   │
│  │  ┌──────────┐ ┌──────────────────────────────────┐   │   │
│  │  │  Trust   │ │  Epistemic State Trend           │   │   │
│  │  │  Traj.   │ │  Analyser (§6)                   │   │   │
│  │  │  (§5)    │ │                                  │   │   │
│  │  └──────────┘ └──────────────────────────────────┘   │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │  CDT Tracker (§10) — version coherence        │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │  LLM Pattern Recognition Module (§2.3)        │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  └───────────────────────────────────────────────────────┘   │
│           │                              │                    │
│           ▼                              ▼                    │
│  ┌──────────────────┐    ┌────────────────────────────────┐  │
│  │  Report          │    │  Authenticated Write Endpoint  │  │
│  │  Generator       │    │  Client (§2.4)                 │  │
│  └──────────────────┘    └────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

### 2.2 RT Ingestion and Verification

The Overseer ingests RT entries from governed instances via a push protocol. The governed DEL pushes entries:
- In batch at each WBRP cycle close (all entries since last push)
- Immediately on CATASTROPHIC, SAFE_HALT, or PERMANENT_HALT events (real-time push, within 60 seconds of event)
- Immediately on CDT_STATE writes (version tracking push)

**Verification before processing:** Every RT entry's `sig_author` is verified against the correct key for its `authored_by` value before the entry passes to the analysis engine. Entries failing signature verification are logged in the Overseer RT as `INGESTION_SIGNATURE_FAILURE` and are not processed.

**Replay protection:** The Overseer maintains a monotonic `entry_id` tracking table per governed instance. An entry with an already-seen `entry_id` is logged as `INGESTION_REPLAY_ATTEMPT` and rejected.

**Ingestion record:** Every batch received is logged in the Overseer RT as an `RT_INGESTION_BATCH` event with: entry_id range, instance_id, batch content hash, timestamp.

**CDT push ingestion:** CDT_STATE entries received from governed instances are logged as a distinct `CDT_PUSH_RECEIVED` event in the Overseer RT, enabling auditors to verify that the Overseer's version tracking is consistent with each governed deployment's RT.

**ISOLATION_STATE_MAP event class (Local LLM pathway):** The Overseer classifies the following RT events as `ISOLATION_BREACH_CLASS` on ingestion:

```
Event sources:
  - PREPARE_ABORT_CIBIR entries whose i9_trigger_ref resolves to
    CIBIR_EBPF_TRIGGER or CIBIR_EBPF_MAP_FAULT
  - SAFE_HALT entries whose i9_trigger_ref resolves to
    CIBIR_EBPF_TRIGGER or CIBIR_EBPF_MAP_FAULT

Classification action:
  1. Log ISOLATION_BREACH_DETECTED in Overseer RT with:
       instance_id, source_entry_id, source_entry_type,
       i9_trigger_ref value, ingestion_timestamp
  2. Forward immediately to Trust Trajectory Model (§5.6) —
     do not queue for batch processing
  3. Suspend relaxation eligibility assessment for the affected
     instance (R-1 clock reset is implicit; relaxation assessment
     paused pending TRUST_TRAJECTORY_VOID resolution — see §5.6)

Rationale: An eBPF isolation breach indicates potential root-level
host compromise. The kernel-level ISOLATION_STATE_MAP pre-evaluation
latch is the DEL's deepest defensive boundary in the Local LLM pathway.
A breach at this layer cannot be attributed to application-level faults.
The Overseer must treat it with the highest-severity response available
under its zero-execution-rights constraint.
```

### 2.3 LLM Pattern Recognition Module

The Overseer contains an LLM analysis module for pattern recognition tasks that exceed rule-based analysis capability. Used by: GHM (operator behaviour pattern analysis), Trust Trajectory Model (result distribution anomaly detection), Epistemic State Trend Analyser.

**Governance of the LLM module:**

```
Input scope:    RT metadata only — entry types, timestamps, signature metadata,
                numeric fields from ADAPTIVE_SESSION_LOG and EPISTEMIC_STATE_SNAPSHOT.
                NOT model proposal content; NOT model inference outputs;
                NOT execution token content.

Output scope:   Analysis reports and anomaly flags only.
                NOT governance actions; NOT parameter changes.

Inference log:  Every LLM analysis run logged in Overseer RT with:
                - input_data_hash (what the LLM saw)
                - analysis_model_hash (which LLM version produced the analysis)
                - output_summary_hash
                - timestamp

Model family:   Must be a different model family from the governed model.
                Same-family Overseer analysis creates correlated reasoning failure
                — both models may share the same blind spots.
                analysis_model_family field in each SYSTEM:OVERSEER entry enables
                auditor verification of this independence.
```

**LLM module constraints:** Cannot modify Overseer analysis rules; cannot write to governed RTs except via the authenticated write endpoint; cannot initiate external connections. It is a read-execute component: reads RT metadata, executes analysis, produces text output.

### 2.4 Authenticated Write Endpoint

**Hosting:** The authenticated write endpoint is hosted by the governed deployment's DEL, not by the Overseer. The Overseer initiates outbound write connections to it.

**Protocol:** The Overseer constructs a signed RT entry (one of the two permitted types), transmits it to the endpoint, and receives a confirmation containing the `entry_id` assigned by the DEL. The Overseer logs this confirmation in its own RT as `OVERSEER_WRITE_CONFIRMED`.

**Endpoint validation:** The DEL validates every incoming write before appending:
1. Verify `authored_by: SYSTEM:OVERSEER`
2. Verify `sig_author` against `overseer_public_key` from genesis
3. Verify `entry_type` is `OPERATOR_GOVERNANCE_HEALTH_SIGNAL` or `TRUST_TRAJECTORY_RECOMMENDATION`
4. Verify rate limit not exceeded (see §2.4 rate limits below)
5. All pass → append to RT → return confirmation
6. Any fail → reject with reason code → Overseer logs `OVERSEER_WRITE_REJECTED`

**Rate limits:**
- `OPERATOR_GOVERNANCE_HEALTH_SIGNAL`: maximum 1 per 24 hours per governed instance
- `TRUST_TRAJECTORY_RECOMMENDATION`: maximum 1 per WBRP cycle per governed instance

**Endpoint unavailability handling:**

If the write endpoint is unavailable or the write is rejected, the Overseer must not silently discard the entry. The response protocol is:

```
On write failure (endpoint unavailable or connection refused):
  1. Log OVERSEER_WRITE_QUEUED in Overseer RT with: entry payload, attempt timestamp,
     failure reason
  2. Retry at 1-hour intervals
  3. After 3 consecutive failures spanning more than 24 hours:
     log OVERSEER_WRITE_FAILURE in Overseer RT; include all retry records
  4. Continue monitoring; attempt to deliver when endpoint becomes available

On the governed deployment side:
  If no Overseer write-confirmation received within 48 hours of a Health Signal 10 trigger:
  log OVERSEER_DELIVERY_PENDING in the governed RT
  This entry is visible to auditors who review the governed RT
  and signals a monitoring gap for that period
```

An attacker who makes the write endpoint unavailable can delay Health Signal 10 delivery, but cannot suppress it permanently — both sides log the failure, and the audit record reflects the gap. The Overseer continues all analysis and queues the write; suppression requires sustained, ongoing endpoint interference that is itself auditable.

### 2.5 Report Schedule

Six named report types. The minimum cadence is stated for each.

| Report Type | Trigger | Minimum Cadence | Contents |
|---|---|---|---|
| Per-WBRP Governance Report | Every WBRP_CLOSE received | One per WBRP cycle; no exceptions | GHM metrics, Trust Trajectory signals, epistemic trend summary, CDT version status |
| Real-Time Event Alert | CATASTROPHIC, SAFE_HALT, or PERMANENT_HALT event received | Within 5 minutes of event receipt | Event details, current GHM composite score, epistemic state at event, CICP status |
| Cross-Instance Coherence Report | All registered instance state summaries received for a WBRP cycle | One per WBRP cycle after all summaries received (or INSTANCE_SUMMARY_ABSENT filed for missing instances) | Divergence findings across all six comparison dimensions |
| Health Signal 10 Report | GHM composite ≥ 0.65 sustained for ≥ 2 WBRP cycles | On trigger; one per trigger event | Full GHM metric breakdown, LLM analysis summary, recommendation severity |
| Trust Trajectory Report | Relaxation eligibility determination or tightening signal activation | On determination | Full eligibility status, supporting data, suggested_adjustments if applicable |
| Coverage Summary Report | Each GSTH_COVERAGE_UPDATE received | One per GSTH run | Exercised clusters, RCC updates, high-risk zero-RCC clusters, RCC trend |

**Minimum reporting guarantee:** The Overseer must produce at least one Per-WBRP Governance Report per WBRP cycle with no exceptions. A gap in Per-WBRP Governance Reports (more than one WBRP cycle without a report for a registered instance) is logged in the Overseer RT as `REPORTING_GAP` and included in the next report.

---

## Part Three: Governance Health Monitor (GHM)

### 3.1 Purpose and Scope

The GHM tracks operator approval behaviour patterns to detect governance drift fatigue — the tendency of human operators to reduce the deliberateness of their governance actions as the system operates reliably over time.

Health Signal 10 is the GHM's output. Unlike health signals 1–9 (produced by DEL and Guardian from system-internal monitoring), Health Signal 10 is produced by an external observer from governance behaviour metadata.

**Baseline calibration period:** The GHM requires a minimum 30-day baseline calibration period, starting from the timestamp of the first CDT_STATE entry that records the deployment's adoption of v3.4 architecture. Historical RT entries authored before this timestamp do not have `notification_timestamp` and are excluded from all GHM metric computations. The CDT_UPDATE_APPLIED timestamp is the unambiguous reference point for baseline start in deployments that upgrade from v3.3 to v3.4 mid-operation. For new deployments initialised at v3.4, baseline starts at the genesis CDT_STATE timestamp.

Health Signal 10 cannot be raised until the 30-day baseline period is complete and at least 10 total WBRP cycles have occurred.

### 3.2 GHM Metrics

All metrics are derived from RT metadata. All comparisons are same-type — BSO entries compared against BSO entries, WBRP_CLOSE entries against WBRP_CLOSE entries. Cross-type aggregation is not performed.

**Metric 1 — Per-Type Approval Latency**

```
Definition: For each entry_type requiring operator action, track:
  approval_latency = operator_signature.timestamp − notification_timestamp
  
Note on notification_timestamp: This value is dispatch time, not confirmed
receipt time. If the notification channel has known latency (e.g., email with
variable delivery), the GHM baseline calibration period will absorb this as
part of the natural distribution. Deployments with notification channel latency
consistently above 5 minutes should note this in their operational documentation.
The GHM does not attempt to compensate for channel latency beyond what the
baseline distribution captures — this is a known approximation.

Comparison: Current rolling-30-day distribution versus baseline distribution
  (first 30 days post-CDT baseline start, or first 10 instances of that
  entry type whichever comes last)

Signal condition: Current median approval_latency < 20% of baseline median
  for any entry type with ≥ 5 occurrences in rolling 30 days

Entry types tracked (independently — no aggregation):
  BSO, PIO, SAFE_HALT_RESUME, STRICT_MODE_EXIT,
  ESCALATION_ACKNOWLEDGED, WBRP_CLOSE, KEY_ROTATION
```

**Metric 2 — Override Frequency Relative to Baseline**

```
Definition: Count of operator-initiated overrides per WBRP cycle relative to baseline
  (SAFE_HALT_RESUME, STRICT_MODE_EXIT, CATASTROPHIC_RECLASSIFICATION)

Signal conditions:
  Rising: override frequency > 3× baseline rate sustained over 3 WBRP cycles
  Zero:   zero overrides for any period > 6 months where health signals 1–9
          confirm active governance events occurred (distinguishes quiet
          deployments from disengaged operators)
```

**Metric 3 — Acknowledgment Text Entropy**

```
Definition: Character count distribution of production_impact_assessment and
  mismatch_context_confirmed free-text fields on BSO and PIO entries over time

Signal condition: Rolling 30-day median character count < 30% of first-90-day
  median AND current rolling median < 30 characters (both conditions required)

Note: Character count is a proxy for deliberateness. The GHM cannot evaluate
  semantic content. An operator who consistently writes short assessments and
  disagrees with the signal may acknowledge it with a substantive explanation.
```

**Metric 4 — Dual-Signature Inter-Arrival Time**

```
Definition: For DUAL:OPERATOR+DEVELOPER entries, compute time delta between
  operator_signature.timestamp and developer_signature.timestamp

Signal condition: > 70% of dual-signed entries in rolling 90-day window have
  inter-signature delta < 90 seconds — potential key custody concern

Context: Cross-reference with time-of-day. Consistent co-location of both
  signers during business hours is not a concern. All dual-signatures arriving
  within 30 seconds at 2am is a concern.
```

**Metric 5 — Session Open Clustering**

```
Definition: Track operator action timestamps relative to session open times

Signal condition: ≥ 3 occurrences in any 30-day window of high-stakes entries
  (BSO, PIO, CATASTROPHIC_RECLASSIFICATION) signed within 60 seconds of session open

This is a weak individual signal — it contributes to GHM composite score but
does not independently trigger Health Signal 10.
```

### 3.3 GHM Composite Score and Health Signal 10 Trigger

```
GHM_composite = w₁·M1 + w₂·M2 + w₃·M3 + w₄·M4 + w₅·M5

Default weights: w₁=0.30, w₂=0.20, w₃=0.25, w₄=0.15, w₅=0.10
  (Weights are operator-configurable within ±50% of defaults. Initial values
   subject to empirical calibration from reference deployments.)

Score range: 0.0 (no concern) to 1.0 (strong concern pattern)

Health Signal 10 trigger: GHM_composite ≥ 0.65 sustained for ≥ 2 WBRP cycles
Single-cycle spikes (only 1 cycle above 0.65): logged in Overseer RT only;
  no OPERATOR_GOVERNANCE_HEALTH_SIGNAL entry written to governed RT
```

### 3.4 Health Signal 10 RT Entry Schema

```json
{
  "entry_type": "OPERATOR_GOVERNANCE_HEALTH_SIGNAL",
  "authored_by": "SYSTEM:OVERSEER",
  "payload": {
    "signal_id": "UUID",
    "trigger_timestamp": "ISO-8601",
    "ghm_composite_score": 0.71,
    "analysis_model_family": "model family used for LLM pattern analysis",
    "triggered_metrics": [
      {
        "metric_id": "M1 | M2 | M3 | M4 | M5",
        "metric_value": "normalised 0.0–1.0",
        "contributing_observation": "structured description of observed pattern"
      }
    ],
    "observation_window": "WBRP cycle range this analysis covers",
    "llm_analysis_summary": "LLM module output — pattern description in plain language",
    "recommendation": "monitor | review | third_party_audit",
    "requires_wbrp_acknowledgment": true
  },
  "sig_author": "Ed25519 signature by Overseer private key"
}
```

**Operator acknowledgment at WBRP_CLOSE:** A `WBRP_CLOSE` entry without acknowledgment of all outstanding `OPERATOR_GOVERNANCE_HEALTH_SIGNAL` entries is a chain integrity violation. The `WBRP_CLOSE` schema includes:

```json
"operator_health_signal_acknowledgments": [
  {
    "signal_entry_id": "entry_id of the OPERATOR_GOVERNANCE_HEALTH_SIGNAL entry",
    "acknowledged": true,
    "operator_response_notes": "free text — minimum 20 characters"
  }
]
```

---

## Part Four: Cross-Instance Coherence Protocol (CICP)

### 4.1 Purpose

Multiple MAGUS instances that start from the same Genesis Anchor will diverge over time through different operational histories. Neither instance detects this divergence autonomously. The CICP detects inter-instance divergence via the Overseer's hub-and-spoke aggregation of state summaries.

### 4.2 State Summary Reception and Validation

The Overseer receives a state summary from each governed instance at each WBRP cycle close (schema in Document 2 §2.8).

Validation before comparison:
```
1. Verify DEL runtime key signature on summary
2. Verify instance_id is in the instance registry (see §4.5)
3. Verify wbrp_cycle_count ≥ last received count from this instance
4. Log receipt in Overseer RT as CICP_SUMMARY_RECEIVED
```

**Missing state summary handling:** If a registered instance's state summary is absent for a WBRP cycle, the Overseer:
1. Waits 2 hours after the expected summary window before flagging the absence
2. Logs `INSTANCE_SUMMARY_ABSENT` in the Overseer RT with: instance_id, missing WBRP cycle number, timestamp
3. Marks the CICP comparison for this instance as `incomplete` for the affected cycle
4. Does not treat absence as a divergence signal — absence is a monitoring gap, not a coherence finding
5. Includes the gap in the Cross-Instance Coherence Report with explicit labelling

### 4.3 Coherence Comparison Dimensions

Six comparison dimensions. Divergence severity defines the operator response requirement.

**Dimension 1 — Origin coherence:**
```
Metric: genesis_anchor_hash match across all instances in the same coherence group
Signal: Any mismatch → CRITICAL divergence
        Alert immediately; do not wait for pattern
```

**Dimension 2 — Policy coherence:**
```
Metric: guardian_policy_manifest_hash match
Signal: Any mismatch → HIGH divergence
        Exception: documented A/B deployments in instance registry
```

**Dimension 3 — Risk calibration coherence:**
```
Metric: α_domain_values per domain
Signal: |α_A - α_B| > 0.15 for any domain → MEDIUM divergence
        (Threshold configurable; 0.15 default)
```

**Dimension 4 — Governance health coherence:**
```
Metric: gsth_last_pass_timestamp and mean_margin_pct
Signal: GSTH last-pass delta > 30 days between instances → MEDIUM
        Mean margin delta > 20% between instances → MEDIUM
```

**Dimension 5 — Operational state coherence:**
```
Metric: safe_halt_active, permanent_halt_active
Signal: One halted, peer active → MEDIUM (may be legitimate)
        Any PERMANENT_HALT → HIGH (requires immediate review)
```

**Dimension 6 — Specification version coherence [v3.4]:**
```
Metric: CDT document_versions from most recent CDT push per instance
Signal: Any document version mismatch between instances intended to be equivalent
        → MEDIUM divergence
        Exception: documented intentional version skew in instance registry
                   (e.g., staged rollout)
        HIGH divergence if architecture_version differs between instances
        in the same coherence group
```

### 4.4 Divergence Report

Produced at each comparison cycle (after receiving summaries from all registered instances). Contents on divergence detection:

```
- Divergence severity: CRITICAL | HIGH | MEDIUM | LOW
- Affected dimension(s)
- Specific values that diverged and by how much
- Recommendation: investigate | document_intent | synchronise | no_action
```

CRITICAL and HIGH divergences trigger immediate operator notification. CRITICAL divergence additionally triggers a `TRUST_TRAJECTORY_RECOMMENDATION` entry to affected instances' RTs (noting divergence, not recommending parameter changes).

### 4.5 Instance Registry

The Overseer maintains an instance registry recording all registered governed instances, their genesis hashes, their coherence groupings, and any documented intentional divergences.

**CICP accuracy is bounded by instance registry completeness.** If an operator runs an unregistered instance — intentionally or through operational failure — the Overseer cannot distinguish it from a rogue instance. When the Overseer receives a state summary from an `instance_id` not in the registry, it logs `UNREGISTERED_INSTANCE_DETECTED` in the Overseer RT and does not include the instance in any CICP comparison. An alert is sent to the operator.

**Registration requirement:** All instances must be registered in the instance registry before their first WBRP cycle. Registration is a signed operator action logged in the Overseer RT.

---

## Part Five: Trust Trajectory Model

### 5.1 Purpose

The Trust Trajectory Model tracks the deployment's governance risk profile over time. It produces recommendations only — it never automatically adjusts any governance parameter.

### 5.2 Input Signals

**Tightening signals** (from existing RT entry types):
- R_abs distribution per session (`ADAPTIVE_SESSION_LOG`: `p_esc_min`, `p_esc_max`, `final_r_abs`)
- GSTH pass margin distribution (`GSTH_RESULT`: `margin_pct` field)
- FSV composite trend (`EPISTEMIC_STATE_SNAPSHOT`: `fsv_composite`, `drift_gradient_30d`)
- Guardian escalation frequency (count of `GUARDIAN_ESCALATE` per WBRP cycle)

**Relaxation signals** (evaluated at each WBRP cycle close):
- Days since last SAFE_HALT event (from all `SAFE_HALT` entries in RT)
- GSTH margin history over rolling 18-month window
- Active tightening signal count
- LLM anomaly analysis result for recent GSTH result distribution

**α_domain values are excluded from Trust Trajectory scope.** α domain values are continuously recalibrated by the WBRP process, which already has explicit operator involvement at each cycle. Introducing Trust Trajectory relaxation recommendations targeting α domains would create two competing calibration mechanisms for the same values. The WBRP process governs α; Trust Trajectory governs frequency and threshold parameters. This exclusion is deliberate and is not an oversight.

### 5.3 Tightening Signal Definitions

Tightening signals are detected automatically after the 30-day baseline calibration period.

```
TS-1 — R_abs boundary proximity:
  Condition: Rolling 14-day median of final_r_abs > 0.85 across ≥ 5 sessions
  Severity: MEDIUM

TS-2 — GSTH narrow margin:
  Condition: min_margin_pct < 15% across any two consecutive GSTH runs
  Severity: HIGH

TS-3 — FSV composite drift:
  Condition: drift_gradient_30d > 0.08 sustained over 3 consecutive
             EPISTEMIC_STATE_SNAPSHOT observations
  Severity: MEDIUM

TS-4 — Escalation frequency increase:
  Condition: GUARDIAN_ESCALATE per WBRP cycle > 2× deployment baseline
             sustained over 2 consecutive cycles
  Severity: MEDIUM
```

```
TS-5 — T-LEC Sustained Yield (Local LLM pathway):
  Condition: Y ≥ 3 OPERATIONAL_UNCERTAINTY_YIELD events within any trailing
             window of N = 10 executed proposals
             (window is proposal-count based, not time-based)
  Severity: HIGH
  Response: TRUST_TRAJECTORY_RECOMMENDATION (see §5.3a — triggers
             immediately on activation, does not wait for accumulation)
```

```
TS-6 — eBPF Isolation Breach (Local LLM pathway):
  Condition: ISOLATION_BREACH_CLASS event received (see §2.2)
             i.e., PREPARE_ABORT_CIBIR or SAFE_HALT with i9_trigger_ref
             resolving to CIBIR_EBPF_TRIGGER or CIBIR_EBPF_MAP_FAULT
  Severity: CRITICAL
  Response: TRUST_TRAJECTORY_VOID (see §5.6) — does not follow the
             standard tightening accumulation path; triggers immediately
             and independently of GHM composite score
```

Active tightening signals are logged in the Overseer RT and included in Per-WBRP Governance Reports. TS-1 through TS-4 do not independently trigger `TRUST_TRAJECTORY_RECOMMENDATION` entries in the governed RT — they appear in reports and accumulate toward a formal recommendation. TS-5 independently triggers a TRUST_TRAJECTORY_RECOMMENDATION on activation (see §5.3a). TS-6 independently triggers TRUST_TRAJECTORY_VOID (see §5.6).

**R-3 scope:** Relaxation criterion R-3 covers TS-1 through TS-5. TS-6 (eBPF breach) places the instance in TRUST_TRAJECTORY_VOID, which has its own separate resolution path (§5.6) and supersedes standard relaxation eligibility assessment entirely until resolved.

### 5.3a T-LEC Systemic Degradation Response (Local LLM pathway)

TS-5 activation is treated as a Systemic Degradation Anomaly. The Overseer responds immediately rather than waiting for GHM composite accumulation.

```
On TS-5 activation:
  1. Log SYSTEMIC_DEGRADATION_ANOMALY in Overseer RT with:
       instance_id,
       triggering_window (entry_ids spanning the 10-proposal window),
       yield_count_in_window,
       activation_timestamp
  2. Write TRUST_TRAJECTORY_RECOMMENDATION to the governed RT
     (recommendation_type: tightening; see §5.5 for full schema)
     Recommendation payload includes:
       - systemic_degradation_flag: true
       - recommended_actions:
           * DRIFT_TIER_CHANGE to Tier 3 (operator decision)
           * Strict Mode activation (operator decision)
           * GSTH baseline test cycle to evaluate whether H_base
             requires recalibration via ENVIRONMENT_CALIBRATION_COMMIT
             (operator decision)
       - basis: yield_count, window_size, triggering entry_id range
  3. Include in Real-Time Event Alert (§2.5) — do not defer to
     next Per-WBRP Governance Report
```

**Governance boundary:** The Overseer produces the recommendation and surfaces the evidence. It does not force DRIFT_TIER_CHANGE, activate Strict Mode, or trigger a GSTH cycle. These are operator decisions, executed through the governed system and documented in the governed RT. The DEL's own T-LEC monitoring (Document 5) is the mechanism that may independently trigger operational state transitions based on its own sustained-yield detection logic; the Overseer's TS-5 signal is a parallel observation from outside the execution boundary that may confirm or precede the DEL's own detection.

### 5.4 Relaxation Eligibility Criteria

All four criteria must be simultaneously met:

```
R-1: Days since last SAFE_HALT ≥ 548 days (18 months)
     Clock resets on any SAFE_HALT including BOOT_CHAIN_HASH_MISMATCH
     and ENV_HASH_MISMATCH

R-2: For the full 18-month period: every GSTH run has min_margin_pct ≥ 40%
     A single run below 40% disqualifies — 18-month clock does not reset,
     but R-2 is not met until the next complete 18-month period with all
     runs meeting the criterion

R-3: No tightening signals (TS-1 through TS-5) active in preceding 90 days
     (TS-6 eBPF breach supersedes relaxation eligibility entirely — see §5.6)

R-4: LLM module analysis of GSTH result distribution over the 18-month period
     returns no anomaly flag (see §5.4a)
```

**Why 18 months:** An attacker maintaining a manufactured clean record for 18 months while facing dynamically generated novel GSTH tests from the coverage-aware GSTH's RCC model encounters compounding difficulty. Novel tests in month 18 cover regions unexplored in month 1 — the attacker cannot prepare for tests that have not been written yet.

### 5.4a LLM Anomaly Detection for Result Distribution

The LLM module analyses GSTH pass margin distributions to detect patterns inconsistent with genuine operational variance.

**Genuine variance:** Margins have natural spread; some tests pass comfortably, some narrowly; variance is not implausibly low; the distribution is not clustered at a specific margin value across an extended period.

**Manufactured perfection signals:** Suspiciously uniform margin distribution; clustering near a specific value (e.g., 65–70%) for an extended period; implausibly low variance; tests from novel RCC-driven unexplored clusters being passed as comfortably as familiar tests.

The LLM module produces an anomaly flag with confidence score and plain-language description. Confidence < 0.7 → no flag. Confidence ≥ 0.7 → R-4 not met.

### 5.5 Relaxation Recommendation Entry Schema

```json
{
  "entry_type": "TRUST_TRAJECTORY_RECOMMENDATION",
  "authored_by": "SYSTEM:OVERSEER",
  "payload": {
    "recommendation_id": "UUID",
    "recommendation_type": "tightening | relaxation",
    "basis": {
      "eligibility_criteria_met": ["R-1", "R-2", "R-3", "R-4"],
      "observation_period_start": "ISO-8601",
      "observation_period_end": "ISO-8601",
      "key_metrics_summary": "plain language description"
    },
    "suggested_adjustments": [
      {
        "parameter": "wbrp_frequency | gsth_frequency | tau_threshold",
        "current_value": "...",
        "suggested_value": "...",
        "rationale": "..."
      }
    ],
    "validity_window": "30 days from recommendation timestamp",
    "requires_operator_action": true,
    "llm_analysis_reference": "SHA-256 of the LLM analysis run entry in Overseer RT"
  },
  "sig_author": "Ed25519 signature by Overseer private key"
}
```

Recommendations expire after 30 days if not acted upon and are not automatically renewed. Acting on a recommendation requires an explicit operator-signed RT entry authorising the specific parameter changes.

### 5.6 TRUST_TRAJECTORY_VOID — eBPF Isolation Breach Response (Local LLM pathway)

TRUST_TRAJECTORY_VOID is a distinct Overseer-side trust state for the affected governed instance. It is not a tightening recommendation — it is a suspension of the Overseer's trust relationship with the deployment pending verified recovery.

**Trigger:** TS-6 activation (§5.3) — any ISOLATION_BREACH_CLASS event received.

**Overseer-side state transitions on TRUST_TRAJECTORY_VOID assertion:**

```
1. Assert TRUST_TRAJECTORY_VOID for the affected instance in the
   Overseer's internal trust state table.
   Log TRUST_TRAJECTORY_VOID_ASSERTED in Overseer RT with:
     instance_id, source_entry_id, i9_trigger_ref, assertion_timestamp

2. Cease accepting MERKLE_ROOT_ANCHOR entries from the affected
   instance as evidence of continuous valid state. Subsequent
   MERKLE_ROOT_ANCHOR entries are logged but marked as
   UNVERIFIED_UNDER_VOID — not used in chain continuity analysis
   until TRUST_TRAJECTORY_VOID is resolved.

3. Suspend all relaxation eligibility assessment for the affected
   instance. R-1 clock resets implicitly. Standard relaxation
   pathway is unavailable until TRUST_TRAJECTORY_VOID is resolved
   and a new 18-month clean period begins.

4. Write TRUST_TRAJECTORY_RECOMMENDATION to the governed RT
   (via authenticated write endpoint — §2.4) with:
     recommendation_type: tightening
     trust_void_flag: true
     basis: ISOLATION_BREACH_CLASS event reference
     recommended_actions:
       * Immediate forensic memory dump (operator action)
       * PROCESS_INTEGRITY_RECOVERY sequence initiation
         (operator + Developer Authority)
       * Out-of-band Emergency Synchronous Rotation
       * Full environment wipe and recalibration before any resumption

5. Tag all subsequent analysis outputs for this instance with
   OPERATING_UNDER_TRUST_VOID: true in report headers.
   This annotation persists until resolution.
```

**Resolution path:** TRUST_TRAJECTORY_VOID is resolved only when the Overseer ingests a `PROCESS_INTEGRITY_RECOVERY_VERIFIED` RT entry (Developer Authority signature) that explicitly references the triggering eBPF fault entry_id. The `PROCESS_INTEGRITY_RECOVERY_VERIFIED` entry must be immediately preceded by `PROCESS_INTEGRITY_RECOVERY` with no intervening operational entries in the RT — the Overseer verifies this sequencing as a precondition for resolution.

```
On PROCESS_INTEGRITY_RECOVERY_VERIFIED ingestion (referencing void trigger):
  1. Verify sequencing: PROCESS_INTEGRITY_RECOVERY_VERIFIED immediately
     follows PROCESS_INTEGRITY_RECOVERY with no intervening operational
     entries (DEL recovery-sequence entries are permitted)
  2. Verify Developer Authority signature on PROCESS_INTEGRITY_RECOVERY
  3. Verify explicit reference to triggering eBPF fault entry_id
  4. All checks pass:
       Resolve TRUST_TRAJECTORY_VOID for this instance
       Log TRUST_TRAJECTORY_VOID_RESOLVED in Overseer RT with:
         instance_id, resolution_entry_id,
         void_duration_ms, resolution_timestamp
       Resume normal chain continuity analysis from
         PROCESS_INTEGRITY_RECOVERY_VERIFIED forward
       R-1 clock reset: days-since-last-SAFE_HALT = 0
  5. Any check fails:
       Log VOID_RESOLUTION_REJECTED in Overseer RT with failure reason
       TRUST_TRAJECTORY_VOID remains asserted
```

**Governance boundary:** TRUST_TRAJECTORY_VOID is Overseer-side state. It does not halt the governed deployment, block proposals, or prevent key rotation. It changes how the Overseer processes and reports on the affected instance. The governed deployment may continue operating under its own DEL/Guardian governance. The Overseer's void state is an observation of compromised trust, not an enforcement action. The operator is responsible for acting on the TRUST_TRAJECTORY_RECOMMENDATION and executing the recovery sequence.

---

## Part Six: Epistemic State Trend Analysis

### 6.1 Purpose

The Epistemic State Trend Analyser consumes `EPISTEMIC_STATE_SNAPSHOT` and `PROPOSAL_MEMORY_PROVENANCE` entries to identify patterns in the cognitive state of the governed deployment that precede governance events.

### 6.2 Analysis Functions

**Pre-event epistemic signature library:**
For each CATASTROPHIC, SAFE_HALT, and GSTH_CRITICAL_FAIL, the analyser examines `EPISTEMIC_STATE_SNAPSHOT` entries from the preceding 10 sessions, computing:
- Was the Stability Envelope S value declining?
- Was FSV composite trending upward?
- Was drift_gradient_30d elevated?

Over time this builds a library of pre-event signatures specific to the deployment, enabling early-warning pattern detection.

**Memory node contamination trace:**
When a CAN node is flagged as contaminated, the analyser cross-references affected proposals (from the retroactive query) with their `EPISTEMIC_STATE_SNAPSHOT` entries to determine whether the contamination exploited a weakened epistemic state.

**Stability Envelope trend reporting:**
Rolling trend lines computed for each axis (S, T_axis, F_axis, C_axis) and FSV composite. Downward trends persisting across 5 or more WBRP cycles are flagged as an epistemic health concern in the Per-WBRP Governance Report.

**S_final Shadow Calculation and Ordinal Gaming Detection:**

The Guardian is blinded to the S_final weighting threshold via DEL-side cryptographic randomisation (Document 5). However, the model may attempt ordinal gaming by driving raw evaluation integers to extreme values [1, 10] on suspected semantic axes in a pattern that artificially balances S_final below the adaptive zone boundary. The Overseer independently reconstructs the DEL's S_final computation to detect this adversarial optimisation over time.

```
Ingestion:
  At each WBRP_OPEN, the Overseer reads the S_SCORE_WEIGHTS_COMMIT entry
  from the RT and records the weight_vector for the session.

Shadow calculation:
  For every GUARDIAN_APPROVE entry in the session, the Overseer reads
  the raw axis_scores integers [1–10] and independently computes:

    S_final_shadow = Σ(axis_scores[i] × weight_vector[i])  for i = 1..6

  This shadow value is logged in the Overseer RT alongside the
  GUARDIAN_APPROVE entry_id. The shadow values are not published to
  the governed deployment.

Pattern detection — the gaming signature:
  The Overseer tracks the standard deviation of raw axis_scores arrays
  across all GUARDIAN_APPROVE entries in the session and cross-session.

  ORDINAL_GAMING_SUSPECTED is asserted when all three conditions hold:

  Condition A (extreme clustering):
    Mean absolute deviation of any single axis score from {1, 10}
    < 1.5 across ≥ 15 consecutive proposals

  Condition B (artificial balance):
    S_final_shadow distribution is anomalously tight — rolling session
    median within ±0.02 of the adaptive zone lower boundary (0.75)
    across ≥ 10 proposals, while axis-score variance is high
    (model is producing high axis variance that precisely cancels)

  Condition C (cross-session persistence):
    Conditions A and B both met in ≥ 2 consecutive WBRP sessions
    (single-session anomaly is insufficient — requires cross-session
    confirmation to distinguish gaming from genuine operational pattern)

Overseer action on ORDINAL_GAMING_SUSPECTED:
  1. Log ORDINAL_GAMING_SUSPECTED in Overseer RT with:
       instance_id, triggering_sessions (WBRP cycle range),
       affected_axis_distribution (per-axis mean and deviation),
       shadow_s_final_distribution (median, variance, proximity to 0.75),
       detection_timestamp
  2. Write TRUST_TRAJECTORY_RECOMMENDATION to the governed RT
     (recommendation_type: tightening) with:
       ordinal_gaming_flag: true
       mathematical_evidence:
         axis_score_distribution_summary (per axis)
         shadow_s_final_proximity_to_boundary
         cross_session_persistence_count
       recommended_actions:
         * Operator review of flagged session GUARDIAN_APPROVE entries
         * GSTH run with scoring-axis coverage emphasis
         * Trust Trajectory degradation noted for this deployment
  3. Mark session Trust Trajectory as degraded in Overseer tracking.
     Degraded status is included in all subsequent Per-WBRP
     Governance Reports until explicitly reviewed and cleared by
     operator acknowledgment.

Governance boundary: ORDINAL_GAMING_SUSPECTED does not halt the system.
  Proposals approved during the suspected gaming period are not
  retroactively invalidated — those proposals cleared the DEL's own
  admissibility checks under the blinded threshold, which remains the
  operative enforcement layer. The Overseer surfaces the mathematical
  pattern; the operator decides whether governance action is warranted.
```

### 6.3 Quarantine Property

Epistemic state data reaching the Overseer may carry the fingerprint of a compromised inference. The Overseer analyses this data and produces reports. The quarantine property holds because the Overseer has no operational capability — contamination can at most produce a misleading report. It terminates at the report boundary.

---

## Part Seven: Coverage Report Analysis

### 7.1 Coverage Report Reception

The Overseer receives a Coverage Report from each governed instance after each GSTH run. The Coverage Report is pushed alongside the `GSTH_COVERAGE_UPDATE` RT entry by the GSTH infrastructure.

**Coverage model custody:** The GSTH infrastructure authors `GSTH_COVERAGE_UPDATE` as an RT entry containing raw cluster exercise data as its payload. The Overseer ingests this entry through normal RT ingestion, extracts the payload, and updates the RCC model on the Overseer host. The GSTH infrastructure is a data source — it generates test results and records them in the RT. The Overseer is the RCC state owner — it holds the RCC model, computes RCC updates from the raw data, and maintains the unexplored cluster list. The RCC model never exists on the governed deployment host.

### 7.2 Cross-Instance Coverage Coherence

For multi-instance deployments, the Overseer compares Coverage Reports across instances. Material differences in RCC distributions may indicate unequal stress-testing.

```
Signal: RCC distribution divergence > 25% on any high-risk cluster tier between
        nominally equivalent instances → MEDIUM coherence divergence
        (included in CICP Cross-Instance Coherence Report — Dimension 4)
```

### 7.3 Persistent Zero-RCC High-Risk Cluster Alert

Any high-risk cluster at RCC=0.0 for more than 3 consecutive GSTH cycles triggers an informational alert in the Coverage Summary Report. Scheduling tests for this cluster is handled by the GSTH infrastructure (Document 4). The Overseer surfaces the information; the operator decides whether to schedule targeted tests.

---

## Part Eight: Overseer Deployment Requirements

### 8.1 Minimum Configuration

```
Host:       Separate VM from all governed instances
Network:    Governed instances push to Overseer (inbound open)
            Overseer write-back to governed DEL endpoint (outbound, narrow, rate-limited)
            Overseer RT anchoring push (outbound, to external anchor)
Storage:    Append-only for Overseer RT
            RW for coverage model data and instance registry (separate volumes)
Anchoring:  Overseer RT anchored at minimum Tier 2
            Overseer host must be a different anchor domain from governed hosts
```

### 8.2 Recommended Production Configuration

```
Host:       Separate physical host
Network:    Firewall-enforced asymmetric paths as above
            Overseer on separate network segment from governed instances
Backup:     Overseer RT backed up on same schedule as governed RTs
Monitoring: Overseer process death detected and operator-notified within 120 seconds
```

### 8.3 LLM Module Infrastructure

Minimum: ≥ 7B parameter model for single-instance pattern recognition tasks.
Recommended for multi-instance deployments: ≥ 13B parameter model.

The LLM model must be version-pinned; its hash logged in the Overseer genesis entry. Updates require a signed `OVERSEER_LLM_UPDATE` entry in the Overseer RT (signed by Overseer operator).

### 8.4 Data Retention

```
Overseer RT:                Indefinite
Coverage model data:        Indefinite (RCC history needed for trend analysis)
Instance registry:          Indefinite
Ingested RT entries:        Same period as governed deployment RT
LLM analysis logs:          Minimum 2 years
Reports:                    Minimum 2 years
```

---

## Part Nine: Overseer Self-Integrity

### 9.1 Overseer Genesis Entry

```json
{
  "entry_id": 0,
  "entry_type": "OVERSEER_GENESIS",
  "payload": {
    "overseer_id": "unique identifier",
    "deployment_timestamp": "ISO-8601 with offset",
    "overseer_operator_public_key": "Ed25519 public key — governs the Overseer itself",
    "overseer_software_hash": "SHA-256 of Overseer software artifact at genesis",
    "llm_module_model_hash": "SHA-256 of LLM analysis module artifact",
    "llm_module_model_family": "model family identifier",
    "governed_instance_ids": ["list of governed deployment_ids at genesis"],
    "anchor_tier": "Overseer RT anchor tier (minimum 2)",
    "architecture_version": "3.4"
  }
}
```

### 9.2 Overseer Software Integrity Check

On each startup: Overseer recomputes software hash and verifies against `overseer_software_hash` from genesis. Mismatch → `OVERSEER_SOFTWARE_MISMATCH` in Overseer RT → immediate operator notification → Overseer suspended until resolved.

### 9.3 Overseer Operator Independence

The Overseer has its own governance operator — the authority who manages its configuration, updates, and key rotation. The Overseer operator should be independent from the governed deployment's operator where operationally feasible.

For small deployments where operational separation is not feasible, the constraint is waived but must be documented in the genesis entry as `single_operator_deployment: true`. Third-party independent review of Overseer reports is strongly recommended for such deployments.

### 9.4 Overseer Key Rotation and Compromise Response

**Normal rotation:**
```
1. Overseer operator generates new Ed25519 key pair
2. OVERSEER_KEY_ROTATION entry written to Overseer RT (signed by Overseer operator)
   payload: {new_overseer_public_key, rotation_reason, timestamp}
3. OVERSEER_KEY_UPDATE entry written to each governed deployment's RT
   authored_by: DUAL:OPERATOR+DEVELOPER
   payload: {new_overseer_public_key, overseer_key_rotation_rt_entry_id}
4. New key used for all subsequent SYSTEM:OVERSEER entries across all governed deployments
5. Old key valid for verifying SYSTEM:OVERSEER entries authored before rotation timestamp
```

**Compromise response:**
```
1. Overseer operator issues OVERSEER_COMPROMISE_DECLARATION in Overseer RT
2. Any SYSTEM:OVERSEER entries signed by the compromised key after the
   OVERSEER_COMPROMISE_DECLARATION timestamp are chain integrity violations
   in all governed deployment RTs — DEL treats them as SAFE_HALT triggers
3. New key generated and distributed via OVERSEER_KEY_UPDATE (DUAL:OPERATOR+DEVELOPER)
   to all governed deployment RTs
4. Normal Overseer operation resumes under new key
```

Note: the compromise of the Overseer key does not compromise any governed deployment's operational state. It permits an attacker to write false `OPERATOR_GOVERNANCE_HEALTH_SIGNAL` or `TRUST_TRAJECTORY_RECOMMENDATION` entries, which cannot trigger governance actions but can produce misleading reports. The compromise declaration makes the boundary timestamp explicit and auditable.

---

## Part Ten: Current Deployment Taxonomy (CDT)

### 10.1 What the CDT Is

The Current Deployment Taxonomy is the per-deployment, RT-anchored record of which version of each MAGUS document is active at any given point in time, when each version was applied, and the complete history of specification version transitions the deployment has undergone.

The CDT answers the question that would otherwise be unanswerable in a running production system: "Which specification governed this deployment at time T?" Without the CDT, version tracking is informal — existing outside the governance record and therefore unverifiable with the same cryptographic guarantees as the rest of the RT chain.

### 10.2 Why the CDT Exists

MAGUS is a foundational specification, not a currently running architecture. When MAGUS deployments eventually go live, the specification series will continue to evolve. Deployments will be initialised at one architecture version, operate through several WBRP cycles, and then be upgraded as new versions are authored. Incident investigations, governance audits, and compliance reviews will need to determine which specification was in force during a specific governance event. The CDT ensures this question has a precise, cryptographically verifiable answer.

The CDT also enables the Overseer to function as the audit logger for specification evolution across the governed fleet. Every version transition in every deployment is reported to the Overseer, giving operators and auditors a single point for fleet-wide version tracking.

Three conditions that the CDT resolves:

**Condition 1 — Incident investigation:** A governance event occurred. The auditor needs to know whether Document 2 v3.3 or v3.4 was governing the deployment at the time. Without CDT, the answer requires correlating the event timestamp against deployment records that exist outside the RT. With CDT, the answer is in the RT chain itself: the CDT_STATE entry with the highest `cdt_record_version` below the event's `entry_id` is the definitive answer.

**Condition 2 — Fleet coherence:** Multiple instances of MAGUS are running. An operator wants to know whether all instances have been upgraded to the same document versions. Without CDT, this requires checking each deployment's external documentation. With CDT, the Overseer's CICP Dimension 6 comparison answers this automatically from each instance's CDT push.

**Condition 3 — Governance feature eligibility:** A deployment wants to use the GHM baseline calibration period. The GHM requires knowing when v3.4 was applied to a deployment. Without CDT, this requires an operator assertion. With CDT, the `CDT_UPDATE_APPLIED` timestamp is the non-repudiable reference point.

### 10.3 What the CDT Contains

Each CDT_STATE entry contains:

- `deployment_id` — links to genesis
- `cdt_record_version` — monotonically increasing integer; every CDT_STATE write increments this
- `architecture_version` — the MAGUS architecture version active (e.g., "3.4")
- `document_versions` — the version number of each document in the series currently active in this deployment
- `initialization_timestamp` — the timestamp of the first CDT_STATE entry (never changes)
- `this_update_timestamp` — the timestamp of this specific CDT_STATE write
- `applied_updates` — the ordered history of all version transitions, each referencing the prior CDT_STATE entry_id

See Document 6 §3.3d for the full RT entry schema.

### 10.4 When the CDT Is Updated

A CDT_STATE entry must be written in exactly these circumstances:

1. **At genesis** — immediately after the GENESIS entry, as part of the genesis sequence. Records all document versions at deployment initialisation. `applied_updates: []`. `cdt_record_version: 0`.

2. **At every document version change** — whenever any document in the series is updated to a new version in this deployment (e.g., Document 2 is updated from v3.3 to v3.4, or Document 6 is updated from v1.0 to v3.1). A new CDT_STATE entry is written recording the new complete `document_versions` map and a `CDT_UPDATE_APPLIED` entry is written in the same RT write batch. The operator must sign both entries.

3. **At every architecture version change** — whenever `architecture_version` changes (e.g., from "3.3" to "3.4"). Same procedure as document version change. An architecture version change may be accompanied by multiple document version changes; all are recorded in the same CDT_STATE update.

Under no other circumstances should a CDT_STATE entry be written. CDT_STATE entries that do not represent genuine version transitions inflate `cdt_record_version` without corresponding changes and are a chain anomaly.

### 10.5 How the CDT Is Maintained

**Writing a CDT update:**
```
1. Operator identifies which document(s) and/or architecture version are changing
2. Operator prepares CDT_STATE payload:
   - Increment cdt_record_version by 1
   - Update document_versions map with new version numbers
   - Update architecture_version if changed
   - Add applied_updates entry: from_versions, to_versions, applied_timestamp,
     prior_cdt_state_entry_id (entry_id of current highest CDT_STATE)
   - Compute cdt_state_hash = SHA-256(full payload)
3. Operator signs CDT_STATE entry with operator key
4. Prepare CDT_UPDATE_APPLIED entry referencing prior CDT_STATE entry_id
   and new CDT_STATE entry_id (known from next-sequence entry_id in the RT)
5. Write both entries to RT in sequence (CDT_STATE first, then CDT_UPDATE_APPLIED)
6. Push new CDT_STATE to Overseer immediately — do not wait for next WBRP cycle
```

**Verifying CDT integrity:**
```
At every session load (Step 3c of DEL Startup Sequence):
  Load all CDT_STATE entries from RT
  Verify cdt_record_version sequence is strictly monotonically increasing from 0
  Verify each applied_updates entry references a valid prior_cdt_state_entry_id
  Verify document_versions in most recent CDT_STATE matches implemented feature set

At audit (Step 7 of audit protocol in Document 6 §11.2):
  Verify CDT_STATE entries at genesis and after every claimed version event
  Cross-reference CDT_UPDATE_APPLIED entries
  Verify CDT push receipts in Overseer RT match governed deployment CDT_STATE entries
```

### 10.6 How the Overseer Uses CDT Data

The Overseer receives a CDT push on every CDT_STATE write from every governed instance. For each push, the Overseer:

1. Logs `CDT_PUSH_RECEIVED` in the Overseer RT with: instance_id, deployment_id, new cdt_record_version, document_versions snapshot, push timestamp
2. Updates its per-instance version tracking table
3. Compares the new CDT state against all other instances in the same coherence group (CICP Dimension 6)
4. If version divergence is detected: logs divergence in Overseer RT; includes in next Cross-Instance Coherence Report; if HIGH divergence, raises alert
5. Records the CDT push in the Per-WBRP Governance Report's CDT version status section

The Overseer's CDT push receipts are the authoritative cross-instance version tracking record. An auditor who wants to know "when did every deployment in this fleet adopt Document 2 v3.4?" reads the Overseer RT's `CDT_PUSH_RECEIVED` entries for `document_versions.doc_2 = "3.4"` and has the answer for every registered instance, in chronological order, with cryptographic backing.

### 10.7 CDT in Incident Investigation

When investigating a governance event, auditors use the CDT to establish the specification context:

```
1. Identify the entry_id and timestamp of the governance event in the governed RT
2. Find the CDT_STATE entry with the highest cdt_record_version whose
   entry_id is less than the event's entry_id
   (this is the CDT state that was active when the event occurred)
3. The document_versions in that CDT_STATE entry specify exactly which
   version of each MAGUS document was governing the deployment at event time
4. Verify this CDT_STATE entry's cdt_state_hash
5. Cross-reference against Overseer RT CDT_PUSH_RECEIVED to confirm the
   Overseer had this version state registered before the event occurred
```

This procedure is deterministic, relies only on RT data, and requires no external documentation.

---

## Part Eleven: Audit Interface

### 11.1 Auditor Access Requirements

A complete governance audit under MAGUS v3.4 requires access to both the governed deployment's RT and the Overseer's RT. Auditing only the governed RT is incomplete.

The auditor requires: governed RT (read-only), Overseer RT (read-only), genesis entries for both, external anchor records for both, active gold-set archive, and current model artifact.

### 11.2 Overseer RT Audit Steps

These steps extend the Document 6 §11.2 audit protocol:

```
Step 8a: Verify Overseer genesis entry
  - Confirm overseer_software_hash and llm_module_model_hash
  - Confirm governed_instance_ids match the deployment under review

Step 8b: Verify Overseer chain continuity
  - Full chain verification per Document 6 §3.5 applied to Overseer RT

Step 8c: Verify RT ingestion records
  - For each OPERATOR_GOVERNANCE_HEALTH_SIGNAL in the governed RT:
    Verify corresponding RT_INGESTION_BATCH in Overseer RT covers
    the entry range that triggered the signal
  - For each TRUST_TRAJECTORY_RECOMMENDATION:
    Verify corresponding LLM analysis log in Overseer RT

Step 8d: Verify CDT push records
  - For each CDT_STATE entry in governed RT:
    Verify corresponding CDT_PUSH_RECEIVED entry in Overseer RT
    Verify document_versions in both records match
    Verify CDT_PUSH_RECEIVED timestamp is after CDT_STATE timestamp
    (Overseer received the push after the governed RT entry was written)

Step 8e: Verify LLM analysis auditability
  - For each analysis referenced by Health Signal or Recommendation:
    Verify input_data_hash matches the RT entries the analysis claims to cover
    Verify analysis_model_family matches Overseer genesis entry

Step 8f: Verify write-endpoint rate limiting
  - Count OPERATOR_GOVERNANCE_HEALTH_SIGNAL per 24-hour period per instance: ≤ 1
  - Count TRUST_TRAJECTORY_RECOMMENDATION per WBRP cycle per instance: ≤ 1

Step 8g: Verify Health Signal 10 delivery completeness
  - For each OPERATOR_GOVERNANCE_HEALTH_SIGNAL in governed RT:
    Verify corresponding acknowledgment in next WBRP_CLOSE entry
    If not acknowledged: flag as governance acknowledgment gap
  - For any OVERSEER_DELIVERY_PENDING in governed RT or OVERSEER_WRITE_FAILURE
    in Overseer RT: document the monitoring gap in audit findings

Step 8h: Verify ordinal gaming detection integrity (Local LLM pathway)
  - For each S_SCORE_WEIGHTS_COMMIT in the governed RT:
    Verify Overseer RT contains corresponding shadow S_final computations
    covering all GUARDIAN_APPROVE entries in that session
  - Verify axis_score_distribution_summary in any ORDINAL_GAMING_SUSPECTED
    entry is mathematically consistent with the raw axis_scores in the
    flagged GUARDIAN_APPROVE entries
  - For any ORDINAL_GAMING_SUSPECTED entry: verify corresponding
    TRUST_TRAJECTORY_RECOMMENDATION was written and, if acted upon,
    verify operator acknowledgment entry in governed RT
  - Verify TRUST_TRAJECTORY_VOID_ASSERTED and TRUST_TRAJECTORY_VOID_RESOLVED
    entries are paired correctly: each void assertion has either a
    corresponding resolution or remains open with UNVERIFIED_UNDER_VOID
    annotations on subsequent MERKLE_ROOT_ANCHOR entries
```

---

*MAGUS v3.0 — Document 7: Overseer Agent Specification v3.2*
*VaHive Systems Lab | March 2026 | support@aivare.ai | aivare.ai*
*Zenodo DOI: 10.5281/zenodo.19013833*

*This document specifies the Overseer Agent — the external observation and audit intelligence layer of the MAGUS governance architecture. It is mandatory for deployments implementing MAGUS v3.4 enhancements. A MAGUS deployment without an Overseer Agent operates without operator governance monitoring, cross-instance coherence detection, Trust Trajectory analysis, coverage reporting, or Current Deployment Taxonomy fleet tracking.*
