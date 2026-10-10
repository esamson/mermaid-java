# Local native-image proof

## Goal

Prove, on a local machine, that `mermaid-java` can be ahead-of-time compiled with GraalVM Native Image and can render four diagram types to SVG and PNG. There is no GitHub Actions job.

The reachability metadata produced for this proof lives in the library JAR, at the path Native Image reads from a dependency. This proof does not publish a consumer guide, a versioning scheme, or a second artifact. A later change can publish that metadata for other applications without moving the files.

## What is in scope

The `Mermaid.render` path, then rasterization of that SVG through `Mermaid.svgToImage` and `ImageIO` PNG encoding, for these four diagrams, in this order, with these exact source strings:

```text
flowchart   graph TD; A-->B;
sequence    sequenceDiagram\nAlice->>Bob: Hello
class       classDiagram\nclass Animal { +int age }
ER          erDiagram\nCUSTOMER ||--o{ ORDER : places
```

`\n` is a newline in the Java string. The other lines are single-line sources.

## What is out of scope

- Layout extraction
- ANTLR source editing
- Diagram types other than the four above
- `Mermaid.renderToImage(String, int)` (the exact-width overload)
- A native-image run of the JUnit suite
- Continuous integration
- Gradle native-image support. The Gradle `application` main stays `com.aresstack.mermaid.MermaidRendererMain`

## Two pieces

**Library metadata.** Files under `src/main/resources/META-INF/native-image/ph.samson/mermaid-java/` describe what `mermaid-java` itself needs. GraalJS keeps its own metadata inside the `js-community` jars.

**Smoke executable.** A Maven profile named `native` builds a native image whose entry point is the smoke harness. The profile chooses the main class and the image name. A downstream application has its own entry point, so those two settings stay in the POM.

A normal `mvn deploy` does not activate the profile. Signing and Central publishing stay on their current path.

## Smoke harness

`com.aresstack.mermaid.NativeImageSmoke` lives in the main source set so `native-image` can use it as the entry point. It ships inside the published JAR. It is a proof harness, not a public API. This proof does not add a way to exclude it from the JAR.

On startup it sets `java.awt.headless` to `true`. It takes no arguments.

Before rendering, it calls `MermaidRenderer.getInstance().isAvailable()`. If that is false, it prints one line that `mermaid.min.js` or `browser-shim.js` is missing from the image and exits 1. It does not attempt the four diagrams.

For each diagram, in the order flowchart, sequence, class, ER:

1. Call `Mermaid.render` once.
2. The SVG check passes when the result is non-null, non-blank, and contains `<svg`.
3. When the SVG check passes, call `Mermaid.svgToImage` on that string, then `ImageIO.write` of the `BufferedImage` as `png` into a byte array. This does not render the diagram a second time through GraalJS, and it does not write a file.
4. The PNG check passes when the image width and height are both greater than zero, the byte array is non-empty, and the bytes start with the PNG signature `89 50 4E 47 0D 0A 1A 0A`.

Stdout gets one line per diagram, in this form:

```text
flowchart svg=1234 png=5678 pass
sequence svg=0 png=- fail
```

`svg=` is the SVG character length, or `0` when `Mermaid.render` returned null. `png=` is the PNG byte length after a passed SVG check, or `-` when the SVG check failed or rasterization produced no image. The last field is `pass` or `fail`. A null image, or an exception thrown while encoding PNG, is `fail` for that diagram. The remaining diagrams still run.

The process exits 0 only when all four diagrams pass. Existing GraalJS tracing on stderr stays as it is. Each GraalJS evaluation keeps the existing 120-second timeout.

Missing-registration errors are `Error`s. The smoke main does not catch them. The process aborts with a stack trace. That stack trace is the signal to add one metadata entry and rebuild.

Font and graphics initialization still depend on the machine. On Linux that includes fontconfig. A failure there is an environment problem. It is not hidden with extra metadata.

## Maven profile

Profile id: `native`.

Plugin: `org.graalvm.buildtools:native-maven-plugin:1.1.14`.

- `extensions` is true
- Goal `compile-no-fork` is bound to `package`
- `imageName` is `mermaid-java-smoke`
- `mainClass` is `com.aresstack.mermaid.NativeImageSmoke`
- `fallback` is false, so a failed native build fails Maven instead of writing a JVM launcher

Maven runs on a GraalVM Community JDK 25, matching the community line of `js-community`. The plugin takes `native-image` from that JDK. The existing `js-community` dependency stays a POM dependency, so Native Image sees the separate GraalJS jars on the runtime classpath. Those jars are not shaded into one archive.

The proof command is:

```bash
mvn -Pnative -DskipTests package
./target/mermaid-java-smoke
```

`src/main/resources/mermaid/mermaid.min.js` is generated from `js-bundle/` before that package run. The binary is `target/mermaid-java-smoke`. The profile does not attach it as a Maven artifact.

The release workflow keeps calling `mvn -B clean deploy` with this profile inactive. The proof command stops at `package`, before the GPG plugin's `verify` execution. `-DskipTests` is part of the proof command so a metadata rebuild does not re-run the JVM suite.

## Reachability metadata

Written up front, because these accesses are dynamic:

`resource-config.json`:

```json
{
  "resources": {
    "includes": [
      { "pattern": "mermaid/browser-shim\\.js" },
      { "pattern": "mermaid/mermaid\\.min\\.js" }
    ]
  }
}
```

`MermaidRenderer.loadResource` loads both through `getResourceAsStream`. Patterns are matched against classpath resource names, without a leading slash.

`reflect-config.json`:

```json
[
  {
    "name": "com.aresstack.mermaid.GraalJsExecutor$JavaBridge",
    "allPublicConstructors": true,
    "allPublicMethods": true
  }
]
```

The browser shim calls `measureTextWidth`, `measureTextFull`, `computeSvgBBox`, `clearBBoxCache`, and `log` on the host object created with `allowAllAccess(true)`.

There is no `native-image.properties` in the library. Build-only settings stay in the Maven profile. `java.awt.headless` stays in the smoke main, so a later GUI consumer is not forced headless.

Batik, AWT beyond the host bridge, and the JDK PNG writer are not pre-registered. `BatikBBoxService` looks up its XML parser by class name. Batik's bridge and transcoder layers load services the same way. `ImageIO.write(..., "png", ...)` looks up a PNG writer through the service loader. When the smoke binary throws a missing-registration or class-loading error, add the smallest reflect, resource, or JNI entry that names the missing type. Do not check in a tracing-agent dump.

This metadata covers the render-and-PNG path only. Layout extraction and the ANTLR editor get no metadata in this proof.

## Testing

`mvn test` is unchanged. The native profile does not run JUnit inside the image. No Surefire test launches `target/mermaid-java-smoke`, because that binary requires a GraalVM JDK.

The AOT check is the two commands in the Maven profile section. Success is exit code 0 and four stdout lines, one each for flowchart, sequence, class, and ER, each marked pass, each with a non-zero SVG length and a non-zero PNG length.

When the Java code changes, `mvn test` is still the JVM regression check. It is a separate command from the native proof.

The README gains a short "Native image proof" note with the GraalVM Community JDK 25 requirement, the need to generate `mermaid.min.js` first, and the two commands above.

## Success criteria

- `mvn -Pnative -DskipTests package`, on GraalVM Community JDK 25, produces `target/mermaid-java-smoke`
- Running that binary exits 0
- Flowchart, sequence, class, and ER each produce a non-empty SVG and a PNG whose bytes start with the PNG signature
- `mvn -B clean deploy` without `-Pnative` is unchanged
