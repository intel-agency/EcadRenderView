# **Architecture Specification: ECAD RenderView Engine**

## **1\. System Overview**

The system is designed as a **data-driven pipeline** that transforms static ECAD JSON definitions into resolution-independent vector graphics (SVG) and rasterized exports (PNG/PDF).

## **2\. Structural Layers**

### **Layer 1: Data Ingress & Validation (The "Guard")**

* **Model:** Powered by Pydantic.  
* **Responsibility:** Enforce the schema and catch "shallow" errors (missing fields, wrong types).  
* **Logic:** Implements the first phase of the "14 Invalid Boards" detection.

### **Layer 2: Core Domain Logic (The "Processor")**

* **Geometry Engine:** Uses Shapely. Converts coordinate lists into Polygon, LineString, and Point objects.  
* **Coordinate Transformer:** A custom module that calculates the "Board-to-Viewport" matrix.  
  * *Normalization:* Handles designUnits (MICRON vs MM).  
  * *Inversion:* Flips the PCB ![][image1]\-axis (up is positive) to the SVG ![][image1]\-axis (down is positive).  
* **Referential Integrity:** Validates that all nets, layers, and footprints referenced in traces/components exist in the metadata.

### **Layer 3: Rendering Engine (The "Artist")**

* **SVG Driver:** Maps domain objects to SVG elements (\<path\>, \<circle\>, \<rect\>).  
* **Layering:** Implements a Z-stack based on the stackup index.  
* **Styling:** A centralized Theme configuration that maps ECAD layers to visual styles (colors, opacity, patterns for keepouts).

### **Layer 4: Interface (CLI & API)**

* **CLI:** Entry point for developers and automated testing.  
* **Web API:** A lightweight FastAPI wrapper for the 20-minute review experience.

## **3\. Data Flow Diagram**

JSON File \-\> Pydantic Model \-\> Geometric Normalizer \-\> Shapely Collection \-\> SVG Generator \-\> Final Export

## **4\. Key Design Patterns**

* **Strategy Pattern:** Different rendering strategies for different output formats (SVG vs. Raster).  
* **Dependency Injection:** The renderer is injected with a Theme and a Transformer, making it highly testable.  
* **Service Object:** The BoardValidator is a standalone service that returns a list of violations rather than throwing a single exception, allowing for comprehensive error reporting.

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAYCAYAAADzoH0MAAAA6ElEQVR4XmNgGAUMKioqfHJycr5AvEFeXv4/kG4D0o4KCgoCMDUgNhAnAvEloNwyoJoYINZGNgekyB1kABCnoEhAAVDeAIgPAZmM6HIwwAzU3AyyRVZWVgdZAig2E2QrAx7NYADUaIvuClFRUR6gAZVAJguSUpyAEWrAA6BhpiBbgey16IrwApBmqCElQLwXaIgWuhq8AKhpNdSACyRrBgGgxiKoAYvQ5YgCQFtnQA2oRpcjCgA1vgYaclJGRkYVXY4oAHO+uro6L7ocTiAlJcUlLS0tDEqeUAOmAbEkUIoJXe0oGAXYAADkdTTLFm4uLQAAAABJRU5ErkJggg==>