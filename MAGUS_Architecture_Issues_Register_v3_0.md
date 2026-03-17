# MAGUS v3.0 — Architecture Issues Register
### Local LLM Pathway | Formal Issues Record

**Version:** 3.0  
**Date:** March 2026  
**Series:** MAGUS v3.0 Architecture — Local LLM Pathway  
**Status:** Sealed — all v3.0 solvable items resolved  

VaHive Systems Lab | aivare.ai | support@aivare.ai  
Zenodo DOI: 10.5281/zenodo.19013833

---

## Purpose

This register records every architectural issue, gap, and open problem identified during the design and formalisation of the MAGUS v3.0 Local LLM pathway. It is the authoritative record of what was found, how it was resolved, and what remains open.

Issues are classified by category:

| Category | Meaning |
|---|---|
| **Cat 1** | Resolved — solution fully specified and implemented in the sealed series |
| **Cat 2** | Direction specified — architectural direction committed; empirical calibration or implementation detail deferred to deployment |
| **Cat 3** | Open hard problem — no currently adequate solution within v3.0 constraints; flagged for v3.5 investigation |
| **Cat 4** | Deferred — out of scope for v3.0; scope established for future version |

Issue identifiers use the following prefix convention:
- **OA-C** — cross-pathway architectural issues
- **LLM-C** — Local LLM pathway-specific issues

v3.5 development is underway. Progress against Category 3 and 4 items will be reported in the v3.5 publication.

---

## Category 1 — Resolved

All items in this category are fully specified and implemented in the v3.0 sealed document series. Document references indicate where the resolution lives.

| ID | Title | Resolution | Document |
|---|---|---|---|
| OA-C1-01 | Discovery Token anti-replay mechanism | Discovery Token with session-scoped nonce partitioning; Ed25519 signed; RT-anchored | Doc 5 §4.2 |
| OA-C1-03 | Model-family concentration risk (Guardian/Worker same family) | `single_family_scrutiny_uplift` — mandatory threshold adjustment when Guardian and Worker share model family | Doc 5 §3.2 |
| OA-C1-04 | Local filesystem write risk weighting | `local_filesystem_write` risk multiplier ×1.25 formalised in DEL | Doc 5 §4.3 |
| OA-C1-06 | T-LEC self-reported confidence attack surface | `state_confidence` field permanently removed; Shannon entropy measurement replaces self-reported confidence entirely | Doc 5 §4.2, AMD-001–005 |
| OA-C1-07 | Behavioural state guard | Behavioural state guard formalised as immutable DEL rule | Doc 5 §5.2 |
| OA-C1-08 | Hard floor R_abs enforcement | R_abs ≥ 0.95 formalised as DEL rule — not configurable below floor | Doc 5 §5.2 |
| OA-C1-09 | Conflict resolution priority stack | Deterministic priority stack implemented as DEL logic — not operator-configurable at runtime | Doc 5 §5.2 |
| OA-C1-10 | Guardian policy hash verification at boot | Policy hash verified against RT anchor at every boot; mismatch → SAFE_HALT | Doc 5 §10.1 |
| OA-C1-11 | HMAC proposal signing specification | K_session-based HMAC proposal signing; full schema specified | Doc 5 §4.2, Doc 6 §4.3 |
| OA-C2-01 | R_abs decay rules (chronological decay and thermodynamic gating) | Three-layer decay spec: state suspension layer, thermodynamic gating layer, 0.20/WBRP chronological cap | Doc 5 §3.6 |
| OA-C2-02 | DEL-side S_final computation | Guardian outputs raw integers; DEL derives weights via HKDF(K_session) at WBRP_OPEN; DEL computes S_final | Doc 5 §4.4a |
| OA-C2-09 | R accumulation formula internal inconsistency | Asymptotic damping formula with logistic escalation multiplier M(R); replaces additive sum | Doc 5 §3.3 |
| OA-C2-10 | Dynamic λ rejection and boundary conditions | Static λ retained; four boundary conditions specified; PG-01 joint validation requirement established | Doc 4 §3.1, Appendix A.1 |
| LLM-C1-06 | Hybrid IIM-eBPF Atomic Kill Switch | Full eBPF latch architecture specified: sk_filter, ISOLATION_STATE_MAP, LSM hooks, user-space double-check; kernel lockdown prerequisite stated | Doc 5 §4.0 |
| LLM-C2-02 Gap 1 | GGUF hash verification chain | fanotify hash verification on DEL-held FD before SCM_RIGHTS delivery | Doc 2 §6.5.1b |
| LLM-C2-02 Gap 2 | GGUF symlink attack surface | O_NOFOLLOW on DEL GGUF open; symlink traversal physically impossible | Doc 2 §6.5.1b |
| LLM-C2-02 Gap 3 | DEL-mediated GGUF file handle acquisition | Five-step DEL-mediated FD acquisition; model process prohibited from direct openat() on GGUF paths via seccomp | Doc 2 §6.5.1b, Doc 5 §10.2 |
| LLM-C2-03 | AKRP PREPARE state machine undefined behaviour | PREPARE_SUSPENDED state added; BSO interrupt path specified; 60-second snap-floor; BSO-per-PREPARE limit; T_active ceiling | Doc 5 §2.3.4 |
| LLM-C2-04 | Bootstrap trust resolution | Bootstrap Trust Resolution as check 0 in cryptographic boot sequence; PROCESS_INTEGRITY_RECOVERY detection; GENESIS_AUTHORITY_PUBKEY verification | Doc 5 §10.1 |
| LLM-C2-06 | Execution Epoch crash-replay vector | Execution Epoch partitioning throughout; `execution_epoch` added to Discovery Token HMAC, boot sequence, and AKRP crash-recovery transitions | Doc 5 §4.2, §10.1 |
| LLM-C2-07 | K_session distribution security | Staged Authenticated EE-ECDH; full key exchange protocol specified | Doc 5 §2.3, LLM-C2-07 Resolution v3.0 |

---

## Category 2 — Direction Specified

These items have a committed architectural direction. Empirical calibration parameters are deferred to deployment-phase validation via the PG-01 and PG-02 calibration programs.

| ID | Title | Direction | Calibration Dependency | Document |
|---|---|---|---|---|
| OA-C2-04 | Genesis Anchor divergence threshold calibration | Thresholds architecturally specified (0.05 per-WBRP notification; 3-cycle floor shift; 5-cycle halt; 0.85 cumulative death-line) | PG-01 must validate against representative task profiles | Doc 4 §3.3 |
| OA-C2-06 | PG-01 ±5% weight variance empirical measurement | Variance bounds locked at ±5%; PG-01 to measure practical decision effect at margin vs. no-effect cases | PG-01 required | Doc 4 Appendix A.1 |
| LLM-C2-04 | Binary attestation limitation | BINARY_ATTESTATION_SOFTWARE_ONLY documented; TPM_MEASURED path specified; limitation acknowledged | Hardware TPM recommended for regulated deployments | Doc 5 §10.1 |
| LLM-C5-03 | Kernel lockdown prerequisites | Full prerequisite list specified: `security=lockdown` integrity mode, signed kernel modules, disabled /dev/mem, /dev/kmem, kexec | Operator provisioning responsibility | Doc 5 §4.0 |

---

## Category 3 — Open Hard Problems

These items have no currently adequate solution within v3.0 architectural constraints. They are published openly because falsifiable identification of unsolved problems is a stronger credibility signal than a specification that claims completeness it does not have.

v3.5 development is actively investigating these items.

| ID | Title | Nature of the Problem | Status |
|---|---|---|---|
| OA-C3-01 / OA-C2-10 | Dynamic λ revisitation | Static λ is correct for v3.0. If PG-01 production data reveals that no single static λ satisfies all four OA-C2-10 boundary conditions across the full deployment task profile range, v4.0 may require a context-aware λ. The direction-inversion failure mode of kinematic and PID coupling approaches would need to be solved first. No solution pathway specified until PG-01 data exists. | Open — awaiting PG-01 data |
| OA-C3-02 / LLM-C3-02 | Behavioural observation as operator intent proxy | The Governance Health Monitor detects operator approval fatigue and drift patterns. But observed behaviour (latency, approval rate, clustering) is an imperfect proxy for intent. A sophisticated insider could deliberately mimic healthy governance patterns while conducting an extended low-rate manipulation campaign that falls within all monitored thresholds. The architecture detects statistical anomalies; it cannot detect intent-consistent deception by a patient adversary. A three-party governance auditor model partially addresses this by making collusion detectable to an external party, at the cost of three-party key management complexity. No complete solution within the two-party MAGUS model exists. | Open — partial mitigation via governance auditor extension (Doc 4 §4.3) |

---

## Category 4 — Deferred

These items are out of scope for v3.0. Scope for a future version has been established.

| ID | Title | Deferral Rationale | Target Version |
|---|---|---|---|
| LLM-C4-01 | Full formal verification of process isolation correctness | OS-level process isolation is specified and GSTH-tested in v3.0. Formal mathematical proof of isolation correctness under all adversarial kernel conditions requires a formal verification toolchain (e.g., Z3 SMT integration) not yet integrated into the MAGUS build pipeline. | v4.0 |

---

## PG-01 and PG-02 Calibration Programs

Several Category 2 items depend on empirical baseline data that cannot be theoretically derived. Two calibration programs are defined:

**PG-01 — Governance Parameter Validation:**
Establishes empirical baselines for R_abs thresholds, λ decay constants, Guardian risk multipliers, and Genesis Anchor divergence thresholds. Must be run against representative deployment task profiles before production certification. Joint validation of OA-C2-01 and OA-C2-10 is mandatory — these parameters cannot be tuned independently.

**PG-02 — Operator Telemetry Baseline:**
Establishes empirical baselines for operator governance behaviour metrics. Required before the Governance Health Monitor thresholds can be production-certified for a given deployment context.

Full specifications for both programs are in Doc 4 (§4.2 for PG-02) and Doc 4 Appendix A.1 (for PG-01).

---

## v3.5 Development Status

The MAGUS v3.5 development workstream is actively addressing the Category 3 open problems identified above, as well as architectural extensions to the v3.0 baseline. A v3.5 specification publication is in preparation.

No v3.5 architectural content is included in this register. This document reflects the v3.0 sealed series state only.

---

*MAGUS v3.0 — Architecture Issues Register v3.0*  
*VaHive Systems Lab | March 2026 | support@aivare.ai | aivare.ai*  
*Zenodo DOI: 10.5281/zenodo.19013833*  
*Part of the MAGUS v3.0 Architecture Design Series — Local LLM Pathway.*
