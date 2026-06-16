# Business rule answers

These answers describe what the **current** legacy `svg-gen` service actually does at its HTTP surface. They are derived from the running code, not the docs. Where the legacy code does not encode a rule, I say so explicitly — those are product calls the rebuild still needs to make.

---

### Q1 — Should the new svg-gen service require an auth token on its HTTP endpoints?

**Today's legacy behaviour:** The legacy `svg-gen` HTTP service has **no authentication of any kind** on any endpoint. `/render`, `/image/fromQuestion`, `/health`, `/test/component`, and `/test/component-tester` are all reachable by any caller that can reach the port. There is no shared-token check, no per-caller credential, no `Authorization` header parsing, and no 401/403 path.

(The optional shared-token check the question refers to lives in a different service — the upstream SAT-Math API — and is not present in svg-gen itself.)

**What the new service should do — product decision needed:**

- **Should `/render` and `/image/fromQuestion` require an auth token in production?** Legacy says "no." If the rebuild should change that, the product owner needs to decide.
- **Same shared token or per-caller credentials?** Not encoded anywhere in the legacy code; this is a green-field product call.
- **401 vs 403, and message text?** Not encoded; this is a green-field product call.
- **`/health` always reachable?** In the legacy service it is unconditionally open and returns `{"status": "ok"}` with HTTP 200. Keeping it open is consistent with how Cloud Run probes consume it today.

I cannot answer the "should" parts from code alone; the product owner must lock the policy.

---

### Q2 — When the service decides a request is "not supported", what should the caller see?

**Today's legacy behaviour:** "Not supported" is **a 4xx error**, not a typed-200 sentinel, and the shape differs slightly by endpoint and by the kind of "not supported":

- `POST /image/fromQuestion` — when the upstream feasibility check decides the request cannot be expressed as a deterministic SceneV1, the caller sees **HTTP 400** with a JSON body of the shape:
  ```json
  {
    "detail": {
      "error": "UnsupportedRequest",
      "message": "Request is not supported by the deterministic SVG renderer. Fallback image generation is disabled.",
      "details": { "feasibility_reasoning": "<free-text reasoning from the feasibility step>" }
    }
  }
  ```
  The `message` is a fixed string. The `details.feasibility_reasoning` is **free-text** explaining why the request was deemed infeasible — there is no fixed catalogue of reason codes.

- `POST /image/fromQuestion` — when the planner emits a component type the renderer does not have implemented, the caller sees **HTTP 501** with `error: "NotImplemented"`. This is the closest legacy analogue of "component type not yet ported."

- `POST /image/fromQuestion` — when the planner produces a structurally invalid scene, the caller sees **HTTP 400** with `error: "InvalidScene"`.

- `POST /render` — there is no feasibility step on this endpoint. A request that violates the SceneV1 schema returns **HTTP 400** with:
  ```json
  {
    "detail": {
      "error": "InvalidScene",
      "message": "Scene validation failed",
      "details": [<pydantic field-level errors>]
    }
  }
  ```
  A request that asks for a component type the renderer doesn't dispatch surfaces as a **500 InternalError** today — there is no `UnknownComponent` path on `/render` at the moment.

**Concretely, mapping back to the questions asked:**

- **Typed-200 sentinel vs 4xx?** Legacy uses 4xx (specifically 400 for "infeasible" and 501 for "not implemented"). There is no typed-200 sentinel today.
- **Which 4xx code?** Legacy uses **400** for "infeasible / cannot express as a SceneV1" and **501** for "component type not implemented." 409 and 422 are not used by the legacy code.
- **Fixed catalogue vs free text?** Today, the top-level `error` field is a **fixed enum** (`UnsupportedRequest`, `NotImplemented`, `InvalidScene`, `UnknownComponent`, `SVGGenerationFailed`, `FeasibilityCheckFailed`, `InternalError`, `ValidationError`). The `message` is fixed per error type. The `details.feasibility_reasoning` is LLM-authored free text.
- **Hint about what *is* supported?** No. The legacy response is opaque about supported types — the only hint is the free-text reasoning string, and even that is not a list of supported components.
- **Same on `/render` and `/image/fromQuestion`?** They **differ**. `/render` has no feasibility gate (it accepts an already-typed scene), so it cannot answer "not supported" the same way — invalid-scene and unknown-component cases come out differently as documented above.

The "fixed catalogue vs free text" and "include hint about what is supported" product decisions for the rebuild are not implied by current behaviour and remain open.

---

### Q3 — When the service fails partway through, what should the caller see?

**Today's legacy behaviour:**

- **HTTP status code:** All three failure modes (planner exhausts retries, geometry/scene validation never converges, renderer fails) currently map to **HTTP 400** on `/image/fromQuestion` when wrapped as `AgentError` of type `UnsupportedRequest`/`InvalidScene`, or **HTTP 500** (`InternalError` / `SVGGenerationFailed` / `FeasibilityCheckFailed`) when they escape as runtime errors. `/render` returns **HTTP 500** with `error: "InternalError"` for any rendering exception.
- **Body shape:** Same envelope as Q2 — `{"detail": {"error": "<type>", "message": "<fixed string>", "details": {...}}}`. The `error` type identifies *which* of the failure modes occurred:
  - Planner retry exhaustion: `error` is `"NotImplemented"` (501) when the model never converged on a supported component, or `"InvalidScene"` (400) when validation failed across all attempts, with a `message` of the form `"Scene validation failed after N attempts"` / `"Failed to plan scene after N attempts"`.
  - Renderer failure: `error` is `"SVGGenerationFailed"` (500) or `"UnknownComponent"` (400).
  - Feasibility-check infrastructure failure (LLM down): `error` is `"FeasibilityCheckFailed"` (500).
  - A label/angle-mark layout that cannot place labels has **no distinct error code** in the legacy code today — it surfaces as a generic `InternalError` / `SVGGenerationFailed`.

  So today the caller can distinguish "planner gave up" vs "renderer blew up" vs "feasibility LLM was unreachable," but not "label layout couldn't place all labels" — that case is opaque.
- **Retry budget the caller waits for:** The planner retry budget is **2 retries (3 total attempts)** by default. The feasibility check has the same **2-retry** default. These are internal — the caller never sees them surfaced.
- **Partial / fallback SVG on failure?** **No.** The legacy code deliberately removed the LLM-authored-SVG fallback ("The LLM fallback was removed deliberately: requests not supported by the deterministic renderer must raise AgentError, not degrade to LLM-authored SVG."). Failures always return an error envelope — never a placeholder image.

The "user-visible cause vs generic" trade-off, the absolute time budget the caller is willing to wait, and whether a placeholder should ever be returned are all product decisions the legacy behaviour does **not** dictate beyond what is listed above.

---

### Q4 — Which diagram/component types must the rebuild render at v0 launch?

**Today's legacy behaviour — the full list the upstream service can ask for** (taken from the live component dispatch table the legacy HTTP server registers):

- **Polygons & quadrilaterals:** `rectangle`, `square`, `circle`, `triangle`, `point`, `parallelogram`, `trapezoid`, `rhombus`, `kite`, `pentagon`, `concavePentagon`, `hexagon`, `heptagon`, `octagon`, `nonagon`, `decagon`, `starPolygon`, `quadrilateral`, `partitionedShape`.
- **Angles & clocks:** `directedAngleDiagram`, `directedAngleDiagramTicks`, `angleDiagram`, `clock`, `digitalClock`.
- **Measurements & manipulatives:** `numberLine`, `faultyNumberLine`, `ruler`, `alignedLengthComparison`, `protractor`, `array`, `tree`, `counterChange`, `gridRectangle`, `gridSquare`, `multiplicationAreaModel`, `rectilinearShape`, `coordinatePlane`, `symmetryFigure`.
- **Charts, tables, 3-D & nets:** `table`, `data_table`, `two_row_table`, `barGraph`, `histogram`, `pictureGraph`, `linePlot`, `faultyLinePlot`, `faultyCircleGraph`, `boxWhiskerPlot`, `lineRelationshipDiagram`, `placeValueBlocks`, `longPlaceValueArithmetic`, `longDivision`, `isometricPrism`, `isometricSolid`, `triangularPrism`, `vennDiagram`, `cubeNet`, `rectangularPrismNet`, `triangularPrismNet`, `rectangularPyramidNet`.

The legacy service also has a runtime "component filter" (`generation_config.json`) that can be used to disable individual components from being advertised in the feasibility prompt — i.e., the in-code list above is the **maximum** capability set; operations can narrow it without a code change.

**What this answer cannot tell you (product calls still needed):**

- **Day-one vs deferred-to-M7:** legacy code treats all listed types as supported; there is no notion of "tier 1" in the code. Splitting the list into v0 vs M7 is a product call.
- **Drop-from-rebuild candidates:** the code has no usage/coverage data and no "deprecated" flag per component. Curriculum + traffic data outside this repo are needed.
- **Curriculum-coverage threshold (e.g. 95%):** not encoded anywhere in the legacy service.

---

### Q5 — How is the 30-scene LLM eval that gates the planner work curated and scored?

I cannot answer this from the legacy codebase. No 30-scene eval harness, scoring rubric, threshold constant, or sign-off owner is encoded in the current `svg-gen` source. There is a `test_all_scenes.py` / `test_e2e_pipeline.py` / `test_parallel_pipeline.py` suite that runs scenes through the pipeline, but it is a developer regression harness — not a curated 30-scene eval with an 85% gate.

This is entirely a forward-looking product/process decision (scene curation source, pass/fail definition, threshold, sign-off authority). The legacy code does not constrain any of those choices.

---

### Q6 — Must SVG output be visually equivalent to today's svg-gen, or are visual changes allowed?

**Today's legacy behaviour:** The current SVG output is the union of (a) the deterministic component renderers, (b) the grid/layout manager, and (c) an SVG-trim step that recomputes the `viewBox` to fit the rendered content. There is no public visual-equivalence contract with consumers — the legacy code does not have a "frozen golden fixture" set of SVGs that downstream services compare against. (There is a `reference-svgs/` directory used as developer-side regression fixtures, not a published consumer contract.)

**Default padding around the diagram:** the SVG-trim step's `default_padding` is **`0.0`** (the trim service crops to the content's bounding box with zero extra padding by default). Per-call padding can be passed at the service layer, but the legacy HTTP `/render` and `/image/fromQuestion` endpoints do not expose a `padding` knob to callers — callers receive whatever the trim service's default produces.

**Style elements consumers might depend on:** the legacy code does not document any consumer-visible style contract (specific stroke colour, specific font size, specific right-angle-mark style). All stroke widths, colours, fonts, and offsets are hard-coded inside the component renderers and are subject to change. The "Figure not drawn to scale." note is the **only** semantically named, conditionally rendered piece of visible chrome (controlled by `showNotDrawnToScaleNote` on the scene).

**Product calls still needed:**

- Pixel-comparable equivalence vs "looks right and passes deterministic checks" — not enforced by current code. Either is implementable; product picks.
- Specific style fixtures that must match — none are pinned today; if any consumer depends on a specific colour/font, surface that and pin it explicitly.
- Default padding — legacy default is 0.0; the rebuild may choose to keep that or raise it.

---

### Q7 — What end-to-end latency budgets do callers expect for `/render` and `/image/fromQuestion`?

The legacy codebase does **not** encode an end-to-end SLO for either endpoint. There is no max-response-time setting, no p95/p99 budget constant, and no caller-visible timeout. The only LLM-side timeout is an internal `FALLBACK_TIMEOUT` (default 30 s) inherited from the removed LLM fallback path and not enforced on the live planner call. The legacy `/render` and `/image/fromQuestion` paths will keep running until the LLM call(s) and renderer return, however long that takes.

**What this means for the rebuild questions:**

- **Max acceptable response time:** not encoded; product decision required.
- **Different budget for `/render` vs `/image/fromQuestion`:** the codebase confirms the two endpoints have different work profiles — `/render` skips feasibility + planning entirely and only runs the deterministic renderer, while `/image/fromQuestion` always runs feasibility + planner (up to 2 retries each) + renderer. A different budget per endpoint is therefore technically reasonable, but no budget is set today.
- **p95/p99 SLO:** none encoded. Product decision required.

I can describe the work each endpoint does; I cannot tell you what the caller's tolerance is from the code.

---

### Q8 — When `/image/fromQuestion` cannot produce a diagram, what should the upstream pipeline see?

**Today's legacy behaviour:** `/image/fromQuestion` returns an HTTP error envelope in **every** failure mode — there is no empty-SVG path, no `{"image": null}` sentinel, and no placeholder image. Specifically:

- **Infeasible (feasibility check said no):** **HTTP 400** with `{"error": "UnsupportedRequest", "message": "Request is not supported by the deterministic SVG renderer. Fallback image generation is disabled.", "details": {"feasibility_reasoning": "<free-text>"}}`.
- **Planner could not produce a valid scene after retries:** **HTTP 400** (`InvalidScene`) or **HTTP 501** (`NotImplemented`) depending on which branch the planner failed in, with a message of the form `"Scene validation failed after N attempts"` / `"Failed to plan scene after N attempts"`.
- **Renderer failed on a known component:** **HTTP 500** with `error: "SVGGenerationFailed"`.
- **Renderer received an unknown component type from the planner:** **HTTP 400** with `error: "UnknownComponent"`, `details: {"component_type": "<name>"}`.
- **Feasibility LLM itself crashed/unreachable:** **HTTP 500** with `error: "FeasibilityCheckFailed"`.
- **Anything else uncaught:** **HTTP 500** with `error: "InternalError"`.

On success, the response is the raw SVG bytes (`Content-Type: image/svg+xml`) with diagnostic headers `X-Generation-Method`, `X-Confidence`, `X-Feasibility`, `X-Duration-Ms`. The endpoint can also return raw PNG bytes (`Content-Type: image/png`) when an upstream image-generation path produces one — that branch exists in the response builder.

So the response **shape on the "feasibility says no" path and the "planner/solver failed at runtime" path is the same envelope**, distinguished only by the `error` field. They are not distinct response shapes today.

**Whether the upstream pipeline should ever serve the question without a diagram:** the legacy `svg-gen` service has no opinion on this — it returns an error and lets the upstream pipeline decide. The current contract (`error` always set, `details` sometimes populated, no `image: null` sentinel) implicitly forces the upstream pipeline to either retry, drop the question, or serve it diagram-less based on its own policy. That product policy is **not encoded in svg-gen** and is a call for the upstream question-generation owner.
