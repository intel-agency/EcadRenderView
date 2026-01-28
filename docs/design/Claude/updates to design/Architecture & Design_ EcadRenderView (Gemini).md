# **Architecture & Design: PCB Board Renderer**

## **1\. Analysis of Requirements vs. Constraints**

### **Platform & App Type Selection**

* **Recommendation:** **Cross-platform Python CLI.**  
* **Justification:** \* **MacOS Compatibility:** Python is natively supported on MacOS. A CLI tool is the most "portable" interface for a technical reviewer to execute in their terminal without installing browser dependencies or GUI runtimes.  
  * **Developer Constraint:** Since you lack a MacOS environment, a CLI ensures that what you build on Linux/Windows will behave identically on a reviewer's Mac.  
  * **Verification Speed:** The "20-minute review" requirement favors a single command (e.g., python render.py design.json \--format svg) over a REST API which requires managing ports and client requests.

### **Output Formats**

* The requirements demand **SVG, PNG, and PDF**.  
* **Selection:** **Matplotlib** or **PyCairo**.  
  * *Matplotlib* is highly recommended here. It treats the PCB as a coordinate system, handles rotations/transforms natively, and its savefig() function supports SVG, PNG, and PDF out of the box with high fidelity.

## **2\. High-Level Architecture**

The application will follow a pipeline architecture to ensure clean separation of concerns:

1. **Ingestion (CLI):** Handles arguments and file paths.  
2. **Validation Layer:** Checks the integrity of the JSON structure and geometry.  
3. **Data Model:** A set of Python Dataclasses/Pydantic models representing the PCB (Board, Component, Trace, Via, Keepout).  
4. **Renderer Engine:** Translates the Data Model into visual primitives (Lines, Circles, Polygons).  
5. **Exporter:** Saves the rendering to the requested file format.

## **3\. Detailed Component Design**

### **A. Validation & Error Handling (Requirement \#6)**

To detect the "14 invalid boards," we will implement a two-stage validation:

* **Structural Validation:** Using Pydantic to ensure mandatory fields (like boundary.coordinates or traces.width) exist and have the correct types.  
* **Business Logic Validation:** \* Check for self-intersecting boundaries.  
  * Check for traces or components located entirely outside the board boundary.  
  * Check for invalid reference designators or missing lookup IDs.  
  * Check for zero-width traces or zero-diameter vias.

### **B. The Renderer (Matplotlib Backend)**

* **Coordinates:** PCB data often uses different units (mils, mm). The renderer will normalize these to a standard unit.  
* **Layers:** We will use a dictionary mapping layer names (e.g., top, bottom) to specific hex colors.  
* **Rotation:** Component outlines will be rotated using a 2D transformation matrix (![][image1]) before plotting.

### **C. Directory Structure**

pcb\_renderer/  
├── main.py                 \# Entry point (CLI logic)  
├── models/  
│   ├── pcb.py              \# Data structures  
│   └── validator.py        \# 14+ Validation rules  
├── rendering/  
│   └── engine.py           \# Matplotlib implementation  
├── tests/  
│   ├── test\_parsing.py     \# Unit tests for invalid boards  
│   └── test\_rendering.py   \# Visual regression tests  
└── requirements.txt

## **4\. Library Recommendations**

| Category | Library | Reasoning |
| :---- | :---- | :---- |
| **Parsing/Validation** | Pydantic | Simplest way to catch missing fields and type mismatches automatically. |
| **Graphics** | Matplotlib | Native support for SVG, PNG, PDF; handles paths and transforms easily. |
| **Geometry** | Shapely | (Optional) Excellent for "Keepout" intersection checks and boundary validation. |
| **CLI** | Typer or argparse | argparse is built-in (no dependencies), Typer is prettier. |

## **5\. Strategy for "14 Invalid Boards"**

The challenge specifically mentions 14 cases. We should implement a BoardValidator class that runs checks such as:

1. MissingBoundaryError: No boundary key.  
2. MalformedCoordinatesError: Coordinates list length is not even or less than 3 points.  
3. InvalidRotationError: Rotation value not a number.  
4. DanglingTraceError: Trace refers to a net or component ID that doesn't exist.  
5. NegativeWidthError: Trace width ![][image2].  
6. EmptyBoardError: Boundary exists but contains no components or traces.  
   ...and so on.

## **6\. Implementation Plan**

1. **Step 1:** Define the Pydantic schemas for the ECAD JSON.  
2. **Step 2:** Write the validator logic to catch the 14 edge cases (this fulfills a major part of the "Backend" challenge).  
3. **Step 3:** Implement the Matplotlib renderer for the board boundary and components.  
4. **Step 4:** Add Trace (lines) and Via (circles) support with layer-based coloring.  
5. **Step 5:** Add Keepout regions using a "hatch" pattern (Matplotlib fill with hatch='///').

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAKgAAAAZCAYAAACl3WVkAAAHGklEQVR4Xu1aW2tdRRTeSQ5a760SU5N0z85F0kStELyVRmk1sYgKilKxtKAQaUV9aJWKNyoWpEibItRaWrVSqGBLoZCSNNjQ0mCxSIOI9CUveetDf0T9vjNrdtaes8/JISY5VueDxZ5Zs9Zc1qyZWTPnRFFAQEBAQEBAQBQlSfKmMWY6juMX/LKAfLS3t9+1YsWKB/n1ywIyKMC32vj1C6oGKhiHc47B4M1+WUAWLS0t98Behzo7O+9kHnZ7FdTuy91IwHjOga6DvvLL5grU9TzoMmzT39raejfSJ0Ef+HJVAYrXRLnOLwvIAnbaixPngsu3tbU9jEnYrGVuNKD/l8RBD/hlcwXqmoKdtkXWpxqQ3g3eiC9XFaD8J48rnx9QCi5m2GuLyyO9ksZHskGJ3VDgKYBxfYPFtsovmwtwujSivrM4bVodD23sBO+KlqsKPNbZQZ8fUAoYeNB3RuRfgv0ONjc336pE/7cQfxrjyeJ4sNFS2O4Ed2ktS9QhiI8h/BjigFvIYBr0lBOA0upax568aGBQfapfdejXOu5OSNdrWQ2OCXJPQ+4j6L+F/P2eSHH8LIfcu1JfBmxb9HdIm+t9GcLFUXRIzQdvO3T39fT03KT5C4mmpqbbaC9+/bJy/YBtm9DXjewv5vtR+oEUFZjm2EG9wqtH/T3CW808xp0wT3tGFUJB2Y2v0ykdT3QnQNOpYGNj4+1g7AIdQ/lxKJ5hY6Ah5Hc7D0f6jahCg4sAOuOwHAFD+G4AvYf0J6BfQIO+AgGZNSj7DTRKw4kTXotkLLzEQOZL8C5K+euU5yXH1QHeWvDO4bscsu1If8x+pI0oYFKfZP00tuMhuQS8A+yrEl1QsK9o7xjoNOgkj1NXhnxvXl+MdczTGNsAFtpDSO+JJW5GfSuNjT+vg3eQPLkIFnmgUfC/pY6zMfKbsi3MgHVQT/OM9TvqXdLM/e7YkU7wRjWm0gOpcJWATh87mUPrjV1tGQI/8evQkF3pB5dnfzk4qK11AwXt0joEypZxwHQax4POa+BNY8JuZl7q2TqjZSF18qgu2sF4E5rnoNKvw1JnOm7XR32cLTTQ3l6O0djdPL07sI/SH+54vg6dL3OR03nOq8gUHZQQJ+XizzhbngNqiH15QUrtZOwmQdsxPLLgJKs0HYgd2IdsAd9nogpH52IBE7sK/Xrf5RlqID+B/vJI6Aa9nHeMcXGhbLxcaCKOz/H2+WViwBMSyI8Yu7L76PRRmZNE+jVOXc1Hfjt5XV1dd2j+QgLtPSdf9vsIT0rmxWYTeTYxdiFehcjnTl5jFgfNXGyqdNCjHo/HOx00N3xKDemvon8bGBvRALNdODDQrRjPMB3RLyMSObYqOOhlJOtQvsnYJzbySJNRzoOy1MdJZnkRdErkj1JPy/qA7hOq/lnJ1y+DBsrG6nKrQpAlWpBI7C6m2zkfqcU4i4PSVimqdNAvfF5c6Z3dWA8e0fHKXMCBcLDVEGR3cIf06ygHBvfQ2aeP7XJIrIOO6nhSI5nFQWMbCxXksZ0TVQ9nb4HeFk4I9bUOj3Dwr+gJRH6QdSX2rW9RYWxMd9gtZHW8MwYvgd41IdMG+lT7wzw6qFs4qd3plLDRBbarBdmRIXdTE6X0psmOQWl5RqEKoI4B3xErETr3iF+HBo2AOj9EsuCOUeglUszL02co71cqRXCnRdmVnNivDupL5YJI58ncuAmxxZHEvl/+7C1atlni2HwdoFPrCUR6DHQqmYMd/ynQ7majYmfazNhNaFzLORjlzAL+/JguxHl0UGff1H7ID8b2UjUTPslk8wbcjcIH8J0y8ksRdyjwfkyFawf368KUHJf7aQx3bNOooHW+kgPKhqmLsWxgXi4OnLSiIWL7bMRdr1ge2aeTF8lHusDJYXusR8qLdcQqrtOgoY0cjZDpj+3xmhuzLjSMva3zZ8liOGJmLiEll0KCZaBD7sThyWHs5bOBm5axt3zK/CRjr5dTg/VOc04oJ09yxynLhZ0XisEuYyh/R9rgC0cmHnXgVvsrOyUKr4D+kMq/A93nK9QC6M/jxq78s6CNMEoXvudB36Pfa6IKDtDR0XFvbJ+RruJ7BnVdAL2tRGjkZ1kfy/H9nWnyWSgOSifn+yif4Q7G9ie/XNtw4iD3tbELZyKq0LdFQD36MC22ou34ZSxdcoMnOE70fRvHxwWI/KSEN+nu6Qj5nd4zE4kLuZc28mRL7jSJfbqbBJ3j/Lh2SkCDsiF3rNPbZQXV0rAlYP/0jsXVWu7ykwc3rnIP1EA9y+VfR3rsBdcuDLmMMnk7pwfGquxbyUVqseHGzb7DGS5yIyp3CVE7XYE6eTvffIL1V2HLgP8ijN310xgQ6ba4RrFwQEAJ6KAMSZjGbtgqYVu3LxcQUBMYe6H5C7SHX8Z5vkxAQEBAQEBAQEBAQBZ/A25LR/tBUr0WAAAAAElFTkSuQmCC>

[image2]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAB4AAAAXCAYAAAAcP/9qAAABk0lEQVR4Xu2UvUvDUBTFk7YoKuLXEEg/0pRApOLmJgii4CSIg5uik4vipjiI4qKbi4OOzuKigzh1E0QQd/Fv0d+tTYmXNE2w6tIDl/dy7rnvvLzcF8PooosfoFwujzuOs8W4z+hCmVqTCJZlDbDIZalUGtE5jUKh0IfZM/p113V95q/U7Xme16u1saBoh3hiEUvnImBiuInZbEDInHhjjbWQLhIZiq8Q1xAv6mQc0K9S96FoE+44gm8iR+E8preM0zxntKANstSeRhmw3mEUX/+GJB6Jc76Rp/NJYNt2PwYXUQaBsWgC4kTOn1hW2tTI5/NjrHMfZyyagDMhJ+RNGVekI8MFaZDWuAkSB8QLom2dSwI2PUr9XZyx7/uDOvcN0s0Ia3ISRvLL37J7WzZXC2S4uzMU3Miok1Ggq5caBtmAq1arPRifpTGuI9QDc0abK8YGHTkpNjAccDKHu6b+Iaz9DeQwesfwiFhozDeE18KOQ36vcuy85W6lUhnS+f8DV2DS+fqJt41isThl/MVxddEpfAJnZ2HHiJnP8gAAAABJRU5ErkJggg==>