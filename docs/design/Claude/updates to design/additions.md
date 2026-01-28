# Features to add to the Design

## Items

2. Automated Testing Solution (The "Impressive" Stack)
To impress the judges, we will go beyond basic unittest.

A. Recommended Stack
Framework: pytest (Standard, extensible, and clean).
Property-Based Testing: Hypothesis. Instead of fixed test cases, it generates edge-case data (e.g., extremely large coordinates, NaN values, nearly-zero widths) to find crashes in your geometry logic.
Snapshot Testing: pytest-regressions or syrupy. This records the SVG output of a "known good" board. If a code change alters the visual output even by 1 pixel, the test fails.
Coverage: pytest-cov integrated into a GitHub Action or local script to enforce the >90% requirement.
B. Coverage for the 14 Error Types
We will create a tests/invalid_boards/ directory containing JSON files specifically crafted to trigger each validator.

Schema Failures: Missing metadata, boundary, or stackup.
Geometric Sanity: Open loops in boundaries, self-intersecting polygons (checked via Shapely).
Reference Integrity: Components referencing non-existent footprints; pins referencing non-existent nets.
Physical Constraints: Via diameter < hole size; trace width <= 0.
Layer Mismatch: Traces or vias on layers not defined in the stackup.

3.Backend Deep Dive: Architecture
A. Data Validation & Modeling (Pydantic)
We define strict models for the ECAD schema. Pydantic handles the initial type-checking and mandatory field presence.

```python
class Via(BaseModel):
center: tuple[float, float]
diameter: float
hole_size: float

@model\_validator(mode='after')  
def validate\_hole\_size(self):  
    if self.hole\_size \>= self.diameter:  
        raise ValueError("Via hole\_size must be smaller than diameter")  
    return self
```
