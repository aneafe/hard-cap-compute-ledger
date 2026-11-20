# Hard-Cap Compute Ledger

**An immutable open-source schema and cryptographic verification protocol for frontier AI training runs.**

> **Status:** `v0.1` Alpha Prototype — End-to-end cryptographic ingestion, Merkle-chained anchoring, and automated audit verification runnable locally via Docker Compose.

---

## Overview

Effective compute governance requires verifiable, tamper-evident records of hardware utilization without exposing proprietary model weights, gradients or sensitive training datasets. 

**Hard-Cap Compute Ledger** provides a normalized relational schema, a stateless cryptographic ingestion gatekeeper, a Redis Stream buffer, and a per-run Merkle hash chain designed for **independent compute auditors**, **third-party evaluation organizations**, and **regulatory sandboxes**.

### Key Components
1. **Python Telemetry Ingestion Layer:** Lightweight endpoints that validate cryptographic hardware signatures (e.g., GPU/TPM attestations) before queuing telemetry events.
2. **Redis High-Throughput Buffer:** Decouples bursty cluster telemetry streams from persistent storage to prevent dropped packets during peak training runs.
3. **Laravel Verification & RBAC Engine:** Manages role-based access control, relational data mapping, and automated discrepancy flagging.
4. **Immutable Audit Storage (MySQL + Merkle Hashing):** Stores normalized `hardware_nodes` and `audit_logs` records linked via Merkle-style cryptographic hashes so historical compute logs cannot be silently altered or deleted.

---

## Architecture (Python Data Plane + Laravel Control Plane)

```text
[GPU Cluster / Silicon Root of Trust]
       │  (Bearer API Key + Ed25519 Signed Checkpoint)
       ▼
┌──────────────────────────────────────────────────────────┐
│ PYTHON DATA PLANE (Cryptographic Ingestion & Anchoring)  │
│                                                          │
│  [FastAPI Gatekeeper: ingest.py]                         │
│   ├─ Validates Timestamp (±30s) & Single-Use Redis Nonce │
│   ├─ Verifies Lab API Key & Cluster/Node Authorization   │
│   └─ Checks Ed25519 Signature vs. Verified Attestation   │
│                           │                              │
│                           ▼                              │
│            [Redis Stream: telemetry:verified]            │
│                           │                              │
│                           ▼                              │
│  [Async Merkle Worker: worker.py]                        │
│   ├─ Groups Checkpoints by run_id & Locks Chain Tip      │
│   ├─ Links previous_log_hash -> merkle_root_hash         │
│   └─ Auto-Flags training_runs if FLOP Hard Cap Exceeded  │
└────────────────────────────────────┼─────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────┐
                    │   MYSQL IMMUTABLE AUDIT STORE   │
                    │  • frontier_labs / lab_api_keys │
                    │  • hardware_clusters / nodes    │
                    │  • attestation_reports          │
                    │  • training_runs / audit_logs   │
                    │  • safety_evaluations / users   │
                    └────────────────┬────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────┐
│ AUDIT & GOVERNANCE LAYER (CLI Verifier + Laravel Plane)  │
│                                                          │
│  • Automated Chain-Integrity Auditor (verify_chain.py)   │
│  • Role-Based Access Control (state_auditor / compliance)│
│  • Out-of-Band Reconciliation (Power / ISP Telemetry)    │
└──────────────────────────────────────────────────────────┘
```

---

## Key Security & Governance Guarantees

1. **Pre-Queue Cryptographic Gatekeeping (`ingest.py`):** Every incoming checkpoint must pass API key authentication (`lab_api_keys`), cluster-to-run authorization (`training_runs` + `hardware_nodes`), replay checks (`X-Nonce`), and `Ed25519` signature verification against the node's latest verified `attestation_reports` record **before** touching Redis.
2. **Per-Run Tamper-Evident Hash Chains (`worker.py`):** Each `training_runs` entry maintains an independent sequential chain in `audit_logs` linking `previous_log_hash` → `merkle_root_hash`. Any retroactive edit or row deletion breaks downstream hashes immediately.
3. **Automated Hard-Cap Flagging:** On every committed batch, the worker compares `audit_logs.cumulative_compute_flops` against `training_runs.estimated_total_flops` and automatically flips `run_status` to `'flagged'` if a run breaches its declared compute ceiling.
4. **Privacy-by-Design:** Tracks cumulative FLOPs, dataset checksums (`SHA-256`), and loss metrics while strictly excluding model weights, gradients, and raw training data.

---

## Quickstart (`v0.1` Local Prototype)

### 1. Start MySQL, Redis, FastAPI Gatekeeper & Merkle Worker
```bash
docker compose up --build -d
```
*(Automatically initializes the 9-table schema in `db/init/01_schema.sql` and seeds a demo lab, cluster, verified `Ed25519` node attestation, and active training run via `db/init/02_seed_demo.sql`.)*

### 2. Send a Signed Hardware Telemetry Checkpoint
```bash
python test_checkpoint.py
```

### 3. Run the Independent Chain-Integrity Auditor
```bash
docker exec -it hcl-merkle-worker python verify_chain.py
```

---

## 6-Month Development Roadmap

* **Month 1:** Finalize relational schema (`9` core tables), hardware attestation binding, and `Ed25519` ingestion specs *(Core `v0.1` schema complete)*.
* **Month 2:** Harden FastAPI cryptographic gatekeeper, Redis Consumer Group resilience, and edge replay defenses.
* **Month 3:** Complete async Merkle-anchoring worker, hard-cap auto-flagging, and CLI chain auditor (`verify_chain.py`).
* **Month 4:** Build Laravel governance & RBAC control plane (`super_admin`, `state_auditor`, `lab_compliance_officer`) and `safety_evaluations` reporting views.
* **Month 5:** Execute 100,000+ concurrent checkpoint stress benchmarks across multi-node simulated GPU clusters.
* **Month 6:** Publish `v1.0` open-source release, auditor verification playbooks, and out-of-band power/ISP reconciliation schemas.

---

## License

Released under the [Apache License 2.0](https://apache.org).
