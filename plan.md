# Plan: Upgrade `@genvid/mcp-utils` to 0.4.0 + adopt helpers (issue #10)

**Issue:** [#10](https://github.com/genvid-holdings/c3-domain-manager/issues/10) — Upgrade `@genvid/mcp-utils` to `0.4.0`
**Branch:** `feat/mcp-utils-0.4.0` (off `main`)
**Validation gate after every task:** `npm run lint && npm run typecheck && npm test`

## Context

`0.4.0` is an additive, drop-in upgrade. Beyond the version bump, the user opted
to adopt the new (and some already-available `0.3.0`) mcp-utils helpers where
`src/mcp/server.ts` currently hand-rolls the equivalent.

**Verified against the published `0.4.0` types** (`npm pack @genvid/mcp-utils@0.4.0`):

- `mcpError(e, { prefix?, extraLines? } | string[])`
- `withMcpErrors(fn, { extraLines?, onError?, prefix? } | (() => string[]))`
  — `onError` is awaited **before** formatting; if it throws, the thrown value
  is formatted and `withMcpErrors` still never throws out.
- `mcpContent(text, footer?)` — single text block, `text` + `footer` joined by a
  single `"\n"`.
- `paginatedContent(fullText, options, footer?)` — single block; range line
  format `lines: <a>-<b> / <total>` (identical to the server's current footer).
- `READ_ONLY` / `REGENERATE` / `MUTATE` annotation constants (already in `0.3.0`).

**Decisions taken (post-verification):**

- **Drop adoption #1 (`{ prefix }`).** `loadConfig` (`domainGenerator.ts:38`) does
  the *inverse* of producing an error — it unwraps a `CallToolResult` back into a
  thrown `Error`. The `loadProjectConfig(<file>):` prefix is added *inside*
  mcp-utils, not hand-rolled here, and is guarded by `domainGenerator.test.ts:176`.
  `{ prefix }` has no valid drop-in target.
- **Include adoption #3 (`onError`).** Net-new error handling + correctness fix.
- **Fold in the `0.3.0` dedups** (annotation constants + `paginatedContent`).

> **Testing caveat:** `src/mcp/server.ts` has no test harness (only `test/domain/`
> and `test/adapters/` are tested). Tasks 2–5 are verified by `typecheck` + lint
> only, not runtime tests. Task 4 carries the only observable behavior change.

## Tasks (one commit each, ordered prep → refactor → feature)

### Task 1 — `chore: Upgrade @genvid/mcp-utils to 0.4.0`
- `package.json`: `"@genvid/mcp-utils": "^0.3.0"` → `"^0.4.0"`.
- `npm install` to refresh `package-lock.json`.
- No code changes; validator should be green (drop-in/additive).

### Task 2 — `refactor: Use mcp-utils tool-annotation constants`
- `server.ts:58-60`: delete the local `READ_ONLY`/`REGENERATE`/`MUTATE`
  `as const` objects; import them from `@genvid/mcp-utils`.
- Pure dedup; shapes confirmed identical.

### Task 3 — `refactor: Adopt mcpContent for txId footers in mutate tools`
- `set-overrides` (`server.ts:318-322`) and `remove-overrides` (`:366`):
  replace the manual `txId: <n>` string joins with `mcpContent(body, footer)`.
- Preserve current spacing by passing the footer as `\ntxId: ${txId}` (leading
  newline), so `mcpContent`'s single-`\n` join reproduces today's blank-line gap
  byte-for-byte.

### Task 4 — `refactor: Adopt paginatedContent for paginated reads`
- Replace the local `paginatedResponse` helper (`server.ts:97-112`) with
  `paginatedContent` in `read-domain-index`.
- **Behavior delta (intentional):** response collapses from **2 content blocks
  → 1** (`paginatedContent`'s purpose). The range-line format is identical. The
  stale-warning is preserved via the `footer` callback (rides after the range
  line in the single block). No runtime test guards this — relying on the
  verified `.d.ts` contract.

### Task 5 — `feat: Harden mutate writes with withMcpErrors onError txId bump`
- Wrap the `set-overrides` / `remove-overrides` write paths in
  `withMcpErrors(fn, { onError: () => { txId++ } })` so an `fs.writeFileSync`
  failure is (a) returned as a `CallToolResult` error instead of propagating
  uncaught, and (b) bumps `txId` so the client reconciles after a possibly-partial
  write (the watcher is suppressed during the write, so without this the client
  stays stale).
- Leave the early-return validation paths (txId-mismatch, override validation)
  unchanged — they return `isError` without throwing, so they correctly bypass
  `onError`.

## Out of scope / deferred
- `{ prefix }` option (#1) — no valid target (see decisions above).
- Standing up a server test harness — pre-existing known gap (`CLAUDE.md`).

## Risks
- **No server regression coverage** — tasks 2–5 verified by typecheck/lint only.
- Task 4 is the only observable behavior change (block count + warning placement).
- Possible `CLAUDE.md` doc touch-up (the "manual `txId` footer" / annotation
  descriptions) — defer to the end-of-run code-reviewer / tech-writer.

## Execution gates
- `genvid-dev:ts-implementer` per task → `genvid-dev:validator` after each →
  `genvid-dev:code-reviewer` at the end (offer `genvid-dev:tech-writer` if doc
  gaps are flagged).
