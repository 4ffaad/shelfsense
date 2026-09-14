# ShelfSense

Local-first computer-vision inventory monitoring for retail shelves.

ShelfSense turns a shelf image into structured inventory observations:

```text
image -> detection -> product identification -> SKU counts -> inventory events -> dashboard
```

The first milestone is deliberately narrow: one shelf image, 3–5 known SKUs, reproducible local inference, and an API that returns detections and counts. Training, OCR, barcode reading, embeddings, video tracking, and a web dashboard follow only after that baseline is measured.

## Project status

Repository initialized. Architecture and delivery plan: `docs/PLAN.md`.

## Planned stack

- Python 3.12, `uv`, Ruff, Pytest, mypy
- FastAPI + Pydantic for the service boundary
- SQLAlchemy + Alembic; SQLite for local development, PostgreSQL for deployment
- OpenCV for image handling and diagnostics
- PyTorch with Apple MPS when available, CPU fallback
- Detector/model adapter kept behind an internal interface; no model is treated as production-ready before evaluation
- Next.js/TypeScript dashboard after the API and vision baseline are stable

## Development principles

1. Keep raw images and model artifacts out of git.
2. Record dataset/model/config versions with every evaluation.
3. Separate detection, identification, inventory logic, and transport layers.
4. Prefer abstention and an explicit `unknown` result over confident guesses.
5. Measure per-SKU precision/recall, count error, coverage, latency, and failure cases.
6. Build the smallest vertical slice before adding infrastructure.

## Initial commands

```bash
uv sync --extra dev
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run python -m shelfsense --help
```

The commands become executable as the application modules are added in the next implementation milestone.

## Repository layout

```text
docs/                 architecture and delivery decisions
src/shelfsense/      application package
  api/                HTTP schemas and routes
  core/               settings, logging, lifecycle
  domain/             product, shelf, observation, event models
  ingestion/          image validation and storage boundaries
  inventory/          counting and alert rules
  vision/             detector/identifier interfaces and adapters
data/                 local-only dataset workspace
models/               local-only model artifacts
notebooks/            exploratory analysis only
scripts/              dataset/evaluation utilities
tests/                unit and integration tests
```
