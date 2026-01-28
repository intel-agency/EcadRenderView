# PCB Renderer Plan – Review & Feedback

This document records detailed feedback on the proposed implementation plan for the Quilter ECAD-to-visualization challenge. It focuses on alignment with the challenge requirements, scope control, technical risks, and concrete improvements to maximize reviewability and correctness.

---

## 1. What the Challenge Actually Requires

The Quilter challenge is narrowly scoped. A correct solution must:

* Parse ECAD JSON input
* Render a PCB visualization including:

  * Board boundary
  * Components with correct transforms (position + rotation)
  * **Reference designators** (R1, C1, U1, etc.)
  * Traces with width and layer coloring
  * Vias rendered as circles
  * **Keepout regions** rendered with a distinct pattern or color
* Detect and report errors when parsing or validation fails
* Perform semantic validation and correctly detect **14 invalid boards**
* Produce readable output
* Be reviewable and verifiable within ~20 minutes
* Include comments and unit tests

The output format may be SVG, PNG, or PDF (SVG is sufficient).

---

## 2. Strengths of the Proposed Plan

### 2.1 Good Architectural Direction

* Vector-first rendering (SVG) is an excellent choice for PCB visualization.
* Clear separation of concerns (parsing → validation → rendering) improves testability and clarity.
* Use of schema-based validation (e.g., Pydantic-style models) is well-suited for early error detection and descriptive failures.
* Explicit inclusion of semantic validation (nets, layers, dimensions) aligns well with the invalid-board requirement.

### 2.2 Awareness of PCB Domain Concepts

* The plan acknowledges multiple design units (MICRON vs MILLIMETER).
* Multi-layer stackups, traces, vias, and component transforms are all recognized as first-class concerns.

---

## 3. Major Weaknesses and Risks

### 3.1 Over-Scoping Relative to the Challenge

The plan proposes a hybrid CLI + REST API architecture with:

* FastAPI server
* Multiple endpoints
* Optional Docker, Postgres/PostGIS, Redis
* Async batch rendering

None of this is required by the challenge and directly conflicts with the requirement that the solution be easily reviewable in ~20 minutes.

**Risk:** Reviewers may not run or verify the solution if setup is complex.

**Recommendation:** Deliver a CLI-first solution. Treat any API/server mode as a future extension, not part of the core submission.

---

### 3.2 Missing Required Features

#### Keepout Regions

* Keepouts are explicitly required by the prompt.
* The plan does not define:

  * A keepout data model
  * Validation rules
  * Rendering strategy

#### Reference Designators

* Although component references exist in example models, the plan does not explicitly commit to rendering readable reference labels.
* Placement, scaling, and readability rules are not defined.

Both features must be first-class parts of the design.

---

### 3.3 Geometry Format Assumptions

The plan implies GeoJSON-style geometry handling, but the provided data does not strictly follow GeoJSON conventions:

* Board boundaries are flat lists of points, not nested rings.
* Traces resemble GeoJSON LineStrings, but polygons do not.

**Risk:** Using off-the-shelf GeoJSON utilities may cause parsing errors or incorrect geometry interpretation.

**Recommendation:** Explicitly document and implement parsers for the actual input schema.

---

### 3.4 Insufficient Definition of Coordinate and Transform Pipeline

Critical details are missing or under-specified:

* SVG coordinate system vs ECAD coordinate system (Y-axis inversion)
* Rotation origin (component center vs outline center)
* Degrees vs radians
* BACK-side component mirroring rules
* Global scaling, padding, and viewBox definition

Without a clearly defined transform pipeline, rotated or back-side components are likely to render incorrectly.

---

### 3.5 Dependency and Review Risk

* Heavy dependencies (e.g., Shapely, Cairo/CairoSVG) can introduce installation issues during review.
* The challenge only requires one output format.

**Recommendation:** Make SVG the primary (and possibly only) required output. Treat PNG/PDF as optional and fail gracefully if dependencies are missing.

---

### 3.6 Validation Coverage Likely Insufficient for 14 Invalid Boards

While some semantic checks are listed, additional validation rules are needed to reliably detect many invalid boards, especially around geometry and transforms.

Example issues observed in provided boards:

* Boundary polygons not explicitly closed
* Empty net names on component pins (valid in some cases)

The plan must distinguish between errors vs warnings and handle edge cases intentionally.

---

## 4. Missing Areas That Should Be Added

### 4.1 Keepout Regions (End-to-End)

Add explicit support for:

* Data model for keepouts
* Geometry validation
* Rendering with hatched or patterned fills
* Optional labeling (e.g., “KEEP-OUT”)

---

### 4.2 Output Readability Rules

Define concrete readability constraints:

* Fixed or minimum canvas size (e.g., 1200px width)
* Board padding as a percentage of bounding box
* Font size scaling relative to board size
* Layer draw order:

  1. Board outline
  2. Copper pours (low opacity)
  3. Traces
  4. Vias
  5. Components
  6. Reference designators
  7. Keepouts (overlay)

---

### 4.3 Concrete Testing Strategy

Beyond “unit tests exist,” explicitly define:

* Parser tests (valid boards parse successfully)
* Semantic validation tests (each invalid board produces a known error)
* Renderer tests (SVG structure, element counts)
* Golden-file or snapshot tests for SVG output

---

## 5. Concrete Improvements to the Plan

### 5.1 Re-Scope to an MVP

Replace the hybrid architecture with:

* Core library: parsing, validation, rendering
* Single CLI entry point:

  * `pcb-render render input.json -o out.svg`
  * `pcb-render validate input.json`

This directly aligns with the challenge constraints.

---

### 5.2 Define a Clear Transform Pipeline

Explicitly document:

1. **Unit normalization**

   * Internal unit (e.g., millimeters)
   * MICRON → mm: divide by 1000
   * MILLIMETER → mm: identity

2. **Board → SVG mapping**

   * Compute bounding box
   * Add padding
   * Define SVG viewBox
   * Optional Y-axis inversion

3. **Component transforms**

   * Local outline centered at its center
   * Rotation applied about center
   * BACK side mirroring (if applicable)
   * Translation by component position

---

### 5.3 Explicit Reference Designator Rendering

* Place text at transformed component centroid
* Keep text upright or clamp rotation for readability
* Scale font size relative to board size
* Use outline/halo stroke to ensure contrast

---

### 5.4 Keepout Rendering Strategy

* Define a reusable SVG hatch pattern
* Fill keepout polygons with pattern + strong outline
* Optional text label

---

### 5.5 Strengthen Semantic Validation

Add checks for:

* Polygon validity (≥3 points, closure)
* LineStrings with ≥2 points
* No NaN/Inf values
* Positive widths/diameters
* Layer and net references exist
* Allow empty net names where valid
* Placement sanity (components/vias inside board bounds, possibly as warnings)

Return structured error codes and paths to failing fields.

---

### 5.6 Reviewer-Friendly Dependency Strategy

* Required output: SVG only
* Optional: PNG/PDF if dependencies are installed
* Clear error messages if optional formats are unavailable

---

### 5.7 Simplified Project Structure

A compact, reviewable layout:

```
pcb_renderer/
  cli.py
  models.py
  parse.py
  validate.py
  render_svg.py
tests/
  test_parse.py
  test_validate.py
  test_render_svg.py
```

---

## 6. Notes Based on Provided Example Boards

* Rotated components (e.g., 45°) require a precise transform pipeline.
* Some boards include many unconnected pins with empty net names; validation must allow this.
* Some boundaries are not explicitly closed; decide whether to auto-close or flag.
* Both MICRON and MILLIMETER boards exist; unit normalization must be explicit and tested.

---

## 7. Summary

The proposed plan demonstrates solid engineering instincts but is significantly over-scoped and misses several explicitly required features. By tightening scope, clarifying transforms, adding keepouts and reference designators, and strengthening validation and testing, the plan can be realigned to the challenge and made far more likely to succeed under the stated review constraints.
