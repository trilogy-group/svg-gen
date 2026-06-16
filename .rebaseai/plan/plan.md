## Overview

The rebuild delivers a from-scratch `svg-gen` service that turns scene/question requests into SVG diagrams via a constraint-based, fully deterministic render pipeline. The LLM is confined to a single role — natural-language description → constraint-graph scene JSON — while a `ConstraintSolver`, a global label/angle-mark layout optimizer, and a math-based viewBox trim eliminate the recurring legacy bugs (clipping, mis-aligned angle marks, label/line overlap, inconsistent label distances). The HTTP and CLI contracts match the prior service; Playwright survives only as an opt-in `--verify-bbox` CI tool.

## Milestones

Milestones are ordered by dependency. Each milestone is shippable end-to-end against the deterministic test suite. The planner-rewrite milestone (M6) is **gated** on the 30-scene eval in M5; if the gate fails, the contingency under M6 (Proposal 2 anchor-model planner) replaces the planner work without invalidating M1–M4.

1. **M1 — Service skeleton & HTTP/CLI contract scaffolding.** Stands up the FastAPI app, CLI entry points, `/health`, scene/question request Pydantic models, and a stub `/render` returning a typed `NotImplemented` response. No dependencies. Capabilities: *Scene-to-SVG rendering HTTP API* (skeleton only), *svg-gen CLI: render*, *svg-gen CLI: serve*, *svg-gen service health endpoint*.
2. **M2 — Constraint engine.** Implements the constraint vocabulary, residual functions, Newton solver, `SolvedGeometry`, and `UnsatisfiableConstraintsError`. Depends on M1's Pydantic scene model. Capabilities: *Deterministic constraint solver*.
3. **M3 — Layout optimizer, math-trim, and verify-bbox.** Implements the global label/angle-mark optimizer, the math-based viewBox union, and the opt-in `--verify-bbox` CLI mode wired into a CI fixture job. Depends on M2 (consumes `SolvedGeometry`). Capabilities: *Global label & angle-mark layout optimizer*, *Math-based SVG trim / viewBox computation*, *--verify-bbox dev/CI tool (new)*.
4. **M4 — v0 renderer set + deterministic test suite.** Ships the three v0 renderers (`Rectangle`, `Triangle`, `CoordinatePlane`) implementing the constraint-graph renderer contract, plus the deterministic geometric test suite enforcing overlap/clipping/angle-mark/label-collision/consistency rules. Depends on M2 and M3. Capabilities: *Deterministic renderer set (constraint-graph contract)*, *Deterministic geometric test suite (new)*.
5. **M5 — 30-scene LLM eval gate.** Runs the curated 30-scene constraint-emission eval against the candidate LLM and records first-try success rate. Depends on M2–M4. Capabilities: *30-scene LLM constraint-emission eval gate (new)*. **Decision point**: if success ≥ 85% proceed to M6 primary path; otherwise trigger M6 contingency.
6. **M6 — LLM planner & question entry point. GATED by M5.** Primary path: implement the constraint-graph planner with retry-on-validation-error and retry-on-`UnsatisfiableConstraintsError`, plus `POST /image/fromQuestion`, replacing the M1 stub. The planner is also responsible for emitting a typed `UnsupportedRequest` outcome when it cannot represent the question as a constraint-graph scene — there is no separate feasibility LLM call. Contingency (M5 fail): swap the constraint-graph planner for Proposal 2's anchor-model planner emitting anchored references — the deterministic solver, optimizer, renderers, trim and test suite from M2–M4 are reused unchanged. Capabilities: *LLM-driven constraint-graph scene planner*, *Question-to-image HTTP endpoint*, and full activation of *Scene-to-SVG rendering HTTP API*.
7. **M7 — Component coverage expansion to launch parity.** Ports the remaining component types listed under *Deterministic renderer set* below to the constraint-graph renderer contract, removes the temporary `UnknownComponent` stub for those types, and finalises Playwright runtime removal. Depends on M6.

### Capability: Scene-to-SVG rendering HTTP API

- **User actions:** A backend caller POSTs a scene description to `/render` to obtain an SVG.
- **Inputs:** JSON body conforming to the `SceneV1` envelope; component payloads use the constraint-graph form (points + relations + label slots, no raw coordinates). `Content-Type: application/json`. `/render` does not run a feasibility check.
- **Outputs:** HTTP 200 with `Content-Type: image/svg+xml` and the SVG bytes in the response body.
- **States & edge cases:**
  - Malformed JSON / schema violation → HTTP **400** with body `{"detail": {"error": "InvalidScene", "message": "Scene validation failed", "details": [<pydantic errors>]}}`.
  - Component type not yet ported (pre-M7) or not in the v0 renderer set → HTTP **500** with body `{"detail": {"error": "UnknownComponent", "message": "Component type not supported", "details": {"component_type": "<name>"}}}`.
  - Constraint solver returns `UnsatisfiableConstraintsError` after the planner's retry budget is exhausted (or directly, for `/render` scenes that already arrive unsatisfiable) → HTTP **500** with body `{"detail": {"error": "SVGGenerationFailed", "message": "<underlying solver text incl. minimal infeasible subset>", "details": {...}}}`.
  - Layout optimizer cannot place labels without violating hard constraints → HTTP **500** with body `{"detail": {"error": "SVGGenerationFailed", "message": "<underlying layout text>", "details": {...}}}`.
  - Any other unhandled exception → HTTP **500** `{"detail": {"error": "InternalError", "message": "<text>", "details": {...}}}`.
  - Never returns a placeholder / fallback SVG; failure is always a typed error envelope.
  - Auth: no authentication on `/render`. Any caller that can reach the bound port may call it. `/health`, `/render`, and `/image/fromQuestion` are open.
- **Acceptance criteria:**
  - For every v0-supported component (Rectangle, Triangle, CoordinatePlane) the endpoint returns HTTP 200 with `image/svg+xml` body and a valid SVG.
  - The full deterministic test suite passes against every successful response: no element overlap, no clipping outside viewBox, angle marks aligned, no label-line collisions, label distances consistent scene-wide.
  - No browser is launched at runtime; the Playwright/Chromium package is absent from the runtime image's installed dependencies.
  - Latency: p95 < 1000 ms on a warm container against the v0 fixture set (no LLM is invoked on `/render`).
  - `/render` returns HTTP 200 for every v0 scene fixture in `tests/fixtures/scenes/`; returns 400 InvalidScene for every fixture in `tests/fixtures/invalid/`; returns 500 UnknownComponent for every fixture in `tests/fixtures/unknown_component/`.
  - The response SVG passes `--verify-bbox` against its CI fixture (math-bbox within 1.0 px of browser-bbox).

### Capability: Question-to-image HTTP endpoint

- **User actions:** A backend pipeline POSTs a SAT-style math question payload to `/image/fromQuestion` and receives an SVG diagram derived from the question.
- **Inputs:** JSON body matching the `ImageFromQuestionRequest` schema (question text + structured context fields), parsed into Pydantic.
- **Outputs:** HTTP 200 with `Content-Type: image/svg+xml` and SVG bytes; also sets the `X-Duration-Ms` response header carrying total wall-clock processing time in milliseconds.
- **States & edge cases:**
  - Planner declares the question unrenderable (it emits a typed `unsupported` outcome instead of a scene, because no constraint-graph representation fits) → HTTP **400** with body
    ```json
    {"detail": {"error": "UnsupportedRequest",
                "message": "Request is not supported by the deterministic SVG renderer. Fallback image generation is disabled.",
                "details": {"planner_reasoning": "<free-text LLM explanation>"}}}
    ```
  - Planner LLM call itself errors / times out → HTTP **500** `{"detail": {"error": "PlannerCallFailed", "message": "<provider text>", "details": {...}}}`.
  - Question maps cleanly to a v0-supported component → SVG returned; deterministic test suite passes.
  - Question requires a component type not yet ported (pre-M7) → HTTP **500** `{"detail": {"error": "UnknownComponent", "message": "Component type not supported", "details": {"component_type": "<name>"}}}`.
  - Planner exhausts the retry budget of **3 total attempts** (initial + 2 retries) without producing a solver-feasible scene → HTTP **500** `{"detail": {"error": "SVGGenerationFailed", "message": "Generated scene failed validation after 3 attempts", "details": {...}}}`.
  - Reserved `NotImplemented` agent-error type → HTTP **501** with `{"detail": {"error": "NotImplemented", "message": "<text>", "details": {...}}}`.
  - Any other unhandled exception → HTTP **500 InternalError**.
  - Never returns a placeholder SVG, an empty SVG, or a `{"image": null}` sentinel — the only success contract is `200 image/svg+xml`. Upstream decides whether to drop the diagram or fail the question when it sees a 4xx/5xx.
  - Auth: none.
- **Acceptance criteria:**
  - For every question in the M5 eval set that was marked successful, the endpoint returns 200 with `image/svg+xml` and an SVG that passes the deterministic test suite.
  - The response envelope on failure matches the shape above field-for-field; callers can branch on `detail.error` to distinguish the typed cases.
  - `X-Duration-Ms` is set on every response (success and failure).
  - Latency: p95 < 30000 ms on a warm container across the M5 eval set; per-LLM-attempt timeout 10000 ms so the worst case (3 planner attempts) stays bounded.
  - When the planner emits a scene (does not flag the request as unsupported) but the retry budget is exhausted, the response body's `error` discriminator is `SVGGenerationFailed` (not `UnsupportedRequest`), so upstream can distinguish "won't render" from "couldn't render".

### Capability: svg-gen CLI: render

- **User actions:** A developer runs `svg-gen render <scene-file>` to produce an SVG locally without booting the HTTP service; optionally appends `--verify-bbox`.
- **Inputs:** Path to a scene JSON file (constraint-graph form), optional `--output <path>` flag, optional `--verify-bbox` flag.
- **Outputs:** SVG written to stdout (default) or to the `--output` path; exit code 0 on success.
- **States & edge cases:**
  - File not found / invalid JSON → exit code 2, human-readable error on stderr (mirrors `/render`'s 400 InvalidScene body in stderr form).
  - Unsatisfiable constraints → exit code 1, error message includes the minimal infeasible subset from the solver.
  - Layout infeasible → exit code 1, error message names the labels that could not be placed.
  - Unknown component type → exit code 1, error message names the component type.
  - `--verify-bbox` set → also launches Chromium, prints the math-vs-browser bbox diff, exits 1 if any per-element diff exceeds the tolerance (default 1.0 px).
- **Acceptance criteria:**
  - Running on every scene fixture in `tests/fixtures/scenes/` produces an SVG that passes the deterministic test suite and exit code 0.
  - `--verify-bbox` passes (exit 0) for every v0 fixture and reports a diff under 1.0 px per element.

### Capability: svg-gen CLI: serve

- **User actions:** An operator or container runtime starts the service with `svg-gen serve`.
- **Inputs:** Optional `--port` (default 8001), optional `--host` (default `0.0.0.0`).
- **Outputs:** A long-running FastAPI process bound to the requested host/port; logs to stdout as JSON lines.
- **States & edge cases:**
  - Port already bound → exits non-zero with a clear error.
  - SIGTERM → graceful shutdown within 10 s; no in-flight render is corrupted.
- **Acceptance criteria:**
  - `svg-gen serve` on a fresh container responds 200 to `GET /health` within 5 seconds of process start.
  - No Playwright/Chromium runtime dependency is installed in the container image used by `serve`.

### Capability: LLM-driven constraint-graph scene planner

- **User actions:** Indirect — runs only inside `/image/fromQuestion`. The planner is the sole LLM call in the entire service: it both decides whether the question can be expressed as a constraint-graph scene and, if so, emits that scene.
- **Inputs:** The question payload plus, on retry, a typed error payload containing either Pydantic validation errors or the solver's minimal infeasible subset.
- **Outputs:** Either (a) a Pydantic-validated constraint-graph scene JSON ready for the solver, or (b) a typed `UnsupportedRequest` outcome carrying free-text reasoning. The two are distinct branches of a single tool/output schema; the LLM picks one per call.
- **States & edge cases:**
  - LLM emits `unsupported` → surfaces as HTTP **400 UnsupportedRequest** per the *Question-to-image HTTP endpoint* contract; reasoning is placed in `details.planner_reasoning` verbatim. No fixed-catalogue mapping. Retries are **not** attempted on `unsupported` (the LLM has decided).
  - LLM emits a scene that fails Pydantic validation → retry with validation errors fed back; counted toward the budget.
  - LLM emits a scene that validates but the solver returns `UnsatisfiableConstraintsError` → retry with the error payload; counted toward the budget.
  - LLM emits a scene that validates and the solver converges → planner is done.
  - LLM provider error / timeout on any attempt → HTTP **500 PlannerCallFailed** with the provider text in `message`. Per-attempt LLM timeout is 10000 ms.
  - **Retry budget: 3 total attempts (initial + 2 retries)**. On exhaustion the caller sees `500 SVGGenerationFailed` with `message: "Generated scene failed validation after 3 attempts"`.
  - Borderline / unsure: planner must fail-closed — if it cannot commit to a scene, it must emit `unsupported`.
  - `/render` does **not** call the planner.
  - Contingency (M5 gate fail at < 85% first-try success): planner is replaced by the Proposal 2 anchor-model variant emitting anchored references; everything downstream is unchanged.
- **Acceptance criteria:**
  - On the M5 eval set the planner produces a solver-feasible scene on the first attempt for ≥ 85% of scenes when the primary constraint-graph path is shipped.
  - Retries never exceed 3 total attempts; the count is logged.
  - The planner output's Pydantic schema rejects any field shaped like a coordinate (no `x`, `y`, `coordinates`, `vertices: [[number, number], …]`), enforced at schema time rather than by post-hoc check.
  - For every entry in `tests/fixtures/unsupported_questions/`, the planner emits `unsupported` on the first attempt and the caller receives the 400 envelope without further retries.
  - The planner is the **only** place the LLM client is imported, verified by a static guard test that fails if any other module imports it.

### Capability: Deterministic constraint solver

- **User actions:** Indirect — runs inside every `/render` and `/image/fromQuestion` call, and is exposed standalone as `svg-gen solve <scene-file>` for debugging.
- **Inputs:** A validated constraint-graph scene (points + constraints from the vocabulary: `distance`, `angle`, `rightAngle`, `parallel`, `perpendicular`, `midpoint`, `onLine`, `congruent`).
- **Outputs:** `SolvedGeometry` — every point given exact coordinates, every length/angle exact.
- **States & edge cases:**
  - Feasible → returns `SolvedGeometry` deterministically (same input ⇒ identical output bytes across runs).
  - Infeasible → raises `UnsatisfiableConstraintsError` carrying the minimal infeasible subset.
  - Under-determined (infinite solutions) → resolves to a deterministic canonical solution using a fixed seed and a documented initial layout (points placed evenly around a unit circle).
  - Newton solver does not converge within the iteration cap → typed convergence error mapped by the pipeline to the same retry/error path as `UnsatisfiableConstraintsError`.
- **Acceptance criteria:**
  - Bit-identical `SolvedGeometry` JSON output across two consecutive runs on the same input.
  - On the M2 fixture set the solver converges in ≤ 200 iterations on 100% of cases.
  - Every right angle in the output is exactly 90° within solver tolerance (1e-6); every declared midpoint is exactly the midpoint within the same tolerance.

### Capability: Global label & angle-mark layout optimizer

- **User actions:** Indirect — runs after the solver, before SVG emission.
- **Inputs:** `SolvedGeometry` and the planner's declared label slots and angle-mark slots; renderer-supplied text bbox estimates.
- **Outputs:** A `LayoutResult` placing every label and angle mark with absolute coordinates and orientations.
- **States & edge cases:**
  - All labels placeable → returns the lowest-cost placement found within the iteration cap.
  - No feasible placement → typed `LayoutInfeasibleError`; surfaces to the caller as 500 SVGGenerationFailed with the underlying message.
  - Two labels would overlap → joint cost minimisation moves both rather than locally winning one.
  - Determinism: fixed seed, fixed anneal/gradient schedule.
- **Acceptance criteria:**
  - On the M4 fixture set: zero label-line collisions, zero label-label overlaps, every angle mark on the correct side of its vertex (up/down) — verified by the deterministic test suite.
  - The `w4` consistency cost term keeps all side-labels in a single scene within a documented narrow distance band of their line (verified by the consistency check in the test suite, default band: max-deviation ≤ 25% of mean).
  - Two runs on the same input produce identical `LayoutResult` bytes.

### Capability: Math-based SVG trim / viewBox computation

- **User actions:** Indirect — the final viewBox of every returned SVG is computed here.
- **Inputs:** The math bboxes reported by every renderer for its emitted geometry, plus the optimizer's placed label/angle-mark bboxes, plus padding.
- **Outputs:** A viewBox set to `BBox.union(all_inputs) + padding`, applied to the emitted SVG root element.
- **States & edge cases:**
  - Default padding constants (matching prior service to keep callers' visual expectations stable): `SVG_TRIM_PADDING = 14.0` applied symmetrically (`(x − 14, y − 14, w + 28, h + 28)`); container-layout constants `LABEL_SLOT_PADDING = 5.0`, `TOP_ALIGNED_BOTTOM_PADDING = 10.0`, container `_TOP_PADDING = _SIDE_PADDING = 20.0`. Callers cannot override padding on the HTTP path; there is no `padding` query/body parameter.
  - Visual output is not pixel-equivalent to any prior implementation — there is no contractual stroke colour, font, or label size in the HTTP contract. Callers receive `image/svg+xml` and consume it as opaque content. The deterministic test suite (not pixel comparison) is the visual gate.
- **Acceptance criteria:**
  - Zero clipping detected by the deterministic test suite on every fixture.
  - `--verify-bbox` reports math-vs-browser bbox diff within 1.0 px per element on every v0 fixture.
  - The Playwright/Chromium dependency is not imported anywhere in the runtime call graph reachable from `/render` or `/image/fromQuestion`.
  - viewBox values for a given fixture are byte-identical across two consecutive runs.

### Capability: Deterministic renderer set (constraint-graph contract)

- **User actions:** Indirect — one renderer per component type emits its SVG fragment.
- **Inputs:** `SolvedGeometry`, the component spec, and a `RenderContext`.
- **Outputs:** SVG fragment string, the renderer's reported math bbox, and its `label_candidates`.
- **States & edge cases:**
  - Component type supported at this milestone → renders normally.
  - Component type not yet ported → the request fails with `UnknownComponent` per the *Scene-to-SVG rendering HTTP API* contract (no renderer invoked).
  - Renderer modules must contain no geometry computation — they only read `SolvedGeometry`. Enforced by a static check (renderer modules may not import `math` beyond an approved helper list).
- **v0 launch component scope** (must work at end of M4, no `UnknownComponent` allowed): `rectangle`, `triangle`, `coordinatePlane`.
- **Launch-parity component scope** (must work at end of M7, no `UnknownComponent` allowed):
  - Polygons / closed shapes: `rectangle`, `square`, `triangle`, `circle`, `point`, `parallelogram`, `trapezoid`, `rhombus`, `kite`, `pentagon`, `concavePentagon`, `hexagon`, `heptagon`, `octagon`, `nonagon`, `decagon`, `starPolygon`, `quadrilateral`, `partitionedShape`.
  - Angles & clocks: `angleDiagram`, `directedAngleDiagram`, `directedAngleDiagramTicks`, `clock`, `digitalClock`.
  - Measurement & manipulatives: `numberLine`, `faultyNumberLine`, `ruler`, `alignedLengthComparison`, `protractor`, `array`, `tree`, `counterChange`, `gridRectangle`, `gridSquare`, `multiplicationAreaModel`, `rectilinearShape`, `coordinatePlane`, `symmetryFigure`.
  - Charts, tables, 3D, nets: `table`, `data_table`, `two_row_table`, `barGraph`, `histogram`, `pictureGraph`, `linePlot`, `faultyLinePlot`, `faultyCircleGraph`, `boxWhiskerPlot`, `lineRelationshipDiagram`, `placeValueBlocks`, `longPlaceValueArithmetic`, `longDivision`, `isometricPrism`, `isometricSolid`, `triangularPrism`, `vennDiagram`, `cubeNet`, `rectangularPrismNet`, `triangularPrismNet`, `rectangularPyramidNet`.
- **Acceptance criteria:**
  - All v0 renderers pass the deterministic test suite on the M4 fixture set.
  - Adding a new component type requires touching exactly one renderer module, one set of declared constraint kinds, and one set of label slots — no central registry edit beyond a single entry.
  - At end of M7 every name in the launch-parity list above renders successfully through `/render` for at least one canonical fixture per type, and `UnknownComponent` is no longer returned for any name in the list.

### Capability: svg-gen service health endpoint

- **User actions:** Container runtime / load balancer probes `GET /health` to confirm the service is alive.
- **Inputs:** None. No auth required.
- **Outputs:** HTTP 200 with `{"status": "ok"}` body.
- **States & edge cases:**
  - Service is up but the LLM provider is unreachable → still returns 200 (health is liveness, not dependency-readiness).
- **Acceptance criteria:**
  - Returns 200 within 50 ms on a warm process.
  - Body equals `{"status": "ok"}` byte-for-byte.

### Capability: --verify-bbox dev/CI tool (new)

- **User actions:** A developer or CI job runs `svg-gen render <fixture> --verify-bbox` to compare the math-computed bbox against the actual Chromium-measured bbox.
- **Inputs:** A scene fixture file; optional `--bbox-tolerance` flag (default 1.0 px).
- **Outputs:** A diff report on stdout listing per-element bbox deltas; exit code 0 if all within tolerance, non-zero otherwise.
- **States & edge cases:**
  - Chromium not installed locally → tool exits with a clear "install via `playwright install chromium`" instructions message and exit code 3.
  - Diff exceeds tolerance → exit 1, CI fails, the offending element is named in the report.
  - This tool is the only place Playwright is touched; runtime code paths never reach it (static guard test).
- **Acceptance criteria:**
  - Runs against the full M4 fixture set in CI on every PR and is required-green to merge.
  - Adding a new fixture and running `--verify-bbox` locally takes one command and < 60 s end-to-end.

### Capability: Deterministic geometric test suite (new)

- **User actions:** Run via `pytest` locally and in CI.
- **Inputs:** Rendered SVG outputs from the fixture set, plus the upstream `SolvedGeometry` and `LayoutResult` for each fixture.
- **Outputs:** Pass/fail per check; failures include the offending coordinates and element ids.
- **States & edge cases:**
  - Suite must never call an LLM (enforced by a static guard) and must never launch a browser.
  - Each check is independently runnable so a single failure pinpoints the bug class.
- **Acceptance criteria:**
  - Covers all five bug classes: element overlap, clipping outside viewBox, mis-aligned angle marks, label-line collisions, label-distance inconsistency.
  - For each bug class, the suite contains at least one positive (passing scene) and one negative (intentionally broken scene that must fail) fixture.
  - Runs under 30 s on the M4 fixture set on a CI runner.

### Capability: 30-scene LLM constraint-emission eval gate (new)

- **User actions:** The team runs the eval once before committing to M6's primary path; CI can re-run on demand when prompts or models change.
- **Inputs:** A curated set of 30 questions plus the candidate LLM provider/model.
  - **Curation rule** (defined for this rebuild since no prior process exists): the 30 scenes are sampled from the upstream question source's recent traffic such that each v0 component type contributes ≥ 5 scenes and the remaining slots cover at least 5 distinct launch-parity component types. Sampling seed and snapshot timestamp are committed alongside the eval artifact.
- **Outputs:** A first-try success rate, a per-scene pass/fail table, and a recorded JSON artifact committed to `.rebaseai/plan/eval/30-scene-{date}.json`.
- **States & edge cases:**
  - **Success definition per scene**: (a) the planner emits a Pydantic-validating constraint-graph scene (not an `unsupported` outcome) on the first attempt, (b) the solver converges on that scene, and (c) the rendered SVG passes the deterministic test suite end-to-end. All three required.
  - **Threshold: ≥ 85%** first-try success on the 30-scene set ⇒ M6 primary path approved; **< 85%** ⇒ M6 contingency (anchor-model planner).
  - LLM provider unreachable / quota-exhausted during eval → eval is marked invalid and rerun; the artifact is not committed in invalid state.
  - **Sign-off**: rebuild tech lead records the path choice in the same commit as the artifact.
- **Acceptance criteria:**
  - Eval is reproducible: rerunning produces an identical pass/fail table given the same provider, model, seed, prompt, and snapshot.
  - The committed artifact contains: provider id, model id, prompt hash, seed, snapshot date, per-scene input + result, aggregate success rate, threshold (0.85), and recorded path choice.
  - The M6 work-item references the artifact and chosen path before any planner code is written.

## Open business questions

All resolved.
