# Local native-image proof Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prove on a local GraalVM Community JDK 25 that `mermaid-java` ahead-of-time compiles and that the resulting binary renders flowchart, sequence, class, and ER diagrams to SVG and PNG.

**Architecture:** Reachability metadata for the two Mermaid scripts and `GraalJsExecutor.JavaBridge` ships inside the library JAR, where Native Image already looks. A Maven profile named `native` builds a smoke executable whose main is `com.aresstack.mermaid.NativeImageSmoke`. The profile is off for a normal deploy. The smoke main is the only place that forces `java.awt.headless`.

**Tech Stack:** Java 25, Maven, JUnit 5, Gson (test scope, already in `pom.xml`), GraalVM Native Image via `org.graalvm.buildtools:native-maven-plugin:1.1.14`, existing `com.aresstack.Mermaid` render and rasterize API.

**Spec:** `docs/superpowers/specs/2026-10-10-native-image-build-design.md`

This spec is one proof, not two subsystems. The metadata is verified by the smoke binary, and the binary cannot pass without the metadata. One plan.

## Global Constraints

- Proof host is a GraalVM Community JDK 25. `native-image` comes from that JDK.
- Plugin is `org.graalvm.buildtools:native-maven-plugin:1.1.14`.
- Profile id is `native`. It is inactive for `mvn -B clean deploy`.
- `extensions` is true. Goal `compile-no-fork` is bound to `package`.
- `imageName` is `mermaid-java-smoke`. `mainClass` is `com.aresstack.mermaid.NativeImageSmoke`. `fallback` is false.
- Proof command is `mvn -Pnative -DskipTests package` then `./target/mermaid-java-smoke`. The binary is `target/mermaid-java-smoke` and is not a Maven artifact.
- `src/main/resources/mermaid/mermaid.min.js` is generated from `js-bundle/` before that package run. Do not commit that generated file.
- Library metadata directory is `src/main/resources/META-INF/native-image/ph.samson/mermaid-java/`. There is no `native-image.properties` in the library.
- `js-community` stays a POM dependency. Do not shade the GraalJS jars into one archive.
- Gradle `application` main stays `com.aresstack.mermaid.MermaidRendererMain`. Do not add Gradle native-image support.
- Do not add a GitHub Actions job. Do not edit `.github/workflows/release.yml`.
- Do not run JUnit inside the native image. No Surefire test launches `target/mermaid-java-smoke`.
- Do not call `Mermaid.renderToImage(String, int)`. Rasterize with `Mermaid.svgToImage(String)` and `ImageIO.write`.
- Diagram types other than flowchart, sequence, class, and ER get no metadata and no smoke coverage. Layout extraction and the ANTLR editor get no metadata.
- Each GraalJS evaluation keeps the existing 120-second timeout. Do not edit `GraalJsExecutor` tracing on stderr.
- `java.awt.headless=true` is set in the smoke main only.
- Missing-registration `Error`s are not caught. Fontconfig failures are environment failures and are not hidden with metadata.
- Do not check in a tracing-agent dump. Do not enable the native-image tracing agent. Do not bind `generateResourceConfig`.
- Batik, AWT beyond the host bridge, and the JDK PNG writer are not pre-registered.

## File structure

- Create: `src/main/resources/META-INF/native-image/ph.samson/mermaid-java/resource-config.json` — classpath patterns for the two scripts `MermaidRenderer.loadResource` reads.
- Create: `src/main/resources/META-INF/native-image/ph.samson/mermaid-java/reflect-config.json` — public constructors and methods of `GraalJsExecutor.JavaBridge`.
- Create: `src/test/java/com/aresstack/mermaid/NativeImageMetadataTest.java` — asserts those two files on the classpath, and that `native-image.properties` is absent.
- Create: `src/main/java/com/aresstack/mermaid/NativeImageSmoke.java` — proof entry point. Ships in the published JAR. Not a supported API. No JAR exclusion.
- Create: `src/test/java/com/aresstack/mermaid/NativeImageSmokeTest.java` — JVM tests of the SVG check, PNG check, stdout line, and the four diagram sources. Does not call `main` or `run`.
- Modify: `pom.xml` — add the `native` profile only. Leave the existing `<build><plugins>` list as it is.
- Modify: `README.md` — add the "Native image proof" note.

Do not modify `build.gradle`, `.github/workflows/release.yml`, `Mermaid.java`, `MermaidRenderer.java`, `MermaidRendererMain.java`, or `GraalJsExecutor.java`.

---

### Task 1: Library reachability metadata

**Files:**
- Create: `src/test/java/com/aresstack/mermaid/NativeImageMetadataTest.java`
- Create: `src/main/resources/META-INF/native-image/ph.samson/mermaid-java/resource-config.json`
- Create: `src/main/resources/META-INF/native-image/ph.samson/mermaid-java/reflect-config.json`

**Interfaces:**
- Consumes: nothing from earlier tasks. `MermaidRenderer.loadResource` already loads `/mermaid/browser-shim.js` and `/mermaid/mermaid.min.js` via `Class.getResourceAsStream`. Native Image matches patterns against classpath names with no leading slash.
- Produces: classpath resources `META-INF/native-image/ph.samson/mermaid-java/resource-config.json` and `META-INF/native-image/ph.samson/mermaid-java/reflect-config.json`, with the JSON shapes below. Later tasks do not add Batik, AWT, or ImageIO entries unless Task 3's smoke binary names a missing type.

- [ ] **Step 1: Write the failing test**

Create `src/test/java/com/aresstack/mermaid/NativeImageMetadataTest.java`:

```java
package com.aresstack.mermaid;

import com.google.gson.JsonArray;
import com.google.gson.JsonElement;
import com.google.gson.JsonObject;
import com.google.gson.JsonParser;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.HashSet;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertNull;
import static org.junit.jupiter.api.Assertions.assertTrue;

class NativeImageMetadataTest {

    private static final String METADATA_DIR =
            "META-INF/native-image/ph.samson/mermaid-java/";

    @Test
    @DisplayName("resource-config includes the two Mermaid scripts without a leading slash")
    void resourceConfigIncludesMermaidScripts() throws IOException {
        JsonObject root = JsonParser.parseString(read("resource-config.json")).getAsJsonObject();
        JsonArray includes = root.getAsJsonObject("resources").getAsJsonArray("includes");
        Set<String> patterns = new HashSet<>();
        for (JsonElement element : includes) {
            String pattern = element.getAsJsonObject().get("pattern").getAsString();
            assertFalse(pattern.startsWith("/"), pattern);
            patterns.add(pattern);
        }
        assertTrue(patterns.contains("mermaid/browser-shim\\.js"));
        assertTrue(patterns.contains("mermaid/mermaid\\.min\\.js"));
    }

    @Test
    @DisplayName("reflect-config exposes JavaBridge public constructors and methods")
    void reflectConfigRegistersJavaBridge() throws IOException {
        JsonArray entries = JsonParser.parseString(read("reflect-config.json")).getAsJsonArray();
        JsonObject bridge = null;
        for (JsonElement element : entries) {
            JsonObject entry = element.getAsJsonObject();
            if ("com.aresstack.mermaid.GraalJsExecutor$JavaBridge"
                    .equals(entry.get("name").getAsString())) {
                bridge = entry;
            }
        }
        assertNotNull(bridge);
        assertTrue(bridge.get("allPublicConstructors").getAsBoolean());
        assertTrue(bridge.get("allPublicMethods").getAsBoolean());
    }

    @Test
    @DisplayName("build-only settings stay out of the library metadata")
    void noNativeImageProperties() throws IOException {
        assertNotNull(read("reflect-config.json"));
        assertNull(NativeImageMetadataTest.class.getClassLoader()
                .getResource(METADATA_DIR + "native-image.properties"));
    }

    private static String read(String fileName) throws IOException {
        String path = METADATA_DIR + fileName;
        try (InputStream in = NativeImageMetadataTest.class.getClassLoader().getResourceAsStream(path)) {
            assertNotNull(in, path);
            return new String(in.readAllBytes(), StandardCharsets.UTF_8);
        }
    }
}
```

The Java string `"mermaid/browser-shim\\.js"` is the pattern text `mermaid/browser-shim\.js`. That is the value Gson returns when the file contains the JSON string `"mermaid/browser-shim\\.js"`. Keep both spellings. Do not "fix" one of them to a single backslash in the file, or to four backslashes in the test.

- [ ] **Step 2: Run the test to verify it fails**

Run: `mvn -Dtest=NativeImageMetadataTest test`

Expected: FAIL. `resourceConfigIncludesMermaidScripts` reports that `META-INF/native-image/ph.samson/mermaid-java/resource-config.json` is null.

- [ ] **Step 3: Write the two metadata files**

Create `src/main/resources/META-INF/native-image/ph.samson/mermaid-java/resource-config.json`:

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

Create `src/main/resources/META-INF/native-image/ph.samson/mermaid-java/reflect-config.json`:

```json
[
  {
    "name": "com.aresstack.mermaid.GraalJsExecutor$JavaBridge",
    "allPublicConstructors": true,
    "allPublicMethods": true
  }
]
```

Do not create `native-image.properties`. Do not add Batik, AWT, ImageIO, or service-loader entries in this task. `JavaBridge` is the host object the browser shim calls (`measureTextWidth`, `measureTextFull`, `computeSvgBBox`, `clearBBoxCache`, `log`) under `allowAllAccess(true)`.

- [ ] **Step 4: Run the test to verify it passes**

Run: `mvn -Dtest=NativeImageMetadataTest test`

Expected: BUILD SUCCESS. Tests run: 3, Failures: 0, Errors: 0.

- [ ] **Step 5: Commit**

```bash
git add \
  src/test/java/com/aresstack/mermaid/NativeImageMetadataTest.java \
  src/main/resources/META-INF/native-image/ph.samson/mermaid-java/resource-config.json \
  src/main/resources/META-INF/native-image/ph.samson/mermaid-java/reflect-config.json
git commit -m "$(cat <<'EOF'
Register the Mermaid scripts and host bridge for native-image.

EOF
)"
```

---

### Task 2: Smoke harness

**Files:**
- Create: `src/test/java/com/aresstack/mermaid/NativeImageSmokeTest.java`
- Create: `src/main/java/com/aresstack/mermaid/NativeImageSmoke.java`

**Interfaces:**
- Consumes:
  - `com.aresstack.Mermaid.render(String diagramCode) -> String` (null when rendering fails)
  - `com.aresstack.Mermaid.svgToImage(String svg) -> java.awt.image.BufferedImage` (null on failure; does not re-enter GraalJS)
  - `com.aresstack.mermaid.MermaidRenderer.getInstance() -> MermaidRenderer`
  - `com.aresstack.mermaid.MermaidRenderer.isAvailable() -> boolean`
- Produces, all on `com.aresstack.mermaid.NativeImageSmoke`:
  - `public static void main(String[] args)` — sets `java.awt.headless` to `true`, then `System.exit(run())`. Ignores `args`.
  - `static int run()` — returns 0 only when all four diagrams pass. Returns 1 when the bundle is missing or any diagram fails. Prints one stdout line per diagram. Does not catch `Error`.
  - `static final String MISSING_BUNDLE_LINE` — `"mermaid.min.js or browser-shim.js is missing from the image"`
  - `static final String[][] DIAGRAMS` — four `{name, source}` rows in the order below
  - `static boolean svgPasses(String svg)`
  - `static int svgLength(String svg)` — `0` when `svg` is null, otherwise `svg.length()`
  - `static boolean pngPasses(java.awt.image.BufferedImage image, byte[] png)`
  - `static String formatLine(String name, int svgLength, String pngField, boolean pass)`

`DIAGRAMS` rows, in order:

| name | source |
|---|---|
| `flowchart` | `graph TD; A-->B;` |
| `sequence` | `sequenceDiagram\nAlice->>Bob: Hello` (`\n` is a newline) |
| `class` | `classDiagram\nclass Animal { +int age }` |
| `ER` | `erDiagram\nCUSTOMER ||--o{ ORDER : places` |

Stdout line shape: `flowchart svg=1234 png=5678 pass` and `sequence svg=0 png=- fail`.

`pngField` is `-` when the SVG check failed, rasterization returned no image, or PNG encoding threw. When encoding returns a byte array, `pngField` is that array's length even if the PNG check fails.

Do not call `run()` or `main` from a test. `run()` renders through GraalJS when the bundle is present, and `mermaid.min.js` is not checked in, so the result depends on a generated file. The native binary in Task 3 is the render proof.

- [ ] **Step 1: Write the failing test**

Create `src/test/java/com/aresstack/mermaid/NativeImageSmokeTest.java`:

```java
package com.aresstack.mermaid;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.ByteArrayOutputStream;
import java.io.IOException;

import static org.junit.jupiter.api.Assertions.assertArrayEquals;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;

class NativeImageSmokeTest {

    @Test
    @DisplayName("the four proof diagrams stay in spec order with the spec sources")
    void diagramSourcesMatchTheProof() {
        assertEquals(4, NativeImageSmoke.DIAGRAMS.length);
        assertArrayEquals(
                new String[]{"flowchart", "graph TD; A-->B;"},
                NativeImageSmoke.DIAGRAMS[0]);
        assertArrayEquals(
                new String[]{"sequence", "sequenceDiagram\nAlice->>Bob: Hello"},
                NativeImageSmoke.DIAGRAMS[1]);
        assertArrayEquals(
                new String[]{"class", "classDiagram\nclass Animal { +int age }"},
                NativeImageSmoke.DIAGRAMS[2]);
        assertArrayEquals(
                new String[]{"ER", "erDiagram\nCUSTOMER ||--o{ ORDER : places"},
                NativeImageSmoke.DIAGRAMS[3]);
        assertEquals(
                "mermaid.min.js or browser-shim.js is missing from the image",
                NativeImageSmoke.MISSING_BUNDLE_LINE);
    }

    @Test
    @DisplayName("SVG check requires non-blank markup that contains <svg")
    void svgCheck() {
        assertFalse(NativeImageSmoke.svgPasses(null));
        assertFalse(NativeImageSmoke.svgPasses(""));
        assertFalse(NativeImageSmoke.svgPasses("   "));
        assertFalse(NativeImageSmoke.svgPasses("not a diagram"));
        assertTrue(NativeImageSmoke.svgPasses("<svg></svg>"));
        assertEquals(0, NativeImageSmoke.svgLength(null));
        assertEquals(4, NativeImageSmoke.svgLength("<svg"));
    }

    @Test
    @DisplayName("PNG check requires a positive image and the PNG signature")
    void pngCheck() throws IOException {
        BufferedImage image = new BufferedImage(2, 3, BufferedImage.TYPE_INT_ARGB);
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        assertTrue(ImageIO.write(image, "png", out));
        byte[] png = out.toByteArray();

        assertTrue(NativeImageSmoke.pngPasses(image, png));
        assertFalse(NativeImageSmoke.pngPasses(image, new byte[0]));
        assertFalse(NativeImageSmoke.pngPasses(image, new byte[]{1, 2, 3, 4, 5, 6, 7, 8}));
        assertFalse(NativeImageSmoke.pngPasses(null, png));
    }

    @Test
    @DisplayName("stdout uses svg length, png length or a dash, then pass or fail")
    void reportLine() {
        assertEquals(
                "flowchart svg=1234 png=5678 pass",
                NativeImageSmoke.formatLine("flowchart", 1234, "5678", true));
        assertEquals(
                "sequence svg=0 png=- fail",
                NativeImageSmoke.formatLine("sequence", 0, "-", false));
        assertEquals(
                "ER svg=40 png=0 fail",
                NativeImageSmoke.formatLine("ER", 40, "0", false));
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `mvn -Dtest=NativeImageSmokeTest test`

Expected: FAIL at compilation with `cannot find symbol` for `NativeImageSmoke`.

- [ ] **Step 3: Write the smoke harness**

Create `src/main/java/com/aresstack/mermaid/NativeImageSmoke.java`:

```java
package com.aresstack.mermaid;

import com.aresstack.Mermaid;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.ByteArrayOutputStream;
import java.io.IOException;

/**
 * Local native-image proof entry point. Ships in the published JAR.
 * This is a proof harness, not a supported API.
 */
public final class NativeImageSmoke {

    static final String MISSING_BUNDLE_LINE =
            "mermaid.min.js or browser-shim.js is missing from the image";

    static final String[][] DIAGRAMS = {
            {"flowchart", "graph TD; A-->B;"},
            {"sequence", "sequenceDiagram\nAlice->>Bob: Hello"},
            {"class", "classDiagram\nclass Animal { +int age }"},
            {"ER", "erDiagram\nCUSTOMER ||--o{ ORDER : places"}
    };

    private static final byte[] PNG_SIGNATURE = {
            (byte) 0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A
    };

    private NativeImageSmoke() {
    }

    public static void main(String[] args) {
        System.setProperty("java.awt.headless", "true");
        System.exit(run());
    }

    static int run() {
        if (!MermaidRenderer.getInstance().isAvailable()) {
            System.out.println(MISSING_BUNDLE_LINE);
            return 1;
        }
        boolean allPassed = true;
        for (String[] diagram : DIAGRAMS) {
            if (!renderOne(diagram[0], diagram[1])) {
                allPassed = false;
            }
        }
        return allPassed ? 0 : 1;
    }

    private static boolean renderOne(String name, String source) {
        String svg = Mermaid.render(source);
        int length = svgLength(svg);
        if (!svgPasses(svg)) {
            System.out.println(formatLine(name, length, "-", false));
            return false;
        }
        BufferedImage image = Mermaid.svgToImage(svg);
        if (image == null) {
            System.out.println(formatLine(name, length, "-", false));
            return false;
        }
        byte[] png;
        try {
            png = encodePng(image);
        } catch (Exception encodingFailure) {
            System.out.println(formatLine(name, length, "-", false));
            return false;
        }
        boolean pass = pngPasses(image, png);
        System.out.println(formatLine(name, length, Integer.toString(png.length), pass));
        return pass;
    }

    static boolean svgPasses(String svg) {
        return svg != null && !svg.isBlank() && svg.contains("<svg");
    }

    static int svgLength(String svg) {
        return svg == null ? 0 : svg.length();
    }

    static boolean pngPasses(BufferedImage image, byte[] png) {
        if (image == null || png == null || png.length < PNG_SIGNATURE.length) {
            return false;
        }
        if (image.getWidth() <= 0 || image.getHeight() <= 0) {
            return false;
        }
        for (int i = 0; i < PNG_SIGNATURE.length; i++) {
            if (png[i] != PNG_SIGNATURE[i]) {
                return false;
            }
        }
        return true;
    }

    static String formatLine(String name, int svgLength, String pngField, boolean pass) {
        return name + " svg=" + svgLength + " png=" + pngField + " " + (pass ? "pass" : "fail");
    }

    private static byte[] encodePng(BufferedImage image) throws IOException {
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        ImageIO.write(image, "png", out);
        return out.toByteArray();
    }
}
```

`catch (Exception encodingFailure)` does not catch `Error`. A missing-registration `Error` from `Mermaid.render` or `Mermaid.svgToImage` still aborts the process with a stack trace. A failure of one diagram does not skip the remaining diagrams. `renderOne` calls `Mermaid.render` once and then `Mermaid.svgToImage` on that string. It does not call `Mermaid.renderToImage` and it does not write a file.

- [ ] **Step 4: Run the test to verify it passes**

Run: `mvn -Dtest=NativeImageSmokeTest,NativeImageMetadataTest test`

Expected: BUILD SUCCESS. Tests run: 7, Failures: 0, Errors: 0.

- [ ] **Step 5: Commit**

```bash
git add \
  src/test/java/com/aresstack/mermaid/NativeImageSmokeTest.java \
  src/main/java/com/aresstack/mermaid/NativeImageSmoke.java
git commit -m "$(cat <<'EOF'
Add a native-image smoke harness for four diagram types.

EOF
)"
```

---

### Task 3: Maven native profile, README note, and local proof

**Files:**
- Modify: `pom.xml` (insert a `<profiles>` block after `</build>`, before `</project>`)
- Modify: `README.md` (insert a section after the `## Building` section, before `## License`)
- Modify only if Step 6 names a missing type: `src/main/resources/META-INF/native-image/ph.samson/mermaid-java/reflect-config.json`, `resource-config.json`, or a new `jni-config.json` in that same directory

**Interfaces:**
- Consumes: `com.aresstack.mermaid.NativeImageSmoke.main(String[])` and the metadata resources from Task 1.
- Produces: inactive-by-default profile `native`. When activated, `package` writes `target/mermaid-java-smoke`. README section `## Native image proof`.

- [ ] **Step 1: Add the native profile**

`pom.xml` currently ends with `</build>` then `</project>`. There is no `<profiles>` block. Insert this between them. Do not add the plugin to the top-level `<build><plugins>` list. Do not change the `js-community` dependency (`<type>pom</type>` stays). Do not add `<skipTests>`, shading, `generateResourceConfig`, or an agent configuration.

```xml
    <profiles>
        <profile>
            <id>native</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.graalvm.buildtools</groupId>
                        <artifactId>native-maven-plugin</artifactId>
                        <version>1.1.14</version>
                        <extensions>true</extensions>
                        <executions>
                            <execution>
                                <id>build-native</id>
                                <goals>
                                    <goal>compile-no-fork</goal>
                                </goals>
                                <phase>package</phase>
                            </execution>
                        </executions>
                        <configuration>
                            <imageName>mermaid-java-smoke</imageName>
                            <mainClass>com.aresstack.mermaid.NativeImageSmoke</mainClass>
                            <fallback>false</fallback>
                        </configuration>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
```

`fallback` false makes a failed native build fail Maven instead of writing a JVM launcher. `compile-no-fork` uses the Maven JVM, so that JVM must be the GraalVM JDK in Step 4. The profile does not attach `target/mermaid-java-smoke` as an artifact. Binding to `package` stops before the GPG plugin's `verify` execution.

- [ ] **Step 2: Confirm the default deploy path does not activate the profile**

Run: `mvn help:active-profiles`

Expected: the `native` profile is not listed. `ph.samson:mermaid-java:jar:0.3.0-beta.2` has no requirement to activate `native`.

Run: `git diff -- .github/workflows/release.yml build.gradle`

Expected: empty diff. The release workflow still runs `mvn -B clean deploy` with no `-Pnative`.

- [ ] **Step 3: Document the proof in the README**

Insert this section after the `## Building` block (after the line `Requires Java 17 JDK or newer.`) and before `## License`. Leave the existing Building section as it is.

````markdown
## Native image proof
Local proof that this library can be ahead-of-time compiled with GraalVM Native Image. There is no GitHub Actions job for it.

Requires GraalVM Community JDK 25. `native-image` comes from that JDK.

Generate `src/main/resources/mermaid/mermaid.min.js` first:

```bash
cd js-bundle && npm install && npm run bundle
```

Then:

```bash
mvn -Pnative -DskipTests package
./target/mermaid-java-smoke
```

The `native` profile stays inactive for `mvn -B clean deploy`.
````

- [ ] **Step 4: Confirm the JDK and generate the Mermaid bundle**

Run: `java -version` and `native-image --version`

Expected: both identify a GraalVM CE build of Java 25 (the runtime line contains `GraalVM CE` and `25`). If either command is missing, or the JDK is not GraalVM CE 25, stop. Do not point Maven at a different `native-image` binary. Do not continue on Temurin.

From the repository root:

```bash
cd js-bundle && npm install && npm run bundle
test "$(wc -c < ../src/main/resources/mermaid/mermaid.min.js)" -gt 1000
```

Expected: `src/main/resources/mermaid/mermaid.min.js` is longer than 1000 bytes. `MermaidRenderer.isAvailable()` treats a shorter bundle as missing.

Do not `git add` `src/main/resources/mermaid/mermaid.min.js` or `js-bundle/node_modules/`.

- [ ] **Step 5: Build the smoke image**

Allow at least 20 minutes. The image build is quiet for long stretches and uses a few gigabytes of RAM.

Run: `mvn -Pnative -DskipTests package`

Expected: `BUILD SUCCESS`. The log does not run `maven-gpg-plugin`. `target/mermaid-java-smoke` exists and is executable. `target/mermaid-java-smoke.jar` does not exist.

```bash
test -x target/mermaid-java-smoke
test ! -e target/mermaid-java-smoke.jar
```

If Maven reports that the native-image executable was not found, stop and return to Step 4. If Maven reports the command line is too long, add `<useArgFile>true</useArgFile>` inside this profile's `<configuration>` and rerun this step. Do not shade dependencies to shorten the classpath.

- [ ] **Step 6: Add the smallest metadata entry the binary names, then rebuild**

Run: `./target/mermaid-java-smoke`

A stack trace whose type is `MissingReflectionRegistrationError`, `MissingResourceRegistrationError`, or `MissingJNIRegistrationError` is the signal to add one entry and rebuild. Copy the class, member, and parameter types from that message. Add nothing else. Then rerun Step 5 and this step.

Do not commit these sample names. Replace every name with the text from the exception.

Reflection (`reflect-config.json`, append one object to the array). Use `methods` when the error names a method, `fields` when it names a field. Include `allPublicConstructors` only when the error names a constructor and does not give a parameter list you can copy. The sample below is the shape; `com.example.Missing` and `load` are stand-ins:

```json
{
  "name": "com.example.Missing",
  "methods": [
    { "name": "load", "parameterTypes": ["java.lang.String"] }
  ]
}
```

Resource (`resource-config.json`, append one object to `resources.includes`). The pattern is a regex of the classpath name from the error, with no leading slash. A literal dot in a filename is written `\\.` in the JSON file. Shape:

```json
{ "pattern": "META-INF/services/com\\.example\\.Service" }
```

JNI (create `jni-config.json` in the same directory only when the error is a JNI registration error). Shape:

```json
[
  {
    "name": "com.example.Missing",
    "methods": [
      { "name": "load", "parameterTypes": [] }
    ]
  }
]
```

After every metadata edit, run `mvn -Dtest=NativeImageMetadataTest test` and keep those three tests green. The new entry must still leave `JavaBridge` registered and must still leave `native-image.properties` absent.

Stop, and do not add metadata, when any of these is true:

- The message mentions fontconfig or `libfontconfig`. That is an environment failure.
- The same missing type is still reported after the entry that names it was added and the image was rebuilt.
- The process prints `mermaid.min.js or browser-shim.js is missing from the image` and exits 1. Return to Step 4. The scripts are not in the image.

A diagram line of `fail` with no registration `Error` is a different case. Read stderr. If stderr names a class or resource that failed to load, add the one entry that names that type and rebuild. If the SVG rendered and the PNG check failed for some other reason, fix the harness only when it disagrees with Task 2. Do not add metadata for a bad diagram.

Do not copy `target/native/agent-output`. Do not run the tracing agent. Do not register a type that neither the stack trace nor that stderr load failure named.

- [ ] **Step 7: Confirm the four diagrams pass**

GraalJS tracing stays on stderr. Capture stdout only:

```bash
./target/mermaid-java-smoke > target/native-image-smoke.out
status=$?
cat target/native-image-smoke.out
test "$status" -eq 0
grep -E '^flowchart svg=[1-9][0-9]* png=[1-9][0-9]* pass$' target/native-image-smoke.out
grep -E '^sequence svg=[1-9][0-9]* png=[1-9][0-9]* pass$' target/native-image-smoke.out
grep -E '^class svg=[1-9][0-9]* png=[1-9][0-9]* pass$' target/native-image-smoke.out
grep -E '^ER svg=[1-9][0-9]* png=[1-9][0-9]* pass$' target/native-image-smoke.out
```

Expected: exit 0, and those four lines. `svg=` and `png=` are non-zero. `target/native-image-smoke.out` is under `target/` and is not committed.

This command is separate from the JVM suite. Run the JVM suite as well:

`mvn test`

Expected: BUILD SUCCESS. That command does not build or launch `target/mermaid-java-smoke`.

- [ ] **Step 8: Commit**

Stage the profile, the README, and any metadata file Step 6 changed. Do not stage `src/main/resources/mermaid/mermaid.min.js`, `js-bundle/node_modules/`, or anything under `target/`.

```bash
git add pom.xml README.md
git add src/main/resources/META-INF/native-image/ph.samson/mermaid-java/
git commit -m "$(cat <<'EOF'
Build the native-image smoke proof from a Maven profile.

EOF
)"
```

`git add` of the metadata directory is a no-op when Step 6 did not change it. That is fine.
