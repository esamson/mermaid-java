# Personal fork!

This is my personal fork for trying out ideas on
[aresstack/mermaid-java](https://github.com/aresstack/mermaid-java). Real work
happens there.

If changes here turn out to be more generally useful, we should work on merging
them back upstream. Otherwise, I don't intend to maintain this beyond my
personal needs.

Don't use this. Use that.

# mermaid-java
Renders [Mermaid](https://mermaid.js.org/) diagram code to SVG or `BufferedImage` in **pure Java** — no browser, no Node.js, no native dependencies.  
Internally uses GraalJS to execute the official Mermaid library and Apache Batik for accurate SVG text measurement and rasterisation.
## Features
- **SVG Rendering** — all 20+ Mermaid diagram types to SVG
- **Rasterisation** — SVG to `BufferedImage` at any resolution
- **Layout Extraction** — extract typed node/edge models from rendered SVGs
- **Source Editing** — ANTLR-based roundtrip editing of Mermaid source code
- **Java 8 compatible** — runs on JRE 8+
## Installation
### Maven
```xml
<dependency>
    <groupId>ph.samson</groupId>
    <artifactId>mermaid-java</artifactId>
    <version>0.2.0-beta.4</version>
</dependency>
```
### Gradle
```groovy
implementation 'ph.samson:mermaid-java:0.2.0-beta.4'
```
## Quick Start
No external files or Node.js required — the Mermaid JS engine is bundled inside the JAR.
```java
import com.aresstack.Mermaid;
// One-liner — get a ready-to-use SVG string:
String svg = Mermaid.render("graph TD; A-->B;");
// Or get a BufferedImage directly:
BufferedImage img = Mermaid.renderToImage("graph TD; A-->B;");
```
### Save as PNG
```java
BufferedImage img = Mermaid.renderToImage("graph TD; A-->B;");
ImageIO.write(img, "png", new File("diagram.png"));
```
### Control the output width (zoom / hi-DPI / thumbnails)
```java
BufferedImage hi    = Mermaid.renderToImage("graph TD; A-->B;", 3840); // 4K
BufferedImage thumb = Mermaid.renderToImage("graph TD; A-->B;", 300);  // thumbnail
```
### Re-rasterise an existing SVG at a different size
```java
String svg = Mermaid.render("graph TD; A-->B;");
BufferedImage normal = Mermaid.svgToImage(svg);       // intrinsic size
BufferedImage large  = Mermaid.svgToImage(svg, 2400);  // exact width
```
### Embed in Swing / JavaFX
```java
// Swing
JLabel label = new JLabel(new ImageIcon(Mermaid.renderToImage("graph TD; A-->B;")));
// JavaFX
ImageView view = new ImageView(SwingFXUtils.toFXImage(Mermaid.renderToImage("graph TD; A-->B;"), null));
```
## Layout Extraction API (new in 0.2.0)
Extract typed node and edge models from rendered SVG — enables diagram editors, click-to-select, and programmatic analysis.
```java
import com.aresstack.mermaid.layout.*;
String svg = Mermaid.render("graph LR\n  A-->B\n  B-->C");
RenderedDiagram diagram = DiagramLayoutExtractor.extract(svg);
// Iterate typed nodes
for (DiagramNode node : diagram.getNodes()) {
    System.out.println(node.getId() + " at (" + node.getX() + ", " + node.getY() + ")");
    if (node instanceof FlowchartNode) {
        System.out.println("  shape: " + ((FlowchartNode) node).getShape());
    }
}
// Iterate typed edges
for (DiagramEdge edge : diagram.getEdges()) {
    System.out.println(edge.getSourceId() + " -> " + edge.getTargetId());
}
// Hit-testing
DiagramNode clicked = diagram.findNodeAt(svgX, svgY);
```
### Polymorphic node types
| Class | Diagram | Properties |
|---|---|---|
| `FlowchartNode` | Flowchart | `shape` (RECTANGLE, DIAMOND, STADIUM, …) |
| `ClassNode` | Class diagram | `members` (fields + methods with visibility) |
| `ErEntityNode` | ER diagram | `attributes` (name, type, PK/FK) |
| `StateDiagramNode` | State diagram | `isStart`, `isEnd`, `isComposite` |
| `SequenceActorNode` | Sequence diagram | actor/participant |
| `SequenceFragment` | Sequence diagram | `fragmentType` (loop, alt, opt, …) |
| `MindmapItemNode` | Mindmap | `depth` |
| `RequirementItemNode` | Requirement | type, risk, verification |
### Polymorphic edge types
| Class | Diagram | Properties |
|---|---|---|
| `FlowchartEdge` | Flowchart | `lineStyle`, `headType`, `tailType` |
| `ClassRelation` | Class diagram | `relationType`, multiplicities |
| `ErRelationship` | ER diagram | `sourceCardinality`, `targetCardinality`, `identifying` |
| `SequenceMessage` | Sequence | `messageType` (SYNC, ASYNC, REPLY, …) |
| `StateTransition` | State diagram | `guard` condition |
## Source Editing API (new in 0.2.0)
ANTLR-based roundtrip editing — modify Mermaid source at exact token positions, preserving all formatting.
```java
import com.aresstack.mermaid.editor.*;
String source = "graph LR\n    A --> B\n    B --> C";
// Rename a node (all references)
String modified = SourceEditBridge.renameNode(source, "flowchart", "A", "A", "Start");
// Reverse an edge
String reversed = SourceEditBridge.reverseEdge(source, "flowchart", edge);
// Delete an edge
String deleted = SourceEditBridge.deleteEdge(source, "flowchart", edge);
// Add a new edge
String added = SourceEditBridge.addEdge(source, "flowchart", "A", "D");
// Change ER cardinality
String erModified = SourceEditBridge.changeErCardinality(source,
    "AUTHOR", "BOOK",
    ErCardinality.ONE_OR_MORE, ErCardinality.ZERO_OR_MORE,
    false, "writes");
// Reconnect edge endpoint to different node
String reconnected = SourceEditBridge.reconnectEdge(source, "flowchart", edge, "X", "Y");
```
### Supported ANTLR grammars
| Grammar | Diagram | Operations |
|---|---|---|
| `MermaidFlowchartLexer/Parser` | Flowchart | rename, edge style, reverse, delete, add |
| `MermaidErDiag` | ER diagram | cardinality, reverse, rename, delete, add |
| `MermaidSequence` | Sequence | message type, rename, reverse, delete, add |
| `MermaidStateDiag` | State diagram | rename, reverse, delete, add |
| `MermaidClassDiag` | Class diagram | rename, reverse, delete, add |
## Roundtrip Editing Support
The following table shows interactive editing support when using `SourceEditBridge` + `DiagramLayoutExtractor` together for visual diagram editors:
### Experimental roundtrip support (Test Suite 1)
| Diagram | Rename | Edge reverse | Edge delete | Edge add (D&D) | Edge reconnect (D&D) | Specialised edits |
|---|---|---|---|---|---|---|
| Flowchart (shapes) | ✅ | ✅ | ✅ | ✅ | ✅ | shape change, edge style |
| Flowchart (edges) | ✅ | ✅ | ✅ | ✅ | ✅ | line style + arrowhead |
| Flowchart (subgraphs) | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| Sequence diagram | ✅ | ✅ | ✅ | ✅ | ✅ | message type |
| ER diagram | ✅ | ✅ | ✅ | ✅ | ✅ | cardinality editing |
| State diagram | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| Class diagram | ✅ | ✅ | ✅ | ✅ | ✅ | relation type |
| Mindmap | ✅ | — | — | — | — | — |
### Not yet supported for roundtrip editing (Test Suite 2)
User Journey, Gantt, Pie, Quadrant, Git Graph, Timeline, Sankey, XY Chart, Block, Kanban, Architecture, Packet, Requirement, C4.
These diagrams are **rendered correctly** and layout extraction provides node/edge models, but ANTLR grammars and `SourceEditBridge` operations are not yet implemented.
## Rendering API Overview
| Method | Returns | Description |
|---|---|---|
| `Mermaid.render(code)` | `String` | SVG with all Batik-compatibility fixes |
| `Mermaid.renderRaw(code)` | `String` | SVG without post-processing |
| `Mermaid.renderToImage(code)` | `BufferedImage` | Rasterised image at intrinsic size, auto-cropped |
| `Mermaid.renderToImage(code, width)` | `BufferedImage` | Rasterised at exact pixel width |
| `Mermaid.svgToImage(svg)` | `BufferedImage` | Convert existing SVG string to image |
| `Mermaid.svgToImage(svg, width)` | `BufferedImage` | Convert existing SVG to image at exact width |
| `Mermaid.autoCrop(image)` | `BufferedImage` | Trim transparent/white edges |
| `Mermaid.renderDetailed(code)` | `JsExecutionResult` | SVG or error details |
## Supported Diagram Types (Rendering)
| Type | Status | Notes |
|---|---|---|
| Flowchart | ✅ | incl. subgraphs |
| Sequence | ✅ | lifeline fixes applied |
| Class | ✅ | |
| State | ✅ | |
| ER (Entity Relationship) | ✅ | label reposition fixes |
| Pie | ✅ | |
| Gantt | ✅ | |
| Journey (User Journey) | ✅ | |
| Mindmap | ✅ | multi-line box expansion |
| Git Graph | ✅ | |
| Sankey | ✅ | |
| Block | ✅ | |
| Architecture | ✅ | |
| Packet | ✅ | |
| Requirement | ✅ | label distribution fix |
| C4 (System Context) | ✅ | `href` → `xlink:href` fix |
| ZenUML | ❌ | requires extra DOM APIs |
| Radar (`radar-beta`) | ❌ | experimental |
| Treemap (`treemap-beta`) | ❌ | experimental |
## Mermaid Version
Currently bundled: **Mermaid 11.4.1** (ESM → IIFE via esbuild).
The bundle is built from `js-bundle/`:
```bash
cd js-bundle && npm install && npm run bundle
```
Output: `src/main/resources/mermaid/mermaid.min.js`
## Architecture
| Class | Purpose |
|---|---|
| `Mermaid` | **Public API** — `render()` / `renderToImage()` / `svgToImage()` |
| `MermaidRenderer` | Singleton — GraalJS context management, Mermaid initialisation |
| `MermaidSvgFixup` | Post-processing: Batik-compatibility DOM fixes |
| `DiagramLayoutExtractor` | Extract typed node/edge models from SVG DOM |
| `RenderedDiagram` | Immutable container for extracted nodes, edges, viewBox |
| `MermaidSourceEditor` | ANTLR-based source editor with `TokenStreamRewriter` |
| `SourceEditBridge` | High-level editing API bridging SVG models ↔ ANTLR editor |
| `GraalJsExecutor` | Internal GraalJS polyglot-context wrapper |
| `BatikBBoxService` | Exact SVG BBox computation via Batik GVT tree |
| `JsExecutionResult` | Immutable result object (success / failure) |
## Building
```bash
./gradlew build
```
Requires Java 8 JDK.
## License
This project is licensed under the [MIT License](LICENSE).
### Bundled third-party software
This JAR includes a bundled copy of [Mermaid.js](https://github.com/mermaid-js/mermaid) (v11.4.1), which is licensed separately under the **MIT License**, Copyright © Knut Sveidqvist. See the [Mermaid license](https://github.com/mermaid-js/mermaid/blob/develop/LICENSE) for details.
