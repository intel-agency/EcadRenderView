# PCB Renderer Development Plan

## Executive Summary

This document provides a comprehensive development plan for building a PCB board renderer CLI tool. The implementation follows a phased approach prioritizing core functionality, correctness, and testability. The system parses ECAD JSON files and generates visual representations in SVG, PNG, or PDF formats while detecting 14 classes of invalid board configurations.

## Project Overview

### Primary Objectives

The system must parse PCB board JSON files containing components, traces, vias, pours, nets, and stackups, then render readable board visuals with required features including keepouts and reference designators. All spatial measurements are normalized to millimeters immediately after parsing to ensure consistency throughout the pipeline.

### Success Criteria

Success is measured by the ability to correctly render all valid board files, detect all 14 categories of invalid boards through validation, produce deterministic and testable outputs, and maintain a codebase that can be reviewed and verified within 20 minutes.

## Technical Foundation

### Technology Stack Rationale

Python 3.11+ serves as the implementation language due to its strong ecosystem for scientific computing, excellent library support for geometric operations, and cross-platform compatibility. The choice of Python 3.11 specifically provides performance improvements through the faster CPython implementation and enhanced error messages that aid debugging.

Pydantic v2 handles data modeling and validation, providing automatic validation of incoming JSON data, comprehensive error reporting with field-level precision, and generation of JSON schemas for documentation. The v2 release offers significant performance improvements over v1, which is critical given the potentially large number of geometric elements in complex boards.

Matplotlib serves as the rendering engine, chosen for its comprehensive support for all three required output formats through a single savefig() API, mature and well-tested SVG generation, and ability to handle complex geometric primitives including polygons, polylines, circles, and text with advanced styling options.

NumPy provides optional acceleration for geometric transforms and batch operations on coordinate arrays, though the core logic remains NumPy-optional to minimize dependencies for simple use cases.

The uv package manager provides reproducible builds through lockfiles, faster dependency resolution than pip, and unified handling of virtual environments and project metadata.

### Development Environment Setup

Developers should install uv following the official installation instructions for their platform. The project initialization begins with cloning the repository, then running uv sync to install all dependencies including development tools. The development environment includes pytest for testing, ruff for linting and formatting, pyright for type checking, and pytest-cov for coverage reporting.

The project requires Python 3.11 or higher, which provides pattern matching syntax, improved error messages, and performance improvements that benefit geometric computation workloads.

## Phase 1: Foundation and Data Models

### Core Data Structures

The foundation begins with defining canonical internal representations for all PCB elements. The Point class represents a two-dimensional coordinate with x and y values in millimeters. The Polygon class contains a list of points forming a closed shape with validation ensuring at least three points and optional automatic closure. The Polyline class represents an open path with at least two points. The Circle class defines center point and radius with positive radius validation.

The Board model serves as the root container, holding metadata including board name, source system, design units, and optional creation date. The boundary is represented as a Polygon defining the board outline. The stackup is a list of layers with properties including name, type (TOP, BOTTOM, MID, PLANE, DIELECTRIC), index, and material characteristics. The model also contains dictionaries of components keyed by reference designator, traces keyed by uid, vias keyed by uid, pours keyed by uid, and keepouts as a list.

Component models include reference designator, footprint name, placement transform with position, rotation in degrees, and side (FRONT or BACK), component outline as a Polygon or rectangle, and pins as a dictionary keyed by pin name. Each pin contains its name, parent component reference, connected net name, pad geometry, position relative to component origin, rotation, and throughhole flag.

Trace models specify a unique identifier, net name for connectivity tracking, layer hash referencing the stackup, path as a Polyline of coordinates, and width in millimeters. Via models define unique identifier, net name, center point, outer diameter, hole diameter with validation ensuring hole is smaller than outer diameter, and layer span with start and end layer references.

### Validation Architecture

Validation operates at multiple levels within the architecture. Model-level validation uses Pydantic validators to enforce field constraints, including type checking, range validation, and cross-field dependencies. The validation decorators use after-mode validators to access the complete model state, enabling complex invariants to be checked after all fields are populated.

Geometric validation ensures coordinate arrays contain no NaN or Infinity values, polygons have at least three points, linestrings have at least two points, and all dimensions (widths, diameters, radii) are positive. The validation layer enforces closure policies for polygons, either auto-closing by duplicating the first point as the last, or raising errors for open polygons depending on configuration.

Reference validation confirms all layer hashes reference actual layers in the stackup, all net names reference declared nets, component pin references point to valid components, and via spans reference layers that exist in the stackup in the correct order.

The error model provides structured reporting with error code from a defined taxonomy, severity level (ERROR, WARNING, INFO), human-readable message, and JSON path to the problematic field. Core error codes include MissingBoundaryError for boards without boundary definitions, MalformedCoordinatesError for unparseable coordinate data, InvalidRotationError for rotation values outside expected ranges, DanglingTraceError for traces referencing nonexistent nets or layers, NegativeWidthError for traces or vias with invalid dimensions, EmptyBoardError for boards with no components or features, InvalidViaGeometryError for vias with holes larger than outer diameter, NonexistentLayerError for features referencing undefined layers, NonexistentNetError for features referencing undeclared nets, SelfIntersectingBoundaryError for board boundaries that cross themselves, ComponentOutsideBoundaryError for components placed beyond board edges, and InvalidPinReferenceError for pins referencing nonexistent components.

### Parsing Strategy

The parsing pipeline begins with loading JSON and performing initial deserialization into Python dictionaries. The parser then normalizes units by detecting the designUnits field and converting all spatial values to millimeters. For MICRON units, values are divided by 1000. For MILLIMETER units, values are passed through unchanged. If units are absent or unrecognized, the parser defaults to microns with a warning.

Coordinate parsing handles multiple observed formats. Some boards provide flat lists like [x1, y1, x2, y2, x3, y3] which must be paired into Point objects. Others provide nested pairs like [[x1, y1], [x2, y2], [x3, y3]] which are directly converted. The parser detects the format by examining the structure of the first coordinate array.

Transform parsing extracts position, rotation, and side from component placement data. Rotation defaults to zero if absent. Side defaults to FRONT if not specified. The parser validates that rotation is numeric and side is one of FRONT or BACK.

Error handling during parsing uses a collector pattern to accumulate all errors rather than failing on the first issue. This provides developers with a complete picture of what needs fixing. The parser catches JSON syntax errors, schema validation failures from Pydantic, and custom geometric validation errors, presenting them all in a unified format.

## Phase 2: Geometric Transforms

### Coordinate System Design

The system defines coordinate systems explicitly to avoid confusion. ECAD coordinates use a right-handed system with origin at the lower-left corner of the board boundary, X increasing rightward, Y increasing upward, and rotation specified in degrees with zero pointing along the positive X-axis. SVG coordinates use origin at the top-left corner of the viewport, X increasing rightward, Y increasing downward, and rotation measured clockwise from the positive X-axis.

The Y-axis inversion is handled by a single transformation applied at render time. The transform.py module provides a flip_y function that takes ECAD coordinates and board height, returning SVG coordinates. This transformation is applied consistently to all geometric elements before SVG emission.

### Transform Pipeline

Component placement transforms operate in a defined sequence to ensure correct results. Position translation applies the component position offset to all pin and outline coordinates. Rotation happens around the component's local origin using the standard rotation matrix. For back-side components, mirroring occurs across the X-axis by negating Y coordinates. Finally, coordinate system conversion applies the Y-axis flip to convert from ECAD to SVG space.

The rotation implementation uses the standard 2D rotation matrix. Given an angle θ in degrees, the transform converts to radians and applies x' equals x times cos(θ) minus y times sin(θ), and y' equals x times sin(θ) plus y times cos(θ). The implementation caches trigonometric values when processing multiple points with the same rotation angle.

Mirroring for back-side components negates the Y coordinate while preserving the X coordinate, effectively reflecting across the board's X-axis. This transform is applied after rotation but before the final coordinate system conversion.

### ViewBox Calculation

The SVG viewBox is computed from the board boundary to ensure the entire board is visible. The algorithm finds the minimum and maximum X and Y coordinates across all boundary points, applies padding of five to ten percent of the board dimensions, and constructs the viewBox string in the format "minX minY width height" suitable for SVG embedding.

The padding calculation uses a configurable padding factor, typically 0.05 to 0.10. The padded boundaries are computed as xMin minus padding times width, yMin minus padding times height, xMax plus padding times width, and yMax plus padding times height.

## Phase 3: Rendering Engine

### Matplotlib Integration

The rendering engine leverages Matplotlib's object-oriented API for precise control. The setup creates a Figure and Axes, sets the aspect ratio to equal to prevent distortion, inverts the Y-axis to match SVG conventions, and configures axis visibility typically disabled for final output.

Layer rendering follows a deterministic order to ensure consistent visual hierarchy. The board outline is drawn first using a Polygon patch with neutral stroke color and no fill. Copper pours and planes are drawn next with low opacity (typically 0.2 to 0.3) to avoid obscuring traces. Traces follow as Line2D objects with width mapped from millimeters to points. Vias appear as Circle patches with filled interior and contrasting outline. Components are drawn as their outline Polygons. Reference designators are rendered as Text objects with halos for contrast. Finally, keepouts overlay everything with distinctive hatching patterns.

The drawing functions use consistent Z-order values to control stacking. Lower Z-order values appear beneath higher values. The typical assignment places board outline at Z-order 1, pours at 2, traces at 3, vias at 4, components at 5, reference designators at 6, and keepouts at 7.

### Styling System

Colors for different layers use a predefined palette that provides sufficient contrast. Top layer copper appears in red, bottom layer in blue, mid layers cycle through green and purple, ground planes use dark gray, and power planes use orange. These colors are applied consistently across traces, pours, and via annular rings on each layer.

Reference designator styling ensures readability through multiple techniques. Font size scales with board dimensions, typically between 0.5mm and 2.0mm in world coordinates. Text color uses high contrast white or black based on background. A halo effect is implemented using Matplotlib's path effects with stroke width of 2 to 3 points in a contrasting color.

Keepout rendering uses distinctive visual treatment. The fill pattern employs diagonal hatching ('///') at high density. Edge color uses a bright warning color like red or orange with thick stroke width. Fill opacity is set to 0.5 to show underlying features while clearly marking the restricted area.

### Output Format Handling

SVG output uses Matplotlib's native SVG backend through the savefig function. The configuration sets format to 'svg', DPI to 72 for web display, bbox_inches to 'tight' to trim whitespace, and pad_inches to 0.1 for minimal padding. The resulting SVG includes embedded style information and is suitable for web display or vector editing tools.

PNG output rasterizes the figure at configurable resolution. Standard DPI values include 150 for preview, 300 for print quality, and 600 for high-resolution archival. The same bbox_inches and pad_inches settings ensure consistency with SVG output.

PDF output generates vector graphics suitable for documentation. The format setting changes to 'pdf' while other parameters remain consistent. The resulting PDF embeds fonts and maintains vector fidelity for all geometric elements.

## Phase 4: Testing Strategy

### Unit Test Organization

The test suite is organized by module with comprehensive coverage of parsing, validation, geometric transforms, and rendering. Test files mirror the structure of the source code with test_parse covering the parsing pipeline, test_validate covering all validation rules, test_transform testing coordinate system conversions and transforms, and test_render verifying the rendering pipeline produces expected outputs.

Each test module uses fixtures to provide reusable test data. Board fixtures create minimal valid boards, invalid boards with specific error conditions, and complex boards with multiple components and layers. Geometric fixtures provide points, polygons, polylines, and circles with various properties. Transform fixtures create different rotation angles, mirror configurations, and coordinate system setups.

### Property-Based Testing

Hypothesis integration provides generative testing for geometric operations. The strategy generates random valid polygons and verifies invariants like the number of points remains constant through transforms and the area is preserved under translation and rotation. Random rotations are generated to verify that rotating by 360 degrees returns to the original position and that rotation matrix properties hold. Random boards are generated within valid parameter ranges to verify that the parsing and rendering pipeline never crashes.

The Hypothesis strategies use composite strategies built from primitive generators. Point strategies generate x and y values in reasonable ranges. Polygon strategies generate three to twenty points ensuring they form valid non-self-intersecting shapes. Board strategies compose components, traces, and vias within the board boundary.

### Snapshot Testing

Visual regression testing uses pytest snapshots to detect unintended changes in rendered output. The snapshot system captures SVG output as text and compares character-by-character against stored baselines. When differences are detected, the test fails and presents a diff showing exactly what changed.

The snapshot update workflow supports the natural development process. During initial development or intentional changes, developers run pytest with the update-snapshots flag to accept new outputs as baselines. During normal testing, any deviation from baselines causes test failure with detailed reporting.

Snapshot organization stores reference outputs in a snapshots directory parallel to tests. Each test module has its own snapshot subdirectory. Individual snapshots are named after the test function that generated them, enabling easy correlation between tests and their expected outputs.

### Invalid Board Testing

The test suite includes crafted invalid boards designed to trigger each of the fourteen error classes. Each invalid board JSON file resides in tests/invalid_boards/ with a name indicating the error type, such as missing_boundary.json, malformed_coordinates.json, or invalid_rotation.json.

The testing strategy validates that each invalid board produces exactly the expected error. The test loads the invalid JSON, attempts parsing and validation, asserts that validation fails, verifies the specific error code appears in the error list, and confirms the error message provides actionable information about the problem.

Edge case testing extends beyond the fourteen invalid boards to cover boundary conditions including empty boards with no components, boards with a single component, traces with only two points forming a minimal line, vias at board boundaries, and components with zero or 360-degree rotations.

## Phase 5: CLI Implementation

### Command Structure

The CLI provides three main commands through a clean interface. The render command takes an input JSON file, output path, optional layer filter, and optional format override. The validate command takes an input JSON file and produces a validation report. The list-boards command scans a directory and summarizes all board files found.

The render command accepts parameters including input as a required positional argument for the board JSON file path, output specified via -o or --output for the destination file path, layers specified via --layers as a comma-separated list defaulting to all layers, format specified via --format as one of svg, png, or pdf with automatic detection from output extension, and verbose flag via -v or --verbose for detailed progress output.

Example render invocations demonstrate the flexibility of the interface. Basic rendering uses pcb-render render board.json -o output.svg. Layer filtering uses pcb-render render board.json -o output.svg --layers TOP,BOTTOM. Format override uses pcb-render render board.json -o output.png --format png. Verbose mode uses pcb-render render board.json -o output.svg -v.

The validate command provides validation without rendering. It accepts input as a required positional argument for the board JSON file path, json flag via --json to output structured error information, and verbose flag via -v or --verbose for detailed validation messages. Example usage includes basic validation via pcb-render validate board.json and JSON output via pcb-render validate board.json --json.

### Error Handling and User Experience

Error reporting follows principles of clarity and actionability. When validation fails, the CLI prints a summary showing the total error count, lists each error with its code, severity, message, and JSON path, and exits with a non-zero status code.

The error output format presents information hierarchically. Summary statistics appear first, showing counts of errors, warnings, and info messages. Detailed errors follow with indentation, showing the severity level, error code, field path, and diagnostic message. Suggestions for fixes appear when available based on the error type.

Progress reporting provides feedback during long operations. When verbose mode is enabled, the CLI prints status messages including "Loading board file", "Validating geometry", "Computing transforms", "Rendering layers", and "Writing output". For batch operations, a progress bar shows percentage complete and estimated time remaining.

### Configuration and Customization

The CLI supports configuration files for repeated operations. The configuration file uses TOML format for readability and is placed in the project root as pcb_render.toml. Configuration sections include rendering options like default format and DPI, styling options like color schemes, validation options like error thresholds, and output options like default directory.

Command-line arguments override configuration file settings, following standard precedence rules. The order of precedence from highest to lowest is explicit command-line arguments, environment variables with PCB_RENDER_ prefix, configuration file settings, and built-in defaults.

## Phase 6: Continuous Integration

### CI Pipeline Architecture

The continuous integration system uses GitHub Actions with matrix builds across operating systems and Python versions. The matrix includes Ubuntu 22.04 with Python 3.11 and 3.12, macOS-latest with Python 3.11 and 3.12, and Windows-latest with Python 3.11 and 3.12.

Each CI job follows a standard workflow of checking out the repository, setting up Python using the actions/setup-python action, installing uv, running uv sync to install dependencies, executing the test suite with pytest, running linters with ruff, performing type checking with pyright, checking security with pip-audit, generating coverage reports, and uploading coverage artifacts.

The testing phase runs pytest with coverage measurement enabled. The configuration uses pytest -v --cov=pcb_renderer --cov-report=xml --cov-report=html --cov-report=term for comprehensive reporting. Coverage thresholds enforce minimum standards with 90% line coverage required on core modules and 80% overall project coverage.

### Quality Gates

Automated quality checks prevent problematic code from merging. The linting gate uses ruff check and ruff format --check to ensure code follows style guidelines. The type checking gate uses pyright to verify type annotations are correct and consistent. The security gate uses pip-audit to detect known vulnerabilities in dependencies. The coverage gate uses coverage report --fail-under=80 to enforce minimum thresholds.

Pull request requirements enforce quality through branch protection rules. Required checks include all matrix build jobs passing, coverage meeting thresholds, linting with zero errors, type checking with zero errors, and security scan with zero vulnerabilities. Additionally, at least one approval from a code owner is required before merging.

### Release Automation

Version tagging triggers automated release builds. When a tag matching the pattern v* is pushed, the CI system builds wheel and sdist packages using python -m build, runs the full test suite on packages, creates a GitHub Release with auto-generated notes, and attaches build artifacts to the release.

The release process follows semantic versioning with major version for incompatible API changes, minor version for backward-compatible feature additions, and patch version for backward-compatible bug fixes. The version number is stored in pyproject.toml and automatically extracted during builds.

## Phase 7: Documentation

### Code Documentation Standards

All public functions and classes include docstrings following Google style. The docstring format includes a brief one-line summary, detailed description explaining purpose and behavior, argument documentation with types and constraints, return value documentation with type and meaning, raises section documenting possible exceptions, and examples demonstrating typical usage.

Type annotations are mandatory for all function signatures. Parameters include explicit types, avoiding Any when possible. Return types are always specified, using None for procedures. Complex types use typing module constructs including Optional, Union, List, Dict, and Tuple.

### User Documentation

The README provides comprehensive guidance for users. The introduction section explains what the tool does, why it exists, and who should use it. The installation section covers system requirements, uv installation, and dependency installation. The quick start section presents basic usage examples with expected outputs. The CLI reference section documents all commands, arguments, and options. The output formats section explains SVG, PNG, and PDF characteristics. The troubleshooting section addresses common issues and solutions.

Advanced topics receive separate documentation files. The validation guide explains error codes in detail, provides examples of valid and invalid boards, and documents how to fix common problems. The rendering guide describes the layer rendering process, explains styling customization, and documents coordinate system transforms. The testing guide explains how to run tests, interpret results, and add new test cases.

### API Documentation

Generated documentation uses Sphinx with the autodoc extension to extract docstrings from code. The configuration includes the Napoleon extension for Google-style docstrings, the viewcode extension for source code links, and the intersphinx extension for cross-references to external libraries.

The documentation structure organizes content logically with a homepage providing an overview, API reference generated from docstrings, user guide with tutorials and examples, developer guide for contributors, and changelog documenting all releases.

## Implementation Timeline

### Phase Breakdown

Phase 1 Foundation requires one week and establishes core data models with Pydantic classes for all PCB elements, parsing logic with unit normalization, and basic validation framework with error reporting.

Phase 2 Transforms requires one week and implements coordinate system conversions, component placement transforms with rotation and mirroring, and viewBox calculation for SVG output.

Phase 3 Rendering requires two weeks and builds the Matplotlib rendering pipeline with layer-specific drawing functions, styling system with colors and patterns, output format handlers for SVG, PNG, and PDF, and reference designator and keepout rendering.

Phase 4 Testing requires one week and creates unit tests for all modules, property-based tests with Hypothesis, snapshot tests for visual regression, and invalid board tests for all fourteen error types.

Phase 5 CLI requires one week and implements command-line interface with argparse or Typer, error reporting and user feedback, configuration file support, and progress reporting for long operations.

Phase 6 CI requires one week and sets up GitHub Actions workflows, matrix builds across platforms and Python versions, coverage reporting and enforcement, and release automation.

Phase 7 Documentation requires one week and writes comprehensive README and user guides, generates API documentation with Sphinx, creates troubleshooting and validation guides, and produces example board files and tutorials.

The total implementation timeline spans seven weeks with phases executed sequentially, though some overlap is possible between Phases 3 and 4, and Phases 5 and 6.

### Risk Mitigation

Technical risks include complex geometric edge cases potentially causing rendering errors, which is mitigated by comprehensive property-based testing. Platform-specific issues particularly on macOS and Windows are addressed through CI matrix builds catching problems early. Performance problems with large boards are handled by profiling during development and optimizing hot paths.

Schedule risks include feature creep expanding scope beyond requirements, managed by maintaining strict focus on the fourteen error types and core rendering requirements. Dependencies on external libraries could introduce compatibility issues, minimized by using stable, well-maintained libraries with conservative version constraints. Testing complexity as the codebase grows is addressed by investing in test infrastructure early and maintaining high coverage throughout development.

## Maintenance and Future Work

### Post-Submission Maintenance

After initial submission, the system should be maintained with regular dependency updates checking for security patches monthly, bug fix releases for critical issues, and documentation updates for clarity improvements or new examples.

The issue tracking system should categorize reports as bugs for incorrect behavior, enhancements for new features, documentation for improvements to guides, and performance for optimization opportunities. Each issue receives a priority label of critical for rendering failures or security issues, high for validation errors or usability problems, medium for nice-to-have improvements, or low for minor documentation tweaks.

### Future Enhancement Opportunities

Beyond the core submission requirements, several enhancements could add value. Interactive rendering could add web-based viewer with pan and zoom, layer visibility toggles, and component highlighting on hover. Advanced validation could include design rule checking for spacing violations, electrical rule checking for connectivity, and manufacturability analysis.

Performance optimization could implement streaming rendering for large boards, parallel processing of independent layers, and caching of computed transforms. Format support could expand to include Gerber export for fabrication, ODB++ for advanced manufacturing, and JSON export of validation results.

Integration features could enable REST API for remote rendering, batch processing of multiple boards, and plugin system for custom validators. These enhancements should be prioritized based on user feedback and usage patterns after the initial submission proves the core functionality.

## Conclusion

This development plan provides a comprehensive roadmap for implementing a PCB board renderer that meets all specified requirements. The phased approach ensures core functionality is established early, with testing and quality assurance integrated throughout the process. The emphasis on validation, deterministic outputs, and comprehensive testing creates a reliable foundation that can be reviewed and verified within the required timeframe.

The architecture prioritizes simplicity and correctness over premature optimization, using well-established libraries and patterns to minimize development risk. The extensive testing strategy, including property-based tests and snapshot comparisons, provides confidence in correctness across a wide range of board configurations. The CI/CD pipeline ensures quality is maintained across platforms and Python versions, while comprehensive documentation supports both users and future maintainers.
