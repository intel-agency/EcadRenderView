# PCB Renderer Improvement Checklist – Prioritized (Expanded)

This document expands the initial checklist into a reviewer-ready implementation plan. Items are organized by **priority**, and each item includes: **(a) description/motivation, (b) reference back to the review, (c) concrete changes to implement, (d) status**, plus a **Feedback / Notes** section.

**Status values:** Not started · Started · Complete

---

# Priority 0 — Must-Have to Meet Challenge Requirements

## P0.1 Re-Scope Architecture to a CLI-Only Submission

**Description / Motivation**
The challenge explicitly optimizes for correctness and rapid reviewability. A service-oriented architecture (API server, async pipelines, persistence, Docker) increases setup time and cognitive load without improving scoring.

**Review reference**

* §3.1 Architectural Over-Scoping
* §5.1 Re-Scope to a Minimal Viable Submission

**Concrete changes to implement**
Short summary: Collapse the solution to a single CLI with a small core library; remove all server and infrastructure concerns from the submission.

* Update the design plan “Architecture” section to remove:

  * FastAPI endpoints
  * Docker, Postgres/PostGIS, Redis
  * async/batch service workflows
* Add a “CLI Interface” section specifying supported commands:

  * `pcb-render render <input.json> -o <out.svg>`
  * `pcb-render validate <input.json>`
* Add a brief “Future Work” subsection listing optional service/deployment ideas explicitly as non-submission scope.

**Status:** Not started

**Feedback / Notes:**

---

## P0.2 Implement Keepout Handling End-to-End

**Description / Motivation**
Keepouts are an explicit rendering requirement. Missing keepout modeling, validation, and rendering risks an automatic “incomplete” evaluation even if everything else works.

**Review reference**

* §3.2 Omission of Required Features (Keepout Regions)
* §4.1 Keepout Handling (End-to-End)
* §5.4 Keepout Rendering Strategy

**Concrete changes to implement**
Short summary: Add a keepout data model, validate its geometry, and render keepouts with a distinct patterned appearance.

* Parsing/Modeling:

  * Add `Keepout` model(s) in `models.py` (or equivalent) with geometry, layer (if applicable), and optional metadata.
  * Extend `parse.py` to parse keepouts from the provided schema.
* Validation:

  * Add keepout geometry validation (min points, closure policy, numeric sanity).
  * Define whether keepouts outside board bounds are errors or warnings.
* Rendering:

  * Use **Matplotlib** as the rendering backend.
  * Render keepouts using `matplotlib.patches.Polygon` with `hatch='///'` for crosshatch pattern.
  * Apply high-contrast outlines via `edgecolor` parameter.
  * Decide keepout draw order (recommend overlay/topmost via `zorder`).
* Documentation:

  * Add a “Keepouts” subsection to the rendering spec.

**Status:** Not started

**Feedback / Notes:**

---

## P0.3 Render Reference Designators Legibly

**Description / Motivation**
Reference designators are explicitly required for interpretability. Omitting them or rendering illegibly undermines the evaluation criteria focused on readability.

**Review reference**

* §3.2 Omission of Required Features (Reference Designators)
* §5.3 Reference Designator Rendering

**Concrete changes to implement**
Short summary: Ensure every component renders an associated reference designator with consistent sizing and contrast.

* Rendering:

  * Use **Matplotlib** `matplotlib.text.Text` for reference designators positioned at the transformed component centroid.
  * Define font size scaling relative to board extents.
  * Apply halo/stroke using `matplotlib.patheffects.withStroke()` for visibility on copper/substrate backgrounds.
* Transform rules:

  * Clamp or normalize text rotation to preserve legibility (e.g., keep text upright).
* Plan updates:

  * Add explicit “Reference Designators” section describing placement, scaling, and legibility rules.

**Status:** Not started

**Feedback / Notes:**

---

## P0.4 Formalize the Coordinate + Transform Pipeline

**Description / Motivation**
Incorrect transforms (rotation origin, axis inversion, mirroring) are a primary failure mode in ECAD visualization. A rigorously defined pipeline reduces subtle geometric errors and ensures correctness for rotated/back-side components.

**Review reference**

* §3.4 Underspecified Coordinate and Transform Pipeline
* §5.2 Formalize the Transform Pipeline
* §6 Observations (45° rotations, mirroring, closure)

**Concrete changes to implement**
Short summary: Write a precise transform spec and implement it consistently across all rendered primitives.

* Design spec additions:

  * Define ECAD axis orientation and SVG axis orientation; specify Y inversion policy.
  * Specify angle units (degrees), direction, and rotation origin (centroid).
  * Define mirroring rules for back-side components.
  * Define global scaling, padding, and viewBox calculation.
* Implementation:

  * Centralize transforms (e.g., a `transform.py` helper) used by components, text, traces, vias, keepouts.
  * Add unit tests for rotated and mirrored placement.

**Status:** Not started

**Feedback / Notes:**

---

## P0.5 Expand Semantic Validation to Reliably Identify 14 Invalid Boards

**Description / Motivation**
The task requires semantic validation that correctly flags 14 invalid boards. Under-specified validation either misses invalid cases or over-rejects valid boards (false positives).

**Review reference**

* §3.6 Insufficient Validation Coverage for Invalid Boards
* §5.5 Expanded Semantic Validation
* §6 Observations (empty net names, boundary closure)

**Concrete changes to implement**
Short summary: Add explicit geometry and numeric validation, enforce layer/net integrity, and distinguish fatal errors from warnings.

* Geometry checks:

  * Polygons: ≥3 points; closure policy (auto-close vs error).
  * LineStrings: ≥2 points.
* Numeric sanity:

  * Reject NaN/Inf; enforce positive widths/diameters.
* Topology/semantic checks:

  * Validate referenced layers exist.
  * Validate nets referenced exist; allow empty net names where appropriate.
* Severity model:

  * Introduce error vs warning categories to avoid false positives.
* Pydantic model-level validation:

  * Use `@model_validator(mode='after')` in Pydantic models to enforce cross-field constraints at parse time:
    ```python
    class Via(BaseModel):
        center: tuple[float, float]
        diameter: float
        hole_size: float

        @model_validator(mode='after')
        def validate_hole_size(self):
            if self.hole_size >= self.diameter:
                raise ValueError("Via hole_size must be smaller than diameter")
            return self
    ```
  * Apply analogous validators for:
    * `Trace.width > 0`
    * `Polygon` with ≥3 points
    * Referenced `layer_hash` exists in stackup

**Status:** Not started

**Feedback / Notes:**

---

# Priority 1 — Strongly Recommended for Correctness & Reviewability

## P1.1 Formalize ECAD Geometry Parsing (Avoid GeoJSON Assumptions)

**Description / Motivation**
Assuming strict GeoJSON conventions risks subtle parsing errors (e.g., outlines as flat point lists). A tailored parser aligned to the actual schema is more robust and reviewable.

**Review reference**

* §3.3 Geometry Schema Assumptions

**Concrete changes to implement**
Short summary: Document the actual geometry forms used and implement schema-accurate parsing into a canonical internal representation.

* Documentation:

  * Add a “Geometry Schema Notes” section summarizing observed shape encodings.
* Parsing:

  * Implement explicit parsers for board outlines, traces, keepouts, vias.
  * Normalize into canonical internal types (e.g., `Polygon`, `Polyline`, `Circle`).
* Testing:

  * Add parsing tests for each geometry type and edge cases.

**Status:** Not started

**Feedback / Notes:**

---

## P1.2 Establish a Canonical Unit System

**Description / Motivation**
Mixed-unit inputs (MICRON vs MILLIMETER) are common; inconsistent normalization yields scaled or misplaced visuals and breaks validation thresholds.

**Review reference**

* §2.2 Domain Awareness (units)
* §5.2 Formalize the Transform Pipeline (unit normalization)
* §6 Observations (both units appear)

**Concrete changes to implement**
Short summary: Normalize all numeric spatial values into millimeters immediately after parsing.

* Models/Parsing:

  * Add explicit unit field handling; convert at parse-time to mm.
* Validation:

  * Warn or fail on unknown/missing units depending on requirement.
* Tests:

  * Add unit conversion tests ensuring equivalent boards render identically after normalization.

**Status:** Not started

**Feedback / Notes:**
NOTE:** Why not use microns (instead of mm's) to normalize to? Advantage is that they are strictly integers. No decimals to deal with.

---

## P1.3 Output Readability Constraints and Deterministic Draw Order

**Description / Motivation**
Readability is graded. Without explicit canvas sizing, padding, layer ordering, and text contrast rules, output can become visually ambiguous or illegible.

**Review reference**

* §4.2 Output Readability Constraints
* §5.7 Simplified and Reviewable Project Layout (indirectly supports clarity)

**Concrete changes to implement**
Short summary: Define visual constraints (canvas size/padding/font scaling) and enforce a deterministic renderer draw order.

* Renderer spec:

  * Define minimum canvas width (e.g., 1200px) or viewBox policy.
  * Define padding (percentage of board bbox).
  * Define font sizing rules relative to board extents.
* Draw order:

  * Enforce order (recommended): board outline → copper/pours (low opacity) → traces → vias → components → refdes → keepouts overlay.
* Tests:

  * Snapshot tests to ensure ordering is stable.

**Status:** Not started

**Feedback / Notes:**

---

## P1.4 Reviewer-Friendly Dependency Policy

**Description / Motivation**
Heavy dependencies reduce the probability a reviewer can run the project quickly. Using a fast, modern package manager and a single rendering library improves install speed and reproducibility.

**Review reference**

* §3.5 Dependency and Reviewability Risk
* §5.6 Reviewer-Friendly Dependency Policy
* §5.5 (optional formats fail gracefully)

**Concrete changes to implement**
Short summary: Use `uv` as the package manager and Matplotlib as the single rendering dependency providing SVG, PNG, and PDF via `savefig()`.

* Package manager:

  * Use **uv** for fast, reproducible dependency management.
  * Include `uv.lock` in repository for deterministic installs.
  * Document `uv sync` or `uv pip install .` as the install command.
* Dependencies:

  * Require **Matplotlib** as the rendering backend (provides SVG, PNG, PDF natively via `savefig()`).
  * Remove CairoSVG, svgwrite, ReportLab from dependencies.
  * No optional extras split needed—single install covers all output formats.
* Documentation:

  * Document "Required Dependencies" succinctly in README.
  * Include quick-start: `uv sync && uv run pcb-render --help`

**Status:** Not started

**Feedback / Notes:**

---

# Priority 2 — Testing, Diagnostics, and Submission Polish

## P2.0 Automated Testing Stack

**Description / Motivation**
A robust testing stack catches edge cases and prevents regressions. Property-based testing and snapshot tests increase confidence in validation and rendering correctness.

**Review reference**

* §4.3 Explicit Testing Strategy
* additions.md Section 2 (Automated Testing Solution)

**Concrete changes to implement**
Short summary: Establish a pytest-based testing stack with property-based and snapshot testing.

* Framework:

  * Use `pytest` as the test runner.
* Property-based testing:

  * Use `Hypothesis` to generate edge-case data (extreme coordinates, NaN, near-zero widths) for geometry and validation logic.
* Snapshot testing:

  * Use `syrupy` or `pytest-regressions` to record "known good" SVG outputs; fail tests if output changes unexpectedly.
* Coverage:

  * Use `pytest-cov` to enforce ≥90% coverage.
* Test data:

  * Create `tests/invalid_boards/` directory with JSON files crafted to trigger each validation error type.

**Status:** Not started

**Feedback / Notes:**

---

## P2.1 Invalid Board Detection Mapping + Regression Tests

**Description / Motivation**
To confidently hit the “14 invalid boards” requirement, the plan should explicitly map invalid cases to rules and lock behavior via regression tests.

**Review reference**

* §5.5 Expanded Semantic Validation
* §3.6 Insufficient Validation Coverage for Invalid Boards

**Concrete changes to implement**
Short summary: Create a validation matrix linking each invalid board to a specific error code and test it.

* Test plan:

  * Create a table: `invalid_board_i → expected error code(s)`.
  * Implement regression tests asserting the expected failures.
* Validation output:

  * Ensure error codes/paths are stable for test assertions.

**Status:** Not started

**Feedback / Notes:**

---

## P2.2 Error Reporting and Diagnostics

**Description / Motivation**
Clear diagnostics improve reviewability and usability. Structured errors also enable robust testing and faster iteration during debugging.

**Review reference**

* §1 Requirements (detect/classify errors)
* §5.5 Expanded Semantic Validation (structured error codes)
* §16 Error Reporting and Diagnostics (in checklist)

**Concrete changes to implement**
Short summary: Standardize structured errors with severity, codes, and JSON paths.

* Error model:

  * Define `ErrorCode`, `severity`, `message`, `json_path`.
  * Implement specific error codes aligned to the 14 invalid boards:
    * `MissingBoundaryError` — No boundary key
    * `MalformedCoordinatesError` — Coordinates list length invalid or <3 points
    * `InvalidRotationError` — Rotation value not a number
    * `DanglingTraceError` — Trace references non-existent net or component
    * `NegativeWidthError` — Trace width ≤ 0
    * `EmptyBoardError` — Boundary exists but no components/traces
    * `InvalidViaGeometryError` — hole_size ≥ diameter
    * `NonexistentLayerError` — Reference to undefined layer
    * `NonexistentNetError` — Reference to undefined net
    * `SelfIntersectingBoundaryError` — Boundary polygon self-intersects
    * `ComponentOutsideBoundaryError` — Component entirely outside board
    * `InvalidPinReferenceError` — Pin references wrong component
* CLI output:

  * Print a concise human summary and (optionally) JSON output for tooling.
* Tests:

  * Assert error structure for representative failures.

**Status:** Not started

**Feedback / Notes:**

---

## P2.3 Renderer Testing (Structure + Golden/Snapshot)

**Description / Motivation**
Rendering correctness is hard to validate purely by eye. Snapshot/golden tests ensure stable SVG outputs and catch regressions in transforms/draw order.

**Review reference**

* §4.3 Explicit Testing Strategy
* §5.7 Suggested project layout (test modules)

**Concrete changes to implement**
Short summary: Add renderer tests validating SVG structure and use golden files for representative boards.

* Unit tests:

  * Validate presence/counts of key elements (outline, traces, vias, refdes, keepouts).
* Snapshot tests:

  * Store golden SVG outputs for sample boards; compare normalized SVG strings.
* Stability:

  * Normalize float formatting where necessary to reduce diffs.

**Status:** Not started

**Feedback / Notes:**

---

## P2.4 Parsing and Validation Testing (Unit + Edge Cases)

**Description / Motivation**
Parsing and validation are the backbone of invalid-board detection. Tests should cover both normal cases and the observed edge cases (closure, empty nets, mixed units).

**Review reference**

* §4.3 Explicit Testing Strategy
* §6 Observations (closure, empty nets, mixed units)

**Concrete changes to implement**
Short summary: Implement unit tests for parsing and semantic validation, emphasizing edge cases from the dataset.

* Parsing tests:

  * Valid examples for each geometry type.
* Validation tests:

  * Closure policy behavior.
  * Empty net names treated as allowed/warned per spec.
  * Mixed-unit equivalence.

**Status:** Not started

**Feedback / Notes:**

---

## P2.5 Documentation for Reviewers

**Description / Motivation**
The rubric includes reviewability within ~20 minutes. A tight README and explicit execution instructions materially improve evaluation outcomes.

**Review reference**

* §1 Requirements (reviewable in ~20 minutes)
* §5.7 Simplified and Reviewable Project Layout

**Concrete changes to implement**
Short summary: Provide a short README covering installation, commands, assumptions, and what reviewers should look for.

* README contents:

  * One-liner purpose and scope
  * Install + run commands
  * CLI usage examples
  * Output description (SVG)
  * Known limitations + future work
* Inline docs:

  * Brief docstrings for parsing/validation/rendering entry points.

**Status:** Not started

**Feedback / Notes:**

---

## P2.6 Final Review Pass (Submission Checklist)

**Description / Motivation**
A final pass ensures no required features are omitted and that the project behaves like a reviewer’s clean install.

**Review reference**

* §7 Concluding Assessment

**Concrete changes to implement**
Short summary: Validate completeness against requirements, run tests, and simulate reviewer workflow.

* Requirements audit:

  * Outline, traces, vias, components, refdes, keepouts, validation, errors.
* Clean run:

  * Fresh environment install; run `validate` and `render`.
* Artifacts:

  * Confirm outputs are readable and stable.

**Status:** Not started

**Feedback / Notes:**

---

# Appendix — Suggested Compact Layout (Reference)

```
pcb_renderer/
  cli.py
  models.py
  parse.py
  validate.py
  render_svg.py
  transform.py
tests/
  test_parse.py
  test_validate.py
  test_render_svg.py
```
