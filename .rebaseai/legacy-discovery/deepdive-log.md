# Deep-Dive Log — EduLLM SAT Math Capability Discovery

**Run date:** 2026-06-15  
**Schema:** capability.schema.json (draft-07)  
**Capabilities produced:** 9

---

## Methodology

1. Read `capability.schema.json` to internalize required fields and constraints.
2. Read `capability-sketch.json` — 9 draft capabilities with preliminary evidence refs and splitting decisions.
3. Read `surfaces.json` — 33 surfaces with evidence locators, triggers, and outcomes.
4. Read `DESIGN.md` — visual design system (GitHub-inspired monochrome).
5. Verified key source files via targeted reads of route handlers and CLI entry points in the actual repo under `/home/ubuntu/workdir/rebase-flow-runs/0a374f1bcd2e4666806e3444d89a0614/`.

---

## Capability-by-Capability Notes

### generation.question-generation
- Confirmed `POST /generate` uses `asyncio.gather` for parallel LLM sub-calls and `verify_auth` as a FastAPI dependency.
- `GET /generate` returns a `GenerateInfoResponse` model — not a freeform dict.
- Auth is explicitly optional: `verify_auth` is a no-op when `SERVICE_AUTH_TOKEN` is unset (local dev mode).
- Integrations confirmed: litellm (Gemini + Claude), Langfuse, GCS, InceptBench.

### asset-browser.gcs-browser
- Five surfaces confirmed: `/browse`, `/object/{key}`, `/image/{key}`, `/view/{key}`, `sync_sat_to_gcs.py`.
- `browse` endpoint injects `GcsReader` and `Jinja2Templates` via FastAPI `Depends`.
- `sync_sat_to_gcs.py` supports `--dry-run` flag — not destructive by default.
- All read surfaces share the `GcsReader` abstraction; the sync script uses `GcsUploader`.

### svg.scene-to-svg
- `POST /render` and `POST /image/fromQuestion` confirmed at lines 212 and 547 of `svg_gen/web/app.py`.
- Internal test endpoints (`GET /test/component-tester`, `POST /test/component`) intentionally excluded per `surfaces.json` self_audit notes.
- Microservice port is 8001.

### svg.svg-to-png
- `POST /convert` supports `X-Output-Width` and `X-Background` request headers for rendering customization — confirmed from source.
- Playwright/Chromium is the rendering engine (not a server-side SVG library).
- Microservice port is 8002.

### image.ai-image-generation
- `POST /generate` accepts `GenerateJsonRequest` with `prompt` and `n` fields.
- OpenAI is the sole provider — no litellm abstraction at this layer.
- Microservice port is 8003.

### question-bank.fetch-and-serve
- 8 surfaces: 2 CLI + 6 HTTP endpoints. Splitting was considered and rejected (sketch `should_split: false` with justification).
- PDF endpoints (`/pdf/{slug}` and `/pdf/{slug}/download`) were initially omitted in the surfaces self-audit but are included in the sketch and surfaces.json; they are covered here.
- Server runs on port 8765.
- Data source: `api.curriculum.inceptapi.com`.

### devtools.question-viewer
- Smallest capability: 1 HTTP endpoint + 1 CLI. Port 7890.
- Explicitly a local dev tool; not included in the production Dockerfile or Cloud Run deployment.

### infra.service-health
- `GET /` returns `RedirectResponse` per source (not a plain JSON welcome — the sketch description is slightly imprecise; the actual implementation may redirect to `/docs` or similar).
- `GET /health` returns `HealthResponse` used by Cloud Run liveness probing.

### infra.gcs-upload
- Single CLI surface exposing the `GcsUploader` component.
- Accepts stdin (`-`) as file path in addition to a local file path.
- Acts as a shared library dependency for both the generation pipeline and the sync script.

---

## Splitting Decisions Summary

All 9 capabilities retained from the sketch without splitting. The sketch's `splitting_check` analysis was sound:
- No capability exceeded a threshold that would justify a split (max 8 surfaces, all coherent around a single concern).
- `question-bank.fetch-and-serve` (8 surfaces) was the closest candidate; download + serve form an inseparable workflow.

---

## Evidence Source Distribution

| Source type     | Count |
|-----------------|-------|
| tool (code ref) | 38    |
| llm-supplement  | 8     |
| extra-read      | 0     |
| doc             | 0     |
| human           | 0     |

All `llm-supplement` entries reference `capability-sketch.json` as the ref, which was itself produced by an earlier tool-grounded analysis pass.

---

## Open Questions / Low-Confidence Areas

- **infra.service-health `GET /`**: Source shows `RedirectResponse` — may redirect to `/docs` rather than returning a JSON welcome body. Confidence remains high for the health check; the root endpoint description is approximate.
- **svg.scene-to-svg LLM pipeline internals**: The LLM provider used by svg-gen was not confirmed from source (only described as "LLM provider" in the surfaces audit). Likely litellm based on project patterns, but not verified at the svg-gen level.
