# Proposal 3 — Aggressive rewrite ✅ PICKED

> **One-line pitch:** svg-gen as a constraint-based diagram engine. Scenes are declared as a graph of geometric relations (“AB perpendicular to BC,” “D is the midpoint of AC,” “label ‘x’ belongs to side BC outside the triangle”); a solver computes positions; rendering is a pure walk over the solved graph. The same three pillars, pushed to the ceiling.

## What this is

Proposal 2 says: *labels are declarative, geometry is partly declarative, layout is local.* Proposal 3 says: **everything is declarative, and the engine is global.**

- A **constraint graph** replaces both today’s direct-coordinate fields and Proposal 2’s anchor refs. The planner declares relations; the engine resolves a consistent geometry.
- A **global layout optimizer** (not a per-label local solver) places labels and angle marks together, minimizing a weighted overlap + readability cost.
- The class hierarchy from Proposal 2 stays, but renderers become *much* thinner: they don’t compute anything, they read the solved graph.

This is the rewrite that says: *the existing band-aids exist because the planner can be inconsistent with itself — right angles that aren’t 90°, midpoints that aren’t midpoints, labels that conflict. Eliminate the inconsistency at the model level by making the planner unable to express it.*

## 🔒 Locked decision — trim is math-based, Playwright is gone

This applies to all three proposals; it is not a Pillar-C choice.

- **Production trim is pure math.** After the constraint solver produces `SolvedGeometry`, every component knows its real coordinates; the layout optimizer places labels with known bboxes. Final viewBox = `BBox.union(solved-geometry bboxes ∪ placed-label bboxes) + padding`. Microseconds, deterministic, no browser.
- **Playwright/Chromium is deleted from the runtime dependency set.** `services/svg_trim/`, `browser_manager.py`, and the `shutdown_svg_trim_service` lifespan hook in `web/app.py` are removed.
- **One opt-in dev tool keeps Playwright available:** `svg-gen render --verify-bbox` launches Chromium, measures the real bbox, prints the diff against the math-bbox. Wired into a CI job over a fixture set so any constraint-solver or optimizer change that breaks bounds fails the build.

This is *especially* clean in Proposal 3: because the constraint solver hands the optimizer exact coordinates and the optimizer outputs exact label positions, the bbox union is a closed-form computation. There is no estimation step that needs verifying against a browser. Playwright literally has nothing to add at runtime.

## Pillar A — Class hierarchy (same as Proposal 2)

See `proposal-2.md` for the full hierarchy. The hierarchy is identical — the difference is what flows through it.

One addition: a `ConstraintSolver` lives next to `LabelSolver`, and the `Renderer` contract changes:

```python
class ComponentRenderer[C: Component](ABC):
    @abstractmethod
    def declare_constraints(self, c: C) -> list[Constraint]: ...
    @abstractmethod
    def draw(self, solved: SolvedGeometry, c: C, ctx: RenderContext) -> str: ...
    @abstractmethod
    def label_candidates(self, solved: SolvedGeometry, c: C) -> list[LabelCandidate]: ...
```

The renderer **does not compute geometry**. It declares relations and reads the solver’s output. Its `RenderResult.bbox` is derived directly from `solved` — feeding the locked math-trim decision above.

## Pillar B — Constraint-graph scene model

### What the planner emits

Not this:

```json
{
  "type": "triangle",
  "vertices": [[100, 100], [300, 100], [200, 250]],
  "sideLabels": ["5", "4", "3"],
  "rightAngleAt": 0
}
```

This:

```json
{
  "type": "triangle",
  "points": [
    {"id": "A"},
    {"id": "B"},
    {"id": "C"}
  ],
  "constraints": [
    {"kind": "distance", "between": ["A", "B"], "value": 5},
    {"kind": "distance", "between": ["B", "C"], "value": 4},
    {"kind": "distance", "between": ["C", "A"], "value": 3},
    {"kind": "rightAngle", "at": "A", "between": ["AB", "AC"]}
  ],
  "labels": [
    {"text": "5", "at": {"kind": "sideMidpoint", "target": "AB", "side": "outside"}},
    {"text": "4", "at": {"kind": "sideMidpoint", "target": "BC", "side": "outside"}},
    {"text": "3", "at": {"kind": "sideMidpoint", "target": "CA", "side": "outside"}}
  ]
}
```

No coordinates anywhere. The planner cannot lie about a right angle because the right angle is a *constraint*, not a *picture*.

### The solver

`ConstraintSolver` takes the declared constraints and produces `SolvedGeometry` (every point now has real coordinates, every line knows its length, every angle is exact). It uses standard 2D geometric constraint solving (numeric Newton solver over the constraint Jacobian; a small library like `scipy.optimize` or hand-rolled).

For v0 we restrict the constraint vocabulary to what SAT diagrams need:

- `distance(A, B, value)`
- `angle(A, B, C, value)`
- `rightAngle(at, between)`
- `parallel(line1, line2)`
- `perpendicular(line1, line2)`
- `midpoint(M, A, B)`
- `onLine(P, A, B)`
- `congruent(seg1, seg2)`

Each constraint maps to a residual function; the solver minimizes residuals.

### Generalized derived geometry

Today’s `geometry/resolver.py` becomes the *entire* geometric layer. `footOfPerpendicular`, `lineIntersection`, `midpoint` are constraints, not special-case derived points. Right-angle truthfulness verification is unnecessary — the solver enforces it as an invariant.

### Validation

Validation becomes “did the solver converge?” If the planner emits contradictory constraints (`AB = 5, BC = 4, CA = 3, rightAngle at A`) and they’re infeasible (these are actually consistent — but `AB = 5, BC = 4, CA = 10, rightAngle at A` is not), the solver fails and returns a typed `UnsatisfiableConstraintsError` with the minimal infeasible subset. The planner sees that as retry feedback.

## Pillar C — Global layout optimizer

### What it does

Given the solved geometry, the optimizer places **all labels and all angle marks at once** by minimizing a cost function:

```
cost = w1 * overlap(label_bboxes, geometry_bboxes)
     + w2 * overlap(label_bboxes, label_bboxes)
     + w3 * sum(distance_from_preferred_position)
     + w4 * sum(distance_from_consistent_offset)
```

Where `w4` penalizes inconsistent label distances (“if all other side labels are 12 units from the side, this one being 30 units is a high cost”). This is the *consistency* requirement you called out, encoded as a cost.

The optimizer is global (across the whole scene) but bounded — it doesn’t need to find the optimum, just a feasible low-cost solution. A few hundred iterations of gradient descent or simulated annealing is enough. **Its output bboxes feed directly into the locked math-trim union.**

### Why this is different from Proposal 2’s solver

Proposal 2 places labels one at a time, locally. If labels A and B both want position X, label A (higher priority) gets it; label B falls through its candidates. This can cause cascading suboptimal placements.

Proposal 3 places A and B together. The optimizer can move A *slightly* to free a better position for B, if the joint cost is lower. The output is provably more consistent at the cost of an opaque solver step in the hot path.

## Smallest valuable slice (v0)

**In:**
- Constraint vocabulary (8 constraint kinds above) + `ConstraintSolver` + `SolvedGeometry`.
- Three renderers ported to the new contract: `Rectangle`, `Triangle`, `CoordinatePlane`.
- Global layout optimizer (label placement only; angle marks deferred to v0.1).
- Math-based trim live for these three (locked decision above). `--verify-bbox` dev tool wired up against a starter fixture set.
- Planner emits new constraint-graph scenes for these three component types only.
- `UnsatisfiableConstraintsError` retry feedback.
- All other components stay on the legacy renderer via the adapter (which still uses Playwright trim until those components are ported).

**Deferred:** porting the rest of the renderers; optimizer for angle marks; richer constraint vocabulary (tangency, area, similarity, …); UI / debug tools; deletion of the Playwright runtime dependency (happens after the last legacy renderer is migrated).

## Natural build sequence

- **Phase 1 — Constraint engine.** Constraint types, residuals, `ConstraintSolver`, `SolvedGeometry`. Unit-tested in isolation against known SAT diagram targets.
- **Phase 2 — Layout optimizer + math trim.** Cost function, gradient/anneal loop, text-metric estimation, **math-bbox trim**, `--verify-bbox` dev tool.
- **Phase 3 — Render contract change.** New `Renderer` ABC with `declare_constraints` + `draw` + `label_candidates`. Port `Rectangle` (simplest).
- **Phase 4 — Port triangle + coord-plane.** The two highest-value bug magnets.
- **Phase 5 — Planner rewrite.** New prompt teaches constraint-graph emission. Eval against held-out scenes.
- **Phase 6+ — Port the rest, expand constraints.**
- **Phase 7 (post-port) — Delete Playwright runtime.** Once every component is constraint-based, drop `services/svg_trim/`, the `playwright` runtime dependency, and the lifespan hook. `--verify-bbox` keeps an opt-in dev-only Chromium install.

## Key risks & open questions

- **R1 (gates v0):** Will current frontier LLMs reliably emit a constraint graph for SAT diagrams? This is the central bet — untested. Mitigation: build a 30-scene eval *before* committing to phase 5; if the first-try success rate is below ~85%, fall back to Proposal 2’s anchor model.
- **R2:** Solver failure modes. Newton solvers can diverge, get stuck in local minima, or be ambiguous (infinite solutions). Need to detect and surface these with useful messages. Mitigation: start from a reasonable initial layout (place points evenly around a circle); cap iterations; classify failures into typed errors.
- **R3:** Optimizer determinism. Gradient descent + annealing is sensitive to seed; identical inputs must produce identical outputs (or downstream caches break). Mitigation: fixed seed, deterministic anneal schedule, snapshot tests on cost values and final positions.
- **R4:** Constraint vocabulary lock-in. Once the planner is trained against 8 constraint kinds, adding a 9th is a planner re-prompt + an eval re-run. Mitigation: explicit deprecation/extension policy; constraint kinds are versioned with the prompt.
- **R5:** Build cost. Realistically 2–3x the engineering of Proposal 2 to first working slice. Mitigation: stop after phase 4 if the bet isn’t paying off; phases 1–4 still give a working triangle + coord-plane subsystem.
- **R6:** Solver in the hot path = latency. A 200-iteration Newton solve per scene adds ~100–300ms. Mitigation: this matches today’s Playwright-trim latency, which the locked decision is removing anyway. Net should be neutral.

## What you cut

- All explicit coordinate fields in scene JSON — the planner can no longer say where points go.
- The dual scene-validator + derived-geometry pipeline — one solver replaces both.
- The today’s `_detect_fraction_expression_error` band-aid — fractions in a distance field are just numeric Pydantic errors, no special case.
- The candidate-anchor local solver from Proposal 2 — the global optimizer subsumes it.
- The Playwright/Chromium runtime dependency (locked decision above). Kept only as the opt-in `--verify-bbox` dev tool; fully removed from the runtime tree in Phase 7.

## What you keep

- The HTTP contract.
- The feasibility → plan → render pipeline shape.
- LiteLLM, three configurable models.
- Langfuse.
- The CLI surface, plus a new `--verify-bbox` mode.

## Why this might be the right call

- **Eliminates the largest class of bugs** by construction: the planner cannot output a geometrically-inconsistent scene because it isn’t expressible.
- **Best long-term ceiling.** Adding a new component is: declare its constraints, declare its label slots. The renderer is ~30 lines.
- **Pays dividends in adjacent domains.** A constraint-based engine generalizes naturally to other SAT topics (transformations, function plots, similarity arguments) without re-architecting.

## Why this might *not* be the right call

- **Highest risk and biggest rewrite.** Two new core systems (constraint solver + global optimizer) that the team hasn’t built before.
- **Largest planner prompt change.** Eval-driven by definition; success rate is the gate.
- **Hardest to roll back.** Once renderers stop computing coordinates, you’re committed to the solver.

---

*Grounded in: `components/svg-gen/src/svg_gen/services/geometry/resolver.py` (the seed pattern we generalize — derived points + claim verification, today only for `coordinatePlane`), `components/svg-gen/src/svg_gen/services/scene_validator.py` (the per-type sanity checks the solver subsumes), `components/svg-gen/src/svg_gen/services/planner_service.py:_detect_fraction_expression_error` and `_attempt_plan` (the per-attempt retry loop that becomes typed `UnsatisfiableConstraintsError` feedback), `components/svg-gen/src/svg_gen/services/renderer/components/geometric/{triangle,quadrilateral,parallelogram}.py` (renderers that today compute coordinates from dimensions — the work the solver takes over), `components/svg-gen/src/svg_gen/services/svg_trim/{browser_manager,svg_trim_service}.py` (Playwright we remove either way; the locked math-trim decision is shared across all three proposals).*