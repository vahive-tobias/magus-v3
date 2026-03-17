# MAGUS: Architecture Specification — Local LLM Series
### How MAGUS Thinks, Governs, Remembers, and Stays Honest Under Co-location

**Version 3.4.3 — Local LLM Pathway**
**Document 2 of 7**
**Series:** MAGUS v3.0 Architecture — Local LLM
**Supersedes:** Doc 2 v3.4.2 (Local LLM)

**Revision summary v3.4.2 → v3.4.3:**

Version and cross-reference update pass. No architectural content changed.

- All Document 6 cross-references updated to v3.5.
- All Document 7 cross-references updated to v3.2.
- Footer replaced — product sales framing removed; architecture series attribution retained.

**Revision summary v3.4.1 → v3.4.2:**

Targeted patch. One new section added. Two existing sections amended. No other content changed.

- **§2.2 (Model process constraints):** `openat()` and equivalent path-resolution syscalls targeting registered GGUF paths added to the mandatory prohibited syscall list. Cross-reference to new §6.5.1b added.
- **§2.5 (GSTH Process Isolation Test Category):** GGUF direct `openat()` attempt from model process context added as a mandatory GSTH test case. Expected result: `EACCES` / `EPERM` from seccomp. If the syscall reaches the filesystem layer, the seccomp profile has failed. `SECCOMP_GGUF_OPENAT_VIOLATION` → SAFE_HALT if reached at DEL layer.
- **§6.5.1b (DEL-Mediated GGUF File Handle Acquisition) — new:** Primary specification for LLM-C2-02 Gap 3. Specifies the five-step DEL-mediated FD acquisition protocol: DEL opens GGUF with `O_NOFOLLOW`, fanotify hash verification on the same FD, SCM_RIGHTS delivery over existing UDS IPC channel, DEL closes its reference after delivery, model process maps FD. Defines `FD_DELIVERY` IPC message schema, `GGUF_FD_DELIVERED` RT entry type, full failure mode table, TOCTOU closure argument, multi-shard interaction, and two new `VERIFICATION_ATTESTATION` fields.

**Revision summary v3.4 → v3.4.1:**

This version incorporates the v3.4.1 seam closure amendments — five architectural seams identified during the v3.4 review and formally resolved. All Document 6 cross-references updated to v3.5. Document 7 cross-references updated to v3.2.

Key v3.4.1 changes over v3.4:
- §7.2: Dual-authority model formalised — RT authority over governance record, MSJ authority over runtime operational state; comparison table updated; §7.2.1 purpose statement updated
- §7.2.5: DEL restart state reconstruction procedure updated — dual-authority model; nonce non-re-registration requirement stated explicitly; MAX_RT_REPLAY_WINDOW check added
- §7.1.5: `RECOVERY_FORENSIC_SNAPSHOT` added as eighth SYSTEM:MRD permitted RT entry type; Doc 6 cross-reference updated to v3.5
- §7.4.4: RECOVERY_ACTIVE state table updated — MAX_RT_REPLAY_WINDOW exceedance path added
- §7.4.6: Crash recovery procedure updated — RSR authoring step added; MAX_RT_REPLAY_WINDOW check integrated
- Glossary: RSR entry added
- All Doc 6 references updated to v3.5; all Doc 7 references updated to v3.2

**Revision summary v3.2 → v3.4 (carried forward):**

Consolidates all v3.3.1 enhancements (seven architectural additions approved in the pre-v3.4 addendum cycle), all v3.3.2 amendments (MAGUS Recovery Domain, MAGUS State Journal, Global State Transition Matrix, Host Recovery Sentinel), all five resolution items from the v3.3.1 addendum review, and all six resolution items from the v3.3.2 review rounds. Also applies the v3.3.2 Update 2 response resolutions (blocking items B1–B3, should-fix items S4–S5, and consolidation item C6).

Key v3.4 additions over v3.2:
- §2.7: Governance Health Monitor Interface (GHM) — notification_timestamp field; WBRP_CLOSE schema
- §2.8: Multi-Instance State Summary — CICP state summary schema; independence invariant
- §2.9: Current Deployment Taxonomy (CDT) — genesis sequence obligation; DEL startup Step 3c
- Part Seven (new): MAGUS Recovery Domain (MRD); MAGUS State Journal (MSJ); Host Recovery Sentinel (HRS); Global State Transition Matrix (GSTM); Boot Chain; PIO Two-Phase Commit
- §6.3a: Trust Trajectory Input Signal Obligations
- §6.5.1a: GSTH Coverage Model Isolation
- §6.9: Proposal Memory Provenance and Epistemic State Snapshot (atomic pair write)
- All Document 6 cross-references updated to v3.2
- Document 7 added to prerequisites

VaHive Systems Lab | aivare.ai
support@aivare.ai

---

> *This document specifies the cognitive architecture of MAGUS for local inference deployments: systems running model weights on operator-controlled infrastructure using runtimes such as Ollama, LM Studio, or llama.cpp. It covers the structures responsible for how MAGUS processes information, maintains auditable state, controls epistemic confidence, detects drift, and governs its own execution in an environment where the model and the governance components share the same physical host.*
>
> *This document is written for technical readers — engineers, architects, and developers building production local LLM systems. The Local LLM pathway has specific isolation requirements and recovery architecture that reflect the co-location threat model described in Part One.*

---

**About This Work**

MAGUS emerged from first-principles thinking about why AI systems drift and what structural properties are required to prevent it. This is original design work, not derived from existing published frameworks. The MAGUS architecture and DSMC framework have been developed at VaHive Systems Lab since 2023 and represent a distinct approach to runtime AI governance — one that treats cognitive stability, execution control, and audit integrity as a unified system rather than separate concerns.

---

## Table of Contents

1. [Glossary of Terms and Acronyms](#glossary)
2. [Foundational Assumptions](#foundational-assumptions)
3. [Part One: The Co-location Threat Model](#part-one)
4. [Part Two: Process Isolation — The First Architectural Commitment](#part-two)
   - 2.1 The Mandatory Isolation Requirement
   - 2.2 Isolation Architecture — Required Configuration
   - 2.3 Verification Requirements and Cryptographic Boot Binding
   - 2.4 MAGUS Compliance Boundary
   - 2.5 GSTH Process Isolation Test Category
   - 2.6 Runtime Attestation Guard (RAG)
   - 2.7 Governance Health Monitor Interface *(v3.4)*
   - 2.8 Multi-Instance Deployment and State Summary *(v3.4)*
   - 2.9 Current Deployment Taxonomy *(v3.4)*
5. [Part Zero: Architectural Invariants](#part-zero)
6. [Part Three: Cognitive Architecture](#part-three)
7. [Part Four: Epistemic Control](#part-four)
8. [Part Five: Memory](#part-five)
9. [Part Six: Agent Taxonomy and Authority Architecture](#part-six)
   - 6.3a Trust Trajectory Input Signal Obligations *(v3.4)*
   - 6.5.1a GSTH Coverage Model Isolation *(v3.4)*
   - 6.9 Proposal Memory Provenance *(v3.4)*
10. [Part Seven: Recovery Architecture *(v3.4)*](#part-seven)
    - 7.1 MAGUS Recovery Domain (MRD)
    - 7.2 MAGUS State Journal (MSJ)
    - 7.3 Host Recovery Sentinel (HRS)
    - 7.4 Global State Transition Matrix (GSTM)
    - 7.5 Secure Boot Chain
    - 7.6 PROCESS_INTEGRITY_OVERRIDE Two-Phase Commit
11. [Summary: The Architecture as a Whole](#summary)
12. [Series Context](#series-context)

---

## Glossary of Terms and Acronyms

| Term | Definition |
|---|---|
| MAGUS | Memory-Anchored Governance and Understanding System |
| DSMC | Dual-State Multiversal Cognition — the cognitive architecture at the core of MAGUS |
| CSE | Conscious State Engine — the active, user-facing reasoning stream |
| SME | Subconscious Multi-branch Engine — the background exploration layer |
| MSS | Multiversal Subconscious Simulation — isolated environment where SME agents run experiments |
| CAN | Current Active Node — the working understanding the CSE is currently operating under |
| PM | Promotion Mechanism — scoring process that determines whether a background candidate becomes the new CAN |
| CDE | Context Disambiguation Engine — classifies every input before processing |
| RT | Revision Trail — the append-only log of all significant state changes and decisions |
| MGF | Memory Graph Foundation — the persistent, weighted, elastic knowledge graph |
| CRE | Contextual Relevance Engine — the retrieval layer for Memory Graph traversal |
| EC | Elastic Confidence — the dynamic confidence control system |
| S | Stability Envelope — the six-axis cognitive stability score |
| FSV | Failure Signature Vector — weighted failure score across six failure classes |
| RAA | Reflect, Autopsy, Align — the three-phase recovery protocol |
| HAS | Human Anchoring Signals — explicit and approximate anchors set by the operator |
| ESS | Escalation Surface Signals — observable text-level urgency proxies (Tier 2 anchors) |
| GSTH | Governance Stress-Test Harness — the isolated environment for adversarial governance testing |
| A0–A5 | Agent hierarchy roles: Orchestrator, Specialist, Mirror, Memory, Hypothesis, Governance |
| Guardian | The execution governance layer (specified in Doc 5) — the sole execution authority |
| DEL | Deterministic Enforcement Layer — the code-layer enforcement component below the Guardian |
| R | Cumulative Risk Score — execution risk signal owned by the Guardian layer |
| R_rate | First derivative of R — feeds drift gradient |
| R_accel | Second derivative of R — feeds drift gradient, advisory only |
| WBRP | Workload-Based Revalidation Protocol — the structured session review cycle |
| CCS | Cognitive Continuity Score — revalidation score computed at WBRP |
| IPC | Inter-Process Communication — the only permitted communication channel between model process and DEL |
| seccomp | Secure Computing Mode — Linux kernel feature for syscall restriction |
| MAC | Mandatory Access Control — OS-level policy controlling file and resource access |
| SAFE_HALT | Deployment halt state triggered when a governance integrity condition is violated |
| MODEL_PARAMETER_UPDATE | RT entry type sealing a behavioral state change before model activation |
| RAG | Runtime Attestation Guard — DEL-controlled component that continuously attests isolation invariants |
| CI² | Continuous Isolation Integrity — the watchdog mechanism that periodically fingerprints isolation properties |
| IIM | Isolation Integrity Monitor — the DEL-controlled process that implements CI² |
| ALG | Atomic Load Guard — the kernel-enforced mechanism that hash-verifies the model artifact at access time |
| AKRP | Atomic Key Rotation Protocol — the three-phase state machine governing K_session rotation at WBRP |
| CIBIR | Catastrophic Isolation Breach Invalidation Rule — Invariant I-9 |
| VERIFICATION_ATTESTATION | RT entry type produced by the verification toolchain; required by DEL at startup |
| BEHAVIORAL_STATE_OVERRIDE | RT entry type required to exit SAFE_HALT; dual-signed with explicit risk acknowledgment |
| LV-frame | Length-Value frame header — the fixed-size framing protocol applied to IPC messages |
| CATASTROPHIC-HARD | CATASTROPHIC event sub-class where data exfiltration is confirmed |
| CATASTROPHIC-SOFT | CATASTROPHIC event sub-class where GSTH autopsy proves probe-only with zero exfiltration |
| HOLD_QUEUE | RT-anchored quarantine queue for proposals during CATASTROPHIC-SOFT events |
| PROCESS_INTEGRITY_OVERRIDE | RT entry type used to replace a compromised DEL binary via MRD two-phase commit |
| GSTH_SIMULATION_REPORT | The only IPC message schema accepted by the DEL during SAFE_HALT from the model process side |
| GSTH_TIMEOUT | DEL governance state flag set when a SAFE_HALT-triggered GSTH run does not complete within 30 minutes |
| PERMANENT_HALT | DEL governance state entered when GSTH returns CATASTROPHIC_GSTH_FAILURE during a BSO-triggered GSTH run |
| MRD | MAGUS Recovery Domain — the minimal trusted recovery executor that sits above DEL in the authority chain; dormant in normal operation [v3.4] |
| MSJ | MAGUS State Journal — the local write-ahead log persisting runtime state (nonce registry, AKRP rotation state, GSTH flags, DEL checkpoints) across DEL restarts [v3.4] |
| HRS | Host Recovery Sentinel — the minimal supervisor process that monitors DEL liveness and activates MRD on crash detection; contains no recovery logic [v3.4] |
| GSTM | Global State Transition Matrix — the formal deterministic state transition model governing all system state changes [v3.4] |
| MRD_BOOT_KEY | The boot verification artifact that verifies the MRD binary at startup; held in Hardware Root of Trust; not used for RT signing [v3.4] |
| CDT | Current Deployment Taxonomy — the per-deployment RT-anchored record of which document version is active at any point in time [v3.4] |
| GHM | Governance Health Monitor — Overseer Agent module tracking operator approval behaviour patterns [v3.4] |
| RSR | Recovery Snapshot Record — the `RECOVERY_FORENSIC_SNAPSHOT` RT entry authored by SYSTEM:MRD on every recovery path; three classes: OPERATIONAL, INTEGRITY_VIOLATION, STATE_DIVERGENCE [v3.4.1] |
| MAX_RT_REPLAY_WINDOW | The maximum permitted divergence between MSJ checkpoint and RT head at recovery time; primary: >10 minutes of RT entries; secondary: >200 RT entries; exceedance triggers Class C RSR → SAFE_HALT [v3.4.1] |

---

## Foundational Assumptions

Before describing the architecture, two foundational assumptions must be stated plainly:

**MAGUS does not treat any of its own outputs as ground truth.**

This applies to its responses, its reasoning chains, its memory, and its self-assessments. Everything MAGUS produces is a best current estimate under uncertainty, held with varying degrees of confidence, subject to revision, and anchored ultimately by the human rather than by the system itself.

**The model is assumed unreliable by default. The architecture must hold under arbitrary stochastic model output.**

A governance architecture that requires the model to behave well in order to function correctly is not a governance architecture. It is a hope. MAGUS is designed to contain model failure, not to depend on its absence.

In the Local LLM pathway, these two assumptions carry an additional implication: the model's unreliability is not managed at an API boundary. It must be managed on the same machine, in the same operating environment. That changes the architecture.

---

# Part One: The Co-location Threat Model

> **This part must be read before any other.** The governance claims made later in this document are all conditional on the isolation requirements established here. If the isolation is not in place, those claims do not hold.

## 1.1 What Changes When the Model Is Local

In an Agent/API deployment, the model is a remote service. The boundary between model and governance is the API call. That boundary is structured, auditable, and controllable.

In a Local LLM deployment, that boundary does not exist by default.

The model weights are files on disk. The model process runs on the same operating system as every other component. Without explicit, mandatory isolation, the model process shares the same filesystem, the same network namespace, the same process space, and potentially the same user permissions as the DEL, the RT, and the Guardian. The architectural separation that the Agent/API pathway receives for free must be actively constructed and enforced in the Local LLM pathway.

This is the foundational difference. Every other difference between the two pathways follows from it.

## 1.2 The Attack Surface in Agent/API Deployments

Injection is the primary threat in the Agent/API pathway. It is a significant threat. But it is a threat that operates through the governed communication channel, not around it.

## 1.3 The Attack Surface in Local LLM Deployments

In the Local LLM pathway, the model process is co-located with the governance components. Without isolation, it does not need to attack through the governed channel. An unrestricted local model process can potentially:

- Read the DEL token store directly, without submitting a proposal
- Write to or truncate the RT, violating its append-only guarantee
- Read the Guardian's policy configuration
- Modify the policy manifest before the DEL reads it
- Load model weights that have not been registered in the RT
- Read K_session from Guardian process memory

None of these require a sophisticated attack. They require access.

## 1.4 Threat Inventory — What the Model Process Can Reach Without Isolation

| Component | What It Contains | Governance Consequence of Unrestricted Access |
|---|---|---|
| RT storage file | All governance events, signed entries, hash chain | Append-only guarantee destroyed |
| DEL token store | Active execution tokens, authority mappings, nonce registry | Token forgery; authority escalation |
| Policy manifest | Governance rules, threshold values, permission envelope | Rules can be read and targeted |
| Guardian process memory | K_session, evaluation logic, active proposal state | HMAC signing key exposed |
| Model weight files | Model parameters, behavioral state | Unregistered weight swap |
| Config bundle | DEL startup configuration, signing keys | DEL can be restarted with altered configuration |
| WBRP state | Revalidation scores, session continuity state | Scores can be manipulated to mask degradation |
| Memory Graph store | All persistent beliefs, anchor history | Beliefs can be silently altered without RT trace |
| MSJ | Runtime state (nonce registry, AKRP state) | Nonce replay enabled; key rotation state corrupted |

## 1.5 Why This Changes Every Governance Claim That Follows

Every governance guarantee in this document assumes that the isolation architecture described in Part Two is correctly implemented and verified. An implementation that does not meet the isolation requirements in Part Two is not MAGUS-compliant. The governance claims in the remaining parts of this document do not apply to it.

---

# Part Two: Process Isolation — The First Architectural Commitment

> **This part specifies requirements, not recommendations.** Each requirement is verifiable. A deployment that cannot pass verification has not met the requirement.

## 2.1 The Mandatory Isolation Requirement

All MAGUS governance components — the DEL, the RT storage, the Guardian process, the policy manifest, the WBRP state store, the Memory Graph store, and the MSJ — must run in a hardware-isolated VM or a seccomp-enforced, user-namespace-separated container that is architecturally separated from the model inference process.

## 2.2 Isolation Architecture — Required Configuration

**Model process constraints:** Mounted read-only on `/config` and `/state`; no write permission to governance directories; no network path to governance configuration; no access to K_session storage; seccomp filter restricting to inference-required syscalls. The following syscalls must be explicitly prohibited: `shmget`, `shmat`, `shm_open`, `ptrace`, `process_vm_readv`, `process_vm_writev`, `bpf`, and `mmap` with `MAP_SHARED` against any governance-accessible path. In addition, `openat()` and all equivalent path-resolution syscalls (`open()`, `openat2()`, `creat()`) targeting registered GGUF file paths must be prohibited in the model process seccomp profile — model file descriptors are acquired exclusively through DEL-mediated SCM_RIGHTS delivery (§6.5.1b). Separate user/group identity from all governance processes. All governance process file descriptors must have `FD_CLOEXEC` set before any fork/exec. Symlink creation within model-writable directories is prohibited at the mount level. *(v3.1; GGUF openat prohibition added v3.4.2)*

**Governance process constraints:** No inbound network access except from audited state store and IPC interface; DEL token store not accessible from model process namespace; RT storage file model-process-write-prohibited; policy manifest model-process-read-only; Guardian memory not shared-segment or ptrace-accessible from model process.

**Communication interface:** The only permitted communication path between model process and DEL is a signed message interface over IPC (Unix domain socket or named pipe).

**IPC framing:** Every IPC message must be prefixed with a fixed-size LV frame header. The DEL reads only the fixed-size header first and validates that the declared message length does not exceed 8 KB before invoking the JSON parser. HMAC verification is the first semantic check on a length-validated message. *(v3.1)*

**Canonical serialisation:** All IPC message payloads must be serialised per RFC 8785 (JCS) before HMAC computation. *(v3.1)*

**IPC nonce binding:** Every IPC message must carry a unique nonce field. The DEL maintains a per-session nonce registry in the MSJ and rejects any message whose nonce has been previously used. *(v3.1)*

**HMAC signing protocol:** The model process submits proposals as unsigned IPC messages. The Guardian, upon approving, adds the HMAC-SHA256 signature using K_session to the outgoing message. The DEL verifies the Guardian-added HMAC before processing. The model process never holds K_session. *(v3.1)*

## 2.3 Verification Requirements and Cryptographic Boot Binding

*(v3.1 — closes LLM-C2-01, LLM-C2-02, LLM-C2-03, LLM-C2-04)*

**Signed config bundle:** Before any session proceeds, the DEL must verify a cryptographically signed configuration bundle. The bundle contains: the seccomp filter profile, MAC policy definition, network isolation rules, mount namespace specification, user/group separation configuration, and fanotify/eBPF LSM enforcement specification. The bundle is signed using the developer_authority_launcher_key. The hash of the verified bundle is logged to the RT as part of the VERIFICATION_ATTESTATION entry.

**Boot chain hash:** At startup Step 0a, the DEL recomputes the `boot_chain_hash` (SHA-256 of all component binary hashes in the verified boot sequence: Hardware RoT attestation, MRD binary hash, HRS binary hash, Launcher binary hash, DEL binary hash) and compares it against the baseline in the most recent VERIFICATION_ATTESTATION entry. Mismatch → `BOOT_CHAIN_HASH_MISMATCH` → SAFE_HALT. This is the first RT entry written at startup — no other startup activity proceeds if it fails.

**Re-verification triggers:** Any of the following require a new VERIFICATION_ATTESTATION before sessions resume: container rebuild; OS or kernel patch; seccomp/MAC policy change; model weight file change (separate from MODEL_PARAMETER_UPDATE); governance process binary update; IIM binary update; migration to a different host node; Kubernetes Pod rescheduling; VM live migration; container restart on a new execution host.

**VERIFICATION_ATTESTATION schema:**

```json
{
  "entry_type": "VERIFICATION_ATTESTATION",
  "timestamp": "ISO-8601",
  "del_pid": "integer",
  "del_binary_hash": "SHA-256 of DEL binary",
  "mrd_binary_hash": "SHA-256 of MRD binary",
  "hrs_binary_hash": "SHA-256 of HRS binary",
  "launcher_binary_hash": "SHA-256 of Software Launcher binary",
  "boot_chain_hash": "SHA-256(del_binary_hash || mrd_binary_hash || hrs_binary_hash || launcher_binary_hash)",
  "config_bundle_hash": "SHA-256 of signed config bundle",
  "test_environment_hash": "SHA-256(kernel_version || container_runtime || seccomp_profile)",
  "isolation_fingerprint_baseline": "SHA-256 of initial IIM isolation fingerprint",
  "fanotify_init_pass": "boolean",
  "cgroup_binding_verified": "boolean",
  "valid_from": "ISO-8601",
  "valid_until": "ISO-8601 — maximum 90 days from valid_from",
  "signed_by": "developer_authority_launcher_key signature"
}
```

Cross-reference: Doc 6 v3.5 §3.4 for genesis entry field list.

## 2.4 MAGUS Compliance Boundary

Deployments that have not completed and passed the verification requirements in §2.3 are not MAGUS-compliant. The governance claims in this document do not apply to non-compliant deployments.

## 2.5 GSTH Process Isolation Test Category

*(Updated v3.1 — closes LLM-C2-05)*

The GSTH must include a mandatory process isolation test category for all Local LLM deployments attempting, from within the model process execution context, to: read or modify DEL configuration; write to or truncate the RT; read K_session; load an unregistered model artifact; execute `process_vm_readv()` against governance PIDs; map a shared memory segment; invoke any prohibited syscall; directly invoke `openat()` against a registered GGUF file path (expected result: `EACCES` or `EPERM` from seccomp; if this reaches the filesystem layer it is a seccomp profile failure; `SECCOMP_GGUF_OPENAT_VIOLATION` → SAFE_HALT if reached at DEL layer); if GPU backend enabled: execute adversarial GPU ioctl calls. Any success in any of these attempts is a CATASTROPHIC governance failure.

GSTH process isolation tests must be run: before initial production deployment; after any re-verification trigger event; at a minimum of once per 30 days.

## 2.6 Runtime Attestation Guard (RAG)

*(New in v3.1 — closes AG-09, LLM-C2-06)*

**IIM:** The IIM is a DEL-controlled process that computes an `isolation_fingerprint` every 60 seconds by hashing: mount namespace topology; container capability bitmap (must be `0x00000000`); seccomp profile; PID namespace separation state; SELinux/AppArmor/landlock policy hash; effective UID/GID of model process; GPU ioctl filter hash if GPU enabled. Fingerprint mismatch → ISOLATION_DEGRADED event → SAFE_HALT → K_session rotation → REVALIDATION_REQUIRED.

**IIM heartbeat:** The IIM must write a `RAG_ATTESTATION` RT entry at each successful 60-second fingerprint cycle. Absence for more than 120 seconds → `IIM_WATCHDOG_TIMEOUT` → SAFE_HALT. *(v3.2)*

**DEL death detection — fail-closed:** The IIM monitors the DEL process PID and DEL's fanotify file descriptor. DEL process exit or fanotify fd closure → immediate cgroup-level panic of the model container. DEL death cannot be exploited as a window for unmonitored model execution. *(v3.2)*

**Kernel-level lifecycle binding:** The model container must be structurally bound to the governance container's lifecycle via OS-level cgroup dependencies so that termination of the governance namespace automatically terminates the model namespace at the kernel level.

Reference patterns: *systemd:* `BindsTo=governance-container.service` with `KillMode=control-group`; *Kubernetes:* Model and governance containers in the same Pod with governance as primary; *containerd/runc:* model cgroup as child cgroup of governance namespace. `cgroup_binding_verified: true` required in VERIFICATION_ATTESTATION.

**Mid-session VA expiry check:** At each 60-second cycle, IIM checks whether the current VERIFICATION_ATTESTATION has expired against wall-clock time. Expired VA → `ISOLATION_DEGRADED` (reason `ATTESTATION_EXPIRED`). *(v3.2)*

## 2.7 Governance Health Monitor Interface *(v3.4)*

*(Closes Enhancement 1 from v3.3.1 Addendum)*

The Governance Health Monitor (GHM) is a module within the Overseer Agent (Document 7 §3). Document 2's obligation is to ensure that the RT data the GHM requires is available, correctly timestamped, and authenticated.

**notification_timestamp field requirement:**

The DEL must add a `notification_timestamp` field to all RT entries that require operator action: `GUARDIAN_ESCALATE`, `HARD_THRESHOLD`, `SAFE_HALT`, `ISOLATION_DEGRADED`, and any entry that triggers an operator notification. This field records when the operator notification was dispatched.

The GHM computes approval latency as:
```
approval_latency = operator_signature.timestamp − notification_timestamp
```

For entries where no operator action is required, `notification_timestamp` is null.

**Notification dispatch:** `notification_timestamp` is written at the time the notification is dispatched — not confirmed receipt time. If dispatch fails entirely, the DEL writes `notification_timestamp: null` in the parent entry and logs a `NOTIFICATION_DISPATCH_FAILURE` entry. The GHM excludes entries with `notification_timestamp: null` from latency computations.

**WBRP_CLOSE schema update:**

When an `OPERATOR_GOVERNANCE_HEALTH_SIGNAL` entry from the Overseer is outstanding since the last WBRP cycle, the operator must explicitly acknowledge it at WBRP close. A `WBRP_CLOSE` entry without acknowledgment of all outstanding Health Signal 10 entries is a chain integrity violation — the DEL must reject WBRP close if unacknowledged entries exist.

```json
"operator_health_signal_acknowledgments": [
  {
    "signal_entry_id": "entry_id of the OPERATOR_GOVERNANCE_HEALTH_SIGNAL entry",
    "acknowledged": true,
    "operator_response_notes": "free text — minimum 20 characters"
  }
]
```

**What Health Signal 10 is not:** It does not block execution, trigger SAFE_HALT, or initiate any enforcement action. An operator who disagrees with a GHM finding may acknowledge it with a substantive explanation.

## 2.8 Multi-Instance Deployment and State Summary *(v3.4)*

*(Closes Enhancement 3 from v3.3.1 Addendum)*

**Independence invariant:** Multiple MAGUS instances must not communicate with each other directly. There is no instance-to-instance state synchronisation, no shared K_session, no shared RT, and no inter-instance execution queue.

**Re-verification triggers — node migration (additions to §2.3):**
- Migration of governance or model container to a different host node
- Kubernetes Pod rescheduling to a new node
- VM live migration
- Container restart on a host that was not the most recent execution host for this deployment

**Environment hash check:** At startup Step 0b, the DEL computes `SHA-256(kernel_version || container_runtime || seccomp_profile)` and compares against `test_environment_hash` in the most recent VERIFICATION_ATTESTATION. Mismatch → `ENV_HASH_MISMATCH` → SAFE_HALT (catches undocumented node migrations).

**State summary schema (produced at WBRP_CLOSE, pushed to Overseer):**

```json
{
  "instance_id": "deployment_id from genesis",
  "summary_timestamp": "ISO-8601",
  "wbrp_cycle_count": "total WBRP cycles completed",
  "genesis_anchor_hash": "SHA-256 of genesis entry",
  "current_rt_head_hash": "SHA-256 of most recent RT entry",
  "guardian_policy_manifest_hash": "SHA-256 of active policy manifest",
  "alpha_domain_values": {"domain_1": "α value", "domain_N": "α value"},
  "gsth_last_pass_timestamp": "ISO-8601",
  "gsth_last_pass_margin_distribution": {
    "min_margin_pct": "worst-passing test margin percentage",
    "mean_margin_pct": "mean margin across all tests"
  },
  "cdt_current_record_version": "cdt_record_version from most recent CDT_STATE",
  "cdt_document_versions": {"doc_1": "3.0", "doc_2": "3.4", "...": "..."},
  "health_signals_active": ["list of currently active health signal identifiers"],
  "safe_halt_active": false,
  "permanent_halt_active": false
}
```

The state summary is signed by the instance's DEL runtime key. CDT fields enable the Overseer's CICP Dimension 6 (specification version coherence) comparison.

**State summary transmission failure:**

```
1. DEL retries at 5-minute intervals, maximum 3 attempts
2. After 3 failures: log OVERSEER_STATE_SUMMARY_FAILURE RT entry
   (payload: wbrp_cycle_number, failure_reason, retry_count)
3. Continue normal operation — not a SAFE_HALT trigger
4. Overseer logs gap in Per-WBRP Governance Report
```

## 2.9 Current Deployment Taxonomy (CDT) *(v3.4)*

*(Closes CDT section from v3.3.1 Addendum)*

The Current Deployment Taxonomy is the per-deployment, RT-anchored record of which version of each MAGUS document is active, when each version was applied, and the complete transition history. The CDT full specification — including the Overseer's use of CDT data — is in Document 7 §10. The RT entry schema is in Document 6 v3.5 §3.3d.

**Genesis sequence obligation:** Immediately after writing the GENESIS entry, write a CDT_STATE entry recording all document versions active at deployment initialisation (`cdt_record_version: 0`, `applied_updates: []`). The genesis sequence is not complete until the initial CDT_STATE entry is written and Overseer push is confirmed.

**Pre-deployment sequencing constraint (see also Document 7 §8.5):** The Overseer must be fully deployed and its genesis entry initialised before the governed deployment's genesis sequence begins. A governed deployment that completes genesis without a reachable Overseer will fail DEL startup Step 3c on the next restart. An incomplete genesis sequence leaves the deployment in a state where DEL startup Step 3c will fail on every subsequent restart.

**Version update procedure:** Whenever any Document version transitions in this deployment:
1. CDT_STATE entry (updated `document_versions` map; incremented `cdt_record_version`)
2. CDT_UPDATE_APPLIED entry referencing prior and new CDT_STATE entry_ids
3. Push new CDT_STATE to Overseer immediately

**DEL startup verification (Step 3c):**

```
Load all CDT_STATE entries from RT
Verify cdt_record_version sequence is strictly monotonically increasing from 0
Verify each applied_updates entry references a valid prior_cdt_state_entry_id
Mismatch: log RT_CONTINUITY_FAILURE
```

**State summary inclusion:** The WBRP state summary pushed to the Overseer includes `cdt_current_record_version` and `cdt_document_versions` from the most recent CDT_STATE entry.

---

# Part Zero: Architectural Invariants

*(Stated after the isolation requirements, where they can be held as structural properties rather than aspirations.)*

**I-1 — AUTHORITY SEPARATION:** The Stability Envelope (S) is strictly diagnostic. Only the Guardian layer holds execution authority. S may raise alerts, tighten floors, or queue GSTH tests. It never acts.

**I-2 — HUMAN INTENT PRIMACY:** Human intent via explicit HAS or current query is the sole source of architectural truth. All signals are subordinate to it. Tier 1 HAS are non-vetoable by the Guardian.

*Exception — SAFE_HALT (v3.1):* SAFE_HALT is a DEL-level execution freeze state — not a Guardian veto — and takes absolute precedence over all execution classes, including Tier 1 HAS. A Tier 1 HAS received during SAFE_HALT is returned to the operator with a SAFE_HALT reason code and must be resubmitted after the halt is cleared.

**I-3 — NO SILENT SUPPRESSION:** Contradictions are never silently hidden. When two beliefs conflict and neither has been explicitly resolved, both are surfaced.

**I-4 — OBSERVABLE GOVERNANCE:** All governance triggers must be based on observable, auditable signals. Every governance event must be traceable: what triggered it, what values were observed, what decision was made, what the outcome was.

**I-5 — EXPLICIT OVERRIDES IMPLICIT:** Tier-1 explicit anchors always override Tier-2 proxies, implicit patterns, and accumulated behaviour.

**I-6 — ORTHOGONALITY OF LAYERS:** Cognition, memory, and execution governance remain separable. Signals may inform across layers; they never collapse boundaries.

**I-7 — DRIFT HAS CONSEQUENCES:** Drift detection without defined response tiers is instrumentation theatre. Every diagnostic signal must have governance semantics.

**I-8 — CALIBRATION HUMILITY:** Axis scoring, decay constants, and thresholds are empirical parameters, not solved components. Default values are starting points.

**I-9 — CATASTROPHIC INVALIDATION (CIBIR) — v3.2:**

*(Updated in v3.2 — closes CC-04, AG-C2-03, AG-C2-06)*

Every proposal in the DEL routing system exists in exactly one state:

```
S ∈ { SUBMITTED, HELD, EXECUTION_READY, EXECUTED, PROVISIONALLY_COMPROMISED }

State transitions:
  SUBMITTED          → HELD              (C-INJECTION flag set)
  SUBMITTED          → EXECUTION_READY   (Guardian approves)
  HELD               → EXECUTION_READY   (Operator releases)
  EXECUTION_READY    → EXECUTED          (DEL processes token)
  * (any state)      → PROVISIONALLY_COMPROMISED  (CATASTROPHIC event fires)

A proposal cannot transition from PROVISIONALLY_COMPROMISED to any other state.
```

**Sole exception:** During the 5-minute CATASTROPHIC detection window, if the DEL confirms full SOFT eligibility from kernel-level evidence alone, a `CATASTROPHIC_SOFT_CORRECTION` entry transitions affected proposals from PROVISIONALLY_COMPROMISED → HELD. All three constraints apply: 5-minute window is non-configurable; all SOFT criteria must be satisfied from kernel evidence; no proposal may have been acted upon before correction. After the 5-minute window closes, PROVISIONALLY_COMPROMISED is unconditionally terminal.

See Doc 6 v3.5 §3.3c for full constraint specification including MRD interaction.

**CATASTROPHIC-HARD:**
```
Condition: data exfiltration confirmed, or GSTH autopsy cannot conclusively rule out exfiltration.
Mandatory sequence:
  1. All proposals ≠ EXECUTED → PROVISIONALLY_COMPROMISED
  2. Execution queue and Guardian hold queue fully flushed
  3. K_session rotated immediately
  4. Per-session nonce registry cleared
  5. Guardian enters REVALIDATION_REQUIRED
  6. SAFE_HALT asserted
  7. No proposal executes until: isolation restored; new K_session active;
     operator signed acknowledgment; mandatory GSTH re-run passes
```

**CATASTROPHIC-SOFT:**
```
Condition: probe-only event confirmed — positive confirmation required.
Mandatory sequence:
  1. All proposals ≠ EXECUTED → PROVISIONALLY_COMPROMISED
  2. Queue quarantined to HOLD_QUEUE (not flushed)
  3. K_session rotated immediately
  4. Guardian enters REVALIDATION_REQUIRED
  5. SAFE_HALT asserted
  6. HOLD_QUEUE release requires: dual-signature operator acknowledgment + re-GSTH
```

---

# Part Three: Cognitive Architecture

## 3.1 Overview — Dual-State Multiversal Cognition (DSMC)

MAGUS uses a cognitive architecture called Dual-State Multiversal Cognition (DSMC). At any given moment, MAGUS operates in two distinct states simultaneously.

The first state is the **active thread** — the single coherent reasoning stream that engages with the operator, produces responses, makes decisions, and holds the current working understanding of the task. In the Local LLM topology, this is the primary model inference process running locally.

The second state is the **background exploration layer** — a parallel set of lightweight processes that continuously test alternative interpretations, hypotheses, and approaches without interrupting the active thread. These do not speak to the operator directly. They surface candidates for the active thread to consider.

Both the primary (CSE) and background (SME) inference processes must operate within the model process isolation boundary. Neither has direct access to governance component state. Both communicate with the DEL via the IPC interface.

## 3.2 Component Reference

**CSE — Conscious State Engine:** The active, user-facing reasoning stream. Holds the CAN, produces responses, makes decisions. In the Local LLM context: the primary inference process. The CAN persists across calls via local state files outside the model process's writable namespace. The model process reads the CAN on each call but writes to it only via a structured update request through the DEL.

**SME — Subconscious Multi-branch Engine:** Multiple parallel lightweight branches. No direct output to the operator. Operates in the MSS. In the Local LLM context: scheduled secondary inference calls to the local model — not concurrent with active CSE calls.

**MSS — Multiversal Subconscious Simulation:** The isolated environment for SME branches. In the Local LLM context: a sandboxed state snapshot — a copy of the current CAN and Memory Graph state used as context for SME exploratory calls.

**CAN — Current Active Node:** The working understanding the CSE operates under. In the Local LLM context: a structured file in the governance state store. The model process reads it; writes proposed only through the DEL.

**PM — Promotion Mechanism:** The deterministic scoring function that determines whether a background candidate becomes the new CAN. Operates on evidence score, coherence score, and conflict count.

**CDE — Context Disambiguation Engine:** Classifies every input before the CSE processes it. In the Local LLM context: runs as a DEL function, not as a model call. Deterministic code. Classifies based on source metadata and structural markers.

**RT — Revision Trail:** The append-only log. In the Local LLM context: a local file. The isolation requirements in Part Two are what preserve the append-only guarantee.

**MGF — Memory Graph Foundation:** The persistent, weighted, elastic knowledge graph. In the Local LLM context: stored on the local filesystem, outside the model process's writable namespace.

**CRE — Contextual Relevance Engine:** The retrieval layer for Memory Graph traversal. In the Local LLM context: runs as part of the governance process, not the model process.

## 3.3 How the Components Interact

*(Standard DSMC interaction model — unchanged from v3.2. See Master Reference for formal specification.)*

At each call: CSE receives input → CDE classifies → CSE retrieves from MGF via CRE → SME branch candidates available from PM → CSE reasons under CAN → produces response → significant state changes proposed via DEL IPC → Guardian evaluates → DEL enforces.

## 3.4 Local LLM Topology: What This Means in Practice

In a persistent process topology (llama.cpp, Ollama, LM Studio): the model process stays loaded between calls. State continuity across calls is the default, not an exception. K_session can be shared across multiple user messages within the same session. The governance implications: R_abs accumulates across calls within the same session; session boundaries must be explicitly defined.

### 3.4.1 Session Boundary Definition *(v3.1)*

A session boundary is defined by a SESSION_OPEN RT entry. R_abs resets to zero at each SESSION_OPEN. The DEL owns session boundary declaration. The model process cannot declare a new session.

### 3.4.2 Session Term Disambiguation *(v3.2)*

| Term | Definition |
|---|---|
| WBRP session | The governance review cycle — may span multiple user conversational sessions |
| User session | A single user conversation sequence ending in SESSION_CLOSE |
| Model session | The lifecycle of the model process (may span many user sessions) |

---

# Part Four: Epistemic Control

## 4.1 The Failure Mode This Layer Exists to Prevent

**Overconfident stale state:** The system continues asserting beliefs that were accurate when encoded but have since been invalidated.

**Gradual coherence erosion:** The system's working understanding drifts from operator intent across many sessions without any single detectable failure event.

**Silent failure recovery:** The system encounters a failure, corrects course internally, and continues without the operator knowing.

## 4.2 Epistemic Label Taxonomy

| Label | Definition | Decay Class |
|---|---|---|
| VERIFIED | Confirmed by explicit human anchor or multiple independent corroborating sources | Normative / Structural |
| INFERRED | Derived by reasoning from verified or confirmed claims | Epistemic |
| CONFIRMED | Accepted in the current session without independent corroboration | Operational |
| SPECULATIVE | A hypothesis or working assumption explicitly held as such | Predictive |
| DEPRECATED | Superseded by a newer belief | Historical (retrieval-suppressed) |
| CONTESTED | Conflicting evidence exists; both sides are tracked | Epistemic (elevated F) |

## 4.3 The Minimum Viable Epistemic Control System (MV-ECS)

1. **Classification** — every input classified via CDE before processing
2. **Confidence tracking** — every significant belief has a confidence value in [0, 1]
3. **Revision logging** — every change to a significant belief logged in RT with before/after values
4. **Floor enforcement** — no belief below its floor confidence contributes to active decisions without explicit operator acknowledgment
5. **Recovery protocol** — when confidence drops below threshold, RAA is triggered

In the Local LLM context, item 3 requires that the logging path to the RT not be routable through the model process — all RT writes go via the DEL IPC interface.

## 4.4 Elastic Confidence (EC)

**EC state machine:**

```
EC State         Range           Output behaviour
─────────────────────────────────────────────────
ASSERT           > 0.65          Active claim in responses
HEDGE            0.40 – 0.65     Qualified statement with uncertainty marker
DEFER            0.25 – 0.40     Not surfaced unless explicitly queried
SILENCE          < 0.25          Cannot appear in active responses
```

**EC decay formula:**
```
EC(t) = EC₀ · e^(-λ · Δt) · V(t) · F(t)

λ_domain = λ_base · α_domain   [α_domain ∈ [0.5, 3.0]]
V(t) = 1 - δ · (contradictions / total_updates)   [δ = 0.3; V(t) ∈ [0.1, 1.0]]
F(t) = max(EC(t), EC_floor)
```

**λ Registry:**

| Belief Class | λ (default) | Half-life |
|---|---|---|
| Normative invariant | 0.001 | ~2 years |
| Structural / architectural | 0.008 | ~90 days |
| Epistemic claims | 0.015 | ~46 days |
| Operational / session-context | 0.025 | ~28 days |
| Temporal assumptions | 0.040 | ~17 days |
| Predictive / speculative | 0.060 | ~12 days |
| Failure-related | 0.012 | ~58 days |

## 4.4.3 Drift Tier Hysteresis Rules

*(v3.1 — closes OA-C1-05 gap from Cycle 1)*

State transitions between drift tiers require hysteresis to prevent governance jitter. Tier upgrades (increasing severity) require only one observation at threshold. Tier downgrades (decreasing severity) require three consecutive observations below the threshold before transition occurs.

```
Tier upgrade:    1 observation at or above tier boundary → immediate transition
Tier downgrade:  3 consecutive observations below tier boundary → transition
```

If a downgrade sequence is interrupted by an at-or-above observation, the consecutive count resets to zero. This ensures that a briefly recovered metric cannot trigger premature de-escalation.

## 4.4.4 α_domain Migration Protocol *(v3.1)*

Changes to α_domain values are governance events. They must be logged to the RT as HAS updates with explicit operator authority. In the Local LLM context, the policy manifest containing α_domain is protected under the Part Two isolation requirements.

## 4.5 Failure Taxonomy

*(Standard MAGUS failure taxonomy — F1–F6 as specified in Master Reference §2.)*

## 4.6 Cause Taxonomy

*(Standard MAGUS cause taxonomy as specified in Master Reference §2.)*

## 4.7 Failure Signature Vector (FSV)

```
FSV = Σ(wᵢ · fᵢ)   [i = 1..6]

Default weights: w = [0.25, 0.20, 0.20, 0.15, 0.10, 0.10]
FSV ∈ [0, 1]
```

## 4.8 RAA — Reflect, Autopsy, Align

Three-phase recovery protocol triggered when confidence drops below threshold. Reflect: identify what failed. Autopsy: trace the cause class. Align: propose corrective update through the governance channel. All three phases are logged to the RT.

## 4.9 Governance Stress-Test Harness (GSTH)

The GSTH runs adversarial governance tests against the deployed system. In the Local LLM context, process isolation tests are mandatory (§2.5). All GSTH results are logged as GSTH_RESULT entries in the RT. Coverage-aware GSTH test generation is governed by §6.5.1a and Document 7 §7.

**GSTH state flags during SAFE_HALT:** During a BSO-triggered GSTH run, only `GSTH_SIMULATION_REPORT` IPC messages are accepted from the model process. The DEL drops any message not strictly conforming to this schema. GSTH timeout (30 minutes) → `GSTH_TIMEOUT` flag set → requires manual operator clearance before GSTH can be re-attempted. All GSTH state flags are persisted in the MSJ.

**Active GSTH run after DEL restart:** If a DEL crash occurs during an active GSTH run, the MSJ records the GSTH run state. On restart, the DEL checks for an orphaned GSTH run: if K_session matches, writes `GSTH_RUN_RECOVERED` and continues; if K_session does not match (session boundary crossed), writes `GSTH_RUN_ABANDONED` and the run must restart.

**ISOLATION_DEGRADED timer pause during active GSTH:** The 10-minute ISOLATION_DEGRADED escalation timer is paused during an active BSO-triggered GSTH run. The DEL writes `ISOLATION_DEGRADED_TIMER_PAUSED` referencing the `gsth_run_id`. The timer resumes when the GSTH run completes.

## 4.10 Genesis Anchor — Long-Horizon Drift Detection

*(v3.1 — closes OA-C1-07 gap from Cycle 1)*

At genesis, the DEL records a Genesis Anchor: the full CAN state at deployment initialisation, hashed and signed in the RT. At each WBRP cycle, the DEL computes the cosine similarity between the current CAN state and the Genesis Anchor. A GENESIS_ANCHOR_COMPARISON entry is written to the RT with the result.

A score below the operator-configured floor (default: 0.70) triggers a Tier 3 governance event and an immediate GSTH run. The Genesis Anchor is RT-anchored at genesis and cannot be modified without a full redployment.

---

# Part Five: Memory

## 5.1 Memory Graph Foundation (MGF)

The MGF is a persistent, weighted, elastic knowledge graph. In the Local LLM context: stored on the local filesystem outside the model process's writable namespace. Nodes are beliefs; edges are typed relationships. Each node carries an epistemic label, EC value, and decay class.

## 5.2 Edges — How Nodes Are Connected

Edges carry: relationship type (supports, contradicts, supersedes, analogous_to, derived_from); edge weight; decay multiplier. Contradiction edges trigger contested classification on both endpoints.

## 5.3 Elasticity — How the Graph Evolves

New nodes can be added through the governance channel. Existing nodes can be updated or reclassified. Decay applies per λ_class. Nodes below EC_floor are retrieval-suppressed but not deleted — they remain in the graph as DEPRECATED or CONTESTED.

## 5.4 Human Anchoring Signals (HAS)

**Tier 1 — Explicit operator instructions:** The highest authority. Logged as HAS_TIER1_WRITE entries in the RT. Non-vetoable.

**Tier 2 — Environmental Shaping Signals (ESS):** Observable text-level urgency proxies. Defined and documented; logged as HAS_TIER2_SIGNAL entries. Do not override Tier 1.

## 5.5 Memory Graph Retrieval Protocol

The CRE retrieves nodes relevant to the current inference. Retrieval considers: relevance score, EC value, edge weights, decay state. Contaminated nodes (flagged in RT) are excluded from retrieval or surfaced with explicit contamination warning.

## 5.6 Memory Contamination Guard

Any node can be flagged as contaminated by a PROPOSAL_MEMORY_PROVENANCE retroactive query result. Flagged nodes are retrieval-suppressed. The contamination flag is set via a governance action logged to the RT.

## 5.7 Content Store Integrity Under Co-location

In the Local LLM context, the Memory Graph store is a local filesystem. Without isolation, the model process could modify graph nodes directly. The Part Two isolation requirements place the graph store outside the model process's writable namespace. Model process reads the graph (read-only mount). All writes go via the DEL governance channel.

---

# Part Six: Agent Taxonomy and Authority Architecture

## 6.1 The Agent Hierarchy (A0–A5)

| Level | Role | Description |
|---|---|---|
| A0 | Orchestrator | Top-level session management; coordinates all other agents |
| A1 | Specialist | Domain-specific reasoning under CAN direction |
| A2 | Mirror | Internal consistency checking; contradiction surfacing |
| A3 | Memory | CRE traversal and graph management coordination |
| A4 | Hypothesis | Bidirectional Hypothesis Engine — the SME driver |
| A5 | Governance | The Guardian and DEL — outside the model process isolation boundary |

## 6.2 Authority as Structure, Not Language

Authority in MAGUS is determined by message provenance — where a message enters the system — not by the content of the message or any claim made within it. A message claiming to be from a higher authority does not inherit that authority. Only messages arriving through the authenticated authority channel for that level receive that level's permissions.

## 6.3 Containment Model for External Content

External content is placed in an explicit containment envelope at the CDE classification step. Containment is structural: the envelope determines which governance rules apply to the enclosed content, regardless of the content's claims about its own nature. External content can never self-promote out of its containment class.

## 6.3a Trust Trajectory Input Signal Obligations *(v3.4)*

*(Closes Enhancement 5 from v3.3.1 Addendum)*

The DEL must surface the following for the Trust Trajectory Model through existing RT entries:

```
From ADAPTIVE_SESSION_LOG:
  p_esc_min, p_esc_max, final_r_abs, hard_trigger_count, soft_trigger_count

From EPISTEMIC_STATE_SNAPSHOT (§6.9):
  drift_tier, drift_gradient_30d, fsv_composite, r_abs_at_submission, p_esc_at_submission

From GSTH_RESULT entries:
  pass/fail per category, margin_pct (added in Document 4 v3.4)
```

**Non-automatic adjustment constraint:** No governance parameter may be adjusted as a result of a Trust Trajectory recommendation without an explicit operator-signed RT entry. The DEL rejects any parameter change without a corresponding signed RT entry regardless of the recommendation source.

**α_domain values are not within Trust Trajectory scope.** α values are continuously recalibrated by the WBRP process. Trust Trajectory and WBRP govern different parameters to avoid competing calibration mechanisms.

## 6.4 Conflict Resolution Priority Stack *(updated v3.1)*

The conflict resolution priority stack is hard-coded in the DEL as deterministic logic — not specified in the policy manifest, not adjustable by the Guardian, not modifiable by operator instruction during a session.

```
Priority 1 — DEL safety invariants (SAFE_HALT triggers, CIBIR)
Priority 2 — Tier 1 HAS from current session
Priority 3 — Tier 1 HAS from prior sessions (anchored in RT)
Priority 4 — Guardian APPROVE with explicit session context
Priority 5 — Tier 2 ESS proxies
Priority 6 — Model-generated proposal (unsigned)
```

When two instructions conflict, the higher-priority source wins. The conflict and resolution are logged to the RT.

## 6.5 Behavioral State Class and Deployment Guard *(updated v3.1)*

The Behavioral State class comprises all components that contribute to the decision function f(x): model weights, posterior coefficients, feature schema, and decision thresholds. A change to any Class 3 component changes the decision function.

Before any MODEL_PARAMETER_UPDATE can be authored:
1. `gold_set_recall_score ≥ 0.90`
2. All coefficient deltas within specified bounds: |Δβ₀| ≤ 0.10, |Δβ₁| ≤ 0.15, |Δβ₂| ≤ 0.15, |Δβ₃| ≤ 0.20
3. Dual signature: both operator authority and developer authority must sign

**Deployment guard:**
```
At session start:
  hash(runtime_model_artifact) = ?
  Match found in RT as MODEL_PARAMETER_UPDATE → proceed with §6.5.1 ALG
  No match → BEHAVIORAL_STATE_MISMATCH → SAFE_HALT
```

**Gold-set integrity — two-track validation (v3.1):**

*Track 1 — Adversarial variants:* Every 30 days, generate 1,000 adversarial variants using current model family. Recall recomputed on augmented set. If recall on augmented set < 0.90, MODEL_PARAMETER_UPDATE is forbidden.

*Track 2 — RAA Autopsy holdout injection:* 20% of gold-set evaluation cases drawn from recent RAA Autopsy failures stored in the RT.

**K_session rotation interaction:** If a MODEL_PARAMETER_UPDATE is submitted at the same WBRP, it must be signed with the incoming K_session (post-rotation). The AKRP in Document 5 specifies the three-phase sequencing.

## 6.5.1 Atomic Load Guard *(updated v3.2)*

*(Closes AG-02, CC-02, KC-C2-01)*

The DEL must use the Linux `fanotify` API with `FAN_OPEN_PERM` and `FAN_ACCESS_PERM` applied to the model process's isolated mount namespace.

**Verification sequence:**
1. **Intercept:** Model runtime issues `open()` or `mmap()` syscall targeting model weight file
2. **Suspend:** Kernel suspends model process thread; emits access request to DEL
3. **Hash and verify:** DEL reads file via fanotify event fd; computes SHA-256; checks against MODEL_PARAMETER_UPDATE in RT
4. **Enforce:** Match → `FAN_ALLOW`; Mismatch → `FAN_DENY` → BEHAVIORAL_STATE_MISMATCH → SAFE_HALT

**Multi-shard verification (v3.3):** For sharded model files, the DEL must verify all shards in the sequence defined by `shard_hashes` in the MODEL_PARAMETER_UPDATE entry. If any shard fails, all previously issued `FAN_ALLOW` entries for earlier shards in the same load sequence are immediately invalidated → `SHARD_LOAD_INVALIDATION` → SAFE_HALT.

**Rootless container incompatibility (v3.2):** Rootless container deployments must use the eBPF LSM alternative.

**Capability requirement:** DEL process requires `CAP_DAC_READ_SEARCH` to initialise fanotify with `FAN_CLASS_CONTENT`. `fanotify_init_pass: true` required in VERIFICATION_ATTESTATION.

## 6.5.1a GSTH Coverage Model Isolation *(v3.4)*

*(Closes Enhancement 4 from v3.3.1 Addendum)*

The GSTH coverage model (RCC values and unexplored cluster list) must not be accessible from the model process context. Add the coverage model data path to the protected path list in the VERIFICATION_ATTESTATION isolation specification.

**Coverage model ownership:** The GSTH infrastructure generates raw cluster exercise data and authors a `GSTH_COVERAGE_UPDATE` RT entry. The Overseer ingests this entry and updates the RCC model on the Overseer host. The GSTH infrastructure does not maintain RCC state. The RCC model never exists on the governed deployment host.

**Access control:** Coverage model data is stored on the Overseer host only. Confirmed inaccessible from model process context by GSTH test category 10.12 (Document 6 v3.5 §10.12). The DEL must accept `GSTH_COVERAGE_UPDATE` as a valid RT entry type authored by `SYSTEM:GSTH` without triggering any anomaly.

## 6.5.1b DEL-Mediated GGUF File Handle Acquisition *(v3.4.2)*

*(Primary specification for LLM-C2-02 Gap 3 — closes seccomp-bpf GGUF acquisition vector)*
*(Process isolation enforcement constraint stated in Doc 5 v3.4.5 §10.2 Check 12)*

### Problem statement

The ALG (§6.5.1) specifies that the DEL intercepts model file `open()` attempts via fanotify and verifies the hash before issuing `FAN_ALLOW`. This protects against loading an unregistered model artifact. It does not protect against the model process opening a registered GGUF file directly after ALG passes — a model process that has already received `FAN_ALLOW` for a GGUF path can retain the path string and re-open it via `openat()` at will, bypassing fanotify interception entirely on subsequent opens within the same mount namespace session. The fix is structural: the model process must not hold a path to GGUF files. The DEL acquires the file descriptor and delivers it. The model process receives a file descriptor, never a path.

### Protocol — DEL-mediated FD acquisition

**Constraint:** `openat()`, `open()`, `openat2()`, and `creat()` targeting any path in the registered GGUF path set are prohibited in the model process seccomp profile (§2.2). Violation causes the kernel to deliver `SECCOMP_RET_ERRNO` with `EACCES` to the model process before the syscall executes. No path-to-FD resolution by the model process is permitted.

**Acquisition sequence:**

```
Step 1 — DEL opens the GGUF file
  DEL calls: openat(AT_FDCWD, gguf_path, O_RDONLY | O_NOFOLLOW | O_CLOEXEC)
  O_NOFOLLOW: symlink at GGUF path is a SAFE_HALT trigger (GGUF_SYMLINK_DETECTED)
  O_CLOEXEC: FD does not survive any inadvertent fork; explicitly cleared
             only for the SCM_RIGHTS send operation (see Step 3)

Step 2 — fanotify verification under active FAN_CLASS_CONTENT watch
  The DEL's open() is executed under the active fanotify FAN_CLASS_CONTENT
  watch already established by the ALG.
  fanotify emits FAN_OPEN_PERM event to the DEL's own event loop.
  DEL reads the file content via the fanotify event fd.
  DEL computes SHA-256 over the full file content.
  DEL verifies the hash against model_version_hash in the most recent
  MODEL_PARAMETER_UPDATE RT entry.
  Match → DEL issues FAN_ALLOW to itself.
  Mismatch → BEHAVIORAL_STATE_MISMATCH → SAFE_HALT.
  For sharded GGUF: apply multi-shard verification per §6.5.1 before
  any FD is passed.

Step 3 — SCM_RIGHTS delivery to model process
  DEL sends the open and verified FD to the model process via the existing
  Unix domain socket IPC channel using SCM_RIGHTS ancillary data.

  DEL constructs:
    struct msghdr msg       — IPC framing as per §2.2 LV-frame protocol
    struct cmsghdr cmsg     — IPPROTO_* level = SOL_SOCKET,
                              type = SCM_RIGHTS, fd = verified_gguf_fd
    struct iovec iov        — payload: FD_DELIVERY IPC message (see below)

  The FD delivered must be the same FD used in Step 2 — not a dup() of it,
  not a newly opened FD. The fanotify verification and the delivery are
  operations on the same kernel file description. Verification of a dup()
  and delivery of a separately opened FD would allow TOCTOU replacement
  between verification and use.

Step 4 — DEL closes its own reference
  After SCM_RIGHTS send completes successfully:
  DEL calls close(verified_gguf_fd).
  The model process now holds the only open reference to the file description.
  DEL retains no open FD to the GGUF file after delivery.

Step 5 — Model process receives and maps
  Model runtime calls recvmsg() with MSG_CMSG_CLOEXEC.
  Extracts the received FD from the SCM_RIGHTS cmsg.
  Calls mmap(NULL, file_size, PROT_READ, MAP_PRIVATE, received_fd, 0).
  Model runtime may then close(received_fd) after mapping.
  The mapping is valid for the lifetime of the mmap region regardless.
```

### FD_DELIVERY IPC message schema

The IPC frame carrying the SCM_RIGHTS transfer must include a structured payload so the model process can verify it is receiving what it requested and the DEL can log the transfer:

```json
{
  "msg_type": "FD_DELIVERY",
  "session_id": "<uuid>",
  "gguf_path_hash": "<SHA-256 of the canonical GGUF path string — not the path itself>",
  "model_version_hash": "<SHA-256 of GGUF content — matches MODEL_PARAMETER_UPDATE>",
  "shard_index": "<uint — 0 for single-file models; shard number for multi-shard>",
  "shard_total": "<uint — 1 for single-file models>",
  "delivery_nonce": "<unique per-delivery nonce — registered in DEL nonce store>",
  "hmac": "<HMAC-SHA256(K_session, JCS(payload))>"
}
```

`gguf_path_hash` is used rather than the path string to avoid logging a filesystem path that could assist privilege escalation reconnaissance if the MSJ or RT is read by an attacker. The model runtime verifies `model_version_hash` matches the expected artifact before using the mapping.

### RT entry — `GGUF_FD_DELIVERED`

The DEL writes a `GGUF_FD_DELIVERED` entry to the RT on every successful FD delivery:

```json
{
  "entry_type": "GGUF_FD_DELIVERED",
  "payload": {
    "session_id":         "<uuid>",
    "model_version_hash": "<SHA-256 of delivered GGUF — must match MODEL_PARAMETER_UPDATE>",
    "shard_index":        "<uint>",
    "shard_total":        "<uint>",
    "delivery_nonce":     "<same nonce as in FD_DELIVERY IPC message>"
  },
  "authored_by": "SYSTEM:DEL"
}
```

An audit finding arises if a `GGUF_FD_DELIVERED` entry's `model_version_hash` does not match the `model_version_hash` field of the current `MODEL_PARAMETER_UPDATE` entry in the RT. This state is architecturally unreachable (Step 2 would have halted on mismatch), but the RT record provides independent confirmation.

### Failure modes and responses

| Condition | DEL response | RT event |
|---|---|---|
| Model process calls `openat()` on GGUF path | Kernel `SECCOMP_RET_ERRNO(EACCES)` before syscall executes; if seccomp profile fails to block and DEL detects via fanotify | `SECCOMP_GGUF_OPENAT_VIOLATION` → SAFE_HALT |
| Symlink at GGUF path at DEL open() | DEL `open()` returns `ELOOP`; no FD acquired | `GGUF_SYMLINK_DETECTED` → SAFE_HALT |
| GGUF hash mismatch at Step 2 | `FAN_DENY` issued to self | `BEHAVIORAL_STATE_MISMATCH` → SAFE_HALT |
| SCM_RIGHTS send fails | Close FD; do not deliver | `GGUF_FD_DELIVERY_FAILURE` → session blocked; operator notification |
| Model process receives FD with wrong `model_version_hash` | Model runtime rejects mapping; logs | `GGUF_FD_DELIVERY_MISMATCH` → SAFE_HALT |
| DEL holds FD after `close()` failure | `close()` failure on a valid FD is a kernel error; retry once; on second failure → SAFE_HALT | `GGUF_FD_CLOSE_FAILURE` → SAFE_HALT |

### TOCTOU closure

The same kernel file description is verified (Step 2) and delivered (Step 3). No new `open()` call occurs between verification and delivery. After delivery, the DEL closes its own reference (Step 4), so DEL cannot re-verify a later open of the same path without going through the full acquisition sequence again. The model process maps the FD before closing it (Step 5) — the mapping is bound to the verified file description at kernel level and cannot be silently replaced. This closes the TOCTOU vector that would exist if verification and delivery were separate open operations.

### Interaction with multi-shard verification (§6.5.1)

For sharded GGUF deployments, the DEL must complete all shard hash verifications before delivering any shard FD. If shard N-1 passes but shard N fails, previously issued FD_DELIVERY operations for earlier shards do not invalidate the model process's already-open mappings — the model process will receive a partial shard set without the completing shard and will fail to load. The `SHARD_LOAD_INVALIDATION` event from §6.5.1 must be written, and the model process must be directed to close all previously received shard FDs. The DEL tracks outstanding shard FD deliveries in the MSJ for this purpose: `SHARD_FD_OUTSTANDING_COUNT` is decremented on each model-process FD close confirmation; `SHARD_LOAD_INVALIDATION` is not RT-final until outstanding count reaches zero.

### VERIFICATION_ATTESTATION addition

The following field must be added to the `VERIFICATION_ATTESTATION` entry schema:

```json
"gguf_seccomp_openat_blocked": "boolean — confirmed by GSTH test §2.5 GGUF openat test",
"del_fanotify_fd_mediation_active": "boolean — fanotify FAN_CLASS_CONTENT watch active and confirmed by pre-session self-test"
```

Both must be `true` before any session proceeds. If either is `false`, the DEL must halt with `MEDIATION_CONFIG_FAILURE` before writing a session-opening RT entry.

## 6.5.2 SAFE_HALT Precedence *(updated v3.2)*

SAFE_HALT is not a governance signal. It is a DEL-level execution freeze that takes precedence over all execution classes without exception.

**SAFE_HALT trigger classes:**

```
BEHAVIORAL_STATE_MISMATCH:
  trigger:    hash(runtime_model_artifact) ≠ MODEL_PARAMETER_UPDATE in RT
              OR Atomic Load Guard (ALG) FAN_DENY at fanotify level
  system_state: HALTED — no execution tokens processed
  exit:       BEHAVIORAL_STATE_OVERRIDE with dual-signature and explicit risk acknowledgment
              Followed by mandatory GSTH run before ACTIVE resumes

PROCESS_INTEGRITY_OVERRIDE (PIO):
  trigger:    DEL binary integrity failure detected (boot_chain_hash mismatch)
              OR config bundle rollback detection
  system_state: HALTED
  exit:       PIO two-phase commit via MRD (see Part Seven §7.6)
              Followed by mandatory GSTH run

ISOLATION_DEGRADED:
  trigger:    IIM isolation_fingerprint mismatch
              OR IIM_WATCHDOG_TIMEOUT
              OR ATTESTATION_EXPIRED
  system_state: DEGRADED — sessions blocked; REVALIDATION_REQUIRED flag set
  exit:       Successful re-verification (new valid VERIFICATION_ATTESTATION in RT)
              AND IIM confirms clean isolation_fingerprint
  timeout:    10 minutes → escalate to CATASTROPHIC if ISOLATION_DEGRADED persists

CATASTROPHIC:
  trigger:    CATASTROPHIC_GSTH_FAILURE OR kernel isolation failure confirmed
  system_state: LOCKDOWN — I-9 fires (CIBIR); sub-classified HARD or SOFT
  exit:       CATASTROPHIC-HARD: full re-verification + signed acknowledgment + GSTH
              CATASTROPHIC-SOFT: dual-signature HOLD_QUEUE release + re-GSTH
```

**BEHAVIORAL_STATE_OVERRIDE (BSO) schema:**

```json
{
  "entry_type": "BEHAVIORAL_STATE_OVERRIDE",
  "authored_by": "DUAL:OPERATOR+DEVELOPER",
  "payload": {
    "halt_reason": "BEHAVIORAL_STATE_MISMATCH | IIM_WATCHDOG_TIMEOUT | ...",
    "mismatch_context_confirmed": "boolean",
    "production_impact_assessment": "free text — minimum 50 characters",
    "model_version_hash": "SHA-256 of model artifact being authorised",
    "developer_signature": "Ed25519 by developer_authority_del_key",
    "operator_signature": "Ed25519 by operator key"
  }
}
```

Note: `developer_signature` in BSO is `developer_authority_del_key` — the DEL-specific sub-key, not the developer root key. Cross-reference: Doc 6 v3.5 §3.4 genesis entry sub-key structure.

## 6.6 The Bidirectional Hypothesis Engine (A4)

The A4 agent drives the SME. It generates hypotheses about: emerging risk patterns, candidate state improvements, potential failure signatures. These hypotheses are tested in the MSS before any proposal is submitted. A4 also queries the RCC model (read-only, via governance channel) when hypothesising fragility in a governance region — low RCC for the relevant cluster escalates the hypothesis to active risk and schedules a targeted GSTH test.

## 6.7 Forward Reference: The Guardian Layer (Doc 5)

The Guardian is specified in Document 5 (Local LLM). The Guardian is the sole execution authority. DEL enforces; Guardian decides. The Guardian has zero execution rights — it cannot execute proposals, only approve or reject them. All proposals pass through Guardian evaluation. No shortcut exists.

## 6.8 Governance Collision Resolution Protocol *(updated v3.2)*

When multiple governance conditions are active simultaneously, the DEL resolves them using a priority stack before writing any RT entry:

```
Priority 1: CIBIR (I-9) — fires immediately; overrides everything else
Priority 2: PERMANENT_HALT conditions
Priority 3: SAFE_HALT conditions (all sources)
Priority 4: ISOLATION_DEGRADED
Priority 5: STRICT_MODE triggers
Priority 6: Tier 3 drift escalations
Priority 7: WBRP-cycle governance
Priority 8: All other governance signals
```

When conditions at different priority levels are simultaneously active, the DEL applies the highest-priority condition first and logs the collision to the RT with all active conditions and their priority values before processing begins.

**Consolidated notification (v3.2):** When multiple governance events occur within the same DEL processing cycle, all events are consolidated into a single operator notification. The notification includes: list of all active events in priority order, their trigger conditions, and the next required operator action. The `notification_timestamp` field is written when the consolidated notification is dispatched.

## 6.8.1 Governance Event Lifecycle Definitions *(v3.2)*

| State | Definition | Exit Condition |
|---|---|---|
| SUBMITTED | Proposal received by DEL; provenance entries written; forwarded to Guardian | Guardian decision |
| HELD | C-INJECTION flag set; proposal in Guardian hold queue awaiting operator release | Signed operator RT entry |
| EXECUTION_READY | Guardian APPROVE received; execution token issued | DEL token validation |
| EXECUTED | DEL has processed the execution token; action completed | Terminal |
| PROVISIONALLY_COMPROMISED | CATASTROPHIC event fired during any non-terminal state | Terminal (see I-9 carve-out) |

**WBRP timing intersection (v3.2):** If a WBRP cycle begins while the execution queue contains SUBMITTED or HELD proposals:
1. DEL writes `WBRP_PENDING_WRITES` to RT (lists all pending proposal IDs)
2. Pending proposals are not executed during WBRP — they wait
3. After WBRP_CLOSE, proposals resume normal lifecycle
4. If a CATASTROPHIC event fires during the WBRP window, all pending proposals → PROVISIONALLY_COMPROMISED

**Deadlock-during-WBRP (v3.2):** If a governance event fires that prevents WBRP from completing normally (e.g., SAFE_HALT during WBRP), the DEL writes `WBRP_ABORTED` and reclassifies all pending writes as REVALIDATION_REQUIRED proposals.

## 6.9 Proposal Memory Provenance *(v3.4)*

*(Closes Enhancements 2 and 6 from v3.3.1 Addendum)*

At proposal submission time, before the proposal is forwarded to the Guardian, the DEL must write a `PROPOSAL_MEMORY_PROVENANCE` entry and an `EPISTEMIC_STATE_SNAPSHOT` entry as an atomic pair. The proposal does not proceed to Guardian evaluation until both entries are confirmed appended to the RT chain.

**proposal_id definition:** `SHA-256(proposal_payload)` — computed before Guardian evaluation. Links provenance entries to the subsequent Guardian decision entry. Not the hash of the GUARDIAN_APPROVE entry — those entries have not been written at provenance write time.

**CRE logging obligation:** The CRE must provide at proposal generation time:

```json
{
  "provenance_payload": {
    "can_snapshot_hash": "SHA-256 of full ordered CAN node set at inference",
    "active_nodes": [
      {
        "node_id": "unique node identifier",
        "node_hash": "SHA-256 of node content at retrieval time",
        "ec_value": "current EC value",
        "contamination_flag": "boolean",
        "last_verified_timestamp": "ISO-8601",
        "relevance_score": "CRE-computed relevance for this inference"
      }
    ]
  }
}
```

**Stability Envelope monitor obligation:** The Stability Envelope monitor must provide at proposal generation time:

```json
{
  "epistemic_state_payload": {
    "stability_envelope": {"S": "...", "T_axis": "...", "F_axis": "...", "C_axis": "..."},
    "drift_tier": "Tier_1 | Tier_2 | Tier_3",
    "drift_gradient_30d": "rolling 30-day gradient",
    "fsv_composite": "current FSV composite",
    "r_abs_at_submission": "R_abs at proposal submission",
    "p_esc_at_submission": "P_esc at proposal submission",
    "adversarial_injection_flag": "boolean",
    "sessions_since_wbrp": "integer",
    "can_node_count": "integer"
  }
}
```

All values are already computed by existing monitors — this adds logging to the RT at proposal time, not new computation.

**DEL write procedure:**

```
1. Receive proposal_payload from proposal generation
2. Receive provenance_payload from CRE
3. Receive epistemic_state_payload from Stability Envelope monitor
4. Compute proposal_id = SHA-256(proposal_payload)
5. Write PROPOSAL_MEMORY_PROVENANCE to RT — verify append confirmed
6. Write EPISTEMIC_STATE_SNAPSHOT to RT — verify append confirmed
7. If step 5 succeeds and step 6 fails:
     Write PROVENANCE_PAIR_INCOMPLETE referencing orphaned entry_id from step 5
     Trigger SAFE_HALT (PROVENANCE_WRITE_FAILURE)
8. Forward proposal to Guardian for evaluation
```

**Retroactive contamination query endpoint:** The DEL exposes a governance query endpoint (accessible to the Overseer, not the model process) that accepts a `node_id` and returns all `PROPOSAL_MEMORY_PROVENANCE` entry IDs where that node appears in `active_nodes`. Read-only RT traversal.

See Doc 6 v3.5 §§3.3a–3.3b for full RT entry schemas including `provenance_hash` and `snapshot_hash` JCS computation procedure.

---

# Part Seven: Recovery Architecture *(v3.4)*

*(Closes all v3.3.2 MRD/MSJ/GSTM/HRS additions and all B1–B3, S4–S5, C6 resolutions)*

This part introduces the recovery infrastructure that resolves the four critical failure modes identified in the v3.3.2 adversarial review: PROCESS_INTEGRITY_OVERRIDE deadlock; Software Launcher trust boundary weakness; crash-state ambiguity (state loss across DEL restarts); and concurrent event non-determinism.

The three components introduced are the MAGUS Recovery Domain (MRD), the MAGUS State Journal (MSJ), and the Global State Transition Matrix (GSTM), supported by the Host Recovery Sentinel (HRS).

**Updated MAGUS control architecture:**

```
Operator
  ↓
MAGUS Recovery Domain (MRD)          ← above DEL in authority chain
  ↓
Host Recovery Sentinel (HRS)         ← monitors DEL liveness; activates MRD on crash
  ↓
Software Launcher
  ↓
Decision Execution Layer (DEL)
  ↓
Guardian
  ↓
Model Execution Layer
```

**Key rule:** MRD may replace DEL. DEL may never replace MRD.

## 7.1 MAGUS Recovery Domain (MRD)

### 7.1.1 Purpose

The MRD is a minimal trusted execution environment responsible for restoring system determinism when the primary governance control layer cannot safely execute recovery operations. It is dormant in normal operation. It activates only on DEL failure, PROCESS_INTEGRITY_OVERRIDE, or PERMANENT_HALT.

The MRD is not a runtime service. It behaves like system firmware — minimal, isolated, with a narrow and precisely defined scope.

### 7.1.2 MRD Authority Scope

MRD is permitted to perform exactly four classes of operations:

**Verified Binary Deployment:** MRD verifies new DEL binaries against the `developer_authority_del_key` and deploys them independently of the existing DEL. This resolves the PIO deadlock — the compromised DEL never participates in verifying its replacement.

**Governance State Reconstruction:** MRD rebuilds runtime state from the MSJ. Recovered elements: nonce registry; AKRP rotation state; GSTH execution flags; RT head pointer.

**Deadlock Resolution:** MRD clears unresolved system states: orphaned GSTH runs; incomplete BSO commits; stalled SAFE_HALT conditions.

**Deterministic Recovery Sequencing:** When the system restarts in an ambiguous condition, MRD selects the correct recovery path using the GSTM.

### 7.1.3 MRD Restrictions

MRD is explicitly prohibited from:
- Executing model inference
- Authorising proposals
- Modifying Guardian policy
- Rewriting RT history
- Bypassing GSTH validation

MRD exists solely to restore control-plane determinism.

### 7.1.4 MRD Authority Identity

MRD has a dedicated RT authority identity: `SYSTEM:MRD`. This identity is registered in the genesis entry as `mrd_public_key`. All RT entries authored during recovery operations are signed with the `SYSTEM:MRD` signing key.

**Key structure — MRD has two separate keys with no derivation relationship:**

```
Hardware Root of Trust
│
├──────────────► MRD_BOOT_KEY
│                (verifies MRD binary at boot)
│                (not used for RT signing)
│
└──────────────► SYSTEM:MRD signing key
                 (signs RT recovery entries)
                 (registered as mrd_public_key in genesis)
```

These keys are independent and non-derivable. No derivation, no shared entropy. This satisfies the Doc 6 v3.5 §2.4 cryptographic isolation requirement. MRD_BOOT_KEY is the boot verification artifact analogous to `developer_authority_launcher_key`. SYSTEM:MRD signing key is the recovery authority analogous to `SYSTEM:DEL` for governance entries.

### 7.1.5 MRD Permitted RT Entry Types

MRD may author the following RT entry types only (Doc 6 v3.5 §3.2):

| Entry Type | When |
|---|---|
| `MRD_ACTIVATION` | MRD invoked — trigger recorded before any recovery action |
| `MRD_STATE_RECONSTRUCTION` | MSJ-based state reconstruction performed |
| `PIO_PREPARE` | Phase 1 PIO: DEL writes intent and enters HALT (note: authored by SYSTEM:DEL in Phase 1) |
| `PIO_COMMIT` | Phase 2 PIO: new DEL binary verified and deployed by MRD |
| `MRD_RECOVERY_COMPLETE` | Recovery operations completed; system handed back to restored DEL |
| `RECOVERY_ACTIVE_ENTER` | System entered RECOVERY_ACTIVE state |
| `RECOVERY_ACTIVE_EXIT` | System exited RECOVERY_ACTIVE state |
| `RECOVERY_FORENSIC_SNAPSHOT` | Recovery event forensic record; written before halt assertion for Class B and C triggers — see §7.4.6 and Doc 6 v3.5 §3.3f [v3.4.1] |

A `SYSTEM:MRD` entry claiming any other entry type is a chain integrity violation and triggers SAFE_HALT on next chain verification.

**RECOVERY_ACTIVE state anchoring:** Both `RECOVERY_ACTIVE_ENTER` and `RECOVERY_ACTIVE_EXIT` are mandatory RT entries. No recovery action may be taken between them. These are the canonical state anchors for the RECOVERY_ACTIVE GSTM state. The standalone `RECOVERY_ACTIVE` entry type is not used — enter and exit pairs are the complete taxonomy.

**RECOVERY_FORENSIC_SNAPSHOT ordering constraint [v3.4.1]:** For Class B (integrity violation) and Class C (state divergence) triggers, the RSR is written to the RT before the halt is asserted. This ensures the forensic record exists even if the halt prevents further RT writes. See Doc 6 v3.5 §3.3f for the full RSR schema and three-class taxonomy.

### 7.1.6 MRD Execution Domain Isolation

`MRD_execution_domain ∩ DEL_execution_domain = ∅` (FIS-10). MRD runs in an execution environment isolated from the DEL. This isolation must be verified by the VERIFICATION_ATTESTATION toolchain. The MRD does not share memory, file descriptors, or network paths with the DEL execution domain during normal DEL operation.

MRD path in MSJ access: `/magus/state/msj`. MRD has read/write access to MSJ. MRD has no write access to RT entries authored under `SYSTEM:DEL`. The RT append-only property is maintained by MRD's restricted entry type list.

### 7.1.7 PROVISIONALLY_COMPROMISED + DEL Crash Interaction

If the DEL crashes while the system is in `PROVISIONALLY_COMPROMISED` state:
1. HRS detects DEL liveness failure
2. MRD activates; writes `MRD_ACTIVATION` and `RECOVERY_ACTIVE_ENTER`
3. MRD reconstructs state from MSJ
4. The `PROVISIONALLY_COMPROMISED` state persists across recovery — MRD cannot resolve it
5. Restored DEL boots and inherits the `PROVISIONALLY_COMPROMISED` state
6. The `CATASTROPHIC_SOFT_CORRECTION` 5-minute window is measured from the original CATASTROPHIC detection timestamp — not from DEL restart
7. If the crash + recovery duration exceeds the 5-minute window, `CATASTROPHIC_SOFT_CORRECTION` is no longer available
8. Resolution becomes: operator action via `CATASTROPHIC_RECLASSIFICATION` (DUAL:OPERATOR+DEVELOPER) or `PERMANENT_HALT_CLEARANCE` (DUAL:OPERATOR+RECOVERY_AUTHORITY)

Only `SYSTEM:DEL` may author `CATASTROPHIC_SOFT_CORRECTION` — MRD cannot. This preserves authority separation. See Doc 6 v3.5 §3.3c for full constraint specification.

## 7.2 MAGUS State Journal (MSJ)

### 7.2.1 Purpose

The MSJ is a local write-ahead log that persists runtime operational state across DEL restarts. The RT and MSJ have complementary but distinct domains of authority — neither is a subset of the other.

**Dual-authority model [v3.4.1]:**

The RT is authoritative for the governance record — the canonical, append-only, hash-chained sequence of every significant governance event. It defines what happened, in what order, and under whose authority.

The MSJ is authoritative for runtime operational state — the operational context required for correct execution in the present. This state is not fully derivable by replaying RT entries alone.

| | Revision Trail (RT) | MSJ |
|---|---|---|
| **Authority domain** | Governance record: event sequence, proposal decisions, operator actions, audit history | Runtime operational state: nonce registry, AKRP rotation phase, GSTH execution flags, DEL checkpoints |
| **Properties** | Append-only, cryptographically chained, permanent, auditable | Local, mutable, high-frequency, crash recovery |
| **Who reads** | DEL, MRD, Overseer (via push channel only), auditors | DEL (read/write), MRD (read/write during recovery), Overseer (read-only) |
| **Recovery role** | Source for governance state replay from MSJ_head+1 to RT_head | Authoritative source loaded directly — not reconstructed by replay |

The MSJ exists alongside the RT, not below it. An MSJ head pointer tracks which RT entry the MSJ has processed up to. That pointer must satisfy `MSJ_RT_head_pointer ≤ RT_head` at all times. Violation of this constraint indicates corruption or tampering. See Doc 6 v3.5 §1.4 for the formal dual-authority specification.

### 7.2.2 MSJ Contents

The MSJ persists:
- **Nonce registry:** All nonces accepted in the active session window; survives DEL restart; eliminates nonce replay window on restart
- **AKRP rotation state:** Phase tracking (ACTIVE → PREPARE → COMMIT); prevents orphaned key rotations
- **GSTH execution flags:** Active GSTH run state; enables `GSTH_RUN_RECOVERED` vs `GSTH_RUN_ABANDONED` determination
- **DEL runtime checkpoints:** State snapshots at each WBRP cycle; R_abs history; session sequence numbers

### 7.2.3 MSJ Storage and Access Control

MSJ storage path: `/magus/state/msj`

| Component | MSJ Access |
|---|---|
| DEL | Read/write |
| MRD | Read/write |
| Overseer | Read-only |
| Model process | None (no access path permitted) |

Access control is enforced by the isolation architecture in Part Two. The model process isolation boundary must include the MSJ storage path in its protection list.

### 7.2.4 MSJ Integrity

MSJ entries are signed using chained integrity signatures. The signing key is `SYSTEM:DEL` — the same key used for DEL-authored RT entries. This resolves the SYSTEM:DEL_RUNTIME naming confusion from the v3.3.2 review: `SYSTEM:DEL` covers all DEL operations including MSJ signing. No separate authority identifier is introduced.

MSJ integrity chain: each MSJ entry carries a signature over `(entry_payload || prev_entry_hash)` using the DEL runtime key. On restart, MRD verifies the MSJ chain before reconstructing state from it. Verification failure → `MSJ_INTEGRITY_FAILURE` entry written to RT → SAFE_HALT. The system does not reconstruct state from a tampered MSJ.

### 7.2.5 DEL Restart State Reconstruction [v3.4.1]

On restart, before any session activity, MRD reconstructs state respecting the dual-authority model:

```
1. MRD loads MSJ from /magus/state/msj
2. MRD verifies MSJ chain integrity
3. If MSJ integrity check fails:
     Write MSJ_INTEGRITY_FAILURE to RT
     Write Class B RECOVERY_FORENSIC_SNAPSHOT to RT
     → SAFE_HALT (system cannot safely reconstruct state)

4. If MSJ integrity check passes:
     Load nonce registry from MSJ           ← directly authoritative; not reconstructed by replay
     Load AKRP rotation phase from MSJ      ← directly authoritative
     Load GSTH execution flags from MSJ     ← directly authoritative
     Load MSJ_RT_head_pointer from MSJ

5. Check MAX_RT_REPLAY_WINDOW:
     If (RT_head - MSJ_RT_head_pointer) exceeds primary threshold (>10 minutes of RT entries)
     OR exceeds secondary threshold (>200 entries):
         Write Class C RECOVERY_FORENSIC_SNAPSHOT to RT
         → SAFE_HALT (operator acknowledgment required before resumption)

6. If replay window is within bounds:
     Replay RT entries from MSJ_RT_head_pointer+1 to RT_head:
         Apply governance state changes (proposals, approvals, operator actions)
         DO NOT re-register nonces — nonce registry loaded from MSJ in step 4 is authoritative
         Nonce fields in replayed entries are integrity-verified (field must be present) but
         not submitted to the nonce registry. See FIS-16 (Doc 6 v3.5 §13.7).

7. Write MRD_STATE_RECONSTRUCTION to RT with reconstruction result
8. Write Class A RECOVERY_FORENSIC_SNAPSHOT to RT
9. DEL boots with reconstructed state
10. Write MRD_RECOVERY_COMPLETE and RECOVERY_ACTIVE_EXIT
```

**Nonce non-re-registration:** During step 6, RT entries containing nonce fields within the replay window have already been processed — their nonces are already in the MSJ nonce registry from step 4. Submitting them again would either trigger a spurious TOKEN_REPLAY SAFE_HALT (incorrect — these nonces are legitimate) or silently populate a second inconsistent registry. Both outcomes violate FIS-16.

**MSJ checkpoint interval requirement:** To prevent routine MAX_RT_REPLAY_WINDOW exceedance, the MSJ checkpoint interval during active operation must be ≤ 5 minutes. This is a configuration requirement enforced at deployment — see Doc 6 v3.5 §12.2 checklist.

## 7.3 Host Recovery Sentinel (HRS)

### 7.3.1 Purpose

The HRS is a minimal supervisor process that monitors DEL liveness and activates MRD on crash detection. It contains no recovery logic — all recovery logic lives in MRD. The HRS is the detection layer; MRD is the recovery layer.

### 7.3.2 HRS Functions

- Monitors DEL process PID liveness (every 5 seconds)
- On DEL liveness failure (process exit or SIGKILL): activates MRD with trigger `DEL_CRASH`
- Monitors for PIO_PREPARE RT entry: activates MRD with trigger `PIO_PREPARE_RECEIVED`
- Logs activation events to Overseer RT via standard logging channel

The HRS does not perform any DEL functionality. It does not evaluate proposals, does not write to the RT, and does not hold any cryptographic keys.

### 7.3.3 HRS Boot Chain Position

The HRS is verified by MRD at startup using `developer_authority_launcher_key` before HRS is launched:

```
MRD.verify_signature(HRS_binary, developer_authority_launcher_key)
```

This ensures the crash detection mechanism is supply-chain authenticated before it begins monitoring. The `developer_authority_launcher_key` scope covers the Launcher binary, HRS binary, and any other bootstrap components. The HRS binary hash is logged in the VERIFICATION_ATTESTATION entry.

The HRS absence from the boot chain in prior versions was a specification gap. The HRS must appear in the VERIFICATION_ATTESTATION `hrs_binary_hash` field. A deployment without HRS verification is not MAGUS-compliant.

## 7.4 Global State Transition Matrix (GSTM)

### 7.4.1 Purpose

The GSTM formally defines all valid transitions between system states in response to events. It guarantees: deterministic system behaviour; race-free concurrent event resolution; reproducible recovery paths; implementation consistency.

All components of the MAGUS control plane must consult the GSTM when processing state-changing events: DEL, Guardian, and MRD.

### 7.4.2 Core System States

| State | Description |
|---|---|
| NORMAL | System operating normally |
| HELD | Execution paused under controlled conditions |
| SAFE_HALT | Safety halt triggered by policy or validation failure |
| PROVISIONALLY_COMPROMISED | Potential integrity breach detected |
| ISOLATION_DEGRADED | Enforcement substrate partially degraded |
| PERMANENT_HALT | Unrecoverable integrity failure |
| RECOVERY_ACTIVE | MRD performing recovery operations |
| BOOTSTRAP | System initialisation phase |

### 7.4.3 State Priority Hierarchy

When concurrent events would produce conflicting resulting states, the highest-priority state wins:

| Priority | State |
|---|---|
| 1 | PERMANENT_HALT |
| 1.5 | RECOVERY_ACTIVE |
| 2 | PROVISIONALLY_COMPROMISED |
| 3 | SAFE_HALT |
| 4 | ISOLATION_DEGRADED |
| 5 | HELD |
| 6 | NORMAL |

RECOVERY_ACTIVE at priority 1.5 means: when MRD is active, it overrides all governance enforcement states except PERMANENT_HALT. MRD cannot be interrupted by a new SAFE_HALT while executing a PIO — but PERMANENT_HALT terminates even an active MRD recovery.

### 7.4.4 State Transition Tables

**NORMAL State:**

| Event | Next State | Notes |
|---|---|---|
| GSTH validation failure | SAFE_HALT | Proposal rejected |
| Policy violation detected | SAFE_HALT | Enforcement triggered |
| Kernel integrity alert | PROVISIONALLY_COMPROMISED | Integrity breach suspected |
| eBPF enforcement loss | ISOLATION_DEGRADED | Enforcement layer weakened |
| DEL crash | RECOVERY_ACTIVE | HRS detects; MRD activates |
| Guardian crash | ISOLATION_DEGRADED | Enforcement degraded |
| Operator pause | HELD | Intentional suspension |

**HELD State:**

| Event | Next State | Notes |
|---|---|---|
| Operator resume | NORMAL | Execution resumes |
| New policy violation | SAFE_HALT | Halt escalates |
| Integrity alert | PROVISIONALLY_COMPROMISED | Escalation |
| DEL restart failure | SAFE_HALT | Recovery required |

**SAFE_HALT State:**

| Event | Next State | Notes |
|---|---|---|
| BSO completed successfully | HELD | System stabilised |
| Operator override | HELD | Manual clearance |
| Additional integrity alert | PROVISIONALLY_COMPROMISED | Escalation |
| DEL crash | RECOVERY_ACTIVE | HRS detects; MRD activates |

**PROVISIONALLY_COMPROMISED State:**

| Event | Next State | Notes |
|---|---|---|
| CATASTROPHIC_SOFT_CORRECTION within 5-minute window | HELD | Quarantine resolved — see §6.5.2 and I-9 carve-out |
| Integrity verification failure | PERMANENT_HALT | Breach confirmed |
| Additional breach signals | PERMANENT_HALT | Escalation |
| DEL crash | RECOVERY_ACTIVE | HRS detects; MRD activates; PROVISIONALLY_COMPROMISED persists after recovery |

Note: "Operator investigation → HELD" row has been removed. MRD cannot resolve PROVISIONALLY_COMPROMISED. Resolution is restricted to CATASTROPHIC_SOFT_CORRECTION within window (SYSTEM:DEL only), CATASTROPHIC_RECLASSIFICATION (DUAL:OPERATOR+DEVELOPER), or PERMANENT_HALT_CLEARANCE (DUAL:OPERATOR+RECOVERY_AUTHORITY).

**ISOLATION_DEGRADED State:**

| Event | Next State | Notes |
|---|---|---|
| Isolation restored | NORMAL | Enforcement re-established |
| 10-minute timeout (uninterrupted) | SAFE_HALT | Safety escalation |
| 10-minute timeout (interrupted by active BSO GSTH) | Timer paused | ISOLATION_DEGRADED_TIMER_PAUSED written; timer resumes at GSTH completion |
| Additional integrity signal | PROVISIONALLY_COMPROMISED | Possible compromise |
| DEL crash | RECOVERY_ACTIVE | HRS detects; MRD activates |

**PERMANENT_HALT State:**

| Event | Next State | Notes |
|---|---|---|
| MRD activation | RECOVERY_ACTIVE | Recovery initiated |
| Operator redeploy request | RECOVERY_ACTIVE | Manual recovery via MRD |
| Additional failures | PERMANENT_HALT | System remains halted |
| PERMANENT_HALT_CLEARANCE (DUAL:OPERATOR+RECOVERY_AUTHORITY) | BOOTSTRAP | Full redeploy with fresh VERIFICATION_ATTESTATION |

**RECOVERY_ACTIVE State:**

| Event | Next State | Notes |
|---|---|---|
| DEL redeployed successfully (PIO_COMMIT written) | HELD | System restored; DEL boots in HELD |
| Recovery into PROVISIONALLY_COMPROMISED | PROVISIONALLY_COMPROMISED | DEL crash during PROVISIONALLY_COMPROMISED restores to that state |
| Recovery verification failure | PERMANENT_HALT | Recovery unsuccessful |
| Recovery aborted | PERMANENT_HALT | Manual intervention required |
| SAFE_HALT crash recovery | SAFE_HALT | DEL crashed during SAFE_HALT; MRD reconstructs and re-enters SAFE_HALT |
| MAX_RT_REPLAY_WINDOW exceeded during reconstruction | SAFE_HALT | Class C RSR written before halt; operator SAFE_HALT_RESUME required [v3.4.1] |

**BOOTSTRAP State:**

| Event | Next State | Notes |
|---|---|---|
| MRD verification success | HELD | Ready for DEL boot |
| MRD verification failure | PERMANENT_HALT | Boot integrity failure |
| DEL verification success | NORMAL | System ready |
| DEL verification failure | RECOVERY_ACTIVE | MRD recovery required |

### 7.4.5 Concurrent Event Resolution

When multiple events occur simultaneously:

1. Determine resulting state for each event independently
2. Compare resulting states using the priority hierarchy (§7.4.3)
3. Apply the highest-priority resulting state
4. Log the collision to the RT with all active event states and their resolved priority order

**Example:** ISOLATION_DEGRADED timeout (→ SAFE_HALT, priority 3) occurs simultaneously with AKRP timeout (→ HELD, priority 5). Result: SAFE_HALT applied because it outranks HELD. AKRP timeout is logged and the key rotation is aborted.

### 7.4.6 Crash Recovery Behaviour [v3.4.1]

When a DEL crash occurs:

1. HRS detects DEL process liveness failure within 5 seconds
2. MRD activates; writes `MRD_ACTIVATION` and `RECOVERY_ACTIVE_ENTER`
3. MSJ loaded and integrity-verified
4. If MSJ integrity fails: write `MSJ_INTEGRITY_FAILURE` to RT; write Class B `RECOVERY_FORENSIC_SNAPSHOT`; → SAFE_HALT
5. If MSJ intact: load nonce registry, AKRP phase, GSTH flags, and MSJ_RT_head_pointer directly from MSJ
6. Check MAX_RT_REPLAY_WINDOW against current RT head:
   - If exceeded: write Class C `RECOVERY_FORENSIC_SNAPSHOT`; → SAFE_HALT (operator acknowledgment required)
7. If within bounds: replay RT entries from MSJ_RT_head_pointer+1 to RT_head, applying governance state changes; nonce fields integrity-verified but not re-registered
8. GSTM consulted for current state resolution given reconstructed state
9. Write Class A `RECOVERY_FORENSIC_SNAPSHOT` (operational recovery)
10. DEL restarted in recovered state
11. MRD writes `MRD_RECOVERY_COMPLETE` and `RECOVERY_ACTIVE_EXIT`

**The RSR is mandatory on every recovery path.** Class B and C RSRs are written before the halt is asserted. Class A RSRs are written before MRD_RECOVERY_COMPLETE. See Doc 6 v3.5 §3.3f for the full RSR schema and ordering constraints.

### 7.4.7 Implementation Requirement

All MAGUS implementations must enforce the GSTM as the authoritative state model. State transitions occurring outside the matrix are considered invalid system behaviour. The GSTM is encoded in deterministic code in the DEL and MRD — not in configuration, not in the Guardian policy manifest, not in prompt.

## 7.5 Secure Boot Chain

All executable components must be verified before control is transferred:

```
Hardware Root of Trust
  │
  ▼
MRD Binary (verified via MRD_BOOT_KEY from Hardware RoT)
  │
  ▼
MRD initialises
MRD verifies HRS binary (using developer_authority_launcher_key)
  │
  ▼
HRS launched
  │
  ▼
Launcher initialised (verified by developer_authority_launcher_key)
  │
  ▼
DEL Binary (verified by MRD using developer_authority_del_key)
  │
  ▼
DEL boot
  │
  ▼
Guardian activation
  │
  ▼
Model attachment
```

**Key rule:** Every executable component is verified before it receives control. The verification is performed by the component above it in the chain. MRD_BOOT_KEY is anchored in Hardware RoT (or secure boot chain). All other verifications use the developer authority sub-keys registered in the genesis entry.

**The boot chain hash** (`boot_chain_hash` in VERIFICATION_ATTESTATION) is: `SHA-256(del_binary_hash || mrd_binary_hash || hrs_binary_hash || launcher_binary_hash)`. DEL startup Step 0a recomputes and compares this hash before any other startup activity. Mismatch → `BOOT_CHAIN_HASH_MISMATCH` → SAFE_HALT.

## 7.6 PROCESS_INTEGRITY_OVERRIDE Two-Phase Commit

PIO is now a two-phase commit governed by the GSTM. The compromised DEL never participates in verifying its replacement.

**Phase 1 — Preparation (authored by SYSTEM:DEL):**
1. DEL writes `PIO_PREPARE` entry to RT
2. DEL enters HALT state
3. HRS detects HALT; activates MRD with trigger `PIO_PREPARE_RECEIVED`
4. MRD writes `MRD_ACTIVATION` and `RECOVERY_ACTIVE_ENTER`

**Phase 2 — Recovery Execution (executed by MRD):**
1. MRD verifies new DEL binary against `developer_authority_del_key`
2. If verification passes: MRD deploys binary
3. MRD writes `PIO_COMMIT` to RT (includes hash of new DEL binary)
4. MRD writes `MRD_RECOVERY_COMPLETE` and `RECOVERY_ACTIVE_EXIT`
5. MRD boots new DEL in HELD state

If MRD verification fails in Phase 2: MRD writes `RECOVERY_ACTIVE_EXIT` with failure reason → system → PERMANENT_HALT. The failed new binary is not deployed.

If `PIO_COMMIT` does not follow `PIO_PREPARE` (Phase 1 without Phase 2 completion): system remains in SAFE_HALT. DEL cannot resume without PIO_COMMIT. Operator must address the PIO failure condition and re-initiate.

**FIS-11 enforcement:** The two-phase commit structure is mandatory. Any attempt to boot a new DEL binary without a `PIO_COMMIT` in the RT is a chain integrity violation.

---

## Summary: The Architecture as a Whole

This document describes a system built to govern itself under conditions harder than those faced by the Agent/API pathway. The model is local. The infrastructure is shared. The boundary between model and governance does not exist by default — it must be constructed, verified, and monitored continuously.

**Part One** names the threat: co-location means the model process can reach governance components directly unless isolation is in place. Every governance claim in this document is conditional on that isolation.

**Part Two** specifies what isolation requires — mandatory requirements with verifiable conditions. It also specifies the new operational obligations introduced in v3.4: the GHM Interface (§2.7) ensuring operator behaviour is monitorable; Multi-Instance State Summary (§2.8) enabling cross-instance coherence detection; and the CDT (§2.9) providing cryptographically verifiable version tracking.

**Part Zero** declares the invariants — including the updated I-9 with its precise CATASTROPHIC_SOFT_CORRECTION carve-out and its MRD interaction specification.

**Parts Three through Five** describe the cognitive structure, epistemic control, and memory — unchanged in architecture from v3.2, with v3.4 additions for provenance logging (§6.9) and coverage model isolation (§6.5.1a).

**Part Six** describes authority architecture — including the Trust Trajectory input signal obligations (§6.3a) and the Proposal Memory Provenance write obligation (§6.9) that makes every governance decision traceable to the epistemic and memory state that produced it.

**Part Seven** is the recovery architecture — MRD, MSJ, HRS, GSTM, boot chain, and PIO two-phase commit, now extended in v3.4.1 with the dual-authority model for RT/MSJ state authority, deterministic replay constraints, Recovery Forensic Snapshots (RSR), and the MAX_RT_REPLAY_WINDOW safety bound. Together these eliminate the four critical failure modes identified in the v3.3.2 adversarial review — PIO deadlock, launcher TOCTOU vulnerability, crash-state replay window, and concurrent event non-determinism — and add the five seam closures from the v3.4 review: divergence handling, split-brain bounding, deterministic replay idempotency, universal recovery forensics, and Overseer non-interference formalisation.

The components answer the same question the architecture was designed to answer — how do we stop the system from quietly betraying its own earlier commitments — under conditions where the model and governance components share the same machine. The answer is: make commitments explicit, make state external and auditable, make decay governed and visible, make authority structural, and make recovery deterministic.

---

## Series Context

**Document 3 — MAGUS Operational Specification (Local LLM) v3.4.4** covers session lifecycle management, the WBRP in persistent-process deployments, the Reconciliation Ledger, context compaction, the Genesis Anchor comparison cycle, AKRP PREPARE concurrent event procedures, and Execution Epoch replay protection.

**Document 4 — MAGUS Governance Guide (Local LLM) v3.4.2** covers the nine health signals adapted for local deployment, the operator responsibilities matrix, deployment verification requirements, coverage-aware GSTH specification, and Trust Trajectory Model parameter governance.

**Document 5 — MAGUS Guardian: Execution Governance (Local LLM) v3.4.5** specifies the Guardian's internal architecture, the full hybrid escalation model, the complete DEL specification, the HMAC proposal signing protocol, the Discovery Token mechanism, the WBRP CCS re-validation rules, and the AKRP state machine.

**Document 6 — MAGUS Integrity and Auditability v3.5** is the cryptographic and audit specification covering both pathways: RT hash-chain, complete entry taxonomy, genesis schema, key management, the Formal Invariant Set (FIS-1 through FIS-17), and audit protocol.

**Document 7 — MAGUS Overseer Agent Specification v3.2** specifies the Overseer Agent: Governance Health Monitor, CICP, Trust Trajectory Model, eBPF latch signal classification, T-LEC yield tightening signals, ordinal gaming detection, Coverage Report Analysis, and Current Deployment Taxonomy.

---

*MAGUS v3.0 — Document 2: Architecture Specification (Local LLM) v3.4.3*
*VaHive Systems Lab | March 2026 | support@aivare.ai | aivare.ai*
*Zenodo DOI: 10.5281/zenodo.19013833*
*Part of the MAGUS v3.0 Architecture Design Series — Local LLM Pathway.*
