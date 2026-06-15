# Surfaces Audit Log

## Q1 — Unmapped paths

Every directory or file group inspected or considered but not mapped to a `surfaces.json` entry, with reasoning.

| Path | Reason not a user-observable surface |
|------|--------------------------------------|
| `src/edullm_sat/generation/` (pipeline.py, strategies.py, verifier.py, ranker.py, tools/) | Internal implementation of the generation pipeline triggered by `api-post-generate`; no independent entry point. |
| `src/edullm_sat/services/orchestrator.py` | Internal service layer coordinating generation steps; not a surface. |
| `src/edullm_sat/services/llm/` | LLM client abstraction over litellm; implementation detail of `api-post-generate`. |
| `src/edullm_sat/services/curriculum.py` | HTTP client that calls the external InceptCurriculum API; internal service, not a surface. |
| `src/edullm_sat/services/tracing.py` | Langfuse tracing plumbing; background observability, not a user-observable surface. |
| `src/edullm_sat/observability/` | Tracing/logging setup; implementation detail. |
| `src/edullm_sat/prompt_strategies/` | Prompt composition logic; implementation detail of `api-post-generate`. |
| `src/edullm_sat/sections/` | Section-registry for SAT sections; internal config. |
| `src/edullm_sat/storage/checkpoint.py` | Local-file JSONL checkpoint helper for offline scripts; not an HTTP or CLI surface. |
| `src/edullm_sat/api/business/generate/` (orchestrator.py, services/) | Business-logic layer invoked by `api-post-generate`; no independent entry point. |
| `src/edullm_sat/api/dependencies.py` | FastAPI dependency factories (GcsReader injection); implementation detail of browse/object/image/view routes. |
| `src/edullm_sat/api/schemas.py`, `constants.py` | Pydantic models and constants; not surfaces. |
| `src/edullm_sat/templates/` | Jinja2 HTML templates rendered by `browse-bucket`, `image-viewer`, `view-markdown`; implementation detail. |
| `components/gcs-uploader/src/gcs_uploader/services/uploader.py` | GcsUploader class used by CLI and scripts; the surface is `cli-gcs-uploader-upload`. |
| `components/gcs-uploader/src/gcs_uploader/services/reader.py` | GcsReader class used by browse/object/image/view routes; implementation detail of those surfaces. |
| `components/svg-gen/src/svg_gen/web/app.py:199 — GET /test/component-tester` | Internal developer test page; not intended as a user-observable surface. |
| `components/svg-gen/src/svg_gen/web/app.py:459 — POST /test/component` | Internal component-tester endpoint; developer tooling, not a user surface. |
| `components/svg-gen/src/svg_gen/services/` (agent_service.py, renderer/, geometry/, etc.) | Internal rendering pipeline; implementation details of `svg-gen-post-render`. |
| `components/svg-gen/src/svg_gen/infrastructure/` | LLM client, Langfuse, prompt-loader wiring; implementation detail. |
| `components/question-bank-fetcher/src/question_bank_fetcher/services/server.py:157 — GET /pdf/{slug}` and `GET /pdf/{slug}/download` | PDF view/download routes on the question-bank-fetcher server. These are secondary read surfaces subordinate to the question-bank-fetcher server and were omitted for conciseness; they could be added if PDF delivery is a primary concern. **Candidate gap — see note below.** |
| `components/question-bank-fetcher/src/question_bank_fetcher/services/server.py:170 — GET /topic/{slug}` (HTML) | Topic HTML viewer page on the question-bank-fetcher local server. Omitted; subordinate to the server. **Candidate gap.** |
| `components/question-bank-fetcher/scripts/` (analyze_outliers.py, classify_substandards.py, etc.) | One-off data-processing scripts; not user-facing surfaces. |
| `components/svg-gen/scripts/` (batch_generate_from_descriptions.py, etc.) | Developer utility scripts; not surfaces. |
| `scripts/sync_sat_to_gcs.py` | CLI-runnable admin script that bulk-uploads `curricula/SAT/` to GCS. **Candidate gap — see Q2.** |
| `scripts/generate_questions.py`, `scripts/build_example_bank.py`, `scripts/run_benchmark_sets.py`, etc. | Offline development/evaluation scripts; not user-observable surfaces in the runtime application. |
| `benchmarks/` | Offline benchmark result JSONL files and analysis scripts; not surfaces. |
| `sat-suite-official/` | Static official SAT question data (JSON/PDF); source data only, no entry point. |
| `curricula/` | Static curriculum YAML/JSON data files; consumed by `src/edullm_sat/services/curriculum.py`, not a surface. |
| `data/` | Local data directory for question-bank-fetcher downloads; storage, not a surface. |
| `openspec/` | OpenAPI specification files; documentation artifact, not a runtime surface. |
| `docs/`, `.planning/` | Architecture docs, ADRs, planning files; not surfaces. |
| `tests/` | Test suite; not surfaces. |
| `.github/` | CI/CD workflow YAML; external infrastructure, not an application surface. |

**Candidate gaps identified:**

- `components/question-bank-fetcher/src/question_bank_fetcher/services/server.py:157` — `GET /pdf/{slug}/download` and `GET /pdf/{slug}` — PDF file delivery routes on the question-bank-fetcher local server. These are read surfaces omitted from surfaces.json for conciseness; they are real user-observable surfaces if the question-bank-fetcher server is considered in scope.
- `components/question-bank-fetcher/src/question_bank_fetcher/services/server.py:170` — `GET /topic/{slug}` (HTML) — topic HTML viewer page.
- `scripts/sync_sat_to_gcs.py` — an argparse CLI script that bulk-syncs `curricula/SAT/` to GCS. This is a write path for GCS content consumed by the browse/object surfaces (see Q2).

---

## Q2 — Write paths for read surfaces

For every surface that reads, lists, searches, or displays domain data, this section traces how that data is created, updated, or deleted.

### `browse-bucket` — `GET /browse[/{prefix}]`

**Data:** GCS bucket object listing. Reads object keys from the `sat-math-api-assets-edullm-491103` GCS bucket via `GcsReader` (`components/gcs-uploader/src/gcs_uploader/services/reader.py`).

**Write paths:**

1. **`cli-gcs-uploader-upload`** — `gcs-uploader upload <FILE> --destination <GCS_PATH>` (`components/gcs-uploader/src/gcs_uploader/cli.py:41`). Uploads a single file to GCS. This is the primary manual upload entry point documented in `src/edullm_sat/api/README.md`.

2. **`scripts/sync_sat_to_gcs.py`** — `python scripts/sync_sat_to_gcs.py` (argparse CLI, `scripts/sync_sat_to_gcs.py:45`). Bulk-syncs the `curricula/SAT/` directory tree to GCS using `GcsUploader`. This is an admin script, not registered as a surface in surfaces.json.

3. **No HTTP write endpoint exists** in the main API. The API is read-only with respect to GCS content; all writes are done via the CLI or admin scripts. There is no `POST /browse` or similar.

### `object-proxy` — `GET /object/{key}`

Same write paths as `browse-bucket` above. Objects are written by `cli-gcs-uploader-upload` or `scripts/sync_sat_to_gcs.py`; the proxy only reads.

### `image-viewer` — `GET /image/{key}`

Same write paths as `browse-bucket`. Image assets (PNGs) are written to GCS by the `gcs-uploader` CLI or sync script. The image viewer only reads via `GcsReader`.

### `view-markdown` — `GET /view/{key}`

Same write paths as `browse-bucket`. Markdown articles are written to GCS by `cli-gcs-uploader-upload` or `scripts/sync_sat_to_gcs.py`. The view route reads via `GcsReader`.

### `api-get-generate-info` — `GET /generate`

**Data:** Static metadata about the POST /generate contract. Defined as a hardcoded `GenerateInfoResponse` literal in `src/edullm_sat/api/routes/generate.py:62`. **No write path exists** — this data is embedded in source code and only changes via code deployment.

### `question-bank-fetcher-serve` — `GET /api/topics`

**Data:** List of available SAT topic bundles. Reads JSON files from a local `data_dir` populated by the `question-bank-fetcher download` command.

**Write path:** **`cli-question-bank-fetcher-download`** — `question-bank-fetcher download` (`components/question-bank-fetcher/src/question_bank_fetcher/cli.py:84`). Fetches bundles from the InceptCurriculum API and writes them as JSON files to the local data directory (`components/question-bank-fetcher/src/question_bank_fetcher/domain/paths.py` via `services/downloader.py`). **No HTTP write endpoint exists** — the server is read-only; data is only created by the CLI download command.

### `question-bank-fetcher-topic-detail` — `GET /api/topic/{slug}`

Same write path as `question-bank-fetcher-serve`. Data is written by `cli-question-bank-fetcher-download`.

### `question-viewer-page` — `GET /`

**Data:** Renders SAT math questions from JSON files on disk. The question-viewer app reads question JSON from the filesystem (same data directory populated by `cli-question-bank-fetcher-download`).

**Write path:** Same as `question-bank-fetcher-serve` — data is created by `question-bank-fetcher download`. **No HTTP write endpoint exists** in the question-viewer server.

### Surfaces that are pure write or compute (no read path for domain data)

- `api-post-generate` — writes new question assets to GCS as a side-effect, but is itself the write path for the browse/object/image/view read surfaces.
- `svg-to-image-post-convert`, `cli-svg-to-image-convert`, `cli-svg-to-image-convert-batch` — compute-only (SVG in → PNG out); no persistent domain data read or written within the surface itself.
- `svg-gen-post-render`, `svg-gen-post-image-from-question`, `cli-svg-gen-render` — compute-only (description/question in → SVG out).
- `image-gen-post-generate`, `cli-image-gen-generate` — compute-only (prompt in → PNG out).
- `cli-gcs-uploader-upload` — write-only; not a read surface.
- `cli-question-bank-fetcher-download` — write-only (creates local JSON files); not a display surface.
- `svg-to-image-get-health`, `api-get-health`, `api-get-root` — return static/runtime status; no domain data involved.

### New surface added to surfaces.json

`scripts/sync_sat_to_gcs.py` is the bulk GCS write path for content shown by `browse-bucket`, `object-proxy`, `image-viewer`, and `view-markdown`. It is an argparse-based CLI script (`scripts/sync_sat_to_gcs.py:45`, `def main()`). This was added to surfaces.json as `cli-sync-sat-to-gcs`.

---

## Q3 — Declared entry points

Every declared runtime entry point found in route files, CLI scripts, container manifests, and CI configuration, with disposition.

### Container entry point

| Entry point | Disposition |
|---|---|
| `Dockerfile:39` — `uvicorn edullm_sat.api.app:app` | Starts the main FastAPI app; covers `api-get-root`, `api-get-health`, `browse-bucket`, `object-proxy`, `image-viewer`, `view-markdown`, `api-get-generate-info`, `api-post-generate`. |

### Main API routes (`src/edullm_sat/api/`)

| Route | Surface ID |
|---|---|
| `app.py:44` GET / | `api-get-root` ✓ |
| `app.py:49` GET /health | `api-get-health` ✓ |
| `routes/browse.py:77` GET /browse | `browse-bucket` ✓ |
| `routes/browse.py:78` GET /browse/{prefix:path} | `browse-bucket` ✓ (parametric variant, same surface) |
| `routes/generate.py:62` GET /generate | `api-get-generate-info` ✓ |
| `routes/generate.py:79` POST /generate | `api-post-generate` ✓ |
| `routes/object.py:33` GET /object/{key:path} | `object-proxy` ✓ |
| `routes/image.py:25` GET /image/{key:path} | `image-viewer` ✓ |
| `routes/view.py:28` GET /view/{key:path} | `view-markdown` ✓ |

### svg-to-image microservice (`components/svg-to-image/src/svg_to_image/web/app.py`)

| Route | Surface ID |
|---|---|
| `app.py:76` GET /health | `svg-to-image-get-health` ✓ |
| `app.py:147` POST /convert | `svg-to-image-post-convert` ✓ |

### svg-gen microservice (`components/svg-gen/src/svg_gen/web/app.py`)

| Route | Surface ID |
|---|---|
| `app.py:189` GET /health | `svg-gen-get-health` — **added in Q3** |
| `app.py:199` GET /test/component-tester | Out of scope: internal developer test page, not a user-observable surface. |
| `app.py:212` POST /render | `svg-gen-post-render` ✓ |
| `app.py:459` POST /test/component | Out of scope: internal component-tester endpoint. |
| `app.py:547` POST /image/fromQuestion | `svg-gen-post-image-from-question` ✓ |

### image-gen microservice (`components/image-gen/src/image_gen/web/app.py`)

| Route | Surface ID |
|---|---|
| `app.py:70` GET /health | `image-gen-get-health` — **added in Q3** |
| `app.py:99` POST /generate | `image-gen-post-generate` ✓ |

### question-bank-fetcher server (`components/question-bank-fetcher/src/question_bank_fetcher/services/server.py`)

| Route | Surface ID |
|---|---|
| `server.py:148` GET /api/topics | `question-bank-fetcher-serve` ✓ |
| `server.py:152` GET /api/topic/{slug} | `question-bank-fetcher-topic-detail` ✓ |
| `server.py:157` GET /pdf/{slug}/download | `question-bank-fetcher-pdf-download` — **added in Q3** |
| `server.py:161` GET /pdf/{slug} | `question-bank-fetcher-pdf-view` — **added in Q3** |
| `server.py:165` GET / | `question-bank-fetcher-index-page` — **added in Q3** |
| `server.py:170` GET /topic/{slug} | `question-bank-fetcher-topic-page` — **added in Q3** |

### question-viewer (`components/question-viewer/src/question_viewer/app.py`)

| Route | Surface ID |
|---|---|
| `app.py:685` GET / | `question-viewer-page` ✓ |

### CLI entry points

| Command | Surface ID |
|---|---|
| `svg-to-image convert` (cli.py:104) | `cli-svg-to-image-convert` ✓ |
| `svg-to-image convert-batch` (cli.py:187) | `cli-svg-to-image-convert-batch` ✓ |
| `svg-to-image serve` (cli.py:223) | `cli-svg-to-image-serve` ✓ |
| `svg-gen render` (cli.py:141) | `cli-svg-gen-render` ✓ |
| `svg-gen serve` (cli.py:161) | `cli-svg-gen-serve` ✓ |
| `image-gen generate` (cli.py:59) | `cli-image-gen-generate` ✓ |
| `image-gen serve` (cli.py:88) | `cli-image-gen-serve` ✓ |
| `question-bank-fetcher download` (cli.py:84) | `cli-question-bank-fetcher-download` ✓ |
| `question-bank-fetcher serve` (cli.py:144) | `cli-question-bank-fetcher-serve` ✓ |
| `question-viewer serve` (cli.py:28) | `cli-question-viewer-serve` ✓ |
| `gcs-uploader upload` (cli.py:41) | `cli-gcs-uploader-upload` ✓ |
| `python scripts/sync_sat_to_gcs.py` (sync_sat_to_gcs.py:45) | `cli-sync-sat-to-gcs` ✓ |

### Scheduled / cron / webhook entry points

None. `.github/workflows/deploy.yml` and `.github/workflows/ci.yml` contain no `schedule:` or `cron:` triggers, and no `on: repository_dispatch` or webhook receiver routes exist in the application code.

---

## Q4 — Non-URL behaviors

Behaviors that are user-observable but not reached by navigating to a stable URL.

### question-viewer page — in-page interactions

The `question-viewer-page` surface (`components/question-viewer/src/question_viewer/app.py:685`) includes the following client-side behaviors triggered without a URL change:

| Behavior | Kind | Disposition |
|---|---|---|
| **Ctrl+Enter keyboard shortcut** — pressing Ctrl+Enter in the JSON paste textarea triggers the same load action as clicking the "Load" button (app.py line ~675 in inline JS). | In-page keyboard shortcut within `question-viewer-page`. | Covered by `question-viewer-page` (kind: `page`). No separate surface needed — the schema does not define a kind for keyboard shortcuts; they are interaction details of the page surface. |
| **Sidebar filter panel** — after loading valid JSON, the `#filters-section` sidebar panel becomes visible with type/difficulty/standard dropdowns that filter which question cards are shown (app.py ~305–340 JS). | In-page panel that toggles visibility without a URL change. | Covered by `question-viewer-page`. This is an in-page state change, not a separate navigable surface. |
| **Bulk section** — a sidebar section (`#bulk-section`) appears after loading, showing bulk-action controls (app.py ~357). | In-page panel, no URL change. | Covered by `question-viewer-page`. |
| **Dark/light theme toggle** — a theme toggle button persists the preference to `localStorage` (app.py ~19–84 CSS/JS). | In-page toggle, no URL change, persisted client-side only. | Covered by `question-viewer-page`. Not a separate surface; no server interaction. |

**No modals, dialogs, toasts, context menus, or drawers** were found in any of the applications (main API Jinja2 templates, question-viewer, question-bank-fetcher server, svg-gen, image-gen, or svg-to-image). All browser-rendered surfaces use full-page responses.

**No background jobs with user-visible effects** exist in the application. Generation (`api-post-generate`) is synchronous within the HTTP request lifecycle using `asyncio.gather`; there are no queues, workers, or push notifications.

**Conclusion:** No new surfaces required from Q4. All in-page interactions belong to `question-viewer-page` (kind: `page`), which is the correct kind for a full page that includes client-side interactivity.

---

## Q5 — Correction pass

Six surfaces were added to `artifacts/surfaces.json` during the Q3–Q4 pass:

| New surface ID | Exposing question | Reason added |
|---|---|---|
| `svg-gen-get-health` | Q3 — Declared entry points | `components/svg-gen/src/svg_gen/web/app.py:189` GET /health was a declared route not previously mapped. Analogous to `svg-to-image-get-health` already in the file; omitting it was inconsistent. |
| `image-gen-get-health` | Q3 — Declared entry points | `components/image-gen/src/image_gen/web/app.py:70` GET /health was a declared route not previously mapped. Same rationale. |
| `question-bank-fetcher-index-page` | Q3 — Declared entry points | `server.py:165` GET / renders an HTML index listing all topic bundles — a genuine user-observable read surface flagged as a candidate gap in Q1 but not yet added. |
| `question-bank-fetcher-topic-page` | Q3 — Declared entry points | `server.py:170` GET /topic/{slug} renders an HTML topic detail page — same candidate gap from Q1. |
| `question-bank-fetcher-pdf-view` | Q3 — Declared entry points | `server.py:161` GET /pdf/{slug} serves a PDF inline — candidate gap from Q1, now confirmed as a real surface. |
| `question-bank-fetcher-pdf-download` | Q3 — Declared entry points | `server.py:157` GET /pdf/{slug}/download serves a PDF as a file attachment — candidate gap from Q1, now confirmed. |

**Final surface count: 34.**

**Defensible omissions upheld:**

- `GET /test/component-tester` and `POST /test/component` on the svg-gen microservice remain excluded. These are developer-only test endpoints (path prefix `/test/`) with no production use case. A rebuild team should accept this because (a) the path naming convention (`/test/`) signals internal tooling, (b) they exist only in the svg-gen component which is not the deployed container, and (c) they have no equivalent in any other component.
