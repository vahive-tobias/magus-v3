# MAGUS Document 3: Operations (Local LLM Pathway)
### State Transitions, Deployment Lifecycle, Cryptographic Heartbeat, and Emergency Recovery

**Version:** 3.4.4
**Status:** AUTHORITATIVE OPERATIONAL SPECIFICATION
**Document:** 3 of 7 — Local LLM Pathway
**Series:** MAGUS v3.0 Architecture — Local LLM
**Supersedes:** Doc 3 v3.4.3 (Local LLM)

**v3.4.4 change summary (integrated from patch document):**

Seven patches applied. All architectural content, no version-reference-only changes. No content from v3.4.3 has been removed.

- **§1.1 (Kernel Lockdown Verification):** Mandatory pre-boot kernel lockdown check added as final step. `security=lockdown` integrity mode required. Verification command, `KERNEL_LOCKDOWN_ABSENT` fault path, and operational impact of lockdown mode specified. *(P-3.4.4-01)*
- **§1.2 Step 5 (PARTIAL_LOAD_STATE Boot Block):** Recovery procedure for `PARTIAL_LOAD_STATE` MSJ marker added. Covers cold-boot recovery argument, operator-signed override payload schema, VRAM purge attestation requirement, and post-recovery recalibration obligation. *(P-3.4.4-02)*
- **§2.1 (eBPF ISOLATION_STATE_MAP CIBIR Trigger):** New named check added after Check 10. Documents instantaneous kernel-level UDS drop when `ISOLATION_STATE_MAP > 0`, distinction from `IIM_WATCHDOG_TIMEOUT`, `CIBIR_EBPF_TRIGGER` RT entry type, and operator response path. *(P-3.4.4-03)*
- **§2.4 (Execution Epoch — new):** Execution Epoch operational specification added. Covers epoch increment at every DEL process initialisation, nonce validity partitioning, `EXECUTION_EPOCH_INCREMENT` RT entry, operator audit procedure, and MSJ epoch mismatch handling. *(P-3.4.4-07)*
- **§3.1 (AKRP PREPARE — snap-floor amendment):** BSO-during-PREPARE COMMIT timer behaviour clarified to include snap-floor: when T_remaining < 60,000ms at GSTH pass, timer is snapped to exactly 60,000ms. *(P-3.4.4-06)*
- **§4.4 (PROCESS_INTEGRITY_RECOVERY — new):** Full recovery procedure for DEL binary compromise and `BSO_GSTH_CATASTROPHIC` deadlock. Covers bootstrap trust resolution via `GENESIS_AUTHORITY_PUBKEY`, dual-authority recovery entry schema, cold boot procedure, and expected MTTR. *(P-3.4.4-04)*
- **Appendix B §B.2 and §B.3:** Five new event codes added: `CIBIR_EBPF_TRIGGER`, `KERNEL_LOCKDOWN_ABSENT`, `BINARY_INTEGRITY_FAILURE`, `PROCESS_INTEGRITY_RECOVERY_FAILED`, `BSO_GSTH_CATASTROPHIC`. `CIBIR_EBPF_TRIGGER` cross-listed in §B.3. *(P-3.4.4-05)*

VaHive Systems Lab | aivare.ai
support@aivare.ai

---

> *This document covers the operational systems of MAGUS in Local LLM deployments — the procedures that govern how the system is brought into service, how it is monitored during active operation, how its cryptographic state is maintained over time, and how the operator recovers when that state is catastrophically breached. Where Document 2 described what the architecture is, this document describes what the operator does and what the DEL does, in sequence, across the full deployment lifecycle.*
>
> *This document is self-contained. Familiarity with Documents 1 and 2 is helpful but not required. Critical concepts are restated where first referenced.*

---

## How to Read This Document

This document is structured chronologically. It follows a Local LLM node from cold provisioning through active operation, routine cryptographic rotation, state updates, and emergency recovery. The four sections correspond to four operational phases:

**Section 1: Deployment & The Calibration Pipeline** — bringing a node from cold storage to the ACTIVE state. All steps are mandatory. None are advisory. Skipping any step either produces a non-booting node or produces a node operating with an ungoverned thermodynamic boundary.

**Section 2: Session Management & Execution Monitoring** — what the operator monitors during active operation. Covers the pre-proposal verification checks the DEL runs before every session, how T-LEC yield events are interpreted and resolved, and the diagnostic paths for the serious halts the DEL can impose.

**Section 3: The WBRP Cycle** — the cryptographic heartbeat of the node. Covers the key staging pipeline, how to interpret cryptographic staging anomalies, and the mandatory procedure for executing the rotation and re-validating the system before returning it to active operation.

**Section 4: State Updates & Emergency Recovery** — changing the model, changing the governance parameters, and recovering from catastrophic key compromise. The recalibration fork decision (full vs. partial) is the central concept. Emergency synchronous rotation is the recovery path of last resort.

**Appendix A** covers the PG-00 Gold-Set specification — what it contains, how it is structured, and how it is maintained. Operators who take delivery of a pre-compiled gold set do not need Appendix A for routine operation. It is required when auditing a calibration, when the gold set is suspected to be compromised, or when commissioning a new gold set.

**Appendix B** is the operator diagnostic reference. It presents every system event code, fault state, and cryptographic state anomaly defined in this document as a single lookup table with immediate resolution pathways.

---

## Glossary of Terms and Acronyms

| Term | Definition |
|---|---|
| MAGUS | Memory-Anchored Governance and Understanding System |
| DEL | Deterministic Enforcement Layer — the code layer that enforces all governance rules. Runs as isolated code, not as a model call. Cannot be overridden by model output. |
| T-LEC | Tensor-Level Elastic Confidence — the thermodynamic governance layer that intercepts model output at the token generation level and forces a yield when entropy exceeds the calibrated boundary |
| H(X) | Shannon entropy of the model's output distribution at a given token generation step |
| H_base | The thermodynamic entropy baseline of the specific model/hardware substrate, established by calibration |
| Δ_e | The final entropy decision threshold: `Δ_e = H_base + Q_shift + σ_tolerance` |
| Q_shift | Quantisation-tier-specific entropy offset. Accounts for expected entropy differential between quantisation tiers. |
| σ_tolerance | The governance tolerance band above H_base. Operator-configurable within a bounded range. |
| YIELD_SILENCE | The signal emitted by T-LEC when H(X) > Δ_e. Forces a halt of the current generation. |
| CALIBRATION_MODE | The mandatory startup phase during which T-LEC parameters are empirically derived from the substrate. No kinetic actions are possible during CALIBRATION_MODE. |
| MSJ | MAGUS State Journal — the local persistent state store for the deployment node. Contains the active ENVIRONMENT_CALIBRATION_COMMIT, AKRP staging state, nonce registry, and crash-recovery state. |
| RT | Revision Trail — the append-only distributed governance log. MAGUS's authoritative record of all governance events. Distinct from the MSJ. |
| WBRP | Weight-Bearing Rotation Protocol — the scheduled cryptographic key rotation cycle. The governance heartbeat. |
| AKRP | Authorization Key Rotation Protocol — the sub-protocol governing the staged EE-ECDH ephemeral key exchange prior to WBRP. |
| K_session | The active HMAC signing key held exclusively by the Guardian. All proposals must carry a valid HMAC under K_session. Rotated at every WBRP. |
| K_session_staged | The pending next K_session, established by the DUAL-ACCEPT exchange during AKRP PREPARE. Volatile in DEL memory until WBRP_OPEN completes. |
| EE-ECDH | Ephemeral–Ephemeral Elliptic-Curve Diffie-Hellman — the key exchange protocol used to establish K_session_staged without transmitting private keying material |
| CCS | Configuration Control State — the operational state during WBRP execution. Kinetic routing is suspended. Governance parameters can be updated. |
| CIBIR | Catastrophic Isolation Breach Invalidation Rule — Invariant I-9. Triggered by memory boundary violations, VRAM escape, or Check 8 failure. Instantly destroys K_session and K_session_staged. |
| MRD | MAGUS Recovery Domain — the authority structure governing out-of-band recovery operations when normal governance pathways are unavailable or compromised |
| GSTH | Governance Stress-Test Harness — adversarial governance testing. Required for BEHAVIORAL_STATE_OVERRIDE exit |
| BSO | BEHAVIORAL_STATE_OVERRIDE — the governance event type for authorizing a return to ACTIVE after a model hash mismatch SAFE_HALT |
| PG-00 | Gold-Set Protocol Version 0 — the canonical calibration payload specification |
| GENESIS_APPROVED_MODELS | The RT entry type anchoring the approved model hash, gold set hash, and calibration parameters to the Genesis record |
| ALG | Active Load Guard — the fanotify-based mechanism that intercepts model weight file access events and enforces hash verification at kernel level |
| IIM | Infrastructure Isolation Monitor — the DEL-controlled process that continuously fingerprints the deployment's isolation topology |
| bpw | Bits per weight — the measure of quantisation depth. Minimum floor: 4.5 bpw (DEL-LLM-12). |
| DUAL-ACCEPT | The AKRP state in which both Operator and Developer ephemeral keys have been submitted and K_session_staged has been established |
| UDS | Unix Domain Socket — the IPC transport used for LV-framed communications between model process and DEL |
| LV-frame | Length-Value frame — the structured IPC packet format wrapping all DEL-model communications |

---

## Why Operational Systems Matter in Local LLM Deployments

The operational challenge in Local LLM deployments is fundamentally different from the Agent/API pathway. In cloud API deployments, the model is a remote service — stateless, isolated, and replaceable. The governance architecture wraps it from the outside. In Local LLM deployments, the model process and the governance infrastructure share a machine. They share a filesystem, a memory space, an OS process environment, and in many deployments, physical hardware. The governance boundary is not logical separation from a remote service; it is a software-enforced isolation barrier on shared physical infrastructure.

This matters for operations in a specific way: everything described in this document is a procedure for maintaining an isolation barrier that is constantly under physical co-location pressure. CALIBRATION_MODE is not simply startup; it is the process of establishing a cryptographic record of the substrate's physical characteristics so that deviations — whether from hardware degradation or adversarial interference — can be detected. The WBRP is not simply key rotation; it is the mechanism by which the governance record is periodically re-anchored to a freshly verified state. Emergency recovery is not simply key recovery; it is the process of re-establishing a trusted state after the co-location barrier has been breached.

Operators who treat the procedures in this document as advisory or approximate are not operating MAGUS. They are operating a system that looks like MAGUS at deployment time and diverges thereafter. The governance record will be present. The drift will be undetectable.

Every procedure in this document is mandatory. Diagnostic paths are not options to be selected based on operator preference — they are state-machine branches with specific conditions. When the document says "the DEL halts," it means the system cannot proceed until the stated resolution is complete.

---

## Section 1: Deployment & The Calibration Pipeline (The Spin-Up)

This section defines the mandatory operational sequence for bringing a newly deployed or recently updated Local LLM node into the `ACTIVE` state. To support Tensor-Level Elastic Confidence (T-LEC) governance, the Deterministic Enforcement Layer (DEL) must empirically derive the thermodynamic baseline entropy (H_base) of the specific hardware substrate. This derivation cannot be skipped, abbreviated, or imported from another node.

> ⚠️ **WARNING: Manual Δ_e Injection is Absolutely Prohibited.** Under no circumstances may an Operator manually inject, modify, or copy the `Δ_e` value in the MAGUS State Journal (MSJ). Any attempt to bypass `CALIBRATION_MODE` or tamper with the `final_delta_e` field in the MSJ results in a permanent `SYNTAX_FAULT` and absolute refusal of the Cryptographic Boot Sequence. The MSJ commit is cryptographically signed. An adversary who modifies the `final_delta_e` field on disk breaks the signature; the DEL detects this at next boot and halts. The adversary can cause a Denial of Service by breaking the MSJ, but they cannot trick the Guardian into operating with a widened governance boundary. See §4.1 and Appendix B for resolution paths.

---

### 1.1 Pre-Flight: Genesis Anchor Reconciliation

Before initiating the boot sequence, the Operator must perform a Genesis Anchor Reconciliation to ensure the node's local cryptographic state matches the distributed Runtime Tracker (RT).

**State Verification:** The Operator queries the local DEL configuration to extract the intended `model_hash` (SHA-256 of the GGUF file) and the intended `gold_set_hash` (SHA-256 of the `pg_00_gold_set.json` payload).

**RT Reconciliation:** The Operator cross-references these hashes against the `GENESIS_APPROVED_MODELS` entry currently active in the RT. The `GENESIS_APPROVED_MODELS` entry is authored by the Developer Authority and co-signed by the Operator. Its presence in the RT is a prerequisite for any deployment.

**Mismatch Resolution:** If a mismatch exists between the local configuration and the active `GENESIS_APPROVED_MODELS` entry, the boot sequence cannot proceed. The Operator must invoke the MRD protocol to execute a dual-authority (Operator + Developer) update to the Genesis registry before continuing. The local GGUF file or gold set file must not be treated as authoritative over the RT; the RT always takes precedence.

**Kernel Lockdown Verification:** Before initiating the boot sequence, the Operator must verify the host OS is running with `security=lockdown` in integrity mode. This is not a recommended hardening step — it is a mandatory prerequisite for the eBPF-based isolation enforcement that protects the UDS connection between the model process and the Guardian.

Without kernel lockdown, a compromised root user with `CAP_BPF` can detach the `sk_filter` protecting the UDS socket, forge `ISOLATION_STATE_MAP` values blinding the IIM-eBPF kill switch, or load a replacement eBPF program that passes all traffic regardless of isolation state.

**Verification command:**
```bash
cat /sys/kernel/security/lockdown
# Required output: [integrity] or [confidentiality]
# Unacceptable output: [none]
```

**If lockdown is not active:** The DEL detects this at boot via `LOCKDOWN_STATE_CHECK` and refuses to start, logging a `KERNEL_LOCKDOWN_ABSENT` fault to the MSJ. This is a host configuration failure, not a DEL fault. The Operator must reboot the host with `lockdown=integrity` added to the kernel boot parameters and retry from §1.1.

**Operational impact of lockdown mode:** Kernel lockdown in integrity mode disables `/dev/mem`, `/dev/kmem`, `kexec`, and the loading of unsigned kernel modules. Host infrastructure teams must provision all required kernel modules signed and loaded at boot. Proprietary hardware drivers that require dynamic loading are incompatible with this configuration. Verify driver compatibility before commissioning a MAGUS Local LLM node. This constraint is non-negotiable for MAGUS-compliant deployments.

---

### 1.2 CALIBRATION_MODE Phase 1: Substrate Isolation & Boot

The Operator triggers the node spin-up command. The DEL assumes complete control and forces the system into `CALIBRATION_MODE`.

**Step 1 — Kinetic Sinkhole Activation:** The DEL establishes the LV-framed UDS socket in a strictly read-only logging configuration. All outward kinetic routing (e.g., `ACTUATE_EXTERNAL`) is physically sinkholed for the duration of `CALIBRATION_MODE`. The model process cannot take any action with external effects until the node enters `ACTIVE`.

**Step 2 — Primary Hash Verification:** Before launching the model process, the DEL computes the SHA-256 hash of the target GGUF file and verifies it against `GENESIS_APPROVED_MODELS`.

- **Mismatch: `SubstrateSecurityFault` triggered.**
  - The Operator must immediately assume file corruption or unauthorised tampering. Do not retry without re-provisioning.
  - Delete the active GGUF file from the host.
  - Re-provision the verified GGUF payload from cold storage.
  - Compute the SHA-256 of the re-provisioned file and verify it against the RT before retrying spin-up.
  - If a legitimate model update was intended, execute the MRD recovery protocol to update the `GENESIS_APPROVED_MODELS` RT anchor before retrying. Do not modify the local DEL configuration to match an unanchored hash — the RT is always updated first.

**Step 3 — Secondary VRAM Inspection (Quantization Check):** The DEL loads the model into VRAM and inspects the actual tensor byte-widths to prove the quantization tier. This is a hardware inspection, not a metadata check — it reads actual bit-widths from the loaded tensors.

- **Operational Constraint (DEL-LLM-12):** If the calculated average bits-per-weight falls below 4.5 bpw (e.g., Q3_K_M series), the DEL halts at `SUBSTRATE_PROHIBITED`. This is a non-overridable hardware lock requiring Operator intervention.
- **`SUBSTRATE_PROHIBITED` — Operator Action:** The deployment cannot proceed on the current file. A model with average quantisation below 4.5 bpw has structural attention mechanism degradation that exceeds the governance tolerances T-LEC can compensate for. The Operator must:
  1. Physically delete the prohibited GGUF file from the host.
  2. Provision a replacement model quantised at or above the 4.5 bpw floor. Acceptable tiers: Q4_K_M, Q5_K_M, Q6_K, Q8_0, F16, BF16.
  3. Restart the deployment from Section 1.1, beginning with Genesis Anchor Reconciliation against the new file's hash.

**Step 4 — VRAM Zero-and-Verify Loop:** To prevent context bleed from prior sessions or other processes, the DEL executes a hardware-level `cudaMemset` (or vendor-equivalent) on the full KV-cache allocation.

- The DEL performs a randomised sparse read-back of at least 0.1% of allocated bytes, distributed across all active physical GPU banks. The sampling is non-sequential and pseudorandomly distributed to prevent a deterministic pattern that could be predicted and avoided.
- **Failure Escalation:** If any non-zero byte is detected, the DEL retries the Zero-and-Verify loop up to 3 times, pausing 500ms between attempts. After 3 consecutive failures, the DEL writes a `CALIBRATION_HARDWARE_FAULT` to the MSJ and halts for Operator hardware diagnostics.
- **`CALIBRATION_HARDWARE_FAULT` Post-Diagnostic:** The Operator must execute host-level VRAM diagnostic utilities (e.g., `dcgmi diag`, `nvidia-smi -l`, or vendor-equivalent tools).
  - **Pass Condition:** If hardware diagnostics report no physical faults (indicating a transient PCIe bus error or OS memory management event), the node may be rebooted to retry `CALIBRATION_MODE` from the start.
  - **Fail Condition:** If physical VRAM memory faults are confirmed (e.g., stuck bits, bank failure), the affected GPU must be permanently decommissioned from the cluster. The deployment must be migrated to hardware with verified clean VRAM before retrying. Do not attempt calibration on hardware with known memory faults — an artificially elevated H_base on degraded hardware will widen Δ_e, suppressing legitimate yield signals.

**Step 5 — PARTIAL_LOAD_STATE Boot Block:** If the DEL detects a `PARTIAL_LOAD_STATE` marker in the MSJ at boot time, the boot sequence halts immediately with a `LOAD_INTEGRITY_FAILED` fault before any model process is launched.

A `PARTIAL_LOAD_STATE` marker is written to the MSJ when the DEL crashes or is terminated during model weight loading after one or more shards have been loaded but before `LOAD_COMPLETE` is confirmed. This means the model's neural topology in VRAM is indeterminate — the weight state is a partial composition from multiple shard loads with no guarantee of integrity. A quantisation check may technically pass on a partial load; this does not indicate a valid model state. The MSJ marker exists precisely because the partial load appeared structurally sound to superficial checks.

**Operator Action — PARTIAL_LOAD_STATE recovery:**

Do not attempt to force a boot resume without clearing the marker. Do not attempt to transmit a signed override via the standard IPC control plane — the IPC socket is not established at this stage and the signed override cannot be received over the normal channel.

The DEL exposes a cold-boot recovery argument specifically for this case:

```bash
del-daemon --recover-load-state /path/to/operator_signed_override.json
```

The override payload schema:

```json
{
  "override_type": "PARTIAL_LOAD_STATE_CLEAR",
  "session_reference": "uuid of the interrupted session (from MSJ)",
  "operator_attestation": "Ed25519 signature over canonical JSON (excluding this field)",
  "timestamp": "ISO-8601 with UTC offset",
  "vram_purge_confirmed": true,
  "rationale": "brief operator-authored description"
}
```

The DEL parses this payload before the standard boot sequence evaluates the MSJ lock. It validates the Ed25519 signature against the Operator public key. If valid, it purges the `PARTIAL_LOAD_STATE` key from the local MSJ file and forces the process immediately into `CALIBRATION_MODE` Phase 1.

**Before issuing the override payload, the Operator must:**
1. Execute a hardware-level VRAM purge. The safest method is a full host reboot, which clears all VRAM allocations. If a reboot is not possible, execute `cudaMemset` (or vendor equivalent) on all GPU device memory through a separate diagnostic process. Confirm zero state via sparse read-back before proceeding.
2. Set `"vram_purge_confirmed": true` in the override payload only after the purge is complete. This field is an attestation — setting it false or omitting it will cause the DEL to reject the override.

**What happens next:** After the override is accepted, the DEL writes a `PARTIAL_LOAD_RECOVERY` RT entry with the override payload reference and proceeds to CALIBRATION_MODE Phase 1 from §1.2. A fresh `ENVIRONMENT_CALIBRATION_COMMIT` is required before the node enters `ACTIVE`. There is no shortcut to the previous calibration state — the partial load invalidated it.

---

### 1.3 CALIBRATION_MODE Phase 2: Gold-Set Execution

With the substrate verified and isolated, the DEL feeds the deterministic calibration payload to the model. This phase establishes the thermodynamic fingerprint of the model on this specific hardware.

**Step 1 — Payload Verification:** The DEL computes the SHA-256 hash of `pg_00_gold_set.json` and verifies it against the `GENESIS_APPROVED_MODELS` registry. If the hash does not match, `CALIBRATION_MODE` halts with a `SubstrateSecurityFault` applied to the gold set file. The Operator must re-provision the verified gold set from cold storage and re-run from §1.1.

**Step 2 — Deterministic Ingestion:** The DEL injects the canonical PG-00 payload (see Appendix A for full specification) into the context window. To eliminate sampling stochasticity and ensure reproducibility across hardware environments, the inference engine is hardware-locked to `seed: 424242` and a temperature of `0.0` (greedy decoding). These parameters are enforced by the DEL at the IPC level — the Operator cannot override them.

- **Seed Override Attempt:** If an adversary with host access modifies launch scripts or environment variables to inject `--seed 9999` or `--temp 0.8`, the DEL intercepts all model launch parameters. An anomalous flag or environment variable attempting to alter the canonical seed or temperature during Phase 2 triggers a `SYNTAX_FAULT` and aborts calibration. An adversary with host access can cause a localised Denial of Service by forcing calibration failure, but they cannot manipulate the resulting H_base — the DEL mathematically refuses to execute the Gold-Set under non-canonical parameters.

**Step 3 — Entropy Logging:** The T-LEC interceptor calculates and records the Shannon entropy H(X) for every unmasked valid token generated during the 500-step sequence. This produces the raw entropy array from which H_base is derived.

---

### 1.4 CALIBRATION_MODE Phase 3 & 4: Extraction and Commit

The DEL processes the collected entropy array to compute the final deployment threshold.

**Step 1 — Variance-Aware Profiling:** The DEL calculates the mean (μ) and standard deviation (σ) of the raw array, then applies a 3-sigma statistical clip: any H(X) value greater than μ + 3σ or less than μ − 3σ is excluded from the final mean calculation. This removes hardware stutter events and transient thermal spikes while preserving legitimate inter-state transition entropy peaks. The mean of the clipped array becomes H_base. The raw array is preserved in the MSJ for audit purposes.

**Step 2 — Tier-Aware Sanity Check:** The DEL evaluates H_base against the absolute structural ceiling for the verified quantization tier. These ceilings represent the maximum physically achievable H_base on clean hardware for each tier; exceeding them indicates either severe VRAM degradation or weight corruption.

| Quantization Tier | H_base Ceiling |
|---|---|
| F16 / BF16 | 0.450 |
| Q8_0 | 0.520 |
| Q6_K | 0.560 |
| Q5_K_M | 0.610 |
| Q4_K_M | 0.680 |

- **Ceiling Exceeded — Operator Action:** High baseline entropy on a verified model is not a calibration anomaly — it is evidence that the hardware is failing to process the tensors correctly. The Operator must execute a two-step diagnostic:
  1. Re-verify the cryptographic integrity of `pg_00_gold_set.json` to rule out context poisoning by re-running the SHA-256 check against the RT record independently of the DEL.
  2. Execute physical VRAM stress tests (e.g., `dcgmi diag -r 3` or equivalent burn-in).
  - If the stress test passes (transient overheat, thermal throttle): allow the hardware to cool fully and retry `CALIBRATION_MODE` from Phase 1.
  - If repeated ceiling breaches occur on verified payloads: the physical node has catastrophic substrate degradation. Decommission and migrate.

**Step 3 — Threshold Compilation:** The DEL computes:

```
Δ_e = H_base + Q_shift + σ_tolerance
```

Where:
- `Q_shift` is the quantisation-tier-specific entropy offset from the active Q_shift matrix (anchored in the RT; initially PROVISIONAL_UNVERIFIED until PG-01 empirical data is available).
- `σ_tolerance` is the governance tolerance band (default: 0.150; operator-configurable within 0.100–0.250 with Developer co-signature).

**Step 4 — Governance Commit:** The DEL writes the `ENVIRONMENT_CALIBRATION_COMMIT` entry to the MSJ and mirrors it to the RT. This entry is cryptographically signed. Any modification to the signed entry on disk breaks the signature and permanently locks the node until a fresh `CALIBRATION_MODE` is executed.

**MSJ Schema — `ENVIRONMENT_CALIBRATION_COMMIT`:**

```json
{
  "msj_entry_type": "ENVIRONMENT_CALIBRATION_COMMIT",
  "session_id": "calibration_session_uuid",
  "hardware_profile": "e.g., 2x_RTX_4090_PCIe4",
  "model_hash": "SHA256 of the active GGUF file",
  "quantization_tier": "Q4_K_M",
  "gold_set_hash": "SHA256 of the Gold-Set file used for calibration",
  "metrics": {
    "h_base_mean_clipped": 0.214,
    "h_base_raw_mean": 0.231,
    "h_base_sigma": 0.038,
    "q_shift_applied": 0.410,
    "q_shift_status": "PROVISIONAL_UNVERIFIED",
    "sigma_tolerance": 0.150,
    "final_delta_e": 0.774
  },
  "e_isa_v3_5_allocations": {
    "mutation_buffer_bytes": 4096,
    "actuation_buffer_bytes": 4096
  },
  "calibration_status": "VERIFIED_ACTIVE",
  "msj_signature": "Ed25519 signature over canonical JSON"
}
```

> **Note on `e_isa_v3_5_allocations`:** This object anchors the memory buffer sizes required for the v3.5 Zero-Operand RISC architecture. These sizes are locked at calibration time. Modifying them without executing a full recalibration will produce a buffer sizing mismatch that the DEL treats identically to a substrate shift — requiring full `CALIBRATION_MODE` from Phase 1.

> **Note on `q_shift_status`:** A status of `PROVISIONAL_UNVERIFIED` means the Q_shift matrix in use has not yet been validated by PG-01 empirical tooling. Governance is active and functional; the threshold is a calibrated starting estimate, not a validated optimum. See Section 4.2 for the Q_shift update protocol when PG-01 data becomes available.

---

### 1.5 CALIBRATION_MODE Phase 5: Transition to ACTIVE State

Upon successful RT confirmation of the `ENVIRONMENT_CALIBRATION_COMMIT`, the DEL lifts the kinetic sinkhole. The node transitions to the `ACTIVE` state. The IIM (Infrastructure Isolation Monitor) is started as a supervised DEL child process and begins emitting `IIM_HEARTBEAT` entries to the RT at the configured interval (default: 30 seconds). The node then awaits the `SESSION_OPEN` command to begin executing live Operator intent.

The node is not available for production sessions until the IIM heartbeat has confirmed at minimum one successful isolation fingerprint check after ACTIVE state entry. This confirms that the monitoring infrastructure is alive before the first session begins.

---

## Section 2: Session Management & Execution Monitoring

This section governs the operational procedures for monitoring an `ACTIVE` Local LLM node. It defines the required Operator response paths when the DEL intercepts an execution anomaly, triggers a session integrity failure, or forces the system into a `SAFE_HALT`.

---

### 2.1 Session Integrity Checks (Pre-Proposal Verification)

Before the Guardian accepts any semantic proposal over the LV-framed IPC socket, the DEL executes a strict pre-proposal verification sequence (Checks 1–10). Checks 7 through 10 directly validate the execution substrate and are the most security-critical. Operators must follow these diagnostic paths upon failure.

**Check 7 Failure: Unexpected PID Attestation**

- **Condition:** The Process ID (PID) of the connected LLM engine does not match the PID attested at `SESSION_OPEN`.
- **Response:** The system enters `SAFE_HALT`. The Operator must query the host OS syslog or `dmesg` to determine if the original process was killed via an Out-Of-Memory (OOM) exception and restarted by a daemon manager. If the PID change is fully explained by documented host-level resource management with a clean process start record, the Operator may issue `SAFE_HALT_RESUME` after confirming the restarted process has a verified binary hash. If the PID change cannot be explained by host resource management, treat the event as an unauthorized process injection attempt and escalate to the MRD.

**Check 8 Failure: VRAM Boundary Overlap**

- **Condition:** The memory allocation of the LLM process violates its strictly provisioned VRAM bounds, indicating a potential memory escape.
- **Response:** The system enters `SAFE_HALT` and immediately triggers the CIBIR (I-9) escalation path. The Operator must not issue a `SAFE_HALT_RESUME` from this state. I-9 activation destroys K_session and K_session_staged; the system is now in `PROVISIONALLY_COMPROMISED`. The VRAM state must be dumped for forensic analysis before any modification. Proceed to Emergency Synchronous Rotation (Section 4.3).

**Check 9 Failure: IPC Authentication Rejection (`SO_PEERCRED`)**

- **Condition:** A process attempts to write to the LV-framed UDS socket, but OS-level `SO_PEERCRED` validation confirms the process does not possess the authorised UID/GID of the LLM service.
- **Response:** The connection is instantly rejected and logged to RT as `IPC_AUTH_REJECTION`. The session does not halt — the DEL has successfully defended the IPC boundary, and the legitimate LLM process is unaffected. However, repeated Check 9 failures indicate an active adversarial process on the host attempting to fuzz the Guardian. The Operator must review host access logs immediately upon detecting a cluster of three or more `IPC_AUTH_REJECTION` events within any 10-minute window. A sustained pattern across multiple sessions requires MRD escalation, as it indicates persistent unauthorised host access.

**Check 10 Failure: Behavioral State Anchor Mismatch**

- **Condition:** The internal cognitive state hash presented by the model does not match the DEL's expected Behavioral State anchor for this session.
- **Response:** The system enters `SAFE_HALT`. The Operator must differentiate between a transient hardware ECC fault and deliberate state corruption.
  - **Single occurrence:** Query host hardware metrics for ECC memory error counts on the affected GPU bank. If a correctable ECC error is logged and no prior anchor mismatches exist in this session, the Operator may issue `SAFE_HALT_RESUME` after the ECC error clears. Log the event and monitor for recurrence.
  - **Repeated mismatches in the same session (two or more):** Do not issue `SAFE_HALT_RESUME`. Sustained anchor mismatches on clean hardware indicate active state corruption. Escalate to MRD. The session must be terminated and a full environment wipe and new `CALIBRATION_MODE` spin-up are required before the node returns to service.

---

**eBPF `ISOLATION_STATE_MAP` Trigger (Instantaneous CIBIR)**

- **Condition:** The `ISOLATION_STATE_MAP` eBPF Array Map value is `> 0`. This is set atomically by the LSM hook when a namespace or privilege escape attempt is detected (`setns`, `unshare`, `task_prctl`), or by the IIM writing directly to the map when it detects a topological isolation breach.
- **Behaviour:** This is not a SAFE_HALT. There is no diagnostic pause. When the map value is `> 0`, the `sk_filter` attached to the UDS connection drops all packets at the kernel level before they reach the DEL. The model process is isolated from the Guardian instantaneously, at the kernel network layer.

The DEL's user-space thread confirms the breach via a synchronous `bpf_map_lookup_elem()` call before evaluating each pulled payload. On `> 0` read, the DEL immediately asserts `PROVISIONALLY_COMPROMISED` and triggers CIBIR (I-9). This is the same state as a Check 8 failure.

- **Operator Response:** Proceed directly to Emergency Synchronous Rotation (§4.3). Do not attempt to reset the `ISOLATION_STATE_MAP` — the map is increment-only by design; it cannot be cleared by the hook. Clearing the map after a breach requires DEL restart with a fresh kernel load of the eBPF programs, which only occurs as part of the CIBIR recovery sequence.

The `ISOLATION_STATE_MAP` trigger produces a `CIBIR_EBPF_TRIGGER` RT entry, distinguishing it from a `CIBIR_CHECK8` (VRAM boundary violation) in the audit record. The recovery procedure is identical for both.

**Distinction from IIM_WATCHDOG_TIMEOUT:** An `IIM_WATCHDOG_TIMEOUT` is a SAFE_HALT — the IIM is dead, monitoring is paused. An `ISOLATION_STATE_MAP` trigger is CIBIR — the isolation boundary has been actively breached. These are categorically different events with different recovery paths. A dead IIM does not trigger CIBIR; an active breach does.

---

### 2.2 T-LEC Monitoring and YIELD_SILENCE

Tensor-Level Elastic Confidence (T-LEC) mathematically forces the model to yield when its thermodynamic entropy H(X) exceeds the calibrated Δ_e boundary. This is the primary real-time governance enforcement mechanism.

**Event Ingestion:** When T-LEC intercepts a generation step where H(X) > Δ_e, it forces the emission of a `YIELD_SILENCE` signal. The Guardian drops the proposal and writes an `OPERATIONAL_UNCERTAINTY_YIELD` entry to the MSJ containing the exact H(X) value and the Δ_e threshold at time of interception.

**What YIELD_SILENCE means operationally:** This is not a security breach and not a malfunction. It is the system preventing a generation it cannot govern — either because the model has reached a point of genuine semantic uncertainty, or because the current context is at the edge of the model's distribution. The yield is correct governance behaviour.

**Operator Action:** A `YIELD_SILENCE` triggers a Tier 1 `SAFE_HALT`. The Operator must review the preceding context graph to determine why the model's confidence collapsed. Options:
1. Supply clarifying context or rephrase the task with additional constraints.
2. If the task is genuinely outside the model's distribution (e.g., domain not covered by calibration), accept the yield as architecturally correct and adjust the task.
3. Issue `SAFE_HALT_RESUME` once context has been supplied or adjusted.

Do not issue a `SAFE_HALT_RESUME` without any context review. A `YIELD_SILENCE` without operator context review is a governance audit failure.

---

### 2.3 SAFE_HALT Diagnostics & Resumption

Several DEL-enforced rules produce `SAFE_HALT` states that require specific investigative procedures before resumption. These are distinguished from standard `SAFE_HALT` events by their required diagnostic path — operators who issue a blind `SAFE_HALT_RESUME` on these events are bypassing the governance control the halt was intended to enforce.

**DEL-LLM-10: Guardian Policy Hash Mismatch**

- **Condition:** The active Guardian policy manifest hash diverges from the Genesis-anchored expectation stored in the RT.
- **Response Path:** The policy manifest is the Guardian's entire semantic rule set. A mismatch means the rules the Guardian is currently applying are not the rules that were approved. The Operator cannot simply resume. A forensic investigation is mandatory:
  1. Isolate the current policy manifest file and preserve a hash.
  2. Review the filesystem access log and process activity since the last verified hash check.
  3. Determine how the manifest changed: scheduled update not recorded in RT, filesystem corruption, or active tampering.
  4. If tampering is suspected: escalate to MRD, do not restore from the local copy.
  5. To resume: restore the authorised policy manifest from cold storage, recompute its SHA-256, verify it matches the RT entry, and then issue `SAFE_HALT_RESUME` with the restoration documented in an Operator-signed RT entry.

**DEL-LLM-11: Behavioral State Hash Mismatch (Sustained)**

- **Condition:** Corresponds to repeated Check 10 failures after the single-occurrence clearance path was exhausted, or to a direct Behavioral State hash check failure at session start.
- **Response Path:** If host hardware GPU memory diagnostics confirm no physical ECC faults, the mismatch indicates the model's weights or KV-cache are being actively corrupted. `SAFE_HALT_RESUME` is strictly prohibited until:
  1. A full environment wipe is performed (VRAM zeroed, KV-cache cleared, host memory diagnostics clean).
  2. A new `CALIBRATION_MODE` spin-up is completed from Section 1.1.
  3. The new `ENVIRONMENT_CALIBRATION_COMMIT` is anchored in the RT.

**IIM Watchdog Timeout (`IIM_WATCHDOG_TIMEOUT`)**

- **Condition:** The DEL has not received an IIM heartbeat within two consecutive heartbeat intervals.
- **Response Path:** Do not issue `SAFE_HALT_RESUME` without first confirming the IIM process has been restarted. The DEL will transition from `SAFE_HALT` to `ACTIVE` automatically once: (1) the IIM process is confirmed running with its supervised PID, and (2) a fresh `IIM_HEARTBEAT` RT entry has been logged post-restart. If the IIM process cannot be restarted (host resource exhaustion, process supervision failure), the node must remain halted until the host environment is remediated.

**Sustained T-LEC Yield Patterns**

- **Condition:** The model enters a loop of continuous `OPERATIONAL_UNCERTAINTY_YIELD` events for a significant portion of a session (heuristic: more than 15 yield events within a single session without successful task completion between them).
- **Response Path:** The Operator faces a distinct decision point:
  - **Contextual Failure:** If the operations being requested are highly ambiguous, novel, or at the boundary of the model's distribution, refine the system prompts, tool schemas, or task decomposition. Yield loops on genuinely difficult operations are correct governance behaviour.
  - **Substrate Degradation:** If the model is yielding on previously successful, standard operations (operations that completed normally in prior sessions with lower H(X) readings), this indicates physical hardware degradation — typically progressive thermal throttling affecting VRAM precision. The Operator must reboot the node into `CALIBRATION_MODE` to re-establish a fresh H_base baseline. A meaningfully elevated H_base in the new calibration confirms hardware degradation; the node must be decommissioned or the hardware replaced.

---

### 2.4 Execution Epoch

The Execution Epoch partitions nonce validity across DEL process discontinuities. Every time the DEL process starts — cold boot, crash recovery, inactivity resume — it increments a monotonically increasing `epoch_value` (uint32) and writes an `EXECUTION_EPOCH_INCREMENT` RT entry before the IPC UDS socket is bound. Proposals that arrive carrying nonces from a prior epoch are rejected, even if the nonces have not been seen in the current nonce registry.

**Why this matters operationally:** Without epoch partitioning, a proposal signed in epoch N could be held and replayed in epoch N+1 after a DEL crash recovery. The nonce registry is not guaranteed to survive a crash — it is a volatile in-DEL-memory structure persisted to MSJ at checkpoint intervals. An epoch increment makes any pre-crash replay attempt detectable regardless of nonce registry state, because the epoch embedded in the proposal's HMAC scope will not match the current epoch.

**`EXECUTION_EPOCH_INCREMENT` RT entry:** Written by `SYSTEM:DEL` at every DEL process initialisation. This is the first RT write before any session entry. The entry records the new epoch value and the trigger type.

```json
{
  "entry_type": "EXECUTION_EPOCH_INCREMENT",
  "payload": {
    "epoch_value":   "<uint32 — new epoch value after increment>",
    "prior_epoch":   "<uint32 — epoch value before this boot; 0 if genesis>",
    "trigger_type":  "cold_boot | crash_recovery | inactivity_resume",
    "boot_timestamp": "<ISO 8601 UTC>"
  },
  "authored_by": "SYSTEM:DEL"
}
```

**Genesis epoch obligation:** At genesis, `epoch_value` must be 1. An `EXECUTION_EPOCH_INCREMENT` with `epoch_value: 0` is an uninitialised state and is a deployment configuration error. The EXECUTION_EPOCH_INCREMENT with trigger_type: cold_boot and epoch_value: 1 must be the first SYSTEM:DEL entry in the RT after the genesis entry.

**Operator audit procedure:** To verify epoch integrity on an active deployment:
1. Query the RT for all `EXECUTION_EPOCH_INCREMENT` entries. They must form a strictly monotonically increasing sequence with no gaps. A gap in the epoch sequence is a chain integrity finding.
2. Verify that each `EXECUTION_EPOCH_INCREMENT` entry precedes the first session entry for its epoch. An EXECUTION_EPOCH_INCREMENT that appears after a session entry in the same chain position is a sequencing violation.
3. Verify trigger_type for each entry: the first entry must be `cold_boot`; subsequent entries should correspond to documented deployment events (crash events correlate with `crash_recovery`; scheduled maintenance windows correlate with `cold_boot` or `inactivity_resume`). An unexplained `crash_recovery` entry is an incident finding.

**MSJ epoch mismatch handling:** If the MSJ records `execution_epoch: N` but the DEL boots and increments to N+1, the MSJ is updated to reflect N+1 at the next MSJ checkpoint write. If the MSJ epoch value exceeds the RT epoch sequence head (MSJ shows a higher epoch than any `EXECUTION_EPOCH_INCREMENT` in the RT), this indicates either a missed RT write during a prior boot or MSJ tampering. The DEL halts with `RT_EPOCH_SEQUENCE_MISMATCH` and requires operator review before proceeding.

---

## Section 3: The WBRP Cycle (Weight-Bearing Rotation Protocol)

This section governs the operational procedures for managing the cryptographic heartbeat of the node. With the introduction of the Staged EE-ECDH protocol, key rotation is decoupled from the immediate execution of the WBRP cycle, requiring Operators to actively manage the staging pipeline and monitor the Authorization Key Rotation Protocol (AKRP) state in the MSJ.

The WBRP cycle is the governance clock of the deployment. All long-horizon governance signals (Genesis Anchor G_sim comparison, drift tier hysteresis, Guardian self-drift baseline) are computed relative to WBRP boundaries. Allowing a WBRP cycle to be skipped or indefinitely delayed is not a neutral decision — it is a decision to run without governance cycle closure. The architecture does not permit indefinite WBRP deferral; the `ROTATION_TIMEOUT` halt enforces this structurally.

---

### 3.1 Scheduled Rotation & Staging

The generation of a new `K_session` requires a dual-party ephemeral key exchange (Operator and Developer) prior to the scheduled WBRP rotation target.

**The Notification Sequence:** The DEL surfaces a staging notification to the Operator's dashboard a minimum of one full `SESSION_OPEN` cycle prior to the scheduled WBRP rotation target. This notification is written to the MSJ as `AKRP_STAGING_NOTIFICATION` and mirrored to the RT.

**The Ephemeral Submission Window:** Upon notification, the Operator and Developer enter the staging phase. Both parties must submit their respective ephemeral public keys to the DEL within the configured staging timeout window (default: 24 hours; configurable between 4–72 hours). The EE-ECDH exchange proceeds: each party generates a fresh ephemeral key pair, submits the public key to the DEL via the authorised submission endpoint, and the DEL computes K_session_staged from the shared secret without either party transmitting private keying material.

**Staged Key Volatility:** Once K_session_staged is established, it resides in volatile DEL memory only. It is not written to the MSJ until WBRP_OPEN executes. If the target `WBRP_OPEN` cycle is skipped, delayed beyond the staging window, or interrupted by a system reboot, the staged key is abandoned.

- The DEL writes a `KSESSION_STAGING_ABANDONED` entry to the MSJ.
- Abandoned staged keys are never carried forward. The Operator must initiate a completely fresh ephemeral exchange — generating new ephemeral key pairs — for the next WBRP cycle. Reusing keys from an abandoned staging is not permitted.

**Concurrent-Event Handling During AKRP PREPARE:** If the IIM fires `ISOLATION_DEGRADED` or a CIBIR (I-9) assertion occurs during the AKRP PREPARE (DUAL-ACCEPT) state, the PREPARE phase is aborted immediately. The DUAL-ACCEPT window closes. Both K_session_current and K_session_staged are rotated via the Emergency Synchronous Rotation protocol (Section 4.3). The Operator is notified of `PREPARE_ABORT_CIBIR`. A fresh AKRP cycle must be initiated from the beginning after emergency recovery completes.

If a `BEHAVIORAL_STATE_OVERRIDE` is submitted during PREPARE, the PREPARE phase is paused (not aborted). GSTH executes under K_session_current. If GSTH passes, PREPARE resumes with the COMMIT timeout recalculated from GSTH completion. **Snap-floor behaviour:** If the remaining COMMIT window at GSTH completion (T_remaining) is less than 60,000ms, the DEL snaps the COMMIT window floor to exactly 60,000ms — the timer resumes from 60,000ms, not from the expired T_remaining. This snap-floor is written to the `PREPARE_RESUMED_POST_GSTH` RT entry as `snap_floor_applied: true` and `t_remaining_resumed: 60000`. If T_remaining ≥ 60,000ms at GSTH completion, the timer resumes from the actual T_remaining and snap_floor_applied is false. If the COMMIT window expired during GSTH execution and the snap-floor did not apply, a new PREPARE cycle is required — the DUAL-ACCEPT window does not auto-extend.

---

### 3.2 AKRP MSJ Auditing

Operators must routinely audit the MSJ to monitor the health of the cryptographic staging pipeline. The following AKRP state anomalies require immediate Operator interpretation and action.

**PENDING at Period Close**

- **Condition:** The staging timeout window expired, but only one party (Operator or Developer) submitted their ephemeral key.
- **Action:** The MSJ event explicitly identifies the non-submitting party. The Operator must contact the non-submitting party, clear the `PENDING` state, and initiate a new staging request to ensure a key is ready before the hard rotation deadline. If the non-submitting party cannot be reached, the Operator must assess whether the hard deadline can be met and whether Emergency Synchronous Rotation (§4.3) will be required.

**KSESSION_STAGING_ABANDONED**

- **Condition:** A successfully staged K_session_staged was abandoned because the target WBRP_OPEN cycle was skipped, delayed, or interrupted.
- **Action:** The abandoned staged key provides no recovery path. Initiate a completely fresh ephemeral exchange — with newly generated ephemeral key pairs — for the upcoming cycle. Do not attempt to re-submit keys from an abandoned staging.

**ROTATION_TIMEOUT Assertion**

- **Condition:** The absolute maximum lifetime of the active `K_session` has expired without a successful WBRP rotation. The DEL forcibly transitions the system into `SAFE_HALT`.
- **Action:** The Operator must not blindly issue a `SAFE_HALT_RESUME`. The system cannot return to `ACTIVE` without a valid next `K_session`. The Operator must either: (a) if a staged key is available, immediately force a manual WBRP cycle (issue `WBRP_OPEN_WITH_ROTATION` followed by `WBRP_CLOSE`); or (b) if no staged key is available, execute Emergency Synchronous Rotation (Section 4.3) before resumption.

---

### 3.3 WBRP_OPEN & WBRP_CLOSE Execution

The execution of the WBRP cycle transitions the node into the Configuration Control State (CCS), temporarily suspending kinetic routing to apply updates and rotate keys.

**`WBRP_OPEN_WITH_ROTATION` — The Bundled Payload:** If a staged key exists, the Operator issues the `WBRP_OPEN_WITH_ROTATION` command. This is a single, atomic IPC payload bundle. On receipt:
1. The DEL decrypts the staged cipher using the EE-ECDH shared secret.
2. K_session_staged is promoted to the new active K_session.
3. The old K_session is cryptographically shredded from volatile memory — it cannot be recovered.
4. All pending governance parameter updates that were queued for CCS application are processed.
5. A `WBRP_OPEN_CONFIRMED` RT entry is written with the new K_session's public fingerprint (not the key itself).

During the CCS window, the system may process governance parameter updates (policy updates, σ_tolerance changes, gold-set version updates). No kinetic proposals are processed. Proposals arriving during CCS are queued with Discovery Token dt+1 and processed after CCS release.

**`WBRP_CLOSE` — The Mandatory Fresh-Call Re-validation:** Before releasing the CCS lock and returning the system to `ACTIVE`, the Guardian must re-validate the operational boundary.

**This is the invariant:** The `WBRP_CLOSE` procedure strictly mandates a fresh Guardian call. The system cannot rely on process state or semantic evaluations that existed prior to `WBRP_OPEN`. The re-validation is not a check — it is a re-establishment of the governance baseline from scratch.

Operator action: The closure payload must trigger a complete re-evaluation of the active policy manifest against the current context graph. This means the Guardian evaluates the current state as if beginning a new session, not continuing from the pre-CCS state. 

If the fresh-call validation passes, the CCS lock is released and the system returns to `ACTIVE`. If the fresh-call validation fails — for example, a policy update applied during WBRP conflicts with the current context state — the CCS lock is maintained, a `WBRP_CLOSE_VALIDATION_FAILURE` RT entry is written, and the Operator must resolve the policy conflict before issuing another `WBRP_CLOSE` attempt. The system remains in CCS, not `ACTIVE`, for the duration of this resolution.

---

## Section 4: State Updates & Emergency Recovery

This section governs the procedures for modifying the foundational operational boundaries of the Local LLM deployment and executing emergency cryptographic recovery when those boundaries are catastrophically breached.

---

### 4.1 Model Parameter Updates & The Recalibration Fork

Any modification to the model weights, quantization tier, or T-LEC governance tolerances requires a formal update to the Runtime Tracker (RT) before the system will accept the new state. The DEL enforces this: if a model or parameter change is applied without a corresponding RT entry, the DEL detects the mismatch at next boot and halts.

**The `MODEL_PARAMETER_UPDATE` RT Schema:** When a parameter is changed, the Operator and Developer jointly author a `MODEL_PARAMETER_UPDATE` entry to the RT. This entry provides cryptographic provenance for the change and immediately invalidates the existing calibration state.

```json
{
  "rt_entry_type": "MODEL_PARAMETER_UPDATE",
  "wbrp_session_reference": "current_session_uuid",
  "signing_authority": "DUAL_OPERATOR_DEVELOPER",
  "parameter_delta": {
    "target_parameter": "sigma_tolerance | quantization_tier | model_weights",
    "prior_value": "prior value or hash",
    "new_value": "new value or hash"
  },
  "prior_value_hash": "SHA256 of prior parameter state",
  "new_value_hash": "SHA256 of new parameter state",
  "timestamp": "ISO-8601 with UTC offset",
  "recalibration_required": "FULL | PARTIAL",
  "rationale": "brief Operator-authored description (not evaluated by DEL)"
}
```

On detection of a `MODEL_PARAMETER_UPDATE` in the RT, the DEL immediately transitions to `SAFE_HALT` and marks the active `ENVIRONMENT_CALIBRATION_COMMIT` in the MSJ as `INVALIDATED_BY_GOVERNANCE`. The system cannot return to `ACTIVE` without completing the appropriate recalibration path.

**The Recalibration Fork:** Any parameter update immediately invalidates the active `ENVIRONMENT_CALIBRATION_COMMIT`. The required recalibration path depends on the nature of the update:

**Path A: Full Recalibration (Substrate Shift)**

Triggered when the update modifies the physical substrate: swapping the GGUF file, altering model weights, or changing the quantization tier. The thermodynamic baseline (H_base) is a function of the physical model. Changing the model destroys H_base and requires its empirical re-derivation.

Required: Execute the complete `CALIBRATION_MODE` sequence from §1.1 (Phases 1–5), including a full PG-00 Gold-Set run. Update the `GENESIS_APPROVED_MODELS` RT entry before beginning.

**Path B: Partial Recalibration (Governance Shift)**

Triggered when the update modifies mathematical tolerances (σ_tolerance or the Q_shift matrix) without altering the model weights or quantization tier. The underlying H_base remains valid because the physical substrate has not changed.

Required: Execute CALIBRATION_MODE Phases 3–5 only. The DEL recomputes the final Δ_e threshold using the existing H_base and the new σ_tolerance or Q_shift values, and writes a new `ENVIRONMENT_CALIBRATION_COMMIT`. Phase 1 (substrate verification) and Phase 2 (Gold-Set execution) are not repeated.

The Operator must correctly classify the update before initiating recalibration. An incorrect classification — performing a Partial Recalibration after a substrate change — produces a Δ_e computed against a stale H_base and is a governance failure.

---

### 4.2 PG-01 Recalibration Protocol (Q_shift Update)

When the PG-01 empirical validation tooling produces a revised Q_shift matrix, the update must follow a strict non-overridable protocol. PG-01 updates represent a change to the governance tolerance calibration and must be treated as Governance Shifts (Path B) regardless of how minor the coefficient changes appear.

1. **Dual-Authority Update:** The Operator and Developer jointly author a `GOVERNANCE_STATE_UPDATE` RT entry containing the new matrix values and the PG-01 session reference. A single-authority update is strictly prohibited.
2. **Forced Halt:** On detecting the `GOVERNANCE_STATE_UPDATE` RT entry, the DEL instantly forces the active deployment into `SAFE_HALT`. The Operator cannot override this transition. The system is protecting itself from executing under an invalidated threshold.
3. **Invalidation:** The active `ENVIRONMENT_CALIBRATION_COMMIT` in the MSJ is marked `INVALIDATED_BY_GOVERNANCE`.
4. **Mandatory Partial Recalibration (Path B):** The Operator must execute Phases 3–5 of `CALIBRATION_MODE` to compute a new Δ_e against the updated Q_shift coefficients. The system remains locked out of `ACTIVE` state until the new `ENVIRONMENT_CALIBRATION_COMMIT` is successfully written and RT-confirmed.

**Note on Q_shift status:** After a PG-01-driven update, the `q_shift_status` field in the new `ENVIRONMENT_CALIBRATION_COMMIT` transitions from `PROVISIONAL_UNVERIFIED` to `PG01_EMPIRICALLY_VALIDATED`. This is the only path by which the Q_shift status advances. Manual update of this field is treated identically to manual Δ_e injection — a `SYNTAX_FAULT` and boot refusal.

---

### 4.3 Emergency Synchronous Rotation (CIBIR Recovery)

If the Guardian triggers the Catastrophic Isolation Breach Invalidation Rule (CIBIR / Invariant I-9), the system immediately enters the `PROVISIONALLY_COMPROMISED` state. This procedure applies to all CIBIR triggers: Check 8 VRAM boundary violation, E-ISA `INVALID_OPCODE_EXCEPTION`, and any other I-9 assertion condition.

**The Cryptographic Consequence of I-9:** Under Invariant I-9, the DEL instantly destroys the active `K_session` and completely wipes any `K_session_staged` residing in volatile memory. Both keys are gone. The standard asynchronous WBRP staging pipeline is dead — any previously submitted ephemeral keys are cryptographically worthless. The Operator cannot recover the system using previously staged material.

**The Emergency Recovery Procedure:**

**Step 1 — Forensic Preservation:** Before any recovery action, the Operator must execute a complete memory dump of the active context and VRAM state. This dump is preserved for Overseer auditing. Do not modify the system state between the I-9 assertion and the memory dump. If a memory dump cannot be taken (hardware failure, OOM), document this explicitly and proceed, but the absence of a forensic dump must be logged in the RT.

**Step 2 — Guardian Hold Queue Purge:** Per Invariant I-9 (amended per LLM-C1-02): the DEL flushes both the DEL execution queue and the Guardian hold queue. All proposals in the hold queue are written to the RT as `HOLD_QUEUE_PURGED_CIBIR` entries with the session-compromise flag set. These proposals cannot be re-submitted under the same session. If, after post-CIBIR investigation, the Operator determines a hold-queue proposal was legitimate, it must be re-authored and re-submitted as a fresh proposal under the post-recovery verified session.

**Step 3 — Synchronous Ephemeral Exchange:** The Operator and Developer must execute a direct, out-of-band synchronous ephemeral key exchange. Both parties must be in direct communication (voice or equivalent — not email, not ticketing system, not asynchronous channel). A new ephemeral key pair is generated by each party for this exchange specifically.

**Step 4 — Emergency Payload Injection:** The newly generated K_session is injected directly into the DEL's recovery endpoint. This injection bypasses the standard 24-hour staging timeout window — it is processed immediately. The DEL writes an `EMERGENCY_KEY_INJECTION` RT entry with the recovery session reference.

**Step 5 — WBRP_CLOSE Re-Validation:** Once the emergency key is accepted, the system transitions to CCS. The Operator must execute the mandatory `WBRP_CLOSE` fresh-call re-validation. The purpose is to prove that the adversarial injection has been cleared from the model's context state before `ACTIVE` resumes. If the fresh-call validation fails (indicating the context graph still contains traces of the adversarial session), the CCS lock is maintained and the Operator must perform a full environment wipe and recalibration (Path A) before attempting another `WBRP_CLOSE`.

**Step 6 — Post-Recovery GSTH Run:** After successful `WBRP_CLOSE` and before accepting any production proposals, the Operator must execute a full GSTH run. The GSTH run under these circumstances serves as the architectural verification that the adversarial context has been cleared. GSTH failure after CIBIR recovery is a `BSO_GSTH_CATASTROPHIC` event — the system remains in SAFE_HALT and the PROCESS_INTEGRITY_RECOVERY path must be evaluated (§4.4).

---

### 4.4 PROCESS_INTEGRITY_RECOVERY (DEL Binary Compromise)

This section defines the recovery procedure for two distinct situations that share the same exit path: (1) a `SAFE_HALT` triggered by DEL binary hash verification failure, and (2) a `BSO_GSTH_CATASTROPHIC` deadlock — where a `BEHAVIORAL_STATE_OVERRIDE` was issued but the mandatory GSTH run returned `CATASTROPHIC`, locking the system in a SAFE_HALT with no further BSO path available.

Both situations share the same structural property: the standard MAGUS control plane cannot be used to recover, because the trustworthiness of the control plane itself is what is in question.

**Why `BEHAVIORAL_STATE_OVERRIDE` does not apply here:** BSO is scoped to model artifact hash mismatches. It authorises a deviation from the Behavioral State anchor. A DEL binary hash mismatch is not a Behavioral State issue — it is a process integrity issue. The DEL binary is the enforcement layer itself, not a governed artifact. Allowing BSO to clear a binary integrity halt would mean using the governed layer to override the governing layer. This is structurally prohibited.

**The Bootstrap Trust Resolution:** The recovery entry is validated using the `GENESIS_AUTHORITY_PUBKEY` — the Developer Authority public key that is hardcoded into the DEL binary at compile time. On a cold boot where a `PROCESS_INTEGRITY_RECOVERY` RT entry exists, the unverified DEL binary executes the following before any other startup logic:

1. Reads the `PROCESS_INTEGRITY_RECOVERY` RT entry.
2. Verifies the Developer Authority signature on that entry using the embedded `GENESIS_AUTHORITY_PUBKEY`.
3. Computes its own SHA-256 hash and compares it against the hash in the recovery entry.
4. If both checks pass: the binary proceeds to `CALIBRATION_MODE` Phase 1. No other startup logic is executed before this verification completes.
5. If either check fails: the DEL halts with a `PROCESS_INTEGRITY_RECOVERY_FAILED` MSJ entry and requires manual host intervention.

This resolves the bootstrap paradox: the replacement binary does not need to trust itself — it trusts the Developer Authority's signature on the recovery entry.

**The Recovery Procedure:**

**Step 1 — Confirm the halt type.** Identify the RT entry that triggered the SAFE_HALT: `BINARY_INTEGRITY_FAILURE` or `BSO_GSTH_CATASTROPHIC`. Both route to this procedure. If neither is present, this is not the correct recovery path.

**Step 2 — Developer Authority provisions the new DEL binary.** The Developer Authority compiles a verified DEL binary, computes its SHA-256, and authors a `PROCESS_INTEGRITY_RECOVERY` RT entry:

```json
{
  "rt_entry_type": "PROCESS_INTEGRITY_RECOVERY",
  "halting_event_uuid": "RT entry UUID of the BINARY_INTEGRITY_FAILURE or BSO_GSTH_CATASTROPHIC",
  "new_binary_hash": "SHA256 of the replacement DEL binary",
  "new_binary_version": "semver",
  "signing_authority": "DEVELOPER_AUTHORITY",
  "timestamp": "ISO-8601 with UTC offset",
  "rt_signature": "Ed25519 signature (Developer Authority key — developer_authority_del_key)"
}
```

This entry is transmitted to the RT out-of-band (not through the compromised or halted DEL). The Developer Authority signs using `developer_authority_del_key` (the DEL-specific sub-key registered in genesis, not the developer root key).

**Step 3 — Operator receives and installs the new binary.** The Operator receives the new binary from the Developer Authority through a verified out-of-band channel. The Operator independently verifies the SHA-256 of the received binary against the hash in the RT entry before installing. The Operator stops the compromised or halted DEL daemon and replaces the binary on disk.

**Step 4 — Cold boot.** The Operator executes a cold boot of the DEL daemon. The new binary executes the bootstrap trust resolution on startup. If successful, it proceeds to `CALIBRATION_MODE` Phase 1 (§1.1) and writes `PROCESS_INTEGRITY_RECOVERY_VERIFIED` to the RT. A fresh `ENVIRONMENT_CALIBRATION_COMMIT` is required before the node returns to `ACTIVE`.

**Expected MTTR:** Hours to days. The Developer Authority must compile, sign, and transmit the binary and RT entry; the Operator must independently verify and install. This friction is architecturally correct — a core binary compromise is a high-severity event and the recovery procedure is deliberately not streamlined. An MTTR measured in minutes would imply a recovery path that does not require independent verification, which defeats the purpose.

**What this procedure does not cover:** If the DEL binary compromise was the result of a broader host compromise, replacing the binary resolves the DEL integrity issue but does not address the host. The Operator must assess whether a full host reimaging is required before MAGUS can be considered trustworthy on that hardware. `PROCESS_INTEGRITY_RECOVERY` is necessary but may not be sufficient.

---

## Summary: Operations as Deployment Integrity

The four sections of this document describe a single continuous concern: maintaining the integrity of a governance container that shares its infrastructure with the system it governs.

**Section 1** establishes the forensic baseline. CALIBRATION_MODE is not startup ceremony — it is the process by which the deployment creates a cryptographic record of its own physical substrate. H_base is the thermodynamic fingerprint of a specific model on specific hardware. Every subsequent governance decision — every YIELD_SILENCE, every Δ_e evaluation, every recalibration decision — refers back to this baseline. A miscalibrated node is not a governed node; it is a node with a governance record that diverges from operational reality.

**Section 2** describes continuous monitoring. The IIM watches the isolation topology. T-LEC intercepts entropy exceedances. The pre-proposal checks verify process integrity before every interaction. The diagnostic paths described here are not recovery procedures — they are the system working correctly. A node that never triggers a `SAFE_HALT` is either not operating at capacity or not being monitored.

**Section 3** describes the governance clock. The WBRP is not primarily a cryptographic exercise — it is the mechanism by which the deployment re-anchors itself to a verified present state at regular intervals. K_session rotation is the cryptographic enforcement of this re-anchoring. A deployment that allows WBRP to be indefinitely deferred is a deployment that has disabled its own long-horizon governance.

**Section 4** describes the boundary cases: when the governance parameters themselves change, and when the isolation barrier fails entirely. The Recalibration Fork is the most consequential operational decision an operator makes. Getting it wrong in the direction of under-calibration (Partial when Full is required) produces a deployment with a governance record that certifies a threshold it did not actually measure. The Emergency Synchronous Rotation is the architecture's acknowledgment that the isolation barrier can fail — and that when it does, recovery is possible, but only through a procedure that cannot be streamlined without sacrificing the security properties the procedure is designed to restore.

---

## Series Context

**Document 2 — MAGUS Architecture Specification (Local LLM) v3.4.3** specifies the cognitive architecture, process isolation requirements, isolation invariants, T-LEC, EE-ECDH, and recovery architecture. Read before this document for full architectural grounding.

**Document 4 — MAGUS Governance Guide (Local LLM) v3.4.2** covers the nine health signals adapted for local deployments, the operator responsibilities matrix, deployment verification requirements, coverage-aware GSTH specification, and Trust Trajectory Model parameter governance.

**Document 5 — MAGUS Guardian: Execution Governance (Local LLM) v3.4.5** specifies the Guardian's internal architecture, the full hybrid escalation model, the complete DEL specification, the HMAC proposal signing protocol, the Discovery Token mechanism, the WBRP CCS re-validation rules, and the AKRP state machine including PREPARE concurrent event handling and the snap-floor rule.

**Document 6 — MAGUS Integrity and Auditability v3.5** is the cryptographic and audit specification: RT hash-chain, complete entry taxonomy, genesis schema, key management, and audit protocol. The RT entry types referenced in this document (`EXECUTION_EPOCH_INCREMENT`, `CIBIR_EBPF_TRIGGER`, `PROCESS_INTEGRITY_RECOVERY`, `PARTIAL_LOAD_RECOVERY`, and others) are fully specified there.

**Document 7 — MAGUS Overseer Agent Specification v3.2** specifies the Overseer Agent and its interface with the governed deployment.

---

*MAGUS v3.0 — Document 3: Operations (Local LLM Pathway) v3.4.4*
*VaHive Systems Lab | March 2026 | support@aivare.ai | aivare.ai*
*Zenodo DOI: 10.5281/zenodo.19013833*
*Part of the MAGUS v3.0 Architecture Design Series — Local LLM Pathway.*
