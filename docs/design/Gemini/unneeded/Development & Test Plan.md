# **Development & Test Plan: ECAD RenderView**

## **Phase 1: The Validation Matrix (Catching the 14 Errors)**

We will implement a ValidationSuite that specifically targets the following failure modes identified in the backend requirements.

| ID | Error Type | Detection Logic |
| :---- | :---- | :---- |
| **V01** | Missing Metadata | Pydantic schema validation. |
| **V02** | Invalid Boundary | Shapely.is\_valid check (closed loop, no self-intersections). |
| **V03** | Unit Mismatch | Checking for illegal designUnits strings. |
| **V04** | Dangling Net | Verifying trace.net\_name exists in nets list. |
| **V05** | Phantom Layer | Verifying trace.layer\_hash exists in stackup.layers. |
| **V06** | Invalid Via Geometry | Check: diameter \<= hole\_size. |
| **V07** | Out-of-Bounds Component | Intersection check: component.outline must be within boundary. |
| **V08** | Trace Width Error | Check: width \<= 0\. |
| **V09** | Empty Stackup | Check: len(layers) \== 0\. |
| **V10** | Overlapping Keepouts | Detection of critical keepout violations (Optional/DRC). |
| **V11** | Malformed Polygon | Less than 3 points in a coordinate list. |
| **V12** | Reference Designator Conflict | Duplicate name or reference in components. |
| **V13** | Through-hole Span Error | Via span references non-existent layers. |
| **V14** | Logic Loop | Component pin referencing its own component recursively (circular refs). |

## **Phase 2: Implementation Sprints**

### **Sprint 1: Foundation (Days 1-2)**

* Project scaffolding (Poetry/Pipenv).  
* Pydantic models for the JSON spec.  
* Unit tests for the 14 validation cases using pytest.

### **Sprint 2: The Geometry Engine (Days 3-4)**

* Integration of Shapely.  
* Implementation of the AffineTransform for coordinate flipping.  
* Logic for component rotation (handling degrees vs radians).

### **Sprint 3: Visuals & Export (Days 5-6)**

* SVG generation logic using svgwrite.  
* Layer coloring and keepout patterning.  
* CairoSVG integration for PNG/PDF.

### **Sprint 4: Interface & Review (Day 7\)**

* CLI implementation via Typer.  
* FastAPI "Preview" server.  
* Dockerization for the MacOS reviewer.

## **Phase 3: Testing & Quality Assurance**

* **Coverage Target:** \>90%. Verified via pytest-cov.  
* **Snapshot Testing:** We will maintain a golden\_snapshots/ folder. Every render of board\_zeta.json must match the reference SVG to within 0.1% visual difference.  
* **Fuzz Testing:** Using Hypothesis to generate random JSON structures to ensure the parser never raises an unhandled 500 or Traceback.