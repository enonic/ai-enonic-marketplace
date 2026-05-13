---
name: xp-app-debugger
description: >
  Debug Enonic XP application errors. Analyzes build failures (Gradle,
  TypeScript) and server runtime errors (Nashorn/JS stack traces in
  server.log). Use when the user asks to debug, troubleshoot, or fix
  errors in an XP app, or pastes XP log output containing ERROR or
  WARN entries.
license: MIT
compatibility: Claude Code
allowed-tools: Bash(tail:*) Bash(grep:*) Bash(./gradlew:*) Read Edit Grep Glob
metadata:
  author: enonic
  xp-version: ">=7.0"
---

## Critical Rules

1. **Never skip analysis** -- Do not jump to a fix before completing Phase 2 (Analyze). Understand the full error context first.
2. **Compiled JS is runtime truth** -- `build/resources/main/` contains what XP actually executes. Always check compiled output when
   debugging runtime errors, then trace back to source.
3. **Server-side logging** -- Use the global `log` object: `log.info()`, `log.error()`, `log.warning()`, `log.debug()`. There is no
   `console.log` on the server.
4. **Large logs get a subagent** -- Never read a full `server.log` in the main context. Spawn a haiku subagent to extract ERROR/WARN
   entries.
5. **Don't deploy unless asked** -- Never run `./gradlew deploy` or equivalent without explicit user approval.
6. **Check sibling repos** -- List the parent directory (`ls ..`) for related apps or libraries that may contribute to the error.
7. **Read logs directly** -- Never trust user descriptions of log output. Users scanning fast-scrolling logs miss entries or misread them.
   Always grep/read the log file yourself before concluding something is present or absent.

## Workflow

### Phase 1: Gather

- **User pastes logs**: Parse directly, identify error type (build or runtime).
- **User says "check the logs"**: Read `$XP_HOME/logs/server.log`. Use a haiku subagent for large files (see Reading Large Logs below).
- **User says "app doesn't load" or "nothing in logs"**: Grep `server.log` for the app name (e.g.
  `grep -i "com.enonic.app.myapp" $XP_HOME/logs/server.log | tail -20`). If there are zero matches, the app was never seen by XP — this
  points to a deploy path mismatch, not an app error. Actual failures (incompatible version, broken code) always produce log entries.
- **User says "debug my app"**: Ask whether to deploy first. If yes, use project-specific deploy instructions from CLAUDE.md/README, or fall
  back to `./gradlew deploy`.
- Identify error type: build failure (Gradle/TS), runtime error (server.log stack trace), or silent failure (app not visible to XP at all).
- Locate relevant files: source (`src/main/resources/`) and compiled (`build/resources/main/`).

**Gate**: Present findings — error type, location, relevant files. Ask user before proceeding to analysis.

### Phase 2: Analyze

- Trace error to exact source location (file path, line number).
- For Nashorn errors: map compiled JS path back to TS source if applicable.
- Check `references/known-patterns.md` for matching heuristics.
- Cross-reference `xp-app-creator` for XP API/domain knowledge (see XP Domain Knowledge table below).
- Check XP platform source at `https://github.com/enonic/xp` via `gh` if error points to XP internals.
- Check sibling repos: `ls ..` for related apps/libs that may be involved.
- Don't assume which value caused the error — understand the full context first.

**Gate**: Present analysis — root cause, evidence, reasoning. Ask user before proceeding to fix.

### Phase 3: Fix

- Propose specific fix with rationale.
- Show exact code change (before/after).
- Get user approval, then apply with Edit tool.

**Gate**: Fix applied. Ask user whether to redeploy and verify.

### Phase 4: Verify

- Redeploy the app (with user approval).
- Tail logs, check for the same error.
- If error persists: return to Phase 2 with new information.
- If resolved: report success. Suggest adding a new entry to `references/known-patterns.md` if the pattern is reusable.

## Log File Locations

| Log          | Typical path                                      | Contains                                                   |
|--------------|---------------------------------------------------|------------------------------------------------------------|
| Server log   | `$XP_HOME/logs/server.log`                        | Runtime errors, Nashorn stack traces, app lifecycle events |
| Build output | Terminal / Gradle console                         | Compilation errors, dependency issues                      |
| Sandbox log  | `~/.enonic/sandboxes/<name>/home/logs/server.log` | Same as server log, sandbox-specific path                  |

`$XP_HOME` is usually `~/.enonic/sandboxes/<name>/home` when running via CLI sandbox.

## Source File Mapping

When an error points to a file, look in the right place:

| Error references          | Check source at                            | Check compiled at                   |
|---------------------------|--------------------------------------------|-------------------------------------|
| `.js` controller          | `src/main/resources/**/<name>.ts` or `.js` | `build/resources/main/**/<name>.js` |
| `.xml` descriptor         | `src/main/resources/**/<name>.xml`         | Same (copied as-is)                 |
| Java class (XP internals) | `https://github.com/enonic/xp` via `gh`    | N/A                                 |
| `node_modules` / lib path | Check `build.gradle` dependencies          | `build/resources/main/lib/`         |

## Reading Large Logs

When `server.log` is large (>500 lines), use a haiku subagent to extract relevant entries:

```markdown
Task tool call:
subagent_type: "Bash"
model: "haiku"
prompt: "Run this command and return only the matching blocks:
tail -1000 $XP_HOME/logs/server.log | grep -E 'ERROR|WARN|Exception|at ' --context=3"
```

The subagent should return only ERROR/WARN blocks with surrounding context. Do not read full log files in the main context.

## Server-Side Logging

Add temporary logging to narrow down issues:

```js
// JS
log.info('Debug: value = %s', JSON.stringify(value));
log.error('Failed at step: %s', stepName);
```

```ts
// TS
log.info('Debug: value = %s', JSON.stringify(value));
log.error('Failed at step: %s', stepName);
```

The `log` object is globally available in all XP controllers. Format strings use `%s` for string substitution.

## Deployment

To deploy and test changes:

1. Check for project-specific instructions in `CLAUDE.md`, `README.md`, or `package.json` scripts.
2. Fall back to `./gradlew deploy` if no specific instructions exist.
3. After deploy, tail the server log to verify: `tail -f $XP_HOME/logs/server.log`

Always ask user before deploying.

## Sibling Repos

XP projects often have related apps or libraries in the same parent directory. Before deep-diving into a single app:

```bash
ls ..
```

Check if the error might originate from a shared library or companion app.

## XP Domain Knowledge

Don't duplicate XP knowledge — load from `xp-app-creator` references when needed:

| Topic                                       | Reference file                                      |
|---------------------------------------------|-----------------------------------------------------|
| Controller patterns (JS/TS)                 | `../xp-app-creator/references/controllers.md`       |
| Component structure (pages, parts, layouts) | `../xp-app-creator/references/components.md`        |
| Build system (Gradle, tsup, webpack)        | `../xp-app-creator/references/build-system.md`      |
| Content API                                 | `../xp-app-creator/references/content-api.md`       |
| Content types and schemas                   | `../xp-app-creator/references/content-types.md`     |
| Input types                                 | `../xp-app-creator/references/input-types.md`       |
| Site configuration                          | `../xp-app-creator/references/site-config.md`       |
| Project structure                           | `../xp-app-creator/references/project-structure.md` |

## Known Patterns

Load `references/known-patterns.md` when analyzing errors. If a resolved bug matches a reusable pattern, suggest adding it to the file (
max ~5 lines per entry, prune at ~30 entries).
