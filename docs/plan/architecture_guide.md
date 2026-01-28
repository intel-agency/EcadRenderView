# PCB Renderer Architecture Guide

## Introduction

This document describes the architecture of the PCB board renderer system, providing a detailed technical reference for developers working with or extending the codebase. The architecture is designed around principles of correctness, testability, and maintainability, with clear separation of concerns and explicit handling of coordinate systems and geometric transforms.

## System Overview

The PCB renderer is a command-line tool built in Python that parses ECAD JSON files and generates visual representations of printed circuit boards. The system operates entirely offline, with no external service dependencies, and produces deterministic outputs suitable for version control and automated testing.

### Architectural Principles

The architecture adheres to several core principles that guide design decisions throughout the system. Correctness takes precedence over performance, with explicit validation at every stage of the pipeline. Testability is achieved through pure functions wherever possible, avoiding global state and side effects. Simplicity is maintained by choosing straightforward implementations over clever optimizations unless profiling demonstrates a real need. Explicitness is valued over implicitness, making coordinate systems, units, and transforms explicit in the code.

### High-Level Architecture

The system is organized into distinct layers with clear responsibilities. The presentation layer consists of the command-line interface implemented with argparse or Typer, handling argument parsing, error display, and progress reporting. The application layer contains orchestration logic that coordinates parsing, validation, transformation, and rendering operations. The domain layer includes core business logic with models representing PCB elements, validation rules expressing correctness constraints, and geometric operations for transforms and measurements. The infrastructure layer provides low-level services including JSON parsing, file I/O operations, and matplotlib integration for rendering.

## Module Organization

The codebase is organized into focused modules with minimal interdependencies. The cli module serves as the application entry point, defining command-line interface and argument handling, orchestrating the rendering pipeline, formatting error messages for user consumption, and managing progress reporting.

The models module contains Pydantic data models representing all PCB elements. The Board class serves as the root model containing metadata, boundary, stackup, components, traces, vias, pours, and keepouts. The Component class represents an electronic component with reference designator, footprint name, placement transform, outline geometry, and pin definitions. The Trace class represents a copper path with unique identifier, net name, layer reference, path geometry, and width. The Via class represents a plated through-hole with unique identifier, net name, center point, diameters, and layer span. The Pin class represents a component connection point with pin name, net assignment, pad geometry, and position. The Keepout class represents a restricted area with geometry, optional layer scope, and keepout type.

The parse module handles JSON deserialization and normalization. The load_board function loads JSON from a file path and returns a Board instance or errors. The normalize_units function converts all spatial values to millimeters based on designUnits field. The parse_coordinates function handles multiple coordinate formats found in the wild. The parse_transform function extracts position, rotation, and side from component placement data.

The validate module implements validation rules at multiple levels. The validate_board function performs top-level validation orchestration. The validate_geometry function checks geometric invariants like positive dimensions and closed polygons. The validate_references function ensures all cross-references are valid. The validate_topology function checks for self-intersections and components outside boundaries. Each validation function returns a list of structured errors for reporting.

The transform module provides coordinate system conversions and geometric transforms. The flip_y function converts ECAD coordinates to SVG coordinates by inverting the Y-axis. The rotate function applies 2D rotation matrix around an origin point. The mirror function reflects geometry across an axis. The compute_viewbox function calculates SVG viewBox from board boundary with padding.

The render_svg module implements the rendering pipeline using matplotlib. The render_board function orchestrates the complete rendering process. The draw_boundary function renders the board outline. The draw_layer function renders all elements on a specific layer. The draw_trace function renders a trace with proper width. The draw_via function renders a via as a circle. The draw_component function renders component outline and reference designator. The draw_keepout function renders keepout with hatching pattern. The style_text function applies halo effect to text for readability.

The geometry module provides geometric primitives and utilities. The distance function calculates Euclidean distance between points. The bbox function computes bounding box of a geometry. The intersects function tests whether geometries overlap. The contains function tests whether a point lies inside a polygon. The transform_points function applies a transformation to a point list.

## Data Flow Architecture

The system processes data through a series of transformations, each producing a more refined representation. The initial stage loads raw JSON as Python dictionaries. The parsing stage deserializes JSON into Pydantic models with validated fields and normalized units. The validation stage checks cross-references, geometric constraints, and topological properties. The transformation stage converts ECAD coordinates to SVG space and applies component placements. The rendering stage draws geometric elements to matplotlib figure. The output stage saves the figure as SVG, PNG, or PDF.

At each stage, errors are collected rather than causing immediate failure, allowing comprehensive error reporting. The data flows through immutable transformations where possible, creating new objects rather than modifying existing ones to improve testability and prevent bugs from shared state.

## Core Data Models

### Board Model

The Board class serves as the root container for all PCB data. The metadata field contains board name, source system identifier, design units specification, and optional creation timestamp. The boundary field is a Polygon defining the board outline in millimeters. The stackup field is a list of Layer objects ordered by index. The nets field is a list of net names declared in the design. The components field is a dictionary mapping reference designators to Component instances. The traces field is a dictionary mapping unique identifiers to Trace instances. The vias field is a dictionary mapping unique identifiers to Via instances. The pours field is a dictionary mapping unique identifiers to Pour instances. The keepouts field is a list of Keepout instances.

The Board model includes validators ensuring boundary is present and valid, stackup contains at least two layers (TOP and BOTTOM), all components reference valid nets, all traces reference valid layers and nets, and all vias reference valid layers and nets.

### Component Model

The Component class represents an electronic part placed on the board. The name field is the reference designator like R1 or U1. The footprint field is the footprint identifier like RES_0603 or QFN32. The transform field contains position as a Point in millimeters, rotation in degrees, and side as FRONT or BACK. The outline field is typically a rectangle defined by width, height, and center. The pins field is a dictionary mapping pin names to Pin instances.

The Component model validates that position is within board boundary, rotation is between 0 and 360 degrees, side is one of FRONT or BACK, outline has positive dimensions, and all pins reference nets declared in the board.

### Trace Model

The Trace class represents a copper path connecting components. The uid field is a unique identifier string. The net_name field is the net to which the trace belongs. The layer_hash field references a layer in the stackup. The path field is a Polyline with at least two points. The width field is the trace width in millimeters.

The Trace model validates that width is positive, path has at least two points, layer_hash references an existing layer, and net_name references a declared net.

### Via Model

The Via class represents a plated through-hole connecting layers. The uid field is a unique identifier string. The net_name field is the net to which the via belongs. The center field is the via position as a Point. The diameter field is the outer diameter in millimeters. The hole_size field is the drill diameter in millimeters. The span field contains start_layer and end_layer names.

The Via model validates that diameter is positive, hole_size is positive and less than diameter, center is within board boundary, both start_layer and end_layer reference existing layers, and net_name references a declared net.

### Geometric Primitives

The Point class represents a two-dimensional coordinate with x and y values as floats in millimeters. The class provides methods for arithmetic operations including addition, subtraction, and scalar multiplication, distance calculation to another point, and rotation around the origin by a given angle.

The Polygon class represents a closed shape defined by a list of Point instances. The class ensures at least three points are present and validates that the polygon is not self-intersecting. Methods include computing the bounding box, testing whether a point is contained within the polygon, and transforming all points by translation, rotation, or mirroring.

The Polyline class represents an open path defined by a list of Point instances with at least two points required. Methods include computing total path length, retrieving the bounding box, and transforming all points consistently.

The Circle class represents a circular shape with a center Point and radius in millimeters. The class validates that radius is positive and provides methods for testing point containment and computing bounding box.

## Validation Architecture

### Validation Levels

Validation occurs at three distinct levels within the architecture. Field-level validation uses Pydantic validators to enforce type constraints, value ranges, and simple invariants. Model-level validation uses Pydantic model validators to check cross-field constraints within a single model instance. Board-level validation uses custom validators to check global constraints across multiple model instances.

The error reporting system collects all errors found during validation rather than failing fast on the first error. Each error is represented as a structured object containing an error code from the defined taxonomy, severity level of ERROR, WARNING, or INFO, human-readable message explaining the problem, and JSON path indicating where the error occurred.

### Validation Rules

Geometric validation checks ensure all coordinates contain finite numeric values with no NaN or Infinity, polygons have at least three points and are closed, linestrings have at least two points, circles have positive radius, and rectangles have positive width and height.

Dimensional validation checks ensure trace widths are positive, via outer diameters are positive, via hole diameters are positive and smaller than outer diameter, component outlines have positive dimensions, and all measurements are in reasonable ranges.

Reference validation checks ensure all layer references point to layers defined in the stackup, all net references point to nets declared in the board, component pins reference the correct parent component, via spans reference layers in correct order, and trace endpoints connect to valid features.

Topological validation checks ensure board boundary does not self-intersect, components are placed within board boundary allowing for reasonable tolerance, traces do not extend beyond board boundary, vias are placed within board boundary, and keepouts have valid geometry.

### Error Codes

The system defines fourteen primary error codes corresponding to the validation requirements. MissingBoundaryError occurs when the board has no boundary definition. MalformedCoordinatesError occurs when coordinate data cannot be parsed or contains invalid values. InvalidRotationError occurs when rotation is outside the valid range or not numeric. DanglingTraceError occurs when a trace references a nonexistent net or layer. NegativeWidthError occurs when a trace or via has zero or negative dimensions. EmptyBoardError occurs when the board contains no components or routing. InvalidViaGeometryError occurs when a via hole is larger than the outer diameter. NonexistentLayerError occurs when a feature references an undefined layer. NonexistentNetError occurs when a feature references an undeclared net. SelfIntersectingBoundaryError occurs when the board outline crosses itself. ComponentOutsideBoundaryError occurs when a component is placed beyond the board edge. InvalidPinReferenceError occurs when a pin references a nonexistent component or net. MalformedStackupError occurs when the layer stack is incomplete or inconsistent. InvalidUnitSpecificationError occurs when designUnits is missing or unrecognized.

## Coordinate System Architecture

### Coordinate System Definitions

The system works with two primary coordinate systems. ECAD coordinates use a right-handed coordinate system with the origin at the bottom-left corner of the board boundary, X-axis pointing to the right, Y-axis pointing up, and rotation measured counter-clockwise from the positive X-axis. SVG coordinates use origin at the top-left corner of the viewport, X-axis pointing to the right, Y-axis pointing down, and rotation measured clockwise from the positive X-axis.

The fundamental difference between these systems is the direction of the Y-axis. ECAD uses a mathematical convention with Y increasing upward, while SVG uses a graphics convention with Y increasing downward. This necessitates a Y-axis flip during the coordinate conversion process.

### Unit Normalization

All spatial values are normalized to millimeters immediately after parsing. The normalization process detects the designUnits field in the metadata, converts micron values to millimeters by dividing by 1000, passes through millimeter values unchanged, defaults to microns with a warning if units are unspecified, and rejects boards with unrecognized unit specifications.

After normalization, all coordinates, dimensions, widths, and diameters are in millimeters. This eliminates the need for unit tracking throughout the rest of the pipeline, simplifying code and reducing the risk of unit mismatch errors.

### Transform Pipeline

The coordinate transform pipeline operates in a well-defined sequence to ensure correct results. For component placement, the process begins with the component defined in its local coordinate system with origin at the component center. Translation applies the component's position offset to move it to the board location. Rotation happens around the component's local origin using the specified angle. Mirroring occurs for back-side components by reflecting across the X-axis. Finally, Y-axis inversion converts from ECAD to SVG coordinate space.

The implementation of these transforms uses matrix operations for efficiency when processing many points. The translation matrix uses an identity matrix with the position offset in the translation column. The rotation matrix uses the standard 2D rotation matrix with cosine and sine of the angle. The mirror matrix uses identity with the Y diagonal element negated. The flip matrix uses identity with the Y diagonal element negated.

Transforms are composed by matrix multiplication in the reverse order of their application, allowing a single matrix multiply per point to achieve the complete transform. The composed transform matrix is computed once per component and then applied to all points in the component outline and pin positions.

## Rendering Architecture

### Matplotlib Integration

The rendering engine uses Matplotlib's object-oriented API for precise control over the drawing process. The rendering begins by creating a Figure and Axes with appropriate size. The aspect ratio is set to equal to prevent distortion of the board geometry. The Y-axis is inverted to match SVG conventions after all coordinate transforms have been applied. Axes are hidden as they are not relevant for the final output.

The rendering process draws elements in a deterministic order to ensure consistent visual hierarchy. Board outline is drawn first using a Polygon patch with dark stroke and no fill. Copper pours follow with Polygon patches using low opacity to avoid obscuring traces. Traces are drawn as Line2D objects with width scaled appropriately. Vias appear as Circle patches with solid fill and contrasting outline. Component outlines are drawn as Polygon patches. Reference designators are rendered as Text objects with halo effects. Keepouts overlay everything with distinctive hatching patterns.

### Layer Rendering

Each copper layer is rendered with a distinctive color to differentiate layers visually. The top layer uses red with hex code #CC0000. The bottom layer uses blue with hex code #0000CC. Inner signal layers alternate between green #00CC00 and purple #CC00CC. Ground planes use dark gray #404040. Power planes use orange #CC6600.

The layer rendering process filters elements by their layer assignment, collecting all traces on the target layer, all vias spanning the target layer, and all component pads on the target layer. For each element, the appropriate drawing function is called with layer-specific styling applied.

### Component Rendering

Component rendering involves multiple steps to produce a complete representation. The component outline is drawn as a Polygon representing the component body boundary. The reference designator is placed at the component centroid with rotation normalized to keep text upright. Pin pads are drawn as rectangles or circles depending on pad geometry. For through-hole pins, a via-like representation shows the hole.

The reference designator styling ensures readability across all backgrounds. Font size scales with board dimensions, typically between 0.5mm and 2.0mm in world coordinates, ensuring text is legible but not overwhelming. Text color uses high contrast white or black based on the underlying layer color. A halo effect is implemented using Matplotlib's path effects, drawing a thick stroke in a contrasting color behind the text.

### Keepout Rendering

Keepouts receive distinctive visual treatment to clearly mark restricted areas. The fill pattern uses diagonal hatching with the matplotlib pattern code '///' applied at high density. Edge color uses a bright warning color such as red #FF0000 or orange #FF6600. Stroke width is set to 2 to 3 points for visibility. Fill opacity is set to 0.5 to show underlying features while still clearly marking the restricted area. Z-order is set to a high value ensuring keepouts appear on top of all other features.

### Text Rendering

Text elements including reference designators require special handling for readability. The font family uses a sans-serif face such as 'DejaVu Sans' for consistency across platforms. Font size is computed based on board dimensions, with a typical formula of font_size equals 0.002 times the maximum board dimension with bounds clamped between 0.5mm and 2.0mm. Text color is determined by the luminance of the background, using white text on dark backgrounds and black text on light backgrounds.

The halo effect uses matplotlib's path effects to create an outline around the text. The implementation uses patheffects.withStroke with linewidth set to 2 to 3 points and foreground color set to the opposite of the text color. This creates a strong contrast that makes text readable even when overlapping complex copper patterns.

### Output Format Handling

The rendering engine produces three output formats using matplotlib's savefig function. For SVG output, the backend is set to 'svg', DPI is set to 72 for web display, bbox_inches is set to 'tight' to remove excess whitespace, and pad_inches is set to 0.1 for minimal border. The resulting SVG is a standalone document with embedded styles that can be opened in browsers or vector editing tools.

For PNG output, the backend is set to 'agg' for anti-aliased raster graphics, DPI is configurable with typical values of 150 for preview, 300 for print, or 600 for archival, bbox_inches is set to 'tight', and pad_inches is set to 0.1. The resulting PNG is suitable for embedding in documents or web pages.

For PDF output, the backend is set to 'pdf', DPI is set to 72 as PDF is vector-based, bbox_inches is set to 'tight', and pad_inches is set to 0.1. The resulting PDF embeds fonts and maintains vector fidelity, making it suitable for documentation or printing.

## Testing Architecture

### Testing Levels

The testing strategy employs multiple levels of testing to ensure correctness. Unit tests verify individual functions and classes in isolation, with each module having a corresponding test module, tests using fixtures to provide test data, and coverage targeting 100% of public interfaces.

Integration tests verify that modules work correctly together, with tests exercising the complete parsing pipeline, tests running full render operations, and tests validating end-to-end workflows from JSON to output file.

Property-based tests use Hypothesis to generate random inputs and verify invariants. Tests generate random valid geometries and verify geometric properties are preserved, tests generate random boards within valid ranges, and tests verify that the system never crashes regardless of input.

Snapshot tests capture rendered outputs and detect unintended changes, with SVG outputs stored as text snapshots, tests failing when outputs differ from baselines, and snapshots updated intentionally using the update-snapshots flag.

### Test Organization

Tests are organized in a tests directory parallel to the source code. The structure includes test_parse.py for parsing logic tests, test_validate.py for validation rule tests, test_transform.py for coordinate transform tests, test_render.py for rendering pipeline tests, test_geometry.py for geometric primitive tests, invalid_boards directory containing crafted invalid JSON files, and snapshots directory containing reference outputs.

Each test file uses fixtures defined in conftest.py to provide common test data. Board fixtures create minimal valid boards, boards with specific features, and invalid boards with known errors. Geometry fixtures provide points, polygons, polylines, and circles. Transform fixtures provide rotation angles, mirror configurations, and coordinate system setups.

### Property-Based Testing

Property-based testing with Hypothesis provides generative coverage of the input space. The strategy defines custom strategies for generating valid geometries including point strategies generating coordinates in reasonable ranges, polygon strategies ensuring at least three points with no self-intersection, trace strategies generating valid paths with positive widths, and component strategies generating valid placements within board boundaries.

Properties verified include geometric invariants such as polygon area being preserved under rotation and translation, component bounding boxes being preserved under rotation, and via hole sizes always being less than outer diameters. Topological properties verified include boards always having a valid boundary, components always being placed within boundaries, and traces always connecting to valid features. Parse-validate roundtrip properties verify that valid boards parse without errors, invalid boards produce expected error codes, and rendering never crashes on valid boards.

### Snapshot Testing

Snapshot testing provides visual regression detection. The implementation uses pytest-regressions or syrupy for snapshot management. For each test, the rendered SVG output is captured as text, compared character-by-character with stored baseline, with test failure and diff generation on mismatch, and baseline updates through the update-snapshots flag.

The snapshot workflow supports natural development iteration. During initial development, tests run with update-snapshots to establish baselines. During maintenance, tests run normally and fail on any output change. On intentional changes, tests run with update-snapshots to accept new outputs. During code review, snapshot diffs are examined to verify changes are intentional.

### Invalid Board Testing

The test suite includes fourteen crafted invalid boards corresponding to the required error types. Each invalid board is stored as a JSON file in tests/invalid_boards/ with a descriptive name. The testing process loads each invalid board JSON, attempts to parse and validate, asserts that validation fails, verifies the expected error code appears, and confirms the error message is actionable.

The invalid boards cover missing_boundary.json demonstrating MissingBoundaryError, malformed_coordinates.json demonstrating MalformedCoordinatesError, invalid_rotation.json demonstrating InvalidRotationError, dangling_trace.json demonstrating DanglingTraceError, negative_width.json demonstrating NegativeWidthError, empty_board.json demonstrating EmptyBoardError, invalid_via_geometry.json demonstrating InvalidViaGeometryError, nonexistent_layer.json demonstrating NonexistentLayerError, nonexistent_net.json demonstrating NonexistentNetError, self_intersecting_boundary.json demonstrating SelfIntersectingBoundaryError, component_outside_boundary.json demonstrating ComponentOutsideBoundaryError, invalid_pin_reference.json demonstrating InvalidPinReferenceError, malformed_stackup.json demonstrating MalformedStackupError, and invalid_unit_specification.json demonstrating InvalidUnitSpecificationError.

## CLI Architecture

### Command Structure

The command-line interface provides a clean user experience with intuitive commands. The main command pcb-render serves as the entry point with subcommands for specific operations. The render subcommand takes input file path, output file path, optional layer filter, and optional format override. The validate subcommand takes input file path and optional json output flag. The help subcommand displays usage information and examples.

The argument parsing uses argparse or Typer with proper type hints, default values, and help text. Positional arguments are used for required inputs like file paths. Optional arguments use flags with sensible defaults. Mutually exclusive options are grouped appropriately.

### Error Reporting

The CLI provides clear, actionable error messages when operations fail. For validation errors, the output includes a summary showing total error count, individual errors with code, severity, message, and JSON path, and suggestions for fixes when applicable. For file I/O errors, messages clearly state what operation failed and why. For unexpected exceptions, a helpful error message directs users to file a bug report with a stack trace.

The error formatting uses indentation and color (when terminal supports it) to improve readability. Summary statistics appear first in bold. Error details are indented under the summary. JSON paths are highlighted to help locate issues. Severity levels use different colors with ERROR in red, WARNING in yellow, and INFO in blue.

### Progress Reporting

For long-running operations, the CLI provides progress feedback. In verbose mode, status messages are printed for each major stage including "Loading board", "Validating geometry", "Computing transforms", "Rendering layers", and "Writing output". For batch operations, a progress bar shows percentage complete and estimated time remaining. Progress information is written to stderr to avoid interfering with structured output on stdout.

### Configuration

The CLI supports configuration files to avoid repetitive command-line arguments. Configuration files use TOML format and are loaded from the current directory or user home directory. Configuration sections include rendering with default format, default DPI, and default layers. Styling includes color schemes and font preferences. Validation includes error threshold levels. Output includes default output directory and file naming patterns.

Command-line arguments override configuration file values following standard precedence rules. Environment variables with the prefix PCB_RENDER_ override configuration file values. Explicit command-line arguments override environment variables. This allows flexible configuration while maintaining explicit control when needed.

## Performance Considerations

### Hot Paths

Performance-critical code paths are identified through profiling and optimized appropriately. The coordinate transform pipeline processes potentially thousands of points per board, so matrix operations are vectorized using NumPy when available. The rendering loop draws many geometric primitives, so matplotlib Artists are created in batch and added to the axes in a single operation. The validation pipeline checks many constraints, so expensive checks are skipped early when possible based on cheaper checks failing.

### Memory Management

The system minimizes memory usage through careful data structure choices. Large coordinate arrays use NumPy arrays when possible for compact representation. Intermediate geometries are not retained after rendering completes. The rendering process streams elements to matplotlib rather than building a complete scene graph in memory.

For very large boards, the system can process elements in chunks rather than loading the entire board into memory at once. This requires restructuring the JSON parsing to use iterative parsing rather than loading the complete document.

### Caching

Computed values that may be reused are cached to avoid redundant computation. The viewBox calculation is cached after first computation. Transformed component geometries are cached after initial placement. Bounding boxes are cached after initial calculation. The caching strategy uses LRU caches from functools for simplicity, with cache sizes tuned based on typical board complexity.

## Extension Points

### Custom Validators

The validation system supports custom validators through a plugin architecture. New validators are defined as functions taking a Board and returning a list of errors. Validators are registered through a decorator or explicit registration call. The validation pipeline automatically discovers and runs all registered validators.

Example custom validators might check for minimum trace spacing based on manufacturing constraints, verify component placement follows design rules, validate that high-speed signals meet length matching requirements, or ensure power distribution meets current capacity requirements.

### Custom Renderers

The rendering system supports custom renderers for different output formats or styles. New renderers implement a common interface with methods for drawing each element type and generating output in the target format. Renderers are registered and selected based on output format or explicit user choice.

Example custom renderers might generate 3D views using matplotlib's 3D toolkit, produce interactive HTML with embedded JavaScript, generate Gerber files for fabrication, or create specialized views for specific analysis tasks like thermal or signal integrity.

### Custom Styles

The styling system supports themes for different visual appearances. Styles are defined as dictionaries or configuration files specifying colors for each layer, fonts and sizes for text elements, patterns for keepouts and pours, line widths for traces and outlines, and opacity values for overlapping elements.

Users can create custom styles by copying and modifying a base style template, then loading the custom style with a command-line argument or configuration file setting.

## Security Considerations

The system processes untrusted JSON files and must handle malicious or malformed input safely. Input validation rejects JSON files larger than a reasonable limit (e.g., 100 MB) to prevent memory exhaustion attacks. Schema validation using Pydantic catches malformed data before it reaches business logic. Numeric validation rejects NaN and Infinity values that might cause rendering issues. String validation limits the length of all string fields to prevent buffer-related issues.

The file I/O operations use safe path handling to prevent directory traversal attacks. Output paths are validated to ensure they don't escape the intended output directory. Temporary files use secure temporary directory creation with appropriate permissions. File operations have appropriate error handling to avoid information leakage through error messages.

## Deployment and Distribution

The system is distributed as a Python package installable via pip or uv. The package structure includes a pyproject.toml with dependencies and metadata, a README with installation and usage instructions, a LICENSE file (MIT or similar permissive license), and tests and documentation in the repository but excluded from the package.

Installation is straightforward using uv with the commands uv pip install pcb-renderer to install from PyPI or uv pip install -e . to install in development mode from the local repository. For end users unfamiliar with Python, pre-built executables can be created using PyInstaller or similar tools, providing single-file executables for Windows, macOS, and Linux that include all dependencies and require no Python installation.

## Monitoring and Observability

The system provides logging for troubleshooting and debugging. The logging configuration uses Python's standard logging module with levels DEBUG for detailed diagnostic information, INFO for normal operation messages, WARNING for recoverable issues, and ERROR for failures requiring user attention.

Log output is formatted consistently with timestamp, level, module name, and message. For development, logs are written to stderr with color coding. For production use, logs can be directed to files with rotation based on size or time. The logging configuration can be controlled through environment variables or configuration files.

## Conclusion

This architecture guide provides a comprehensive technical reference for understanding and extending the PCB board renderer system. The architecture prioritizes correctness through explicit modeling of coordinate systems and geometric transforms, testability through pure functions and deterministic outputs, maintainability through clear module boundaries and comprehensive documentation, and extensibility through plugin points for validators, renderers, and styles.

The system demonstrates how careful attention to fundamentals such as coordinate system handling, validation design, and testing strategy creates a robust foundation that can handle diverse PCB designs while remaining simple enough to understand and modify. Future enhancements can build on this foundation without requiring architectural changes, ensuring the system remains maintainable as requirements evolve.
