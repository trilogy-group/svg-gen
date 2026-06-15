# Rebuild Roadmap Draft

Source repo: `https://github.com/trilogy-group/EduLLM-SAT-Math.git`
Pinned SHA: `59abf786197225afb84882bb75a8287135660591`
Generated from run: `.flow/runs/discovery/2026-06-15T08-21-45.484Z`

---

## How To Read This Document

Phases are ordered by dependency and confidence: earlier phases unblock later ones. Within each phase, capabilities are listed in the order they should be implemented — foundational infrastructure first, then high-confidence leaf capabilities, then medium-confidence capabilities that depend on resolved open questions.

Open questions that are still unresolved are flagged inline. The two before-rebuild blockers (Q-003, Q-004) must be answered before Phase 3 and Phase 4 respectively; work in Phases 1 and 2 can proceed without them.

---

## Pre-Work: Resolve Before-Rebuild Blockers

Before cutting any production-bound code, answer the following two questions. Phases 1 and 2 are independent and can start immediately; they do not depend on these answers.

### Q-003 — Auth error-path HTTP status code

**Capability affected:** `generation.question-generation`
**How to answer:** Start the service locally with `SERVICE_AUTH_TOKEN=test`, call `POST /generate` without a token, and record the HTTP status code and response body.
**Expected answer:** Either 401 (with `WWW-Authenticate: Bearer` header, idiomatic for Bearer tokens) or 403 (FastAPI default for a failed dependency).
**Impact on rebuild:** The rebuilt service must return the same status code, since InceptBench or other callers that check status codes will fail integration tests against a mismatched error response.

### Q-004 — question-bank-fetcher production vs. dev-only scope

**Capability affected:** `question-bank.fetch-and-serve`
**How to answer:** Ask the product owner. The component is absent from the main Dockerfile and Cloud Run config, strongly suggesting dev-only.
**Impact on rebuild:** If dev-only: treat as `devtools` scope, no production infrastructure needed. If production: provision a separate Cloud Run service or sidecar.

---

## Phase 1 — Shared Infrastructure Primitives (no open questions)

These capabilities have zero open blockers and are prerequisites for everything else. Implement them first.

### 1-A. `infra.gcs-upload` — GCS upload shared library and CLI
**Confidence:** high
**Dependencies:** Google Cloud Storage SDK
**Why first:** The `GcsUploader` component is imported by both the main generation pipeline and the bulk sync script. Nothing that writes to GCS can be tested without it.
**Surfaces:**
- `cli-gcs-uploader-upload` — `gcs-uploader upload <FILE> --destination <GCS_PATH>`

**Rebuild scope:**
- `components/gcs-uploader/` package
- `GcsUploader.upload` function with optional content-type override
- Click CLI entrypoint

---

### 1-B. `infra.service-health` — Health and root endpoints
**Confidence:** high (Q-001 is nice-to-have, does not block)
**Dependencies:** FastAPI
**Why here:** Cloud Run will not route traffic until liveness probes pass. These are the first endpoints to bring up in any deployment.
**Open question (nice-to-have):** Q-001 — confirm whether `GET /` returns JSON or redirects to `/docs`. Implement whichever is correct; default to `RedirectResponse` to `/docs` since that is what source evidence shows.
**Surfaces:**
- `api-get-health` — `GET /health` returns JSON `{"status": "ok"}`
- `api-get-root` — `GET /` returns root response or redirect

**Rebuild scope:**
- `src/edullm_sat/api/app.py` FastAPI app init
- Health and root route handlers

---

## Phase 2 — Leaf Microservices (high confidence, independent)

These three microservices have no dependencies on each other or on the generation pipeline. They can be rebuilt in parallel after Phase 1.

### 2-A. `svg.svg-to-png` — SVG-to-PNG rasterization microservice
**Confidence:** high
**Dependencies:** Playwright/Chromium (~300–500 MB), FastAPI, Click
**Open question (nice-to-have):** Q-005 — confirm accepted ranges for `X-Output-Width` and `X-Background` headers. Rebuild to accept them as optional; default behavior should match source if ranges are unconfirmed.
**Surfaces:**
- `svg-to-image-post-convert` — `POST /convert` (port 8002), returns PNG bytes
- `svg-to-image-get-health` — `GET /health`
- `cli-svg-to-image-convert` — `svg-to-image convert <INPUT> <OUTPUT>`
- `cli-svg-to-image-convert-batch` — `svg-to-image convert-batch <DIR>`
- `cli-svg-to-image-serve` — `svg-to-image serve`

**Rebuild scope:**
- `components/svg-to-image/` package
- Playwright headless browser setup and PNG rendering pipeline
- HTTP microservice and CLI entrypoints
- `GET /health` endpoint
- **Deployment note:** deployment topology (sidecar vs. separate Cloud Run service) is unresolved; design to be independently deployable and let infrastructure config decide.

---

### 2-B. `image.ai-image-generation` — OpenAI photorealistic image microservice
**Confidence:** high
**Dependencies:** OpenAI image API, FastAPI, Click
**Open question (post-rebuild):** Q-006 — whether model selection (`dall-e-2` vs `dall-e-3`) is supported via request body or env var. Default to env-var-configured with `dall-e-3` as default; the API surface is unaffected.
**Surfaces:**
- `image-gen-post-generate` — `POST /generate` (port 8003), accepts `{"prompt": "...", "n": 1}`, returns PNG bytes
- `image-gen-get-health` — `GET /health`
- `cli-image-gen-generate` — `image-gen generate <prompt>`
- `cli-image-gen-serve` — `image-gen serve`

**Rebuild scope:**
- `components/image-gen/` package
- OpenAI image API integration
- HTTP microservice and CLI entrypoints
- `GET /health` endpoint

---

### 2-C. `svg.scene-to-svg` — LLM-backed SVG diagram microservice
**Confidence:** medium (Q-002 unresolved: LLM provider unknown; pipeline internals not fully read)
**Dependencies:** LLM provider (litellm likely; confirm via Q-002), FastAPI, Click
**Open question (post-rebuild):** Q-002 — which LLM provider does svg-gen use? If litellm, it inherits the same Gemini/Claude alternates as the main pipeline. Wire up litellm with `gemini/gemini-3-flash-preview` as default and let env vars override, matching the main pipeline pattern.
**Surfaces:**
- `svg-gen-post-render` — `POST /render` (port 8001), accepts scene JSON, returns SVG
- `svg-gen-post-image-from-question` — `POST /image/fromQuestion`, accepts question object, returns SVG
- `cli-svg-gen-render` — `svg-gen render <scene>`
- `cli-svg-gen-serve` — `svg-gen serve`
- `svg-gen-get-health` — `GET /health`

**Rebuild scope:**
- `components/svg-gen/` package
- LLM-backed scene-to-SVG rendering pipeline
- `POST /render` and `POST /image/fromQuestion` handlers
- CLI entrypoints and health endpoint
- **Risk note:** pipeline internals were not read in full during discovery. Read `components/svg-gen/src/svg_gen/` before rebuilding; the pipeline may contain non-obvious prompt engineering or schema validation logic.

---

## Phase 3 — Core Generation Pipeline (depends on Q-003 answer + Phases 1 and 2)

Do not start this phase until:
1. Q-003 is answered (auth error-path HTTP status code).
2. Phases 1 and 2 are complete (GCS uploader and all three microservices are functional).

### 3-A. `generation.question-generation` — Main SAT question generation API
**Confidence:** medium (HTTP contract is clear; pipeline internals are acknowledged scaffold)
**Dependencies:** litellm (Gemini/Claude/OpenAI), GCS (`infra.gcs-upload`), Langfuse, svg-gen (`svg.scene-to-svg`), svg-to-image (`svg.svg-to-png`), image-gen (`image.ai-image-generation`), InceptBench API
**Surfaces:**
- `api-post-generate` — `POST /generate`, InceptBench-compatible JSON body, optional Bearer / X-API-Key auth, returns generated question JSON
- `api-get-generate-info` — `GET /generate`, returns JSON contract summary

**Rebuild scope:**
- `src/edullm_sat/api/routes/generate.py` — POST and GET handlers
- `verify_auth` FastAPI dependency — implement with answer from Q-003:
  - If 401: raise `HTTPException(401)` with `WWW-Authenticate: Bearer` header
  - If 403: raise `HTTPException(403)` (FastAPI default)
  - No-op bypass when `SERVICE_AUTH_TOKEN` is unset
- Generation pipeline wired via `asyncio.gather`, calling svg-gen, svg-to-image, and image-gen microservices in parallel
- Langfuse tracing callback registration
- GCS asset write via `GcsUploader`
- Default LLM model: `gemini/gemini-3-flash-preview` via litellm
- **Scaffold risk:** The repo is described as "guardrail-first boilerplate" with the question-generation algorithm incomplete. Rebuild should faithfully reproduce the scaffold structure first; fill in generation logic as a second pass once the contract is verified end-to-end.

---

## Phase 4 — GCS Asset Browser and Sync (depends on Phase 1; Q-004 answer informs devtools placement)

Phase 4 can start after Phase 1-A (`infra.gcs-upload`) is complete. Q-004 affects only whether `question-bank.fetch-and-serve` is placed in this phase or in Phase 5.

### 4-A. `asset-browser.gcs-browser` — GCS bucket browser and asset viewers
**Confidence:** high
**Dependencies:** GCS SDK (read), Jinja2, `infra.gcs-upload` (for sync CLI)
**Surfaces:**
- `browse-bucket` — `GET /browse` and `GET /browse/{prefix}`, HTML directory listing via Jinja2
- `object-proxy` — `GET /object/{key}`, streams raw GCS bytes with content-type detection
- `image-viewer` — `GET /image/{key}`, HTML image viewer page
- `view-markdown` — `GET /view/{key}`, HTML Markdown renderer from GCS
- `cli-sync-sat-to-gcs` — `python scripts/sync_sat_to_gcs.py [--dry-run]`, bulk upload from `curricula/SAT/`

**Rebuild scope:**
- `src/edullm_sat/api/routes/browse.py`, `object.py`, `image.py`, `view.py`
- `GcsReader` FastAPI dependency for bucket object listing and reads
- Jinja2 templates in `src/edullm_sat/templates/` (GitHub-inspired monochrome design: dark charcoal `#24292f` header, grey `#f6f8fa` page, white `#ffffff` cards, blue `#0969da` accent; see `artifacts/DESIGN.md` for full token YAML)
- `scripts/sync_sat_to_gcs.py` with `--dry-run` flag
- **ARIA debt (high priority):** `image-viewer` needs descriptive `alt` text and SVG `<title>`/`<desc>`; back/navigation links need accessible labels. Address before shipping.
- **ARIA debt (medium priority):** `browse-bucket` table needs `scope` headers; breadcrumb needs `<nav aria-label>`; `view-markdown` heading hierarchy and breadcrumb `aria-current`.

---

## Phase 5 — Developer Utilities (can run after Phase 3 or independently; Q-004 determines question-bank-fetcher placement)

These are dev-only or scope-uncertain tools. They do not block production but should be rebuilt so engineers can inspect generated output locally.

### 5-A. `devtools.question-viewer` — Local question HTML viewer
**Confidence:** high
**Dependencies:** FastAPI, Jinja2
**Surfaces:**
- `question-viewer-page` — `GET /` (port 7890), HTML question viewer
- `cli-question-viewer-serve` — `question-viewer serve`

**Rebuild scope:**
- `components/question-viewer/` package
- `GET /` handler rendering SAT math questions with SVG/HTML assets
- `question-viewer serve` CLI
- Not in main Dockerfile; dev-only.

---

### 5-B. `question-bank.fetch-and-serve` — SAT question bank downloader and local server
**Confidence:** high (contract well-evidenced; scope is the only open question)
**Dependencies:** InceptCurriculum API, FastAPI, Jinja2, Click
**Open question (before-rebuild):** Q-004 — production vs. dev-only. Implement the component either way; only the deployment target changes.
**Surfaces:**
- `cli-question-bank-fetcher-download` — `question-bank-fetcher download`, fetches from InceptCurriculum API
- `cli-question-bank-fetcher-serve` — `question-bank-fetcher serve <data-dir>`, starts server on port 8765
- `question-bank-fetcher-serve` — `GET /api/topics`, JSON topic list
- `question-bank-fetcher-topic-detail` — `GET /api/topic/{slug}`, JSON topic bundle
- `question-bank-fetcher-index-page` — `GET /`, HTML topic index
- `question-bank-fetcher-topic-page` — `GET /topic/{slug}`, HTML topic detail
- `question-bank-fetcher-pdf-view` — `GET /pdf/{slug}`, inline PDF
- `question-bank-fetcher-pdf-download` — `GET /pdf/{slug}/download`, PDF attachment

**Rebuild scope:**
- `components/question-bank-fetcher/` package
- Download CLI with assessment/domain/output-dir filtering
- FastAPI server with JSON API, HTML pages, and PDF routes
- Jinja2 templates for index and topic pages
- **ARIA debt (medium priority):** topic page needs semantic list markup and math image alt text; PDF download link needs accessible label with file type/size.
- **ARIA debt (low priority):** index page topic group headings should use semantic `h2`/`h3`.
- **Deployment:** if Q-004 resolves to production — provision as a separate Cloud Run service (it is not in the main Dockerfile). If dev-only — ship as a local dev tool alongside `question-viewer`.

---

## Dependency Graph Summary

```
Phase 1 (no deps)
  1-A infra.gcs-upload
  1-B infra.service-health

Phase 2 (after Phase 1, can run in parallel)
  2-A svg.svg-to-png          (independent)
  2-B image.ai-image-generation (independent)
  2-C svg.scene-to-svg        (medium confidence; pipeline internals risk)

Phase 3 (after Phases 1+2, requires Q-003 answer)
  3-A generation.question-generation

Phase 4 (after Phase 1-A)
  4-A asset-browser.gcs-browser

Phase 5 (after Phase 1; independent of Phase 3)
  5-A devtools.question-viewer
  5-B question-bank.fetch-and-serve (requires Q-004 answer for deployment target)
```

---

## Open Questions Status at Roadmap Time

| ID | Blocker tier | Affects | Status |
|---|---|---|---|
| Q-001 | nice-to-have | `infra.service-health` (Phase 1-B) | open — implement `RedirectResponse` to `/docs` per source evidence |
| Q-002 | post-rebuild | `svg.scene-to-svg` (Phase 2-C) | open — default to litellm + Gemini; verify after rebuild |
| Q-003 | before-rebuild | `generation.question-generation` (Phase 3-A) | open — **must answer before Phase 3** |
| Q-004 | before-rebuild | `question-bank.fetch-and-serve` (Phase 5-B) | open — **must answer before Phase 5-B deployment** |
| Q-005 | nice-to-have | `svg.svg-to-png` (Phase 2-A) | open — accept headers as optional; confirm ranges post-rebuild |
| Q-006 | post-rebuild | `image.ai-image-generation` (Phase 2-B) | open — default `dall-e-3` via env var; confirm post-rebuild |

---

## ARIA Remediation Backlog

Six HTML surfaces carry ARIA debt identified during discovery. Address before shipping to any human-facing environment.

| Priority | Surface | Key gap |
|---|---|---|
| High | `image-viewer` | Image `alt` text; SVG `<title>`/`<desc>`; nav link labels |
| High | `question-viewer-page` | Math SVG accessible description; answer list semantics; MathML `aria-label` |
| Medium | `browse-bucket` | Table column `scope`; breadcrumb `<nav aria-label>` |
| Medium | `view-markdown` | Breadcrumb `aria-current`; heading hierarchy |
| Medium | `question-bank-fetcher-topic-page` | Question list semantics; math image alt; PDF link label |
| Low | `question-bank-fetcher-index-page` | Semantic heading elements for topic groups |

Full details in `artifacts/needs-aria.json`.

---

## Visual Identity

All HTML surfaces must use the GitHub-inspired monochrome design system documented in `artifacts/DESIGN.md`:
- Header: dark charcoal `#24292f`
- Page background: light grey `#f6f8fa`
- Content cards: white `#ffffff`
- Link accent: blue `#0969da`
- Monospace stack: `ui-monospace, "Cascadia Code", "Fira Mono"`
- Sans-serif stack: `-apple-system, BlinkMacSystemFont, "Segoe UI"`
- Base font: 16px / 1.6 line-height
- Border radii: 4px (sm), 6px (md)
