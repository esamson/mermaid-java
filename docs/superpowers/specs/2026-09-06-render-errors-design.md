# Spec: Surface mermaid render failures as exceptions

**Issue:** [aresstack/mermaid-java#3](https://github.com/aresstack/mermaid-java/issues/3)
**Status:** Draft — awaiting review before implementation plan
**Date:** 2026-09-06

## Objective

Callers of mermaid-java must be able to **handle and present mermaid rendering failures** to their users. Today those failures are written to `System.err` and the public render methods return `null`, so the error details never reach application code.

Success looks like: an editor or CLI that calls `Mermaid.render(userSource)` can `catch` a typed exception, read a message suitable for display, and show it — without scraping stderr or treating `null` as a silent failure.

## Current behavior

`MermaidRenderer.renderToSvg(String)` (and therefore `Mermaid.render`, `renderRaw`, `renderWithLayout`, `renderToImage`):

1. Returns `null` for null/blank input without an error message.
2. Returns `null` if the bundled mermaid script cannot be loaded.
3. Catches mermaid JS failures inside the GraalJS script, prefixes them with `ERROR:`, prints them to `System.err`, and returns `null`.
4. On GraalJS/engine failure or empty output, prints a truncated diagram plus `JsExecutionResult.getErrorMessage()` to `System.err` and returns `null`.

`Mermaid.renderDetailed` / `MermaidRenderer.renderToSvgDetailed` already exist as a non-throwing inspection path, but they are incomplete:

- Mermaid JS errors caught in the script come back as **`JsExecutionResult.success("ERROR:…")`**, not as `failure(…)`.
- Callers of the methods people actually use (`render`, `renderToSvg`) still only see `null`.

There is no exception type in the library today.

## Assumptions

These are locked for this spec. Correct them in review if they are wrong.

1. **Throw, do not keep `null`.** The issue asks to throw instead of returning `null` on mermaid render failure. This is a breaking behavioral change. The library is still pre-1.0 (`0.3.0-beta.1` on `main`), so a documented break is acceptable; a parallel null-returning API is not.
2. **Unchecked exception.** `MermaidRenderException extends RuntimeException`. Call sites stay compilable; callers who already ignored `null` will now fail loudly instead of NPEing later. Checked exceptions would add `throws` noise to every façade method and every test.
3. **One exception type.** No error-code enum, no sealed hierarchy. `getMessage()` is enough for a UI to display.
4. **Scope is mermaid rendering**, not Batik rasterisation. `svgToImage` / Batik failures may still return `null` and log via `java.util.logging`. That is a separate problem.
5. **`renderDetailed` stays** as the non-throwing inspection API, but mermaid JS errors must become real failures (not `success` with an `ERROR:` prefix).
6. **Stop printing render failures to `System.err`.** The exception *is* the signal. GraalJS timing/debug prints in `GraalJsExecutor` are out of scope.
7. **Java 17 / current `main`.** Implementation targets `main` (`0.3.0-beta.1`), not the older Java 8 `upstream` line.

## Approaches considered

### A — Throw from the existing methods (recommended)

Change `renderToSvg` / `Mermaid.render*` so mermaid/engine failure throws `MermaidRenderException` instead of returning `null`. Keep `renderDetailed` as the Result-style escape hatch, and fix it so JS errors are `failure`, not `success("ERROR:…")`.

- **Pros:** Matches the issue; one obvious path; UIs get a normal `try/catch`; no second set of method names; `renderDetailed` already covers “inspect without throwing”.
- **Cons:** Breaking for anyone who tests `== null`. Existing tests that `assertNull` on empty input must change.

### B — Dual API (`render` stays null, add `renderOrThrow`)

Leave current methods unchanged; add throwing variants.

- **Pros:** Source-compatible.
- **Cons:** The methods people use still hide the error. Two styles forever. Does not solve #3 as written (“instead of returning `null`”).

### C — Result type as the primary API

Make `JsExecutionResult` (or a richer `RenderResult`) the only public contract; drop or deprecate String-returning methods.

- **Pros:** No exceptions; forces callers to branch on success.
- **Cons:** Largest break; `JsExecutionResult` is an internal GraalJS artifact, not a mermaid-domain type; worse fit for “present this error to the user” than `catch`.

**Recommendation: A.** The issue is specifically about throwing. The library is beta. `renderDetailed` already is approach C for callers who want it — it just needs to report JS errors correctly.

## Design

### Exception

```java
package com.aresstack.mermaid;

/**
 * Thrown when Mermaid diagram rendering fails.
 * {@link #getMessage()} is suitable for display to an end user.
 */
public class MermaidRenderException extends RuntimeException {
    public MermaidRenderException(String message) { super(message); }
    public MermaidRenderException(String message, Throwable cause) { super(message, cause); }
}
```

- Package: `com.aresstack.mermaid` (same as `MermaidRenderer`), not `com.aresstack`, so the façade class `Mermaid` does not grow siblings for every domain type.
- `getMessage()` is the mermaid/JS error text with the `ERROR:` sentinel stripped, or a short Java-side reason (`"Empty diagram code"`, `"Mermaid bundle not available"`, `"JS evaluation timed out after 120 seconds"`, `"Empty render output"`). Guest JS stacks that mermaid already concatenates into `__renderError` stay in that string; do not add a second copy. `JsExecutionResult` has no `Throwable` field, so this change does not wrap `PolyglotException` as `getCause()`. The two-arg constructor exists so later work can attach a cause without another API break.
- No diagram-source field on the exception (YAGNI). Callers already have the source they passed in.

### Method contracts

| Method | On mermaid/engine failure | On null/blank input |
|---|---|---|
| `MermaidRenderer.renderToSvg` | throw `MermaidRenderException` | throw `MermaidRenderException` (`"Empty diagram code"`) |
| `MermaidRenderer.renderWithLayout` | same (propagates) | same |
| `Mermaid.render` / `renderRaw` / `renderWithLayout` | same | same |
| `Mermaid.renderToImage` (both overloads) | same, because they call `render` | same |
| `Mermaid.renderDetailed` / `renderToSvgDetailed` | return `JsExecutionResult.failure(message)` — **never throw** for mermaid errors | return `failure("Empty diagram code")` as today |
| `Mermaid.svgToImage` / Batik path | unchanged (`null` + logger) | unchanged |

`renderToImage` does **not** start throwing for Batik rasterisation failure. If SVG render succeeds and Batik fails, behavior stays `null` + warning log.

### Control flow (renderToSvg)

1. Null/blank diagram → throw `MermaidRenderException("Empty diagram code")`.
2. Missing preamble/bundle → throw `MermaidRenderException("Mermaid bundle not available")`.
3. Run the existing GraalJS script (unchanged mermaid API / `ERROR:` sentinel).
4. If `JsExecutionResult` is unsuccessful → throw `MermaidRenderException(result.getErrorMessage())`. `JsExecutionResult` does not carry a `Throwable` today; do not add one in this change. The exception message is the engine/mermaid text.
5. If output is empty → throw `MermaidRenderException("Empty render output")`.
6. If output starts with `ERROR:` → throw `MermaidRenderException(output.substring("ERROR:".length()))`.
7. Otherwise post-process SVG and return it. **Do not** `System.err.println` on these paths.

`renderToSvgDetailed` uses the same script and classification, but returns `failure(message)` at steps 1–6 instead of throwing, and `success(svg)` (raw, currently un-post-processed — keep that existing quirk; do not expand this issue into unifying post-processing).

### Deduplication

`renderToSvg` and `renderToSvgDetailed` currently duplicate the JS setup script. Implementation should extract one private helper that returns `JsExecutionResult`, then:

- `renderToSvgDetailed` returns it (after classifying `ERROR:` / empty as failure).
- `renderToSvg` maps failure → throw, success → `postProcessSvg`.

That is an internal refactor, not a public API change.

### Call-site impact inside this repo

- Tests that `assertNull(renderer.renderToSvg(null))` become `assertThrows(MermaidRenderException.class, …)`.
- Happy-path `assertNotNull(svg)` tests stay valid (success still returns a String).
- Diagnostic / Swing demos that treat `svg == null` as render failure should catch `MermaidRenderException` (or keep a local try/catch). Not required for the first slice if those classes are manual mains.
- `MermaidRendererMain` should print the exception message instead of `"Rendering FAILED"`.

### Versioning and docs

- **Breaking** change: document under `Changed` in `CHANGELOG.md` for the next `0.3.0-beta.*` (or whatever the next release is). Do not bump the version in this design work; the implementation PR does that if the project’s release process expects it.
- Update README Rendering API table: `Mermaid.render(code)` returns SVG or throws; `renderDetailed` remains the non-throwing result object.
- Javadoc `@return` / `@throws` on every changed public method.

## Testing strategy

JUnit 5, same module as `MermaidRendererTest`.

Must add (or rewrite) tests that fail before the change:

1. Invalid mermaid source (e.g. `graph TD; A---`) throws `MermaidRenderException` whose message contains mermaid’s parse/render text (not a generic `"null"` / empty message).
2. Null and blank input throw `MermaidRenderException` with `"Empty diagram code"`.
3. `renderToSvgDetailed` on invalid source: `isSuccessful() == false`, `getErrorMessage()` non-blank, output is not a string starting with `ERROR:`.
4. Valid flowchart still returns SVG containing `<svg` (existing test remains the regression guard).
5. `Mermaid.render` / `renderRaw` propagate the same exception (thin façade tests, not a second copy of mermaid-error parsing).

No need for a test that asserts `System.err` is quiet; removing the prints is specified above.

## Boundaries

- **Always:** Java 17; keep public method names; keep `renderDetailed`; document the break in CHANGELOG + README; tests first for the throw/failure cases.
- **Ask first:** Throwing from `svgToImage` / Batik; adding error codes; changing `JsExecutionResult` into a mermaid-domain type; silencing `GraalJsExecutor` debug `System.err` prints.
- **Never:** Swallow mermaid errors again; return `null` from `renderToSvg` / `Mermaid.render*` on mermaid failure; treat `ERROR:` output as success in `renderDetailed`.

## Out of scope

- Batik rasterisation error reporting.
- `DiagramLayoutExtractor.extract(null)` still returning `null`.
- GraalJS context-creation debug logging.
- New CLI flags or a structured error JSON schema.
- Changing mermaid initialize config or the JS render script beyond error plumbing.

## Success criteria

- Invalid diagram source thrown from `Mermaid.render` / `renderToSvg` with a message a UI can show.
- Null/blank input throws with a stable message, not `null`.
- `renderDetailed` reports mermaid JS errors as `failure`, not `success("ERROR:…")`.
- Valid diagrams still render.
- README + Javadoc + CHANGELOG describe the new contract.
- No `System.err` print on the mermaid-failure path in `MermaidRenderer.renderToSvg`.

## Open questions

None blocking implementation if the assumptions above stand. The one product choice to confirm in review:

> Is a **breaking** change from `null` → throw on `Mermaid.render*` acceptable for `0.3.0-beta.x`, or should we keep null-returning methods and only add throwing variants (approach B)?

This spec assumes **acceptable** (approach A).
