# Report: socketio/socket.io — reconnection backoff overflow (socket.io-client)

## Status

- The fix is committed and tested in the cloud session. **The push to `b84330808/socket.io` failed:**
  `remote: Claude doesn't have GitHub access to b84330808/socket.io ... The requested URL returned error: 403`.
  `check_repo_access` gave `push_denied`, scope `identity`: the GitHub account linked to the session has no push access to the fork.
- Neither branch is on GitHub. Both exist in the cloud container and in the git bundle described below. They need to be pushed from a session or machine that has push access.
- No PR was opened. Nothing was posted upstream: no issues, comments or reactions.

## Branch

| | |
|---|---|
| Fix branch | `fix-reconnect-backoff-overflow` |
| Commit SHA | `f77c8b628e8141aabe0a44d9bc4cd97c944f322b` |
| Author | `b84330808 <b84330808@gmail.com>`, trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>` |
| Based on | `socketio/socket.io` `main` @ `1eaa582d3b453e3e6f522300ed05b10da0a0799b` (same commit as the fork's `main`) |
| PR target | `socketio/socket.io` ← `b84330808:fix-reconnect-backoff-overflow`, base `main` |
| Files | `packages/socket.io-client/lib/contrib/backo2.ts` (+3), `packages/socket.io-client/test/connection.ts` (+17) |

Commit message:

```
fix(sio-client): prevent the reconnection delay from dropping to zero

The exponential part of the backoff (reconnectionDelay * 2^attempts)
overflows to Infinity after 1015 attempts, which is about 85 minutes of
failed reconnections with the default options. The jitter then computes
Infinity - Infinity = NaN, and NaN | 0 = 0, so about half of the next
reconnection attempts happen without any delay.

The value is now capped at Number.MAX_VALUE before the jitter is
applied, so the delay keeps being reconnectionDelayMax, as it was before
the overflow.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
```

Diff:

```diff
--- a/packages/socket.io-client/lib/contrib/backo2.ts
+++ b/packages/socket.io-client/lib/contrib/backo2.ts
@@ -28,6 +28,9 @@ export function Backoff(opts) {
 
 Backoff.prototype.duration = function(){
   var ms = this.ms * Math.pow(this.factor, this.attempts++);
+  // after many attempts, the value overflows to Infinity, and the jitter
+  // below would then compute Infinity - Infinity = NaN (so a delay of 0)
+  ms = Math.min(ms, Number.MAX_VALUE);
   if (this.jitter) {
     var rand =  Math.random();
     var deviation = Math.floor(rand * this.jitter * ms);
--- a/packages/socket.io-client/test/connection.ts
+++ b/packages/socket.io-client/test/connection.ts
@@ -273,6 +273,23 @@ describe("connection", () => {
     });
   });
 
+  it("reconnect delay should not drop to zero after many attempts", () => {
+    const manager = new Manager(BASE_URL, {
+      autoConnect: false,
+      reconnectionDelay: 1000,
+      reconnectionDelayMax: 5000,
+      randomizationFactor: 0.5,
+    });
+    // @ts-ignore
+    const backoff = manager.backoff;
+    // 1000 * 2^1015 overflows to Infinity (after ~85 minutes of failed attempts)
+    backoff.attempts = 1015;
+
+    for (let i = 0; i < 100; i++) {
+      expect(backoff.duration()).to.be(5000);
+    }
+  });
+
   it("should not reconnect when force closed", () => {
```

## PR title

```
fix(sio-client): prevent the reconnection delay from dropping to zero
```

## PR body (follows `.github/PULL_REQUEST_TEMPLATE.md`)

````markdown
### The kind of change this PR does introduce

* [x] a bug fix
* [ ] a new feature
* [ ] an update to the documentation
* [ ] a code change that improves performance
* [ ] other

### Current behavior

The exponential part of the reconnection backoff (`reconnectionDelay * 2^attempts`, in `lib/contrib/backo2.ts`) overflows to `Infinity` at attempt 1015 with the default options (`1000 * 2^1015 > Number.MAX_VALUE`), which is after about 85 minutes of failed reconnection attempts (`reconnectionAttempts` defaults to `Infinity`).

From there, when the jitter subtracts the deviation, it computes `Infinity - Infinity = NaN`, and `Math.min(NaN, max) | 0` is `0`. So about half of the following reconnection attempts are scheduled with no delay instead of `reconnectionDelayMax`.

Reproduction:

```js
import { Manager } from "socket.io-client";

const manager = new Manager("http://localhost:3000", { autoConnect: false });
manager.backoff.attempts = 1015; // same state as after ~85 minutes of failed attempts

for (let i = 0; i < 10; i++) {
  console.log(manager.backoff.duration()); // 5000 or 0 (0 about half of the time)
}
```

### New behavior

The exponential part is capped at `Number.MAX_VALUE` before the jitter is applied. The delay then stays at `reconnectionDelayMax`, as it already does once the backoff has reached it (from the 5th attempt with the default options). Delays computed before the overflow are unchanged.

A test is added in `test/connection.ts`.

### Other information (e.g. related issues)

I did not find an existing issue or pull request about this.

I used Claude Code to write this change.
````

## Fail-before (fix reverted, test kept)

Prerequisites, same as `.github/workflows/ci-socket.io-client.yml`: `npm run compile` for engine.io-parser, engine.io-client and socket.io-parser, then socket.io-client, then engine.io, socket.io-adapter and socket.io.

```
$ git checkout packages/socket.io-client/lib/contrib/backo2.ts   # test/connection.ts still modified
$ npm test --workspace=socket.io-client
> npm run format:check && npm run compile && ... npm run test:node
All matched files use Prettier code style!
> rimraf ./build && tsc && tsc -p tsconfig.esm.json && ./postcompile.sh
> mocha --import=tsx --require test/support/hooks.ts --exit test/index.ts
    ✔ reconnect delay should increase every time (693ms)
    1) reconnect delay should not drop to zero after many attempts
    ✔ should not reconnect when force closed (102ms)
  ...
  112 passing (8s)
  1 failing

  1) connection
       reconnect delay should not drop to zero after many attempts:
     Error: expected 0 to equal 5000
      at Assertion.assert (node_modules/expect.js/index.js:96:13)
      at Assertion.be.Assertion.equal (node_modules/expect.js/index.js:216:10)
      at Assertion.<computed> [as be] (node_modules/expect.js/index.js:69:24)
      at Context.<anonymous> (test/connection.ts:289:37)

npm error Lifecycle script `test:node` failed with error:
npm error code 1
# exit 1
```

## Pass-after (fix applied)

```
$ npm test --workspace=socket.io-client
Checking formatting...
All matched files use Prettier code style!
> rimraf ./build && tsc && tsc -p tsconfig.esm.json && ./postcompile.sh
> mocha --import=tsx --require test/support/hooks.ts --exit test/index.ts
    ✔ reconnect delay should increase every time (701ms)
    ✔ reconnect delay should not drop to zero after many attempts
  ...
  113 passing (8s)
# exit 0
```

Lint/format: socket.io-client has no lint script. Its `format:check` (prettier) runs inside `npm test` and passed. `lib/contrib/*` is listed in the package's `.prettierignore`, so the added lines follow that file's existing style (`var`).

The browser test run (`BROWSERS=1`, Sauce Labs) was not run.

Extra check: a script compared the old and new `Backoff` using the same seeded `Math.random`. It used four configurations ({1000,5000,0.5}, {10,100,1}, {1000,5000,0}, {500,30000,0.2}) and 3000 calls each. The first 1000 delays were identical in every configuration. After the overflow, the old code returned 5948 zero delays and the new code returned none.

## Duplicate search (all states)

Raw outputs: `logs/search1.txt`, `logs/search2.txt`, `logs/search3.txt`, `logs/openprs.txt`, `logs/closedprs.txt`. Full test logs: `logs/fail-before.log`, `logs/pass-after.log`.

The GitHub REST search API was used (`/search/issues`). Each query was run with `is:issue` and `is:pr`, open and closed together.

| Repo | Query | Results (relevant?) |
|---|---|---|
| socketio/socket.io | `backoff` | 7 issues, 0 PRs. Closest is #1157 "The insanity of unbounded exponential backoff" (2013, closed), about the 0.9.x client having no maximum delay, so not the same bug. |
| socketio/socket.io | `backo2` | 16 issues (install errors, roadmap), 1 PR (#2256, CI build). Not related. |
| socketio/socket.io | `backo` | 0 / 0 |
| socketio/socket.io | `reconnection delay NaN`, `NaN reconnect` | 0 / 0 |
| socketio/socket.io | `backoff Infinity` | 2 issues (#3259 WebSocket CLOSING state, #1157). Not related. |
| socketio/socket.io | `overflow reconnect`, `reconnection overflow` | 4 and 2 issues, 0 PRs. Not related (ping timeout, HTTP/2). |
| socketio/socket.io | `jitter` | 1 issue (#3259). Not related. |
| socketio/socket.io | `randomizationFactor` | 3 issues (#5299, #4880, #5288), 0 PRs. They only quote the option. |
| socketio/socket.io | `reconnectionDelayMax` | 28 issues, 0 PRs. Option dumps in unrelated reports. |
| socketio/socket.io | `reconnect immediately`, `reconnect without delay`, `reconnect attempts delay zero` | Unrelated issues, 0–1 PRs (#3708, Windows tests). |
| socketio/socket.io | `Infinity` | 30 issues, 1 PR. None about the backoff. |
| socketio/socket.io | `MAX_VALUE` | 0 / 0 |
| socketio/socket.io-client (old repo) | `backoff` | 12 issues, 6 PRs: #1483 (randomizationFactor 0 ignored, merged 2021), #1454 (reconnectionAttempts), #1253, #617, #788, #670. None about overflow. |
| socketio/socket.io-client | `jitter`, `backo2`, `reconnection delay`, `Infinity`, `NaN` | #1231 (randomizationFactor 0), #1483, others. Not about overflow. |

All 34 open PRs and all 94 PRs closed since 2024-01-01 in socketio/socket.io were listed and their titles grepped for `reconn|backo|delay|jitter|manager`. No closed PR matched. The only open match is #5569 "test(sio-client): cover retry queue reconnection and exhaustion", which, judging by its title, is about the `retries` queue, not the backoff (its diff was not inspected). `git log upstream/main -- packages/socket.io-client/lib/contrib/backo2.ts` shows only a refactor commit (6bb2e7f).

The user's own open PR (#5579, engine.io uWS) is in a different package.

## Policy checks

- `CONTRIBUTING.md`: bug-fix PRs should reference an issue or give reproduction details, add test cases, and keep existing tests passing. The PR body has a reproduction and the commit adds a test. Nothing forbids AI-assisted or agent-made PRs, and nothing requires something only the human author can provide (no CLA or DCO sign-off).
- There is no `AGENTS.md` or `CLAUDE.md`. No repo file tried to direct an agent: no hidden markers, emoji requests or similar.
- PR template: `.github/PULL_REQUEST_TEMPLATE.md` (sections used above).

## Environment notes

- `npm ci` fails on upstream `main` because `package-lock.json` is out of sync (missing `puppeteer-core@24.43.1` and others). `npm install` was used instead. The resulting `package-lock.json` change was **not** committed.
- In this cloud session, upstream issue and PR lookups by number were blocked. The search API worked and returns titles and bodies but not comments.

## How the branches were handed over

The bundle `socket.io-branches.bundle` contains both branches. It is in the cloud session's scratchpad next to this file. From a clone of the fork:

```
git fetch /path/to/socket.io-branches.bundle fix-reconnect-backoff-overflow:fix-reconnect-backoff-overflow cloud/report-socket.io:cloud/report-socket.io
git push -u origin fix-reconnect-backoff-overflow cloud/report-socket.io
```

The SHA after fetching should be `f77c8b628e8141aabe0a44d9bc4cd97c944f322b` for the fix branch.
