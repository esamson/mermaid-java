# Changelog

All notable changes to **mermaid-java** will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/), and this project adheres to [Semantic Versioning](https://semver.org/).

---

## [0.3.0-beta.1] — 2026-09-03

### Changed

- **GraalJS upgraded 21.2.0 → 25.3.4.1** — migrated from the legacy artifacts (`org.graalvm.sdk:graal-sdk`, `org.graalvm.js:js`) to the current Maven coordinates: `org.graalvm.polyglot:polyglot` + `org.graalvm.polyglot:js-community` (the open-source-licensed GraalJS distribution)
- **BREAKING: minimum Java version raised from 8 to 17** — GraalJS 25.x requires Java 17+; `maven.compiler.source`/`target` and Gradle `sourceCompatibility`/`targetCompatibility` are now 17
- **Test JVMs disable Truffle-internal assertions** — Surefire and Gradle enable `-ea` by default, which trips a Truffle 25.x internal assertion (`DynamicObjectSupport`: "before != after") on JS property deletes inside dagre/graphlib; tests now run with `-da:com.oracle.truffle...` (matching production JVMs, which run without assertions) and `--enable-native-access=ALL-UNNAMED`

## [0.2.0-beta.2] — 2026-03-25

### Fixed

- **Intl polyfill for GraalJS** — Mermaid 11+ uses `Intl.Segmenter` in `SplitTextToChars()` for grapheme-cluster text splitting. GraalJS does not provide the `Intl` API by default, causing `ReferenceError: Intl is not defined` at render time. Added a minimal polyfill to `browser-shim.js`:
  - `Intl.Segmenter` — splits by code points with surrogate-pair handling
  - `Intl.NumberFormat` — formats numbers via `String()`
  - `Intl.DateTimeFormat` — formats dates via `toISOString()`
  - `Intl.Collator` — compares strings via `<` / `>` operators
- **ANTLR grammar path** — moved grammars to Maven-convention path (`src/main/antlr4/<package>/`) so the ANTLR4 Maven plugin places generated Java files in the correct package directory
- **Maven build** — added `antlr4-maven-plugin` 4.9.3 + `antlr4-runtime` 4.9.3 to `pom.xml` for CI/CD pipeline compatibility

## [0.2.0-beta.1] — 2026-03-24

### Added

- **Layout Extraction** — `DiagramLayoutExtractor` extracts typed node/edge models from rendered SVG
- **`RenderedDiagram`** — immutable container with hit-testing (`findNodeAt`, `findNodeById`, `findEdgesFor`)
- **Polymorphic node types**: `FlowchartNode`, `ClassNode`, `ErEntityNode`, `StateDiagramNode`, `SequenceActorNode`, `SequenceFragment`, `MindmapItemNode`, `RequirementItemNode`
- **Polymorphic edge types**: `FlowchartEdge`, `ClassRelation`, `ErRelationship`, `SequenceMessage`, `StateTransition`
- **`MermaidSourceEditor`** — ANTLR-based roundtrip editing with `TokenStreamRewriter`
- **`SourceEditBridge`** — high-level API for: `renameNode`, `reverseEdge`, `deleteEdge`, `addEdge`, `reconnectEdge`, `changeErCardinality`
- **6 ANTLR grammars** — flowchart, ER, sequence, state, class (all Java 8–compatible with ANTLR 4.9.3)
- **Edge endpoint drag-and-drop** — direct grab-and-drag reconnection using SVG path endpoints
- **ER cardinality editing** — with label preservation from ANTLR model
- **`Mermaid.renderWithLayout(code)`** — new façade method returning `RenderedDiagram` (SVG + nodes + edges)

### Fixed

- **ER SVG extraction** — use actual path endpoints instead of bounding-box midpoints
- **ER relationship labels** — always preserve label (Mermaid ER requires `: label`)
- **`reconnectEdge`** — use `findEdgeRobust` + strip SVG prefixes from node IDs

## [0.1.0-beta.1] — 2026-03-23

### Added

- **SVG Rendering** — render all 20+ Mermaid diagram types to SVG using GraalJS + bundled Mermaid 11.4.1 (ESM → IIFE via esbuild)
- **Rasterisation** — SVG to `BufferedImage` at any resolution via Apache Batik
- **Public API**: `Mermaid.render()`, `Mermaid.renderRaw()`, `Mermaid.renderToImage()`, `Mermaid.svgToImage()`, `Mermaid.autoCrop()`
- **`MermaidSvgFixup`** — Batik-compatibility DOM post-processing (lifeline fixes, label repositioning, `href` → `xlink:href`, multi-line box expansion)
- **`BatikBBoxService`** — exact SVG BBox computation via Batik GVT tree
- **`GraalJsExecutor`** — internal GraalJS polyglot-context wrapper
- **`browser-shim.js`** — DOM/CSS polyfills for headless GraalJS (document, window, navigator, getComputedStyle, etc.)
- **esbuild bundler** — `js-bundle/` for building Mermaid IIFE bundle from npm
- Java 8 compatible

---

[0.2.0-beta.2]: https://github.com/aresstack/mermaid-java/compare/v0.2.0-beta.1...v0.2.0-beta.2
[0.2.0-beta.1]: https://github.com/aresstack/mermaid-java/compare/v0.1.0-beta.1...v0.2.0-beta.1
[0.1.0-beta.1]: https://github.com/aresstack/mermaid-java/releases/tag/v0.1.0-beta.1
