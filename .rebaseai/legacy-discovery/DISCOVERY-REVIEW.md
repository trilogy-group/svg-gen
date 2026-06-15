# Discovery Review Packet

Source repo: `https://github.com/trilogy-group/EduLLM-SAT-Math.git`
Source ref: `main`
Pinned SHA: `59abf786197225afb84882bb75a8287135660591`
Generated from run: `.flow/runs/discovery/2026-06-15T08-21-45.484Z`

## How To Review This Packet

This document is the reviewer-friendly view of the discovery JSON artifacts. It is intentionally detailed, but it is not the source of truth.

Use this packet to approve, correct, split, merge, or reject the inferred app model. When you need exhaustive evidence, inspect the JSON files:

- `acquire.json`
- `surfaces.json`
- `capability-sketch.json`
- `coverage.json`
- `capabilities/**/capability.json`
- `open-questions.json`
- `needs-aria.json`
- `DESIGN.md`


## Executive Summary

Discovery inferred an AI-powered SAT math question generation service named **EduLLM SAT Math**. The system accepts InceptBench-compatible HTTP requests on a primary FastAPI API, orchestrates an LLM generation pipeline via `asyncio.gather` (using Google Gemini as the default provider, with Anthropic Claude and OpenAI as alternates through the litellm SDK), stores generated question assets in Google Cloud Storage, and returns structured question JSON to the caller. The main API is deployed as a Google Cloud Run service.

The primary callers are automated systems and evaluation pipelines (InceptBench) rather than human end-users. The only human-facing browser surfaces are utility HTML pages: a GCS bucket browser for inspecting generated assets, a question viewer for local development inspection, and a question bank fetcher UI for browsing downloaded SAT curriculum bundles from InceptCurriculum. No JavaScript framework is used; all HTML is rendered server-side via Jinja2.

The codebase is structured as a `uv` workspace of six internal Python components (`svg-gen`, `svg-to-image`, `gcs-uploader`, `image-gen`, `question-bank-fetcher`, `question-viewer`) alongside the main `src/edullm_sat/` package. Three of these components expose standalone microservices on ports 8001–8003 (svg-gen, svg-to-image, image-gen). Discovery confidence is **medium-high** for the API contract and infrastructure shape; it is **lower** for the generation pipeline internals, as the repo is explicitly described as "guardrail-first boilerplate" with the core question-generation algorithm acknowledged as scaffold rather than production-complete logic.

Two questions must be answered before rebuild can proceed: the auth error-path behavior under `SERVICE_AUTH_TOKEN` enforcement (Q-003), and whether the `question-bank-fetcher` server is a production dependency or a local dev tool only (Q-004). All 34 surfaces are covered by 9 capabilities; no orphans were found.


## Review Actions Needed

### Must Answer Before Rebuild

**Q-003** — `generation.question-generation`

- **Question:** What does `POST /generate` return when `SERVICE_AUTH_TOKEN` is set and the caller omits or sends a wrong token — HTTP 401 or 403 — and is this status documented in the OpenAPI schema?
- **Why this blocks rebuild:** The rebuild must reproduce the auth contract precisely. If the error code is wrong, InceptBench or other callers that check status codes will fail integration tests against the rebuilt service.
- **Likely options:** FastAPI's default for a failed dependency raises `HTTPException(status_code=403)`; however the dependency name `verify_auth` suggests it may use 401 with a `WWW-Authenticate` header as is idiomatic for Bearer tokens.
- **Who can answer:** runtime observation — start the service locally with `SERVICE_AUTH_TOKEN=test`, call `POST /generate` without a token, and record the status code and response body.
- **Source:** `capabilities/generation.question-generation.json`, `surfaces.json#api-post-generate`

---

**Q-004** — `question-bank.fetch-and-serve`

- **Question:** Is the `question-bank-fetcher` server (port 8765) intended to be deployed in production alongside the main Cloud Run API, or is it strictly a local developer utility like `question-viewer`?
- **Why this blocks rebuild:** If it is a production dependency, the rebuild must provision and deploy it (separate container or Cloud Run service). If it is dev-only, it belongs with `devtools` and does not need production infrastructure.
- **Likely options:** (a) Dev-only — it is not referenced in the main Dockerfile or Cloud Run config, making this the more likely answer. (b) Production sidecar — an operator may have a separate deployment config not captured in this repo.
- **Who can answer:** source app / product owner
- **Source:** `capabilities/question-bank.fetch-and-serve.json`, `open-questions.json#Q-004`

---

### Confirm Or Correct

- **svg-gen LLM provider (Q-002):** The `svg-gen` component's rendering pipeline is labelled "LLM provider" without a specific provider. Confirm whether it calls litellm (and thus supports the same Gemini/Claude alternates) or has a hard-coded provider.
- **GET / redirect vs. JSON (Q-001):** Source shows a `RedirectResponse` at `src/edullm_sat/api/app.py:44`, but the capability describes a JSON welcome response. Confirm the actual redirect target (likely `/docs`). This is nice-to-have; it does not block rebuild.
- **Supabase check-ins scope:** `acquire.json` notes that `agent_checkins` writes in `services/orchestrator.py` are unclear — whether they occur on the production request path or only via agent tooling affects observability rebuild decisions.
- **svg-to-image deployment mode:** Whether the svg-to-image microservice (port 8002) runs as a sidecar in the same Cloud Run container or as a separate deployed service is not specified in deployment config.

### Possible Missing Or Misclassified Surfaces

- **`POST /test/component` and `GET /test/component-tester`** (svg-gen, `components/svg-gen/src/svg_gen/web/app.py:459` and `:199`) — omitted by discovery as internal developer testing surfaces. If QA or integration testing calls these endpoints in automation, they should be promoted to the surface list.
- **`GET /pdf/{slug}` and `GET /pdf/{slug}/download`** on the question-bank-fetcher — these were included in `surfaces.json` as `question-bank-fetcher-pdf-view` and `question-bank-fetcher-pdf-download` (confirmed covered), but were flagged in the surfaces audit self-note as "secondary surfaces subordinate to question bank fetcher." Review whether PDF delivery is a primary user-facing feature that warrants its own capability card.


## Inferred App Model

EduLLM SAT Math is an AI-backed question generation service for SAT math. Its core loop is: an external evaluation platform (InceptBench) POSTs a generation request to `POST /generate`; the main API validates an optional Bearer or API-Key token, then fans out LLM calls in parallel via `asyncio.gather` to produce a math question with associated visual assets; assets (SVGs, PNGs, HTML) are written to a GCS bucket (`sat-math-api-assets-edullm-491103`); the generated question JSON is returned synchronously to the caller. LLM providers are abstracted through litellm, defaulting to Google Gemini (`gemini/gemini-3-flash-preview`) with Anthropic Claude and OpenAI as configured alternates.

The visual asset pipeline is broken into two independent microservices that the main API calls internally: **svg-gen** (port 8001) generates SVG diagrams from structured scene descriptions using an LLM pipeline, and **svg-to-image** (port 8002) rasterizes those SVGs to PNG using a headless Playwright/Chromium browser. A third microservice, **image-gen** (port 8003), generates photorealistic images from text prompts via the OpenAI image API. All three microservices are independently deployable and expose their own CLI `serve` commands and `GET /health` endpoints.

Three supporting utilities provide developer-time and evaluation support: the **question-bank-fetcher** downloads SAT topic bundles from the InceptCurriculum API and serves them locally for reference; the **question-viewer** provides a local HTML page for visually inspecting rendered questions; and the **gcs-uploader** component is a shared library used by the generation pipeline and a bulk-sync script (`sync_sat_to_gcs.py`) to move curricula assets into the GCS bucket. The main API also hosts a human-readable GCS bucket browser at `/browse` for engineers inspecting generated output.

Authentication is single-tier: a flat token check in `verify_auth` that is bypassed entirely when `SERVICE_AUTH_TOKEN` is unset (local dev mode). There is no RBAC, no session management, and no user-facing login flow.


## Surface Coverage

### Coverage Summary

| Metric | Count |
|---|---|
| Total surfaces | 34 |
| Covered surfaces | 34 |
| Uncovered surfaces | 0 |
| Orphan surface references | 0 |
| Total capabilities | 9 |

Coverage is complete — every enumerated surface is assigned to at least one capability.

### Covered Surfaces

| Surface ID | Kind | Label | Trigger | Observed Outcome | User Role | Capability IDs |
|---|---|---|---|---|---|---|
| `api-post-generate` | public-api | POST /generate | HTTP POST with InceptBench JSON body, optional auth header | Runs generation pipeline, stores assets in GCS, returns question JSON | none (optional token) | `generation.question-generation` |
| `api-get-generate-info` | public-api | GET /generate | HTTP GET | Returns JSON describing POST /generate contract and supported parameters | none | `generation.question-generation` |
| `api-get-health` | public-api | GET /health (main API) | HTTP GET | Returns JSON health status | none | `infra.service-health` |
| `api-get-root` | public-api | GET / (main API) | HTTP GET | Returns root welcome/info response (or redirect — see Q-001) | none | `infra.service-health` |
| `browse-bucket` | page | GET /browse | HTTP GET to `/browse` or `/browse/{prefix}` | Renders HTML page listing GCS bucket objects at the given prefix | none | `asset-browser.gcs-browser` |
| `object-proxy` | public-api | GET /object/{key} | HTTP GET | Streams raw GCS object bytes with content-type headers | none | `asset-browser.gcs-browser` |
| `image-viewer` | page | GET /image/{key} | HTTP GET | Renders HTML page displaying a GCS-stored PNG or SVG image | none | `asset-browser.gcs-browser` |
| `view-markdown` | page | GET /view/{key} | HTTP GET | Renders HTML page displaying a GCS-stored Markdown article | none | `asset-browser.gcs-browser` |
| `cli-sync-sat-to-gcs` | cli-command | sync_sat_to_gcs.py | `python scripts/sync_sat_to_gcs.py [--dry-run]` | Walks `curricula/SAT/`, uploads all files to GCS bucket | none | `asset-browser.gcs-browser` |
| `svg-gen-post-render` | public-api | POST /render | HTTP POST with scene JSON to svg-gen port 8001 | Returns rendered SVG diagram | none | `svg.scene-to-svg` |
| `svg-gen-post-image-from-question` | public-api | POST /image/fromQuestion | HTTP POST with question data JSON to svg-gen | Returns SVG image asset for the question | none | `svg.scene-to-svg` |
| `cli-svg-gen-render` | cli-command | svg-gen render | `svg-gen render <scene>` | Generates SVG from scene description, writes to stdout or file | none | `svg.scene-to-svg` |
| `cli-svg-gen-serve` | cli-command | svg-gen serve | `svg-gen serve` | Starts svg-gen FastAPI server on port 8001 | none | `svg.scene-to-svg` |
| `svg-gen-get-health` | public-api | GET /health (svg-gen) | HTTP GET to svg-gen | Returns JSON health status | none | `svg.scene-to-svg` |
| `svg-to-image-post-convert` | public-api | POST /convert | HTTP POST with SVG content to svg-to-image port 8002 | Returns PNG bytes via Playwright/Chromium | none | `svg.svg-to-png` |
| `svg-to-image-get-health` | public-api | GET /health (svg-to-image) | HTTP GET to svg-to-image | Returns JSON health status | none | `svg.svg-to-png` |
| `cli-svg-to-image-convert` | cli-command | svg-to-image convert | `svg-to-image convert <INPUT> <OUTPUT>` | Converts single SVG file to PNG | none | `svg.svg-to-png` |
| `cli-svg-to-image-convert-batch` | cli-command | svg-to-image convert-batch | `svg-to-image convert-batch <DIR>` | Converts all SVGs in a directory to PNGs | none | `svg.svg-to-png` |
| `cli-svg-to-image-serve` | cli-command | svg-to-image serve | `svg-to-image serve` | Starts svg-to-image FastAPI server on port 8002 | none | `svg.svg-to-png` |
| `image-gen-post-generate` | public-api | POST /generate (image-gen) | HTTP POST with prompt JSON to image-gen port 8003 | Returns PNG bytes via OpenAI | none | `image.ai-image-generation` |
| `image-gen-get-health` | public-api | GET /health (image-gen) | HTTP GET to image-gen | Returns JSON health status | none | `image.ai-image-generation` |
| `cli-image-gen-generate` | cli-command | image-gen generate | `image-gen generate <prompt>` | Generates PNG from prompt via OpenAI, writes to file or stdout | none | `image.ai-image-generation` |
| `cli-image-gen-serve` | cli-command | image-gen serve | `image-gen serve` | Starts image-gen FastAPI server on port 8003 | none | `image.ai-image-generation` |
| `question-bank-fetcher-serve` | public-api | GET /api/topics | HTTP GET to question-bank-fetcher | Returns JSON list of SAT topic bundles by assessment type | none | `question-bank.fetch-and-serve` |
| `question-bank-fetcher-topic-detail` | public-api | GET /api/topic/{slug} | HTTP GET | Returns full question bundle JSON for the given topic slug | none | `question-bank.fetch-and-serve` |
| `cli-question-bank-fetcher-download` | cli-command | question-bank-fetcher download | `question-bank-fetcher download` | Downloads SAT question bundles from InceptCurriculum API to local JSON | none | `question-bank.fetch-and-serve` |
| `cli-question-bank-fetcher-serve` | cli-command | question-bank-fetcher serve | `question-bank-fetcher serve <data-dir>` | Starts local FastAPI server on port 8765 | none | `question-bank.fetch-and-serve` |
| `question-bank-fetcher-index-page` | page | GET / (question-bank-fetcher) | HTTP GET | Renders HTML index of all SAT topic bundles with links | none | `question-bank.fetch-and-serve` |
| `question-bank-fetcher-topic-page` | page | GET /topic/{slug} | HTTP GET | Renders HTML detail page for the given topic slug | none | `question-bank.fetch-and-serve` |
| `question-bank-fetcher-pdf-view` | public-api | GET /pdf/{slug} | HTTP GET | Serves topic PDF inline (browser view) | none | `question-bank.fetch-and-serve` |
| `question-bank-fetcher-pdf-download` | public-api | GET /pdf/{slug}/download | HTTP GET | Serves topic PDF as file attachment download | none | `question-bank.fetch-and-serve` |
| `question-viewer-page` | page | GET / (question-viewer) | HTTP GET to question-viewer port 7890 | Renders HTML page for viewing SAT math questions with SVG/HTML assets | none | `devtools.question-viewer` |
| `cli-question-viewer-serve` | cli-command | question-viewer serve | `question-viewer serve` | Starts question-viewer FastAPI server on port 7890 | none | `devtools.question-viewer` |
| `cli-gcs-uploader-upload` | cli-command | gcs-uploader upload | `gcs-uploader upload <FILE> --destination <GCS_PATH>` | Uploads a local file to GCS with optional content-type override | none | `infra.gcs-upload` |


## Capability Map

### Domain: generation

| ID | Summary | Surfaces | Confidence | Tests | Dependencies | Reviewer Note |
|---|---|---|---|---|---|---|
| `generation.question-generation` | Core SAT question generation via LLM pipeline; stores assets in GCS; returns question JSON | `api-post-generate`, `api-get-generate-info` | medium | unknown — repo described as scaffold | litellm (Gemini/Claude/OpenAI), GCS, Langfuse, InceptBench API | Core business logic described as intentionally deferred/stub. Verify pipeline completeness before rebuild target is set. |

### Domain: asset-browser

| ID | Summary | Surfaces | Confidence | Tests | Dependencies | Reviewer Note |
|---|---|---|---|---|---|---|
| `asset-browser.gcs-browser` | GCS bucket browser: HTML directory navigation, raw proxy download, image viewer, Markdown viewer, bulk sync CLI | `browse-bucket`, `object-proxy`, `image-viewer`, `view-markdown`, `cli-sync-sat-to-gcs` | high | unknown | GCS SDK, Jinja2 | Coherent browsing capability; no known gaps. |

### Domain: svg

| ID | Summary | Surfaces | Confidence | Tests | Dependencies | Reviewer Note |
|---|---|---|---|---|---|---|
| `svg.scene-to-svg` | LLM-backed SVG diagram generation from scene descriptions or question objects | `svg-gen-post-render`, `svg-gen-post-image-from-question`, `cli-svg-gen-render`, `cli-svg-gen-serve`, `svg-gen-get-health` | medium | unknown | LLM provider (unconfirmed — see Q-002) | Provider identity unconfirmed; internal pipeline not read. |
| `svg.svg-to-png` | Playwright/Chromium SVG rasterization to PNG; HTTP microservice and CLI | `svg-to-image-post-convert`, `svg-to-image-get-health`, `cli-svg-to-image-convert`, `cli-svg-to-image-convert-batch`, `cli-svg-to-image-serve` | high | unknown | Playwright/Chromium | Header validation for `X-Output-Width`/`X-Background` unconfirmed (Q-005); nice-to-have. |

### Domain: image

| ID | Summary | Surfaces | Confidence | Tests | Dependencies | Reviewer Note |
|---|---|---|---|---|---|---|
| `image.ai-image-generation` | OpenAI-backed photorealistic PNG generation from text prompts; HTTP microservice and CLI | `image-gen-post-generate`, `image-gen-get-health`, `cli-image-gen-generate`, `cli-image-gen-serve` | high | unknown | OpenAI image API | Model selection (dall-e-2 vs dall-e-3) unconfirmed (Q-006); post-rebuild concern. |

### Domain: question-bank

| ID | Summary | Surfaces | Confidence | Tests | Dependencies | Reviewer Note |
|---|---|---|---|---|---|---|
| `question-bank.fetch-and-serve` | Downloads SAT topic bundles from InceptCurriculum API; serves locally via FastAPI with topic listing, detail, and PDF endpoints | 8 surfaces | high | unknown | InceptCurriculum API | Production vs. dev-only deployment scope is a before-rebuild blocker (Q-004). |

### Domain: devtools

| ID | Summary | Surfaces | Confidence | Tests | Dependencies | Reviewer Note |
|---|---|---|---|---|---|---|
| `devtools.question-viewer` | Local HTML viewer for rendered SAT math questions; FastAPI on port 7890 | `question-viewer-page`, `cli-question-viewer-serve` | high | unlikely | FastAPI, Jinja2 | Dev-only tool; not in main Dockerfile. |

### Domain: infra

| ID | Summary | Surfaces | Confidence | Tests | Dependencies | Reviewer Note |
|---|---|---|---|---|---|---|
| `infra.service-health` | Health check and root info endpoints on the main API; used for Cloud Run liveness probing | `api-get-health`, `api-get-root` | high | none expected | FastAPI | Q-001: confirm GET / returns JSON or redirects to /docs. |
| `infra.gcs-upload` | CLI utility for uploading individual files to GCS; consumed by generation pipeline and sync script | `cli-gcs-uploader-upload` | high | unknown | GCS SDK | Single-surface capability; a shared library more than a user-facing feature. |


## Capability Details

### generation.question-generation

**Summary:** Accepts an InceptBench-compatible POST body, fans out LLM calls in parallel with `asyncio.gather`, stores produced assets in GCS, and returns the generated SAT math question JSON. Also exposes a GET introspection endpoint.

**Surfaces:** `api-post-generate`, `api-get-generate-info`

**Key behavior:**
- Auth is optional: `verify_auth` is a no-op when `SERVICE_AUTH_TOKEN` is unset.
- Under enforcement, error-path HTTP status code is unconfirmed (Q-003 — before-rebuild blocker).
- Default LLM model: `gemini/gemini-3-flash-preview` via litellm.
- Langfuse tracing is integrated as an LLM observability callback.
- Generation pipeline is described in the repo as intentionally incomplete scaffold ("guardrail-first boilerplate").

**Evidence highlights:**
- `src/edullm_sat/api/routes/generate.py:79` — POST handler triggers generation pipeline with `asyncio.gather`
- `src/edullm_sat/api/routes/generate.py:62` — GET handler returns contract summary

**Tests:** None confirmed; pipeline described as scaffold.

**Rebuild decisions needed:** (1) Confirm auth error HTTP status (Q-003). (2) Determine whether the generation pipeline stub is in scope for rebuild or is a known intentional placeholder.

**Review prompts:** Is the generation pipeline expected to be complete or is scaffold acceptable in the rebuild target? Which LLM model is the authoritative default?

---

### asset-browser.gcs-browser

**Summary:** Provides HTML-based navigation of the GCS bucket, proxy download of stored objects, an image viewer, a Markdown article renderer, and a bulk sync CLI.

**Surfaces:** `browse-bucket`, `object-proxy`, `image-viewer`, `view-markdown`, `cli-sync-sat-to-gcs`

**Key behavior:**
- All browser routes read from GCS via a `GcsReader` dependency injected into FastAPI routes.
- Jinja2 templates are in `src/edullm_sat/templates/`.
- `sync_sat_to_gcs.py` supports `--dry-run` for safe validation.

**Evidence highlights:**
- `src/edullm_sat/api/routes/browse.py:88` — `reader.list_prefix(prefix)` enumerates bucket contents
- `src/edullm_sat/api/routes/object.py:33` — streams raw bytes with content-type detection
- `scripts/sync_sat_to_gcs.py:45` — bulk upload from `curricula/SAT/`

**Tests:** None confirmed.

**Rebuild decisions needed:** None blocking.

---

### svg.scene-to-svg

**Summary:** Generates SVG diagrams from structured scene descriptions or question objects via an LLM pipeline. Exposed as HTTP microservice on port 8001 and as a CLI.

**Surfaces:** `svg-gen-post-render`, `svg-gen-post-image-from-question`, `cli-svg-gen-render`, `cli-svg-gen-serve`, `svg-gen-get-health`

**Key behavior:**
- `POST /render` accepts a scene JSON body; `POST /image/fromQuestion` accepts a SAT question object.
- LLM provider identity unconfirmed (Q-002 — post-rebuild, not blocking).
- Internal SVG rendering pipeline at `components/svg-gen/src/svg_gen/` was not read in full.

**Evidence highlights:**
- `components/svg-gen/src/svg_gen/web/app.py:212` — POST /render handler
- `components/svg-gen/src/svg_gen/web/app.py:547` — POST /image/fromQuestion handler

**Tests:** None confirmed.

**Rebuild decisions needed:** Confirm LLM provider for rebuild environment configuration.

---

### svg.svg-to-png

**Summary:** Rasterizes SVG to PNG using Playwright/Chromium. Available as HTTP microservice on port 8002 and three CLI commands (single, batch, serve).

**Surfaces:** `svg-to-image-post-convert`, `svg-to-image-get-health`, `cli-svg-to-image-convert`, `cli-svg-to-image-convert-batch`, `cli-svg-to-image-serve`

**Key behavior:**
- Playwright/Chromium (~300–500 MB dependency) is bundled in the Docker image for request-time rendering.
- `POST /convert` accepts SVG as raw body or JSON; configurable via `X-Output-Width` and `X-Background` headers (ranges unconfirmed — Q-005, nice-to-have).
- Batch CLI processes all `.svg` files in an input directory.

**Evidence highlights:**
- `components/svg-to-image/src/svg_to_image/web/app.py:147` — POST /convert handler
- `components/svg-to-image/src/svg_to_image/cli.py:187` — convert-batch CLI

**Tests:** None confirmed.

**Rebuild decisions needed:** Confirm deployment topology (sidecar vs. separate service).

---

### image.ai-image-generation

**Summary:** Generates photorealistic PNG images from text prompts via the OpenAI image generation API. Available as HTTP microservice on port 8003 and CLI.

**Surfaces:** `image-gen-post-generate`, `image-gen-get-health`, `cli-image-gen-generate`, `cli-image-gen-serve`

**Key behavior:**
- `POST /generate` accepts `{"prompt": "...", "n": 1}` (and potentially a model field — Q-006).
- Returns raw PNG bytes.

**Evidence highlights:**
- `components/image-gen/src/image_gen/web/app.py:99` — POST /generate handler
- `components/image-gen/src/image_gen/cli.py:59` — generate CLI

**Tests:** None confirmed.

**Rebuild decisions needed:** Confirm OpenAI model selection behavior (Q-006, post-rebuild).

---

### question-bank.fetch-and-serve

**Summary:** Downloads SAT topic bundles from the InceptCurriculum API to local JSON files, then serves them via a local FastAPI server (port 8765) with topic listing, detail, PDF view, and PDF download endpoints.

**Surfaces:** `cli-question-bank-fetcher-download`, `cli-question-bank-fetcher-serve`, `question-bank-fetcher-serve`, `question-bank-fetcher-topic-detail`, `question-bank-fetcher-index-page`, `question-bank-fetcher-topic-page`, `question-bank-fetcher-pdf-view`, `question-bank-fetcher-pdf-download`

**Key behavior:**
- Download CLI supports filtering by assessment type, domain, and output directory.
- Serve CLI reads from a local data directory and exposes both JSON API and HTML pages.
- Not referenced in the main Dockerfile or Cloud Run config.

**Evidence highlights:**
- `components/question-bank-fetcher/src/question_bank_fetcher/cli.py:84` — download CLI
- `components/question-bank-fetcher/src/question_bank_fetcher/services/server.py:148` — GET /api/topics

**Tests:** None confirmed.

**Rebuild decisions needed:** Q-004 (before-rebuild blocker): production vs. dev-only deployment scope.

---

### devtools.question-viewer

**Summary:** Local developer tool for visually inspecting rendered SAT math questions as HTML with SVG and HTML assets rendered server-side.

**Surfaces:** `question-viewer-page`, `cli-question-viewer-serve`

**Key behavior:** Serves a FastAPI HTML page on port 7890. Not in main Dockerfile.

**Evidence highlights:**
- `components/question-viewer/src/question_viewer/app.py:685` — GET / handler
- `components/question-viewer/src/question_viewer/cli.py:28` — serve CLI

**Tests:** None expected for a dev utility.

**Rebuild decisions needed:** None.

---

### infra.service-health

**Summary:** Health check (`GET /health`) and root info (`GET /`) endpoints on the main API, used by Cloud Run liveness probes.

**Surfaces:** `api-get-health`, `api-get-root`

**Key behavior:** Stateless; no auth required; `GET /` may redirect to `/docs` (see Q-001).

**Tests:** None expected for health probes.

**Rebuild decisions needed:** Q-001 (nice-to-have): confirm redirect target.

---

### infra.gcs-upload

**Summary:** CLI utility (`gcs-uploader upload`) for uploading individual files to a GCS bucket path with optional content-type override. Consumed internally by the generation pipeline and the sync script.

**Surfaces:** `cli-gcs-uploader-upload`

**Key behavior:** Thin wrapper around the GCS SDK; also used as a library by other components.

**Tests:** None confirmed.

**Rebuild decisions needed:** None.


## Confidence And Risk

### High Confidence Capabilities

- `asset-browser.gcs-browser` — all routes and their GCS read patterns are directly evidenced.
- `svg.svg-to-png` — Playwright rendering contract is clearly defined; only header validation details are open (nice-to-have).
- `image.ai-image-generation` — OpenAI integration is confirmed; model selection is the only open point (post-rebuild).
- `infra.service-health` — trivial stateless endpoints.
- `infra.gcs-upload` — single CLI command with clear GCS SDK usage.
- `question-bank.fetch-and-serve` — download and serve contract is well-evidenced; deployment scope is a before-rebuild question, not a contract question.
- `devtools.question-viewer` — straightforward dev tool.

### Medium Confidence Capabilities

- `generation.question-generation` — the HTTP contract is clear, but the generation pipeline internals are acknowledged as scaffold/stub. Confidence in rebuilding the pipeline to correct behavior is low until the completeness question is resolved.
- `svg.scene-to-svg` — HTTP contract is clear, but the internal LLM provider and rendering pipeline were not fully read.

### Zero-Test Capabilities

All 9 capabilities have zero confirmed qualifying tests. The repo is described as "guardrail-first boilerplate," so this is expected at this stage. Risk is highest for `generation.question-generation` and `svg.scene-to-svg` where internal pipeline logic is unverified.

### What Would Improve Confidence

1. A live runtime test of `POST /generate` with a real InceptBench payload would confirm the auth error path (resolving Q-003) and validate the generation pipeline produces valid output.
2. Reading `components/svg-gen/src/svg_gen/` pipeline source would confirm the LLM provider (resolving Q-002).
3. Inspecting the Cloud Run deployment config or speaking with the product owner would resolve Q-004 (question-bank-fetcher production scope).


## ARIA / Runtime Observation Needed

Six HTML surfaces have ARIA concerns identified in `needs-aria.json`. Two are high priority, three medium, one low.

**High priority:**

- `image-viewer` (`GET /image/{key}`) — displayed images need descriptive `alt` text beyond the filename; SVG images embedded inline should carry `<title>` and `<desc>` elements; back/navigation link needs a clear accessible label.
- `question-viewer-page` (`GET /`) — math question SVG content needs an accessible description for screen readers; answer choice lists should use semantic list markup with `role="radiogroup"` if interactive; LaTeX/MathML rendered inline should carry `aria-label` or descriptive wrapper text.

**Medium priority:**

- `browse-bucket` (`GET /browse`) — directory listing table lacks column header `scope` attributes; breadcrumb navigation not wrapped in `<nav aria-label>`; file/folder type distinction is visual-only.
- `view-markdown` (`GET /view/{key}`) — breadcrumb should use `<nav aria-label="Breadcrumb">` with `aria-current="page"` on the last item; Markdown heading hierarchy should be validated for no skipped levels.
- `question-bank-fetcher-topic-page` (`GET /topic/{slug}`) — question list items need semantic list markup; math expressions rendered as images need `alt` text; PDF download link should indicate file type and size in accessible label.

**Low priority:**

- `question-bank-fetcher-index-page` (`GET /`) — topic group headings should use semantic `h2`/`h3` elements, not styled divs; topic links need descriptive text beyond just the slug.

Reviewers should capture runtime screenshots of all 6 HTML surfaces and verify the ARIA concerns listed above. Full details in `needs-aria.json`.


## Visual Identity

The app uses a GitHub-inspired monochrome design system extracted from Jinja2 templates in `src/edullm_sat/templates/`. The palette is dark charcoal header (`#24292f`), light grey page background (`#f6f8fa`), white content cards (`#ffffff`), and a blue link accent (`#0969da`). Monospace type (`ui-monospace, "Cascadia Code", "Fira Mono"`) is used for shell chrome; a system sans-serif stack (`-apple-system, BlinkMacSystemFont, "Segoe UI"`) is used for prose content. Base font size is 16px with 1.6 line-height; border radius tokens are 4px (sm) and 6px (md).

Full design token YAML is in `artifacts/DESIGN.md`.


## Exclusions

This packet explicitly excludes:

- Cloned source files and raw grep output
- Dependency version dumps and lock file contents
- Environment variable values (only boundaries and integration points are described)
- Internal helper functions that do not constitute user-observable behavior
- Non-qualifying tests (infrastructure tests, internal unit tests without behavioral assertions)
- Full evidence arrays from capability JSON files (see `capabilities/*.json` for complete evidence)
- The `.beads/` organizational memory directory and `.claude/` agent tooling checked into the repo


## Next Step

1. **Answer the two before-rebuild blockers** — Q-003 (auth error path, via runtime observation) and Q-004 (question-bank-fetcher production scope, via product owner).
2. **Confirm or correct** the svg-gen LLM provider (Q-002) and the GET / redirect behavior (Q-001) if they affect your rebuild target.
3. **Mark any misclassified surfaces** — in particular, decide whether the svg-gen internal test endpoints (`POST /test/component`, `GET /test/component-tester`) should be promoted to the surface list.
4. **Confirm split/merge notes** — the question-bank-fetcher PDF routes are grouped into `question-bank.fetch-and-serve`; split them if PDF delivery is considered a distinct user-facing feature.
5. **Resume the workflow** once all before-rebuild questions are answered.
