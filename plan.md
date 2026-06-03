# Plan: Adopt c3source 1.1.0 event extractors (issue #5)

**Branch:** `fix/5-adopt-c3source-extractors` · **Release:** `0.2.0`

## Context

`@genvid/c3source` 1.1.0 landed both upstream requests issue #5 was waiting on:

- **c3source#23** — `ExtractedFunction` now carries the signature
  (`{ kind, name, objectClass?, params: FunctionParameter[], returnType }`).
- **c3source#24** — `extractIncludes(sheet) → IncludeReference[]` (`{ includeSheet, jsonPath }`).

Both take a full `EventSheet`. This puts us on issue #5's **clean path**: delete the
hand-rolled event-tree walks in `src/domain/extraction.ts` and map c3source's typed
output to our `FunctionDef`.

Key facts making the swap behavior-preserving:
- c3source's `FunctionParameter` is byte-identical to our local one → `formatParams`
  (`"name: type"`) still applies.
- `formatting.ts` renders custom-ACEs as `objectClass.aceName(...)`, so
  `aceName` ← `f.name` when `f.kind === "custom-ace"`.

## Tasks (≈ one commit each)

0. **prep** — bump `@genvid/c3source` to `^1.1.0`, project version to `0.2.0`
   (package.json + package-lock.json), commit plan.
1. **types.ts** — drop local `FunctionParameter`; re-export c3source's.
2. **domainGenerator.ts** — inline c3source `extractFunctions`/`extractIncludes`.
   Add one exported pure seam `extractFunctionDefs(sheet, sheetName): FunctionDef[]`
   (param formatting + `sourceSheet`/`objectClass`/`aceName` mapping). Includes
   call site becomes `extractIncludes(sheet).map(r => r.includeSheet)`.
3. **delete `src/domain/extraction.ts`** + remove its `index.ts` re-export.
4. **tests** — rewrite the `extractFunctions`/`extractIncludes` blocks in
   `domainFormatter.test.ts` to the new signatures (wrap fixtures in a minimal
   `EventSheet`, target `extractFunctionDefs`).
5. **docs** — update `CLAUDE.md` dependency note + the `extraction.ts` paragraph
   under Architecture (file retired).

## Validation

`npm run lint && npm run typecheck && npm test && npm run build`, then code review.
Closes #5.

## Out of scope

Pushing a release tag / npm publish (separate explicit step). New 1.1.0 surface
unrelated to us (manifest-drift, scene-graph, SID tooling).
