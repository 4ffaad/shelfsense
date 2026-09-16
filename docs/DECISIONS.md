# Architecture decisions

## ADR-001 — Start with a small vertical slice

**Decision:** Begin with one shelf image and 3–5 known SKUs.

**Reason:** A small fixture set makes the complete path observable: upload, inference, counting, persistence, and evaluation. Expanding before the baseline works would hide failures behind infrastructure.

## ADR-002 — Keep vision and inventory boundaries separate

**Decision:** Detector and identifier adapters produce candidate evidence. The inventory domain validates and aggregates it.

**Reason:** Model output is probabilistic; stock events must be deterministic, auditable, and testable without a model.

## ADR-003 — Add RAG after structured facts work

**Decision:** Implement structured retrieval before embeddings and generation.

**Reason:** We need a non-LLM baseline for correctness. Otherwise a fluent answer can conceal incorrect retrieval or missing data.

## ADR-004 — Use OMLX as a provider, not a domain dependency

**Decision:** Call OMLX through HTTP-backed embedding, reranking, generation, and optional VLM/OCR adapters.

**Reason:** OMLX is a good local Apple Silicon serving layer, but model-serving choices can change. The application should be able to compare providers without changing domain rules.

## ADR-005 — Allow a direct-VLM prototype, require a hybrid path for trust

**Decision:** We may prototype whole-image VLM inventory extraction, but the target architecture uses detector boxes plus OMLX-assisted identification/evidence.

**Reason:** Whole-image VLM output is useful for learning and rapid experiments. Dense repeated retail products need localization, per-object diagnostics, and deterministic aggregation for a trustworthy inventory system.

## ADR-006 — Prefer abstention to unsupported certainty

**Decision:** Unknown products, missing documents, unsupported questions, and weak evidence produce `unknown`, warnings, or an explicit abstention.

**Reason:** A wrong stock count or fabricated explanation is more harmful than an incomplete result that can be reviewed.

## Decision flow

```mermaid
flowchart TD
    Need["New capability"] --> Evidence["What evidence does it need?"]
    Evidence --> Vision{ "Image localization?" }
    Vision -- "Yes" --> Detector["Detector / CV adapter"]
    Vision -- "No" --> Knowledge{ "Stored context lookup?" }
    Knowledge -- "Yes" --> Retrieval["Structured or hybrid retrieval"]
    Knowledge -- "No" --> Generation["Generation / UI layer"]
    Detector --> Validate["Validate candidate output"]
    Retrieval --> Validate
    Generation --> Validate
    Validate --> Domain["Deterministic domain rules"]
```
