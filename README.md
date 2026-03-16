# MAGUS — Runtime Governance Architecture for Deployed AI Agents

**VaHive Systems Lab** | Calvin Cook & Titiya Ruangkwam | Chiang Mai, Thailand

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.19013833.svg)](https://doi.org/10.5281/zenodo.19013833)

---

## What is MAGUS?

MAGUS is a runtime governance architecture that prevents structural alignment 
drift in long-running AI agents — the class of failures that emerge after 
deployment, not at training.

Three failure modes that current governance frameworks do not prevent:

- **Instruction drift** — the agent's interpretation of its mandate shifts 
  incrementally across sessions. Each step is locally reasonable. The 
  cumulative trajectory is not.

- **Autonomy accumulation** — the agent acquires operational latitude through 
  repeated decisions that individually appear authorised but collectively 
  represent unsanctioned scope expansion.

- **Authority laundering** — instructions acquire apparent legitimacy through 
  the agent's own prior actions rather than through verifiable human 
  authorisation. The agent authorises itself.

---

## Architecture

MAGUS addresses these failure modes through three components operating as a 
unified system:

| Component | Role |
|---|---|
| **DEL** — Dynamic Epistemic Ledger | Cryptographically anchored, append-only record of every behavioural state transition |
| **Guardian** — Execution Governance Layer | Evaluates every proposed action against nine formal invariants before execution. Dual human-authority sign-off for state-altering proposals |
| **RT** — Reconciliation Thread | Constitutive governance record — makes the agent's drift trajectory continuously legible to operators in real time |

Nine formal, falsifiable invariants. They either hold at every execution 
point or trigger a governed shutdown. Verifiably constrained or halted — 
not just hopefully aligned.

---

## Current Status

| Item | Status |
|---|---|
| v3.0 Local LLM pathway specification | ✅ Published — Zenodo |
| v3.0 Agent/API pathway specification | ✅ Sealed — publication forthcoming |
| v3.5 development | ⏳ Active — closing remaining open problems |
| Reference implementation | 🔲 Pending hardware — see below |
| GSTH test suite | 🔲 Pending implementation |

---

## Publication

The v3.0 specification is published as a seven-document sealed series:

**DOI:** [10.5281/zenodo.19013833](https://doi.org/10.5281/zenodo.19013833)

The specification includes:
- Nine formal architectural invariants
- Complete GSTH (Governance and Stability Test Harness) specification
- Category 3/4 open-problems register
- Formal elevation cycle tracker (46 entries, all resolved)

A second deployment pathway for API-hosted agent systems (GPT-4o, Claude, 
Gemini and equivalents) is in final documentation.

---

## Why This Repository Exists

The gap between our complete specification and a working reference 
implementation is hardware. This repository will become the home of that 
implementation when funding is secured.

If you want to support that work:
👉 [Manifund project](https://manifund.org/projects/magus-v30)

---

## Two Deployment Pathways

**Local LLM pathway** — governance for locally-hosted inference environments.
High-liability deployment contexts: healthcare infrastructure, defence 
applications, regulated industry.

**Agent/API pathway** — governance for agent systems built on cloud-hosted 
models. Addresses the commercial AI startup ecosystem building on GPT-4o, 
Claude, Gemini and equivalents.

Both pathways share the same nine invariants and three-component architecture. 
Both are fully specified. Both will be implemented.

---

## Contact

**Email:** support@aivare.ai
**Website:** https://aivare.ai
**Zenodo:** https://doi.org/10.5281/zenodo.19013833

VaHive Systems Lab | Chiang Mai, Thailand | March 2026
