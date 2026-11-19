markdown
# Hard-Cap Compute Ledger

**An immutable open-source schema and cryptographic verification protocol for frontier AI training runs.**

> **Status:** Early Architecture & Specification (`v0.1` in active planning)

---

## Overview

Effective compute governance requires verifiable, tamper-evident records of hardware utilization without exposing proprietary model weights or sensitive training data. 

**Hard-Cap Compute Ledger** provides a standardized relational schema, high-throughput ingestion pipeline, and Merkle-tree audit log designed for **independent compute auditors**, **third-party evaluation organizations**, and **regulatory sandboxes**.

---

## Core Architecture

The protocol separates low-level hardware telemetry collection from high-integrity audit orchestration:

```text
[GPU Cluster / TPM Attestation] 
       │  (Signed Hardware Telemetry)
       ▼
[Python Ingestion Endpoints] ──► [Redis High-Throughput Buffer]
                                        │
                                        ▼
                       [Laravel Verification & RBAC Engine]
                                        │
                       ┌────────────────┴────────────────┐
                       ▼                                 ▼
         [MySQL Relational Schema]         [Merkle Hash Audit Chain]
       (hardware_nodes, audit_logs)        (Tamper-Evident Proofs)
```

### Key Components
1. **Python Telemetry Ingestion Layer:** Lightweight endpoints that validate cryptographic hardware signatures (e.g., GPU/TPM attestations) before queuing telemetry events.
2. **Redis High-Throughput Buffer:** Decouples bursty cluster telemetry streams from persistent storage to prevent dropped packets during peak training runs.
3. **Laravel Verification & RBAC Engine:** Manages role-based access control, relational data mapping, and automated discrepancy flagging.
4. **Immutable Audit Storage (MySQL + Merkle Hashing):** Stores normalized `hardware_nodes` and `audit_logs` records linked via Merkle-style cryptographic hashes so historical compute logs cannot be silently altered or deleted.

---

## Threat Model & Verification Principles

* **Tamper-Evidence:** Once ingested, log batches are hashed into an append-only Merkle chain; any retroactive modification invalidates downstream root hashes.
* **Privacy-by-Design:** The schema captures verifiable compute metrics (FLOP estimates, active node counts, runtime windows) while strictly excluding model weights, gradients, and training datasets.
* **Out-of-Band Cross-Referencing:** Schema fields are structured to support reconciliation against independent physical signals such as facility power consumption and ISP network telemetry.

---

## 6-Month Development Roadmap (`v0.1`)

* **Month 1:** Finalize core relational schema (`hardware_nodes`, `training_runs`, `audit_logs`) and cryptographic signature verification specs.
* **Month 2:** Build Python ingestion endpoints and Redis buffering pipeline.
* **Month 3:** Implement Laravel verification service, RBAC permissions, and Merkle-tree hashing engine (`v0.1` Alpha).
* **Month 4:** Develop synthetic multi-node GPU cluster telemetry generator for reproducible testing.
* **Month 5:** Execute high-concurrency ingestion benchmarks and publish audit verification documentation.
* **Month 6:** Release `v0.1` open-source package with deployment guides for third-party auditors and governance researchers.

---
## Ingestion Security & Cryptographic Gatekeeping

To prevent queue poisoning and spoofed telemetry, the Python FastAPI ingestion layer acts as a stateless cryptographic gatekeeper **before** writing any data to Redis:

1. **Freshness & Replay Protection:** Validates `X-Timestamp` within a strict ±30-second drift window and enforces single-use `X-Nonce` tracking.
2. **Hardware Identity Binding:** Resolves `X-Node-ID` against registered public keys bound to hardware TPM / GPU attestation certificates.
3. **Deterministic Canonical Hashing:** Computes a `SHA-256` digest over `timestamp + nonce + canonical_json` so payload formatting cannot be manipulated.
4. **Pre-Queue `Ed25519` Verification:** Verifies `X-Signature` in memory; unauthenticated or tampered packets fail immediately (`401`/`403`) without touching Redis memory.
5. **Verified Stream Handoff:** Only cryptographically verified envelopes—preserving the original signature and `payload_hash`—are appended to the `telemetry:verified` Redis Stream for downstream Laravel Merkle-tree anchoring.

## License

This project will be released under the [Apache License 2.0](https://apache.org).
