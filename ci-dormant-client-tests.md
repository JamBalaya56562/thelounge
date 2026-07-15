# Investigation: dormant client-side unit tests not run by CI

**Status:** investigation notes for a future PR (do not implement until the in-flight PRs
#5121 / #5122 / #5123 are merged).
**Related upstream issue:** [#3014 "Improve overall test coverage of our codebase"](https://github.com/thelounge/thelounge/issues/3014)
**Discovered while working on:** #1784 / #1783 (emoji `parse()` fix) and #4930 (#5121, `findNames`).

---

## TL;DR

The core message-parsing unit tests under `test/client/js/helpers/` are **not executed by
CI**. Two independent causes:

1. **Not collected** — vitest's `include` globs don't match these files (they aren't named
   `*Test.ts`/`*test.ts` and don't live under one of the whitelisted directories).
2. **Would not all pass even if collected** — the component test `parse.ts` throws on import
   under jsdom, `@vue/test-utils` output format changed, and `parseIrcUri.ts` has 2 real
   failures.

Net effect: the tests I updated in `parse.ts` (#1784) and added in `findNames.ts` (#5121)
never gate CI. The message-parsing helpers are a coverage blind spot.

---

## Current state (verified)

### CI entry point
- `.github/workflows/build.yml` and `release.yml` both run `yarn test`.
- `package.json`:
  ```json
  "test": "run-p --aggregate-output --continue-on-error lint:* test:vitest",
  "test:vitest": "vitest run"
  ```
- So the whole test selection is controlled by a single `vitest.config.ts`.

### The include globs (`vitest.config.ts`)
```ts
include: [
    "test/**/*Test.ts",
    "test/**/*test.ts",
    "test/tests/**/*.ts",
    "test/server.ts",
    "test/client.ts",
    "test/commands/**/*.ts",
    "test/models/**/*.ts",
    "test/plugins/**/*.ts",
    "test/shared/**/*.ts",
],
```
`test/client/**` is **not** a whitelisted directory, so under `test/client/` only files whose
name ends in `Test.ts`/`test.ts` are picked up.

### Which client tests run vs. are dormant

| State | Files |
|---|---|
| ✅ Run in CI | `test/client/js/authTest.ts`, `constantsTest.ts`, `helpers/friendlysizeTest.ts`, `helpers/localetimeTest.ts`, `helpers/roundBadgeNumberTest.ts` |
| ❌ Dormant (never run) | `helpers/parse.ts`, `helpers/parseIrcUri.ts`, `helpers/ircmessageparser/{findNames,findEmoji,merge,findChannels,parseStyle,anyIntersection,fill}.ts` |

The 9 dormant files are exactly the message-parsing core (parse / findNames / findEmoji /
merge / …) — the area touched by #5121 and #1784.

---

## Root cause detail

### Cause 1 — collection
The dormant files are named after the function under test (`findNames.ts`, `merge.ts`, …),
not `*Test.ts`, and they sit in `test/client/js/helpers/…`, which no include glob covers.
Most likely a leftover from an earlier test-tooling migration where `include` was rewritten
around the `*Test.ts` convention and the older-named files were not renamed/moved.

### Cause 2 — they don't all pass as-is
Ran locally with a temporary broadened include (`test/client/**/*.ts`):

```
Test Files  2 failed | 12 passed (14)
     Tests  2 failed | 72 passed (74)
```

**12 of 14 files pass today.** The pure-logic `ircmessageparser/*` tests
(findNames, findEmoji, merge, findChannels, parseStyle, anyIntersection, fill) **pass as-is**.
Only two files fail:

1. **`parse.ts` — fails to even load (0 tests collected).**
   `parse.ts` mounts `ParsedMessage.vue`, which transitively imports `client/js/socket.ts`.
   That module runs at import time:
   ```ts
   // client/js/socket.ts:7
   transports: JSON.parse(document.body.dataset.transports || "['polling', 'websocket']"),
   ```
   Under jsdom `document.body.dataset.transports` is unset, so the fallback string
   `"['polling', 'websocket']"` is passed to `JSON.parse` — **invalid JSON (single quotes)** →
   `SyntaxError` on import, so the whole test file dies before any test runs.

2. **`@vue/test-utils` output format changed.**
   The current version pretty-prints `wrapper.html()` with newlines/indentation between tags
   (e.g. `Hello\n<span …>`), whereas the expected strings in `parse.ts` were written for the
   old compact serialization. ~28 cases fail on whitespace / attribute-order alone even though
   the DOM structure is correct.

3. **`parseIrcUri.ts` has 2 genuinely failing assertions**
   (`should not parse invalid port`, `should not parse plus in port`) — needs a look at actual
   vs expected.

---

## Impact

- `parse.ts` test updates (#1784) and `findNames.ts` additions (#5121) are **not enforced by
  CI**. Both fixes were still validated out-of-band (rendered `parse()` HTML directly;
  `nickhighlights.ts` under `test/tests/**` does run and covers the network highlight side).
- The parsing helpers can regress without CI noticing.

---

## Proposed remediation (phased — do after the in-flight PRs merge)

### Phase 1 — low risk, high value: wire in the pure-logic tests
- Rename the `ircmessageparser/*` (+ `parseIrcUri.ts`) test files to `*Test.ts`, **or** move
  them under `test/tests/`, **or** add `test/client/**/*.ts` to `include` (see Phase 3 caveat).
- Fix `parseIrcUri.ts`'s 2 failing assertions (verify real vs expected first — could be a real
  parser bug or a stale expectation).
- These recover most of the parsing coverage cheaply since they already pass.

### Phase 2 — medium: revive the `parse.ts` component test
- Make `socket.ts` not throw at import under jsdom. Options:
  - A jsdom setup file that sets a valid value before modules load:
    ```ts
    // e.g. test setup
    if (typeof document !== "undefined" && document.body) {
        document.body.dataset.transports = '["polling","websocket"]';
    }
    ```
  - or mock `client/js/socket.ts` in the test.
  - (Cleaner long-term: make `socket.ts` not do top-level side effects / lazy-connect.)
- Reconcile with the new `@vue/test-utils` serialization: either regenerate all `expected`
  strings to the pretty-printed form, or normalize whitespace in `getParsedMessageContents`
  (e.g. collapse inter-tag `\n\s*`). Prefer explicit expected strings if attribute order is
  stable; otherwise a normalization helper is more robust.
- The correct post-fix outputs for the emoji cases are already encoded in the #1784 PR
  (`test/client/js/helpers/parse.ts`), so reuse those.

### Phase 3 — broaden include safely
- Only widen `include` to `test/client/**/*.ts` **after** Phase 1+2, otherwise `parse.ts`
  (load error) and `parseIrcUri.ts` (2 failures) immediately turn CI red.

---

## Reproduction (local)

Temporary vitest config used during investigation (place at repo root so `node_modules`
resolves; do not commit):

```ts
// vitest.tmp.config.ts
import {defineConfig} from "vitest/config";
import vue from "@vitejs/plugin-vue";
import path from "path";

export default defineConfig({
    plugins: [vue()],
    resolve: {extensions: [".ts", ".js", ".vue"], alias: {debug: path.resolve(__dirname, "scripts/noop.js")}},
    define: {__VUE_PROD_DEVTOOLS__: false, __VUE_OPTIONS_API__: false},
    test: {
        include: ["test/client/**/*.ts"],
        exclude: ["test/fixtures/**", "test/public/**"],
        environment: "node",
        // add a jsdom setup that sets document.body.dataset.transports to valid JSON to let parse.ts load
        setupFiles: ["test/fixtures/env.ts"],
        globals: true,
        testTimeout: 25000,
    },
});
```

Run everything:
```bash
npx vitest run --config vitest.tmp.config.ts
```
Run just the parser:
```bash
npx vitest run --config vitest.tmp.config.ts test/client/js/helpers/parse.ts
```

Note: `parse.ts` has `// @vitest-environment jsdom` at the top; the per-file directive
overrides the global `environment: "node"`.

---

## Appendix — key file references
- `vitest.config.ts` — the `include`/`exclude`.
- `client/js/socket.ts:7` — the top-level `JSON.parse(document.body.dataset.transports || …)`.
- `test/client/js/helpers/parse.ts` — component tests via `mount(ParsedMessage).html()`.
- `test/client/js/helpers/ircmessageparser/*.ts` — pure-logic tests (pass today, just not run).
- `test/fixtures/env.ts` — existing setup file (server config home); a jsdom setup would sit
  alongside it.
