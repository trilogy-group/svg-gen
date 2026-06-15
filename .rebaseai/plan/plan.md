## Overview

The rebuild delivers a from-scratch `svg-gen` service that turns scene/question requests into SVG diagrams via a constraint-based, fully deterministic render pipeline. The LLM is confined to two roles — feasibility check and natural-language → constraint-graph scene emission — while a `ConstraintSolver`, a global label/angle-mark layout optimizer, and a math-based viewBox trim eliminate the recurring legacy bugs (clipping, mis-aligned angle marks, label/line overlap, inconsistent label distances). The legacy HTTP and CLI contracts are preserved verbatim; Playwright survives only as an opt-in `--verify-bbox` CI tool.

## Milestones

Milestones are ordered by dependency. Each milestone is shippable end-to-end against the deterministic test suite. The planner-rewrite milestone (M6) is **gated** on the 30-scene eval in M5; if the gate fails, the contingency work-item under M6 (Proposal 2 anchor-model fallback) replaces the planner work without invalidating M1–M4.

1. **M1 — Service skeleton & HTTP/CLI contract scaffolding.** Stands up the FastAPI app, CLI entry points, health endpoint, scene/question request Pydantic models, and a stub render pipeline returning a typed "not supported" response. No dependencies. Capabilities: *Scene-to-SVG rendering HTTP API* (skeleton only), *svg-gen CLI: render*, *svg-gen CLI: serve*, *svg-gen service health endpoint*.
2. **M2 — Constraint engine.** Implements the constraint vocabulary, residual functions, Newton solver, `SolvedGeometry`, and `UnsatisfiableConstraintsError`. Depends on M1's Pydantic scene model. Capabilities: *Deterministic constraint solver*.
3. **M3 — Layout optimizer, math-trim, and verify-bbox.** Implements the global label/angle-mark optimizer, the math-based viewBox union, and the opt-in `--verify-bbox` CLI mode wired into a CI fixture job. Depends on M2 (consumes `SolvedGeometry`). Capabilities: *Global label & angle-mark layout optimizer*, *Math-based SVG trim / viewBox computation*, *--verify-bbox dev/CI tool (new)*.
4. **M4 — v0 renderer set + deterministic test suite.** Ships the three v0 renderers (`Rectangle`, `Triangle`, `CoordinatePlane`) implementing the constraint-graph renderer contract, plus the deterministic geometric test suite enforcing overlap/clipping/angle-mark/label-collision/consistency rules. Depends on M2 and M3. Capabilities: *Deterministic renderer set (constraint-graph contract)*, *Deterministic geometric test suite (new)*.
5. **M5 — 30-scene LLM eval gate.** Runs the curated 30-scene constraint-emission eval against the candidate LLM and records first-try success rate. Depends on M2–M4 (eval needs the solver + renderers to score "rendered cleanly"). Capabilities: *30-scene LLM constraint-emission eval gate (new)*. **Decision point**: if success ≥ ~85% proceed to M6 primary path; otherwise trigger M6 contingency.
6. **M6 — LLM planner & feasibility, plus question entry point. GATED by M5.** Primary path: implement the constraint-graph planner with retry-on-`UnsatisfiableConstraintsError`, the feasibility short-circuit, and the `POST /image/fromQuestion` endpoint, replacing the M1 stub. Contingency (M5 fail): swap the constraint-graph planner for Proposal 2's anchor-model planner emitting anchored references instead of free-form constraints — the deterministic solver, optimizer, renderers, trim and test suite from M2–M4 are reused unchanged. Capabilities: *LLM-driven feasibility check*, *LLM-driven constraint-graph scene planner*, *Question-to-image HTTP endpoint*, and full activation of *Scene-to-SVG rendering HTTP API*.
7. **M7 — Component coverage expansion.** Ports remaining legacy component types to the constraint-graph renderer contract until launch parity (scope defined by Q4) is met, deletes any temporary "not supported" stubs for those types, and finalises Playwright runtime removal. Depends on M6.

### Capability: Scene-to-SVG rendering HTTP API

- **User actions:** A backend caller (the upstream question-generation service or a developer) POSTs a scene description to `/render` to obtain an SVG.
- **Inputs:** JSON body conforming to the legacy `/render` request schema, parsed into a Pydantic `SceneRequest` whose component payloads use the constraint-graph form (points + relations + label slots, no raw coordinates). `Content-Type: application/json`.
- **Outputs:** HTTP 200 with the rendered SVG (response body shape matches legacy verbatim — same MIME, same wrapping JSON keys where legacy wrapped them).
- **States & edge cases:**
  - Malformed JSON or schema violation → HTTP 422 with a Pydantic validation error body (no LLM involved).
  - Component type not yet ported (pre-M7) → typed "not supported" response (shape per Q2).
  - Feasibility check returns "not supported" → typed "not supported" response (shape per Q2).
  - Constraints are unsatisfiable after the planner's retry budget is exhausted → typed error response (shape per Q3).
  - Solver converges but optimizer cannot place all labels without violating hard constraints → typed error response (shape per Q3).
  - Auth: behaviour depends on Q1.
- **Acceptance criteria:**
  - Given a legacy `/render` request body for a supported component, the new service returns an SVG byte-for-byte equivalent in structure to the legacy response wrapper (visual equivalence governed by Q6).
  - The full deterministic test suite (M4) passes against every response: no element overlap, no clipping outside viewBox, angle marks aligned, no label-line collisions, label distances consistent scene-wide.
  - No browser is launched at runtime for any `/render` call (verified by absence of Playwright import in the runtime dependency tree).
  - Auth enforcement matches the answer to Q1; latency budget matches Q7.
  - For a v0-supported component type, the response SVG passes `--verify-bbox` against its CI fixture (math-bbox within tolerance of browser-bbox).

### Capability: Question-to-image HTTP endpoint

- **User actions:** The upstream generation pipeline POSTs a SAT math question payload to `/image/fromQuestion` and receives an SVG diagram derived from the question.
- **Inputs:** JSON body matching the legacy `ImageFromQuestionRequest` (question text + structured context fields), parsed into Pydantic.
- **Outputs:** HTTP 200 with an SVG asset in the same response shape the legacy endpoint produced.
- **States & edge cases:**
  - Question contains content the feasibility check classifies as not renderable → response per Q8.
  - Question maps cleanly to a v0-supported component → SVG returned; deterministic test suite passes.
  - Question requires a component type not yet ported → response per Q8 (until M7 closes coverage per Q4).
  - Planner returns an unsatisfiable constraint set after retries → response per Q3 (potentially Q8-shaped if Q8 differs from Q3 for this endpoint).
- **Acceptance criteria:**
  - For the held-out scenes in the M5 eval set that the eval marked as successful, the endpoint returns a passing-test-suite SVG on first call (no retry loop visible to the caller beyond the planner's internal budget).
  - The endpoint preserves the legacy request/response field names and types verbatim.
  - Latency budget matches Q7.

### Capability: svg-gen CLI: render

- **User actions:** A developer runs `svg-gen render <scene-file>` to produce an SVG locally without booting the HTTP service; optionally appends `--verify-bbox`.
- **Inputs:** Path to a scene JSON file (constraint-graph form), optional `--output <path>` flag, optional `--verify-bbox` flag.
- **Outputs:** SVG written to stdout (default) or to the `--output` path; exit code 0 on success, non-zero on failure (codes match legacy CLI behaviour where defined).
- **States & edge cases:**
  - File not found / invalid JSON → non-zero exit with a human-readable error on stderr.
  - Unsatisfiable constraints → non-zero exit, error message includes minimal infeasible subset from the solver.
  - `--verify-bbox` set → also launches Chromium, prints the math-vs-browser bbox diff, exits non-zero if diff exceeds tolerance.
- **Acceptance criteria:**
  - Running on every scene fixture in `tests/fixtures/scenes/` produces an SVG that passes the deterministic test suite.
  - `--verify-bbox` passes for every v0 fixture and reports a diff under the configured tolerance.

### Capability: svg-gen CLI: serve

- **User actions:** An operator or Cloud Run container starts the service with `svg-gen serve`.
- **Inputs:** Optional `--port` (default 8001), optional `--host` (default `0.0.0.0`).
- **Outputs:** A long-running FastAPI process bound to the requested host/port; logs to stdout in the legacy format.
- **States & edge cases:**
  - Port already bound → exits non-zero with a clear error.
  - SIGTERM → graceful shutdown within the Cloud Run grace window; no in-flight render is corrupted.
- **Acceptance criteria:**
  - `svg-gen serve` on a fresh container responds 200 to `GET /health` within 5 seconds of process start.
  - No Playwright/Chromium runtime dependency is installed in the container image used by `serve`.

### Capability: LLM-driven feasibility check

- **User actions:** Indirect — callers see the result via `/render` and `/image/fromQuestion`. The check is invoked before the planner runs.
- **Inputs:** The user-facing request (scene description or question payload).
- **Outputs:** Either a `feasible=true` signal to the planner, or a typed "not supported" response surfaced to the caller (shape per Q2).
- **States & edge cases:**
  - LLM provider error / timeout → behaviour per Q3 (treated as a transient pipeline failure).
  - Borderline cases → must fail-closed (treat as not supported) to avoid the planner emitting nonsense scenes; exact threshold for "not supported" governed by Q2.
- **Acceptance criteria:**
  - For every entry in the curated "not supported" fixture set (defined alongside Q2/Q4), the check returns the typed not-supported response without invoking the planner.
  - For every entry in the M5 eval set marked feasible, the check returns `feasible=true`.
  - The feasibility check is the only place the LLM is called outside the planner — verified by a static check in the test suite.

### Capability: LLM-driven constraint-graph scene planner

- **User actions:** Indirect — the planner produces the scene the deterministic pipeline renders.
- **Inputs:** The feasibility-approved request plus, on retry, an `UnsatisfiableConstraintsError` payload containing the minimal infeasible subset from the solver.
- **Outputs:** A Pydantic-validated constraint-graph scene JSON ready for the solver.
- **States & edge cases:**
  - Output fails Pydantic validation → retry with validation errors fed back, up to the retry budget (Q3 fixes the user-visible result when the budget is exhausted).
  - Output validates but solver returns `UnsatisfiableConstraintsError` → retry with the error payload, up to the same budget.
  - Output validates and solver converges → planner is done; downstream pipeline takes over.
  - Contingency (M5 gate fail): planner is replaced by the Proposal 2 anchor-model variant emitting anchored references; downstream stages remain unchanged.
- **Acceptance criteria:**
  - For ≥ ~85% of the M5 eval scenes (or the threshold set in Q5), the planner produces a solver-feasible scene on the first attempt.
  - Retries never exceed the budget set by Q3; on budget exhaustion the user-visible response matches Q3.
  - No coordinate fields appear in any planner output (enforced by Pydantic schema, not by post-hoc check).

### Capability: Deterministic constraint solver

- **User actions:** Indirect — the solver runs inside every `/render` and `/image/fromQuestion` call.
- **Inputs:** A validated constraint-graph scene (points + constraints from the vocabulary: `distance`, `angle`, `rightAngle`, `parallel`, `perpendicular`, `midpoint`, `onLine`, `congruent`).
- **Outputs:** `SolvedGeometry` — every point given exact coordinates, every length/angle exact.
- **States & edge cases:**
  - Constraints feasible → returns `SolvedGeometry` deterministically (same seed, identical output across runs).
  - Constraints infeasible → raises `UnsatisfiableConstraintsError` with the minimal infeasible subset.
  - Constraints under-determined (infinite solutions) → resolves to a deterministic canonical solution using a fixed seed and a documented initial layout heuristic; never random.
  - Newton solver fails to converge within the iteration cap → typed convergence error (treated by the planner as unsatisfiable for retry purposes).
- **Acceptance criteria:**
  - Bit-identical `SolvedGeometry` across two consecutive runs on the same input.
  - On the M2 fixture set (known-good SAT diagrams), solver converges in ≤ 200 iterations on 100% of cases.
  - Every right angle in the output is exactly 90° (within solver tolerance); every declared midpoint is exactly the midpoint (likewise) — verified by the deterministic test suite.

### Capability: Global label & angle-mark layout optimizer

- **User actions:** Indirect — runs after the solver, before SVG emission.
- **Inputs:** `SolvedGeometry` and the planner's declared label slots / angle-mark slots; rendered text bbox estimates.
- **Outputs:** A `LayoutResult` placing every label and angle mark with absolute coordinates and orientations.
- **States & edge cases:**
  - All labels can be placed without violating hard constraints → returns the lowest-cost placement found within the iteration cap.
  - No feasible placement exists for a given label → typed `LayoutInfeasibleError`; user-visible behaviour per Q3.
  - Two labels would overlap → optimizer's joint cost minimisation moves both rather than picking a local winner.
  - Determinism: fixed seed, fixed anneal/gradient schedule — identical input produces identical output.
- **Acceptance criteria:**
  - On the M4 fixture set: zero label-line collisions, zero label-label overlaps, every angle mark on the correct side of its vertex (up/down) — verified by the deterministic test suite.
  - The `w4` consistency cost term causes all side-labels in a single scene to sit within a documented narrow distance band from their line (verified by the consistency check in the test suite).
  - Two runs on the same input produce identical `LayoutResult` byte-for-byte.

### Capability: Math-based SVG trim / viewBox computation

- **User actions:** Indirect — the final viewBox of every returned SVG is computed here.
- **Inputs:** The math bboxes reported by every renderer for its emitted geometry, plus the optimizer's placed label/angle-mark bboxes, plus the configured padding.
- **Outputs:** A viewBox set to `BBox.union(all_inputs) + padding`, applied to the emitted SVG root element.
- **States & edge cases:**
  - Any geometry or label bbox would extend beyond the union → impossible by construction (union covers everything).
  - Padding configured per scene or globally; default value must match what legacy callers visually expect (governed by Q6).
- **Acceptance criteria:**
  - Zero clipping detected by the deterministic test suite on every fixture.
  - `--verify-bbox` reports math-vs-browser bbox diff within tolerance on every v0 fixture.
  - The Playwright/Chromium dependency is not imported anywhere in the runtime call graph reachable from `/render` or `/image/fromQuestion`.

### Capability: Deterministic renderer set (constraint-graph contract)

- **User actions:** Indirect — one renderer per component type emits its SVG fragment.
- **Inputs:** `SolvedGeometry`, the component spec, and a `RenderContext`.
- **Outputs:** SVG fragment string plus the renderer's reported math bbox and its `label_candidates` (for the optimizer).
- **States & edge cases:**
  - Component type supported at this milestone → renders normally.
  - Component type not yet ported → `/render` returns the typed "not supported" response per Q2 (no renderer is invoked).
  - Renderer must contain no geometry computation — it only reads `SolvedGeometry`. Enforced by a static check (no math imports in renderer modules beyond approved helpers).
- **Acceptance criteria:**
  - All v0 renderers (`Rectangle`, `Triangle`, `CoordinatePlane`) pass the deterministic test suite on the M4 fixture set.
  - Adding a new component type requires touching exactly one renderer module, one set of declared constraint kinds, and one set of label slots — no central registry edit beyond a single entry.
  - The v0 launch set of supported component types matches the answer to Q4.

### Capability: svg-gen service health endpoint

- **User actions:** Cloud Run / load balancer probes `GET /health` to confirm the service is alive.
- **Inputs:** None.
- **Outputs:** HTTP 200 with a JSON `HealthResponse` matching the legacy body verbatim.
- **States & edge cases:**
  - Service is up but the LLM provider is unreachable → still returns 200 (health is liveness, not dependency-readiness — matches legacy behaviour).
- **Acceptance criteria:**
  - Returns 200 within 50 ms on a warm process.
  - Body equals the legacy `HealthResponse` payload field-for-field.

### Capability: --verify-bbox dev/CI tool (new)

- **User actions:** A developer or CI job runs `svg-gen render <fixture> --verify-bbox` to compare the math-computed bbox against the actual Chromium-measured bbox.
- **Inputs:** A scene fixture file; optional tolerance flag (default value documented).
- **Outputs:** A diff report on stdout listing per-element bbox deltas; exit code 0 if all within tolerance, non-zero otherwise.
- **States & edge cases:**
  - Chromium not installed locally → tool exits with a clear "install via `playwright install`" instructions message.
  - Diff exceeds tolerance → exit non-zero, CI fails, the offending element is named in the report.
  - This tool is the only place Playwright is touched; runtime code paths never reach it.
- **Acceptance criteria:**
  - Runs against the full M4 fixture set in CI on every PR and is required-green to merge.
  - Adding a new fixture and running `--verify-bbox` locally takes one command and < 1 minute end-to-end.

### Capability: Deterministic geometric test suite (new)

- **User actions:** Run via `pytest` locally and in CI.
- **Inputs:** Rendered SVG outputs from the fixture set, plus the upstream `SolvedGeometry` and `LayoutResult` for each fixture.
- **Outputs:** Pass/fail per check; failures include the offending coordinates and element ids.
- **States & edge cases:**
  - Suite must never call an LLM (enforced by a static guard) and must never launch a browser.
  - Each check is independently runnable so a single failure pinpoints the bug class.
- **Acceptance criteria:**
  - Covers all five required bug classes: element overlap, clipping outside viewBox, mis-aligned angle marks, label-line collisions, label-distance inconsistency.
  - For each bug class, the suite contains at least one positive (passing scene) and one negative (intentionally broken scene that must fail) fixture.
  - Runs under 30 seconds on the M4 fixture set on a CI runner.

### Capability: 30-scene LLM constraint-emission eval gate (new)

- **User actions:** The team runs the eval once before committing to M6's primary path; CI can re-run on demand when prompts or models change.
- **Inputs:** A curated set of 30 input scenes (composition and ownership governed by Q5) plus the candidate LLM provider/model.
- **Outputs:** A first-try success rate, a per-scene pass/fail table, and a recorded artifact (JSON) committed to the repo.
- **States & edge cases:**
  - Success rate ≥ ~85% (exact threshold per Q5) → M6 primary path is approved.
  - Success rate < threshold → M6 contingency activates: the Proposal 2 anchor-model planner replaces the constraint-graph planner; M2–M4 deliverables are reused unchanged.
  - LLM provider unreachable or quota-exhausted during eval → eval is marked invalid and rerun, never silently passed.
- **Acceptance criteria:**
  - Eval is reproducible: rerunning produces an identical pass/fail table given the same provider, model, seed, and prompt.
  - Definition of a "successful" emission per scene is documented and matches the answer to Q5.
  - The eval artifact is committed and dated; the M6 path chosen is recorded in the same commit.

## Open business questions

- **Q1** — gates the auth-related acceptance criteria of *Scene-to-SVG rendering HTTP API* and *Question-to-image HTTP endpoint*.
- **Q2** — gates the "not supported" response shape used by *Scene-to-SVG rendering HTTP API*, *Question-to-image HTTP endpoint*, *LLM-driven feasibility check*, and *Deterministic renderer set*.
- **Q3** — gates the user-visible error response on retry-budget exhaustion / unsatisfiable constraints / layout infeasibility across *Scene-to-SVG rendering HTTP API*, *LLM-driven constraint-graph scene planner*, *LLM-driven feasibility check*, and *Global label & angle-mark layout optimizer*.
- **Q4** — gates the v0 launch component-type scope of *Deterministic renderer set* and M7's exit criteria.
- **Q5** — gates the curation, success definition, and threshold of the *30-scene LLM constraint-emission eval gate*, and through it the M6 path choice.
- **Q6** — gates the visual-equivalence acceptance criterion of *Scene-to-SVG rendering HTTP API* and the padding default of *Math-based SVG trim / viewBox computation*.
- **Q7** — gates the latency acceptance criteria of *Scene-to-SVG rendering HTTP API* and *Question-to-image HTTP endpoint*.
- **Q8** — gates the "diagram cannot be produced" response of *Question-to-image HTTP endpoint*.
