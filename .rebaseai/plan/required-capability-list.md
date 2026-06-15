## Required capabilities

The selected proposal (Proposal 3 — Aggressive rewrite) scopes the rebuild to the legacy `svg-gen` component, recast as a constraint-based diagram engine. The HTTP contract and CLI surface of legacy `svg-gen` are preserved; the rest of the legacy estate is out of scope (see dropped list). Capabilities below are what the new repo must deliver.

### Scene-to-SVG rendering HTTP API
A user-facing HTTP service that accepts a scene description (JSON body) on `POST /render` and returns a rendered SVG, with the legacy request/response contract preserved verbatim. Derives from legacy capability `svg.scene-to-svg` (surfaces `svg-gen-post-render`, `svg-gen-get-health`). Required because the proposal explicitly locks "HTTP contract unchanged" so existing callers of svg-gen continue to work after the rewrite.

### Question-to-image HTTP endpoint
A `POST /image/fromQuestion` endpoint that takes a SAT math question payload and returns a rendered diagram (SVG asset), preserving the legacy request/response shape. Derives from legacy capability `svg.scene-to-svg` (surface `svg-gen-post-image-from-question`). Required because the proposal keeps the "feasibility → plan → render pipeline shape" and the question-shaped entry point is the primary integration used by the upstream generation pipeline.

### svg-gen CLI: render
A `svg-gen render` command-line entry point that takes a scene description file and writes an SVG locally, used by developers and CI without needing the HTTP service. Derives from legacy capability `svg.scene-to-svg` (surface `cli-svg-gen-render`). Required because the proposal explicitly lists "the CLI surface" among what is kept, and local rendering is how developers iterate on scenes and exercise the deterministic pipeline.

### svg-gen CLI: serve
A `svg-gen serve` command that boots the HTTP service (default port 8001) so the component runs as a standalone microservice with the same lifecycle as today. Derives from legacy capability `svg.scene-to-svg` (surface `cli-svg-gen-serve`). Required because the proposal keeps the microservice deployment shape and the CLI is how operators and Cloud Run start the service.

### LLM-driven feasibility check
An LLM-backed step that decides whether a requested question/scene is renderable and short-circuits with a typed "not supported" response when it is not. Derives from legacy capability `svg.scene-to-svg` (LLM pipeline entry point). Required because the proposal locks "LLM is used ONLY for: feasibility check ... and natural-language → JSON scene spec" — feasibility is one of only two permitted LLM uses and gates the rest of the deterministic pipeline.

### LLM-driven constraint-graph scene planner
An LLM-backed planner that turns the input request into a constraint-graph scene JSON (points + relations + label slots), with retry feedback on `UnsatisfiableConstraintsError`. Derives from legacy capability `svg.scene-to-svg` (legacy planner pipeline), reshaped per the proposal. Required because the proposal designates this as the *only other* permitted LLM role, and it is the source of the scene model that the deterministic render pipeline consumes.

### Deterministic constraint solver
A `ConstraintSolver` that takes the planner's declared relations (distance, angle, rightAngle, parallel, perpendicular, midpoint, onLine, congruent) and produces `SolvedGeometry` with exact coordinates via a Newton residual solver. Derives from legacy capability `svg.scene-to-svg` (legacy `geometry/resolver.py` + per-type validators, generalized). Required because Pillar B locks the constraint graph as the single source of truth and the solver is what makes "right angles that aren't 90°" and similar planner inconsistencies un-expressible.

### Global label & angle-mark layout optimizer
A deterministic, seed-fixed optimizer that places **all** labels and angle marks together in one pass over the solved geometry, minimizing a cost function whose `w4` term enforces label-distance consistency scene-wide. Derives from legacy capability `svg.scene-to-svg` (legacy per-element label placement), reshaped per the proposal. Required because Pillar C locks global, joint placement as the mechanism that kills the "labels overlapping lines" and "inconsistent label distances" bug classes the proposal explicitly targets.

### Math-based SVG trim / viewBox computation
A deterministic viewBox computation that unions the math bboxes reported by each renderer plus the optimizer's placed label bboxes plus padding, with zero browser involvement at runtime. Derives from legacy capability `svg.scene-to-svg` (legacy Playwright-based `svg_trim`), reshaped. Required because the proposal locks "production trim is pure math" and the closed-form union is the mechanism that eliminates the "images getting clipped off" recurring bug while removing Playwright from the hot path.

### Deterministic renderer set (constraint-graph contract)
A set of per-component renderers (`Rectangle`, `Triangle`, `CoordinatePlane` in v0, others ported in later phases) that implement `declare_constraints` / `draw` / `label_candidates` and contain **no** geometry computation, reading exact coordinates from `SolvedGeometry`. Derives from legacy capability `svg.scene-to-svg` (legacy renderer set). Required because the proposal locks "renderers become much thinner: they don't compute anything, they read the solved graph" — they are the means by which the constraint model becomes SVG output.

### svg-gen service health endpoint
A `GET /health` endpoint on the svg-gen service used by Cloud Run / load balancers for liveness probing. Derives from legacy capability `svg.scene-to-svg` (surface `svg-gen-get-health`). Required because the HTTP-contract lock covers the existing health surface and the service still needs to be deployable behind Cloud Run with the same probe configuration.

### --verify-bbox dev/CI tool (new)
A new `svg-gen render --verify-bbox` CLI mode that launches Chromium opt-in to measure the real browser bbox and diff it against the math-bbox, for use in a CI fixture job. Derives from: **new** (introduced by the proposal). Required because the proposal explicitly locks this as the single sanctioned home for Playwright post-rewrite, and it is how CI guarantees that constraint-solver/optimizer changes never silently regress the math-trim bounds.

### Deterministic geometric test suite (new)
A test suite of pure geometric/bbox checks that deterministically detects element overlap, clipping outside viewBox, mis-aligned angle marks, label-line collisions, and label-distance inconsistency, with no LLM in the test path. Derives from: **new** (introduced by the proposal / additional guidance). Required because the additional guidance locks this as a deliverable and it is the only mechanism that converts the proposal's targeted bug classes into hard CI gates.

### 30-scene LLM constraint-emission eval gate (new)
A reproducible 30-scene evaluation harness run **before** the planner-rewrite phase (Phase 5), measuring first-try success rate of the LLM at emitting valid constraint-graph scenes, with a documented fallback to Proposal 2's anchor model if success rate is below ~85%. Derives from: **new** (introduced by R1 mitigation in the proposal). Required because the additional guidance explicitly gates Phase 5 on this eval and designates the anchor-model contingency as a planned work-item rather than an ad-hoc rescue.
