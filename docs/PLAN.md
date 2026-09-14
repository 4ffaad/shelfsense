# ShelfSense plan

## 1. Product goal

ShelfSense is a local-first retail shelf inventory platform. A user submits a shelf image, the system detects visible products, identifies known SKUs when evidence is sufficient, aggregates counts, stores an observation, and emits actionable events such as low stock, out of stock, unknown product, or misplaced product.

The system is an inventory-observation tool, not a source of truth for warehouse stock. It reports what is visible on a shelf and preserves uncertainty.

## 2. First release boundary

The first useful vertical slice supports:

- one uploaded image;
- one configured shelf;
- 3–5 known SKUs;
- local inference on the M3 using MPS or CPU;
- detection results with bounding boxes and confidence;
- SKU counts plus `unknown` detections;
- SQLite persistence;
- a documented evaluation command and test fixtures;
- a small FastAPI endpoint and machine-readable response.

Not in the first slice: live cameras, multi-store tenancy, user accounts, cloud deployment, automatic model training, OCR, barcodes, embeddings, Next.js UI, or alerts delivered to external systems.

## 3. Architecture

```text
API route
  -> image ingestion/validation
  -> vision pipeline
       -> detector
       -> product identifier
       -> post-processing / abstention
  -> inventory aggregation
  -> observation repository
  -> response + event records
```

The domain must not depend on FastAPI, OpenCV, or a specific model vendor. Vision implementations depend on a small detector/identifier protocol. The API calls an application service. Persistence is behind repositories so inference and domain rules can be tested without a database.

## 4. Technology decisions

### Runtime and tooling

- Python 3.12: strong CV/ML ecosystem and compatible with the target hardware.
- `uv`: reproducible environment and lockfile.
- Ruff: formatting and linting with one fast tool.
- Pytest: unit and integration tests.
- mypy: type checking at stable boundaries.

### Backend

- FastAPI: typed HTTP API and generated OpenAPI.
- Pydantic v2: request/response/settings validation.
- SQLAlchemy 2 + Alembic: explicit persistence and migrations.
- SQLite initially; PostgreSQL when deployment or concurrent workloads justify it.

### Vision

- OpenCV/Pillow: decode, validate, resize, annotate, and inspect images.
- PyTorch: model runtime, with explicit MPS/CPU device selection on Apple Silicon.
- Detector and identifier behind interfaces. The initial adapter may use a small YOLO-family detector, but model choice is an experiment recorded with weights, license, config, and metrics—not a hidden architecture dependency.
- OCR, barcode, and embedding matching are separate later adapters. They must not silently override a high-confidence barcode or catalog decision.

### Frontend and deployment

- Next.js + TypeScript only after the API contract stabilizes.
- Docker Compose for PostgreSQL and local service parity when needed.
- No cloud provider is selected for v0; local processing is the privacy and cost default.

## 5. Core domain objects

- `Product`: SKU, name, barcode/aliases, expected shelf capacity, active status.
- `Shelf`: stable identifier, store/area metadata, allowed SKUs, capacity rules.
- `ShelfImage`: immutable source metadata, dimensions, checksum, capture time, source type.
- `Detection`: box, detector confidence, crop reference, class/model metadata.
- `ProductMatch`: SKU or `unknown`, method, confidence, evidence, abstention reason.
- `ShelfObservation`: image, pipeline/model version, detections, counts, latency, status.
- `InventoryEvent`: event type, SKU, shelf, observed count, expected capacity, observation ID.

Every observation is append-only. Reprocessing creates a new observation linked to the same image rather than overwriting history.

## 6. API outline

- `GET /healthz`: process health.
- `GET /readyz`: dependency/model readiness.
- `POST /v1/observations`: upload an image and request inference.
- `GET /v1/observations/{id}`: retrieve a stored observation.
- `GET /v1/shelves/{shelf_id}/inventory`: latest visible counts and event state.
- `GET /v1/products`: catalog listing for configuration and later UI use.

The upload endpoint enforces content type, maximum bytes, image dimensions, and decode success. Responses include pipeline version, model version, confidence values, unknown count, and warnings.

## 7. Delivery phases

### Phase 0 — foundation

Create package boundaries, settings, structured errors, health route, database migration setup, CI-quality commands, and test fixtures.

Exit: service starts; health/readiness behavior is tested; no model is required for unit tests.

### Phase 1 — image-to-detections baseline

Implement image validation, detector interface, one local detector adapter, annotated output, and deterministic inference metadata.

Exit: a fixture image produces valid boxes; malformed/oversized images fail safely; latency and device are recorded.

### Phase 2 — known-SKU counting

Add catalog, product identification for the selected 3–5 SKUs, abstention to `unknown`, count aggregation, and observation persistence.

Exit: per-SKU count error and identification precision/recall are measured on a held-out set.

### Phase 3 — inventory rules and API

Add expected capacity, low-stock/out-of-stock/misplaced rules, event persistence, and the complete observation API.

Exit: rule tests cover boundaries and API integration tests cover upload-to-response.

### Phase 4 — evaluation and diagnostics

Add dataset manifest, checksum/provenance records, confusion/error reports, per-SKU and condition slices, and annotated diagnostic images.

Exit: baseline report is reproducible and documents failure modes, coverage, and limitations.

### Phase 5 — dashboard

Build a small Next.js client for upload, observation results, shelf inventory, and event history.

Exit: UI uses only the versioned API; it does not duplicate inventory rules.

### Phase 6 — recognition expansion

Add barcode/OCR/embedding adapters behind explicit evidence-ranking rules, then video tracking and multi-shelf workflows only if measurements justify them.

## 8. Dataset and evaluation plan

Start with a small, licensed dataset containing the exact 3–5 target SKUs and representative shelf conditions. Capture a separate held-out set across lighting, angle, occlusion, spacing, packaging orientation, and empty slots. Split by physical scene/capture session—not random near-duplicate images.

Track:

- detection precision, recall, and mAP/IoU;
- identification precision/recall per SKU and unknown coverage;
- absolute count error and inventory event precision;
- latency, memory, device, image size, and failure rate;
- slices for occlusion, blur, lighting, viewpoint, and shelf density.

A model may abstain. A high aggregate score is insufficient if one SKU or condition fails.

## 9. Risks and decisions to revisit

- Product packaging changes and visually similar SKUs may require catalog updates.
- Occlusion means visible count is not always shelf stock.
- Public dataset licensing and model-weight licensing must be recorded before redistribution.
- MPS behavior and memory must be measured on representative images; CUDA assumptions are prohibited.
- Thresholds are configuration, not hard-coded constants, and must be evaluated with the model revision.
- PostgreSQL, object storage, authentication, and background jobs are deferred until actual concurrency/deployment needs exist.

## 10. Definition of done for v0.1

A clean checkout can install dependencies, run tests and linting, start the API, submit a fixture image, receive a schema-valid observation, persist it in SQLite, and reproduce an evaluation report with model/data/config identifiers. Known limitations and failed cases are documented.
