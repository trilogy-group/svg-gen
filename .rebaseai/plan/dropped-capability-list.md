## Dropped capabilities

The selected proposal (Proposal 3 — Aggressive rewrite) scopes the rebuild to the legacy `svg-gen` component as a constraint-based diagram engine. Every legacy capability outside that scope is dropped from this rebuild; capabilities still needed by the wider product continue to live in the legacy estate and are simply not the subject of this repo.

### generation.question-generation
The end-to-end SAT math question generation pipeline (`POST /generate`, `GET /generate`) that orchestrates LLM calls, asset writes to GCS, and InceptBench-compatible responses. Dropped because the proposal explicitly limits LLM usage in this rebuild to feasibility checks and constraint-graph scene emission for diagrams — the upstream question-generation orchestrator is a separate service and is not in the svg-gen rebuild's scope.

### asset-browser.gcs-browser
The HTML GCS-bucket browser (`GET /browse`, `/browse/{prefix}`, `/object/{key}`, `/image/{key}`, `/view/{key}`) plus the `sync_sat_to_gcs` bulk-upload CLI. Dropped because asset browsing is an operator tool tied to the question-generation service's storage layout, with no role in the svg-gen rendering contract the proposal locks; it stays in the legacy estate and the rebuild does not reproduce it.

### svg.svg-to-png
The standalone Playwright/Chromium SVG-to-PNG rasterization microservice on port 8002 (`POST /convert`, `GET /health`, batch and single-file CLI commands). Dropped because the proposal locks "Playwright/Chromium is deleted from the runtime dependency set" and replaces browser-based rendering with a pure-math pipeline; downstream PNG needs, if any, are handled outside this repo and rasterization is no longer a capability the rebuild ships.

### image.ai-image-generation
The OpenAI-backed text-prompt → PNG image generation microservice on port 8003 (`POST /generate`, `GET /health`, generate/serve CLI). Dropped because photorealistic image generation is unrelated to the constraint-based vector-diagram engine the proposal defines, and the proposal restricts the rebuild's LLM surface to feasibility + scene-spec emission only.

### question-bank.fetch-and-serve
The InceptCurriculum question-bank acquisition CLI and local serving UI (`GET /api/topics`, `/api/topic/{slug}`, `/`, `/topic/{slug}`, `/pdf/{slug}`, `/pdf/{slug}/download`). Dropped because question-bank acquisition and browsing is a dev-time data-pipeline concern feeding the upstream generation service; the svg-gen rebuild consumes scene descriptions, not question banks, and has no use for these surfaces.

### devtools.question-viewer
The local FastAPI page on port 7890 that renders SAT math questions with their SVG/HTML assets for engineer inspection. Dropped because it is a viewer for full questions produced by the upstream generation pipeline, not for svg-gen output in isolation; the rebuild's own developer feedback loop is covered by the new deterministic test suite and the `--verify-bbox` dev tool.

### infra.service-health
The `GET /health` and `GET /` endpoints on the **main edullm_sat API** (`src/edullm_sat/api/app.py`). Dropped because these belong to the question-generation service, which is not part of this rebuild; the svg-gen service has its own health endpoint that is retained under the required `svg.scene-to-svg` capability.

### infra.gcs-upload
The `gcs-uploader` CLI for uploading individual files to a GCS bucket with optional content-type override. Dropped because the svg-gen rebuild returns rendered SVGs over its HTTP contract and does not own asset persistence; GCS upload remains a concern of the upstream generation service and is not reproduced here.
