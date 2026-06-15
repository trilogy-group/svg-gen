# Business rule questions

These questions are for the product manager of the legacy `svg-gen` product. We are rebuilding the diagram-generation service and need to lock the user-visible behaviour before we finalise contracts. Every question is about what users (the upstream generation service, developers, operators) should see — not about how we build it.

### Q1 — Should the new svg-gen service require an auth token on its HTTP endpoints?

Today's main API has an optional shared-token check that only activates when a deployment variable is set. We need to know what the new `svg-gen` HTTP service should do:

- Should `/render` and `/image/fromQuestion` require an auth token in production?
- If yes, is it the same single shared token used elsewhere, or per-caller credentials?
- What should an unauthenticated request see — a 401 ("missing/invalid token") or a 403 ("forbidden") — and what user-facing message?
- Should `/health` always be reachable without auth (for Cloud Run probes)?

This question blocks the auth criterion on the *Scene-to-SVG rendering HTTP API* and *Question-to-image HTTP endpoint*.

### Q2 — When the service decides a request is "not supported", what should the caller see?

The new pipeline begins with a feasibility check that can decide a scene or question simply cannot be rendered (e.g. asks for a 3D figure, requires a chart type we don't support, references a component type not yet ported). It will also return this response for component types still on the not-yet-ported list during phased rollout.

- Should "not supported" be an HTTP 200 with a typed JSON body (e.g. `{"status": "not_supported", "reason": "..."}`), or a 4xx error code?
- If a 4xx, which code (`400`, `409`, `422`, `501`)?
- What should the user-facing reason text look like — a fixed catalogue of reasons, free text from the LLM, or just a generic "this diagram type isn't supported"?
- Should the response include a hint about what *is* supported, or stay opaque?
- Is this behaviour the same on `/render` and on `/image/fromQuestion`, or do they differ?

This question blocks the "not supported" path across *Scene-to-SVG rendering HTTP API*, *Question-to-image HTTP endpoint*, *LLM-driven feasibility check*, and *Deterministic renderer set*.

### Q3 — When the service fails partway through (planner cannot produce a valid scene after retries, or the layout cannot be drawn cleanly), what should the caller see?

The new pipeline has a few internal failure modes that all map to "we tried, we cannot give you a usable SVG":

- The LLM planner kept producing scenes the geometry solver rejected, and the retry budget ran out.
- The geometry solver could not converge.
- The label/angle-mark layout step could not place all labels without overlapping the diagram.

For each, we need to know:

- What HTTP status code should the caller see?
- Should the response body explain the cause (which one of the three above) in user-facing terms, or stay a generic "could not render"?
- What is the maximum number of internal retries the user is willing to wait for before we give up? (Drives the planner retry budget.)
- Should we ever return a partial/fallback SVG (e.g. a placeholder image with a "render failed" label), or always an error?

This question blocks the failure-path criteria on *Scene-to-SVG rendering HTTP API*, *LLM-driven constraint-graph scene planner*, *LLM-driven feasibility check*, and *Global label & angle-mark layout optimizer*.

### Q4 — Which diagram/component types must the rebuild render at v0 launch?

The rewrite plan brings up three component types first (`Rectangle`, `Triangle`, `CoordinatePlane`) and ports the rest in a follow-up milestone. Before we lock the launch scope we need to know:

- What is the full list of diagram types the upstream generation service relies on today?
- Which of those are "must work on day one of the new svg-gen" vs "can return not-supported until M7 lands"?
- Are there any diagram types the legacy svg-gen supports today that should be **dropped** from the rebuild even at parity (e.g. rarely used, deprecated by the curriculum team)?
- Is there a usage-share or curriculum-coverage threshold the launch set must hit (e.g. "must cover 95% of recent question traffic")?

This question blocks the v0 launch scope of *Deterministic renderer set* and the exit criteria of milestone M7.

### Q5 — How is the 30-scene LLM eval that gates the planner work curated and scored?

Before we build the LLM constraint-graph planner we will run a 30-scene eval; if the LLM cannot reliably emit constraint-graph scenes (target ≥ ~85% first-try success), we fall back to a different planner design. We need product input on:

- Who owns the 30 input scenes — are they sampled from production SAT question traffic, hand-authored by the curriculum team, or a mix?
- What counts as a "successful" emission for one scene — does the resulting SVG need to render visually correct end-to-end and pass review, or is "the geometry solver accepts it" enough?
- Is ~85% the right pass threshold for production, or should it be higher (e.g. 95%) or lower (e.g. 75%)?
- Who signs off on the eval result and the path choice (primary constraint-graph planner vs anchor-model fallback)?

This question blocks the design of the *30-scene LLM constraint-emission eval gate* and through it the M6 path choice.

### Q6 — Must SVG output be visually equivalent to today's svg-gen, or are visual changes allowed?

The plan locks the HTTP contract verbatim, but the new render pipeline is a clean rewrite — fonts, label offsets, line widths, colours, padding, and stroke styles will differ unless we deliberately match the legacy. Please confirm:

- Should the new SVGs be visually indistinguishable from today's (pixel-comparable on a fixture set)?
- Or is "looks correct and clean, but may differ in details" acceptable as long as the deterministic test suite passes (no overlap, no clipping, etc.)?
- Are there specific style elements the upstream consumers depend on (e.g. a fixed stroke colour for right-angle marks, a fixed label font size)?
- What padding around the diagram do callers expect by default?

This question blocks the visual-equivalence criterion of *Scene-to-SVG rendering HTTP API* and the default padding of *Math-based SVG trim / viewBox computation*.

### Q7 — What end-to-end latency budgets do callers expect for `/render` and `/image/fromQuestion`?

The new pipeline still calls the LLM (once for feasibility, then for planning, plus retries) on top of the new solver/optimizer (~100–300 ms). We need to know what's acceptable:

- What is the maximum response time before the upstream generation service treats the call as failed?
- Should `/render` (no LLM if scene is already constraint-graph form) and `/image/fromQuestion` (always LLM) have different budgets?
- Is there a p95 / p99 SLO the rebuild must meet?

This question blocks the latency acceptance criteria on *Scene-to-SVG rendering HTTP API* and *Question-to-image HTTP endpoint*.

### Q8 — When `/image/fromQuestion` cannot produce a diagram, what should the upstream pipeline see?

`/image/fromQuestion` is consumed by the upstream question-generation pipeline, which may behave differently from a direct `/render` caller when it gets a failure. Specifically:

- If the question is judged not renderable (feasibility check), should `/image/fromQuestion` return an error, an empty SVG, a typed JSON sentinel (e.g. `{"image": null, "reason": "..."}`), or a placeholder image so the question can still be served without a diagram?
- If the planner/solver/optimizer fail at runtime, should the response shape be the same as the not-renderable case, or distinct?
- Should the upstream pipeline ever proceed to serve the question without a diagram, or is "no diagram" considered a question-generation failure?

This question blocks the failure-path response shape of *Question-to-image HTTP endpoint* and may differ from Q3's general failure-path answer.
