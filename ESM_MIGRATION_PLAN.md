# ESM Migration Plan

Convert the project from CommonJS (CJS) to ECMAScript Modules (ESM).

## Background

- The project currently uses `require()` / `module.exports` throughout
- `@actions/github` v9+ is ESM-only; we're pinned to v8.x as a workaround
- The action runs on `node24` which supports `require(esm)` natively, but the source should be idiomatic ESM
- `@vercel/ncc` bundles everything into `dist/index.js` — ncc has known issues with ESM output (it emits CJS-style `__nccwpck_require__` internally), but can *consume* ESM input since it uses webpack under the hood
- The bundled `dist/index.js` is what actually runs in GitHub Actions (via `action.yml` → `runs.main`)

## Key Risks

1. **ncc bundling**: ncc can bundle ESM *input* but its output is still CJS-flavored. Since `action.yml` uses `runs.using: node24` and points to `dist/index.js`, and ncc's output is self-contained, this works fine — the output doesn't depend on `package.json` `"type"`.
2. **`local.js` CLI tool**: Uses `#!/usr/bin/env node` shebang and is referenced as `"bin"` in `package.json`. With `"type": "module"`, `.js` files are treated as ESM, which is what we want.
3. **Test suite**: mocha 11.x supports ESM natively. Tests use `require("@actions/github")` to mutate `context` between tests — this pattern needs reworking since ESM imports are live bindings, not mutable copies.

## Step-by-step Plan

### Step 1: Add `"type": "module"` to `package.json`

- Add `"type": "module"` to `package.json`
- Upgrade `@actions/github` from `^8.0.1` to `^9.1.0`
- Upgrade `@vercel/ncc` from `^0.36.1` to `^0.38.4` (latest, better ESM input handling)

### Step 2: Convert `code-build.js` to ESM

Replace:
```js
const core = require("@actions/core");
const github = require("@actions/github");
const { CloudWatchLogs } = require("@aws-sdk/client-cloudwatch-logs");
const { CodeBuild } = require("@aws-sdk/client-codebuild");
const assert = require("assert");

module.exports = { runBuild, build, waitForBuildEndTime, inputs2Parameters, githubInputs, buildSdk, logName };
```

With:
```js
import core from "@actions/core";
import github from "@actions/github";
import { CloudWatchLogs } from "@aws-sdk/client-cloudwatch-logs";
import { CodeBuild } from "@aws-sdk/client-codebuild";
import assert from "node:assert";

export { runBuild, build, waitForBuildEndTime, inputs2Parameters, githubInputs, buildSdk, logName };
```

Note: Check whether `@actions/core` and `@actions/github` v9 use default vs named exports and adjust accordingly.

### Step 3: Convert `index.js` to ESM

Replace:
```js
const core = require("@actions/core");
const { runBuild } = require("./code-build");
const assert = require("assert");

if (require.main === module) {
  run();
}

module.exports = run;
```

With:
```js
import core from "@actions/core";
import { runBuild } from "./code-build.js";
import assert from "node:assert";
import { fileURLToPath } from "node:url";

if (process.argv[1] === fileURLToPath(import.meta.url)) {
  run();
}

export default run;
```

Key change: `require.main === module` → `process.argv[1] === fileURLToPath(import.meta.url)` for ESM entry point detection.

### Step 4: Convert `local.js` to ESM

Replace:
```js
const uuid = require("uuid/v4");
const cp = require("child_process");
const cb = require("./code-build");
const assert = require("assert");
const yargs = require("yargs");
```

With:
```js
import { v4 as uuid } from "uuid";
import cp from "node:child_process";
import * as cb from "./code-build.js";
import assert from "node:assert";
import yargs from "yargs";
import { hideBin } from "yargs/helpers";
```

Also update the yargs usage — `yargs` v15 ESM usage requires `yargs(hideBin(process.argv))` instead of just `yargs.option(...)`. Consider upgrading `yargs` to a version with proper ESM support (v17+).

Note: The `uuid` import path changes from `"uuid/v4"` to a named import `{ v4 }` from `"uuid"`. The `uuid` package should be upgraded from `^3.4.0` to `^9.0.0` or later for ESM support.

### Step 5: Convert `test/code-build-test.js` to ESM

Replace:
```js
const { logName, githubInputs, inputs2Parameters, waitForBuildEndTime, buildSdk } = require("../code-build");
const { expect } = require("chai");
const forEach = require("mocha-each");
```

With:
```js
import { logName, githubInputs, inputs2Parameters, waitForBuildEndTime, buildSdk } from "../code-build.js";
import { expect } from "chai";
import forEach from "mocha-each";
```

**Critical**: The tests mutate `require("@actions/github").context` between test cases:
```js
const { context: OLD_CONTEXT } = require("@actions/github");
// later...
const { context } = require("@actions/github");
context.payload = { ... };
```

In ESM, `import` gives you live bindings to the module's exports, but you can't re-import to get a fresh reference mid-test. The fix:

```js
import github from "@actions/github";

// In beforeEach/afterEach, mutate the existing object:
const originalPayload = github.context.payload;
const originalEventName = github.context.eventName;

afterEach(() => {
  github.context.payload = originalPayload;
  github.context.eventName = originalEventName;
});

// In tests:
github.context.payload = { pull_request: { head: { sha: pullRequestSha } } };
```

This works because `github.context` is a mutable object — we're mutating properties on the same reference, not replacing the module binding.

### Step 6: Update `uuid` and `yargs` dependencies

- Upgrade `uuid` from `^3.4.0` to `^9.0.0` (ESM support, new import path)
- Upgrade `yargs` from `^15.3.1` to `^17.7.2` (ESM support)
- Remove the `mocha` overrides for `diff` and `serialize-javascript` if no longer needed after dependency resolution

### Step 7: Update build script and verify ncc bundling

- Update `@vercel/ncc` to `^0.38.4`
- Run `npm run build` and verify `dist/index.js` is generated correctly
- The ncc output will still be CJS-flavored internally (webpack bundle), which is fine — it's self-contained
- Verify the bundled output doesn't reference `import.meta` or other ESM-only constructs that would break in ncc's wrapper

### Step 8: Update ESLint configuration

- `eslint@8.x` config uses CJS (`module.exports = ...` in `.eslintrc.js` if present)
- If using a `.eslintrc.js` file, rename to `.eslintrc.cjs` or migrate to `eslint.config.js` (flat config)
- Alternatively, if config is in `package.json` or `.eslintrc.json`, no change needed
- Update ESLint `parserOptions.sourceType` to `"module"`

### Step 9: Run tests and lint

- `npm test` — all 23 tests should pass
- `npm run lint` — no lint errors
- `npm audit` — 0 vulnerabilities
- `npm run build` — dist/index.js builds successfully

### Step 10: Verify end-to-end

- Confirm `action.yml` still points to `dist/index.js` (no change needed)
- Confirm CI workflows don't need changes (they run `npm ci`, `npm test`, `npm run build`)
- Test `local.js` CLI: `node local.js --help` should work

## Files Changed Summary

| File | Change |
|------|--------|
| `package.json` | Add `"type": "module"`, upgrade deps (`@actions/github`, `uuid`, `yargs`, `@vercel/ncc`) |
| `code-build.js` | `require` → `import`, `module.exports` → `export` |
| `index.js` | `require` → `import`, `require.main` → `import.meta.url` check |
| `local.js` | `require` → `import`, update `uuid` and `yargs` usage |
| `test/code-build-test.js` | `require` → `import`, rework `@actions/github` context mutation |
| `package-lock.json` | Regenerated |

## Dependencies Upgraded

| Package | From | To | Reason |
|---------|------|----|--------|
| `@actions/github` | `^8.0.1` | `^9.1.0` | ESM-only from v9, which is now compatible |
| `uuid` | `^3.4.0` | `^9.0.0` | ESM support, `uuid/v4` path removed |
| `yargs` | `^15.3.1` | `^17.7.2` | ESM support |
| `@vercel/ncc` | `^0.36.1` | `^0.38.4` | Better ESM input handling |
