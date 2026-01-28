# PCB Renderer Design Plan (CLI-Only Submission)

This plan focuses on a **single CLI** with a small core library to maximize correctness and reviewability. All optional service/deployment ideas are explicitly out of scope for submission and listed under Future Work.

## Goals

1. Parse PCB board JSON files (components, traces, vias, pours, nets, stackups).
2. Render readable board visuals with required features (keepouts, refdes).
3. Normalize units to **millimeters (mm)** immediately after parsing.
4. Provide deterministic, testable validation that flags the 14 invalid boards.

## CLI Interface

- `pcb-render render <input.json> -o <out.svg>`
- `pcb-render validate <input.json>`

## Architecture (CLI + Core Library)

```text
┌───────────────────────────────┐
│ CLI (typer or argparse)       │
│  - parse args                 │
│  - call core library          │
└──────────────┬────────────────┘
                             │
┌──────────────▼────────────────┐
│ Core Library                  │
│  parse.py     validate.py     │
│  render_svg.py transform.py   │
│  models.py    geometry.py     │
└───────────────────────────────┘
```

## Technology Stack (Reviewer-Friendly)

- Python 3.11+
- Pydantic v2 (models + validation)
- Matplotlib (single rendering dependency; SVG/PNG/PDF via `savefig()`)
- NumPy (optional helper for transforms)

Package manager: **uv** (include `uv.lock`).

## Geometry Schema Notes (Explicit Parsing)

Do not assume strict GeoJSON. Document observed encodings and parse explicitly:

- **Board boundary**: may be flat point lists or nested coordinate pairs.
- **Traces**: may be point lists (polyline) with width.
- **Vias**: center + diameter/hole size (circle-like).
- **Keepouts**: polygons (layer-scoped or global).

Parsing step normalizes all shapes into canonical internal types:

- `Polygon(points: list[Point])`
- `Polyline(points: list[Point])`
- `Circle(center: Point, radius: float)`

## Coordinate + Transform Pipeline (Formal Spec)

- **Units**: normalize all spatial values to **mm** at parse time.
- **Axes**: define ECAD axes (right-handed) vs SVG axes (Y down). Apply Y inversion once at render time.
- **Rotation**: degrees, clockwise vs counterclockwise explicitly defined; rotation origin is component centroid.
- **Mirroring**: back-side components are mirrored across the board X-axis (after rotation).
- **ViewBox**: computed from board boundary bbox + padding (e.g., 5–10%).

All rendering and validation use a centralized `transform.py` helper to avoid inconsistencies.

## Rendering Requirements

### Keepouts (Required)

- Model keepouts as explicit geometry in `models.py`.
- Validate geometry (min points, closure policy, numeric sanity).
- Render with Matplotlib `Polygon` using `hatch='///'` and high-contrast `edgecolor`.
- Draw keepouts **last** (topmost) using `zorder`.

### Reference Designators (Required)

- Render with Matplotlib `Text` at component centroid.
- Font size scales relative to board extents.
- Apply halo with `matplotlib.patheffects.withStroke()` for contrast.
- Clamp rotation so text remains upright and legible.

### Readability + Draw Order

Deterministic order:

1. Board outline
2. Copper / pours (low opacity)
3. Traces
4. Vias
5. Components
6. Reference designators
7. Keepouts (overlay)

## Validation & Error Reporting

### Model-Level Validation (Pydantic)

Use `@model_validator(mode='after')` to enforce cross-field constraints:

- `Trace.width > 0`
- `Via.hole_size < Via.diameter`
- `Polygon` has ≥ 3 points
- `layer_hash` references a real layer

### Geometry + Numeric Checks

- Reject NaN/Inf
- Positive widths/diameters
- Linestrings have ≥ 2 points
- Polygon closure policy (auto-close or error)

### Error Model

Structured errors with: `code`, `severity`, `message`, `json_path`.

Core error codes (examples):

- `MissingBoundaryError`
- `MalformedCoordinatesError`
- `InvalidRotationError`
- `DanglingTraceError`
- `NegativeWidthError`
- `EmptyBoardError`
- `InvalidViaGeometryError`
- `NonexistentLayerError`
- `NonexistentNetError`
- `SelfIntersectingBoundaryError`
- `ComponentOutsideBoundaryError`
- `InvalidPinReferenceError`

CLI output shows concise summary and optional JSON diagnostics.

## Testing Strategy

- **pytest** as runner
- **Hypothesis** for geometry/property-based testing
- **syrupy** or `pytest-regressions` for SVG snapshot tests
- Add `tests/invalid_boards/` with crafted JSONs mapping to each error code
- Coverage target ≥ 90%

## Project Layout (Compact)

```text
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

## Documentation (Reviewer-Facing)

README outline:

- Purpose and scope
- Install + run (`uv sync`, `pcb-render --help`)
- CLI examples (render, validate)
- Output description (SVG)
- Known limitations and future work

## Future Work (Out of Submission Scope)

- FastAPI service / REST API
- Docker packaging and hosting
- Postgres/PostGIS persistence
- Multi-user or batch rendering services
