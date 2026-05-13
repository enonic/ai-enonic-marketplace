---
name: xp-app-upgrader
description: >
  Use when upgrading or migrating an Enonic XP 7 application to XP 8 —
  converting descriptors (application.xml, site.xml,
  parts/layouts/pages/content-types, admin tools, APIs, services, webapp) to
  the new YAML `kind:` format, bumping `xpVersion` and the `com.enonic.xp.app`
  Gradle plugin to 4.x, or finishing/fixing partial xp8migrator runs. Also
  triggers on post-upgrade XP 8 deployment errors. Skip for brand-new XP 8
  apps and for upgrades between XP 7.x minor versions.
license: MIT
compatibility: Claude Code, Codex
allowed-tools: Bash(./gradlew:*) Bash(gradle:*) Bash(ls:*) Bash(find:*) Bash(grep:*) Bash(cat:*) Bash(mv:*) Bash(rm:*) Bash(curl:*) Bash(./migrator:*) Read Write Edit
metadata:
  author: enonic
  xp-version: "7.x → 8.x"
---

## What this skill does

Upgrades a single XP application's source tree from XP 7 to XP 8. Out of scope: dumping/loading content data between XP 7 and XP 8
instances (covered briefly at <https://raw.githubusercontent.com/enonic/doc-xp/refs/heads/8.0/docs/release/upgrade.adoc>), and
upgrading 3rd-party market apps (replace those with their XP 8 versions yourself).

The skill leans on the official [`xp8migrator`](https://github.com/enonic/xp8migrator) tool for descriptor conversion (`application.xml`,
the entire `site/` tree, admin tools, APIs, webapp → YAML with `kind:`) and only does the things the migrator doesn't handle by hand:
build-system reorganization (settings plugin, `xplibs.*` catalog, `gradle.properties` flow), TypeScript/bundler config, code-level breaking
changes, `logback.xml`, and post-migration cleanup of the orphaned `site/` directory. See
<https://raw.githubusercontent.com/enonic/doc-code/refs/heads/master/docs/upgrade.adoc> for the upstream
Enonic upgrade guide (source of truth for descriptor shapes and breaking-change lists), `references/manual-schemes-migration.md` for the
full set of transformations the migrator performs, and `references/examples.md` for the full `xplibs.*` alias tables, worked `build.gradle`
examples, and TypeScript wiring.

## Workflow (must follow in order)

The upgrade has two strict phases: **inventory & plan** (read-only, presented to the user), then **execute & validate** (only after the user
approves). Never edit files before the user approves the plan — silent edits frustrate users who want to inspect the change set first.

```
1. Detect      → read gradle.properties, build.gradle, app descriptor, list directories
2. Inventory   → enumerate every file the upgrade will touch (group by category)
3. Plan        → present a numbered checklist of changes; ask for approval
                 (descriptor changes are run via xp8migrator, not hand-edited)
4. Execute     → run xp8migrator for descriptors; edit build/code by hand
5. Validate    → run `./gradlew clean build` BEFORE any cleanup (so a failure leaves the originals intact for diagnosis)
6. Cleanup     → only after the build is green AND the user confirms: `rm -rf src/main/resources/site` and delete `application.xml`
7. Deploy      → offer `./gradlew clean deploy` and run it only after the user confirms
```

### 1. Detect

Read these to understand the app:

- `gradle.properties` — current `xpVersion`, `appName`, `appDisplayName`, `vendorName`, `vendorUrl`, `projectName`, `version`
- `settings.gradle` — does it declare `com.enonic.xp.settings`? If absent, the upgrade adds it (this plugin is the central XP 8 plumbing)
- `build.gradle` — current `com.enonic.xp.app` plugin version (pinned in XP 7, versionless in XP 8), the `app{}` block contents (XP 7 wires
  metadata here; XP 8 leaves it empty), how `com.enonic.xp:lib-*` deps are written (XP 7 long-form vs. XP 8 `xplibs.*` aliases)
- `gradle/wrapper/gradle-wrapper.properties` — Gradle version (must be 9.x for plugin 4.x)
- `src/main/resources/application.xml` (XP 7) or `application.yaml` (already XP 8)
- `src/main/resources/site/` — if present, this is a site app. The whole tree migrates to `cms/` (see
  `references/manual-schemes-migration.md`).
- `src/main/resources/webapp/` — if present, includes a webapp
- `src/main/resources/admin/tools/`, `admin/widgets/` — admin surface (widgets become `admin/extensions/` in XP 8)
- `src/main/resources/apis/`, `services/`, `tasks/` — HTTP endpoints / background tasks
- `src/main/resources/idprovider/` — ID provider (rare)
- `src/main/resources/cms/` — already partially migrated? If both `site/` and `cms/` are present, the migration is incomplete.
- **Stray descriptors outside the above locations.** Run
  `find src/main/resources -name '*.xml' -not -path '*/site/*' -not -path '*/admin/*' -not -path '*/apis/*' -not -path '*/services/*' -not -path '*/tasks/*' -not -path '*/webapp/*' -not -path '*/idprovider/*' -not -path '*/import/*' -not -name 'application.xml'`.
  Files like `lib/<name>/<name>.xml` (Page descriptors stashed inside `lib/`) are silently skipped by `xp8migrator` and stay as XP 7 XML in
  the upgraded app. Surface any hits to the user during the plan step — choices are: (a) delete if vestigial, (b) move to the appropriate
  standard location and re-run the migrator, (c) hand-convert in place.

Don't assume only one of these. A single app can be all three (site app + admin tool + webapp).

### 2. Inventory

Build a list of every file that needs to change. Group by category — the user will scan this in step 3, and a flat dump of 50 file paths is
unreadable. Categories:

| Category        | What to look for                                                                                                                                                                  |
|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Settings plugin | `settings.gradle` — needs `com.enonic.xp.settings` plugin in XP 8 (drives every other build-file change)                                                                          |
| Build           | `gradle.properties`, `build.gradle` (plugin pin, `app{}` block, dependency form), `gradle/wrapper/gradle-wrapper.properties` (Gradle version), `package.json` (`@enonic-types/*`) |
| App descriptor  | `src/main/resources/application.xml`                                                                                                                                              |
| Site → CMS      | `src/main/resources/site/**/*.xml` — the entire tree relocates to `cms/` (`site.xml`, `styles.xml`, parts, layouts, pages, content-types, mixins, x-data, macros, form-fragments) |
| Admin tools     | `src/main/resources/admin/tools/*/*.xml`                                                                                                                                          |
| Admin widgets   | `src/main/resources/admin/widgets/*/*.xml` (renamed to `admin/extensions/` in XP 8)                                                                                               |
| Services        | `src/main/resources/services/*/*.xml` (descriptors migrate to YAML; `kind: "Service"`)                                                                                            |
| APIs            | `src/main/resources/apis/*/*.xml` (rare in XP 7)                                                                                                                                  |
| Tasks           | `src/main/resources/tasks/*/*.xml`                                                                                                                                                |
| Webapp          | `src/main/resources/webapp/webapp.xml` (if present)                                                                                                                               |
| ID provider     | `src/main/resources/idprovider/idprovider.xml` (if present)                                                                                                                       |
| Server config   | `logback.xml` anywhere in the project                                                                                                                                             |
| Code            | controllers using removed admin URL helpers, services that should migrate to APIs                                                                                                 |
| Cleanup         | leftover `src/main/resources/site/` after a successful migrator run (it leaves orphan `.js`/`.svg` files behind)                                                                  |

### 3. Plan

Present the plan as a numbered checklist grouped by category. For each item state **what** changes and **why** in one line. Then ask the
user to approve, modify, or skip individual items. Wait for explicit approval before any write.

The user can grant approval up-front in the same turn as the original request (e.g. "show plan then apply", "go ahead and fix it", "do
whatever it takes"). Treat that as approval — show the plan and proceed straight into execution. Pause for a fresh confirmation only when
the request was open-ended ("what do I need to change?", "can you upgrade this?") without an explicit "go ahead".

Example shape:

```
## XP 7 → XP 8 upgrade plan for <app-name>

Detected: JS site app, xpVersion 7.9.0, plugin 3.6.2, no settings plugin, `app{}` block has explicit name/displayName/vendor wiring, dependencies use long-form `com.enonic.xp:lib-*:${xpVersion}` coordinates, Gradle wrapper 8.5, no admin tools.

### Build system (settings plugin + catalog)
1. `settings.gradle` — add the XP 8 settings plugin: `id("com.enonic.xp.settings") version "<latest>"`. This is the centerpiece of the XP 8 build; it supplies the `com.enonic.xp.app` plugin version and the `xplibs.*` catalog. Pick the latest released version (alpha/beta releases like `4.0.0-A3` / `4.0.0-B1` are fine — do not use `-SNAPSHOT`).
2. `build.gradle` — drop the `version '3.6.2'` pin from `id 'com.enonic.xp.app'`; in XP 8 the version is supplied by the settings plugin.
3. `build.gradle` — omit the `app { }` block (the verified XP 8 reference apps don't carry it). Keep `app { createDefaultDevTask = false }` only if the project registers its own custom `dev` task (e.g. an `NpmTask`).
4. `build.gradle` — migrate Enonic library dependencies from `"com.enonic.xp:lib-X:${xpVersion}"` to `xplibs.X`.
5. `gradle/libs.versions.toml` (new file) — extract third-party `com.enonic.lib:*` deps (e.g. `lib-thymeleaf`, `lib-xslt`, `lib-asset`, `lib-static`) into a Gradle version catalog, and reference them as `libs.<alias>` in `build.gradle`. Skip if the app has zero or one such dep.
6. `gradle.properties` — bump `xpVersion` to the highest XP 8 version available (prefer stable; fall back to alpha/beta like `8.0.0-A3` / `8.0.0-B1`; use `-SNAPSHOT` only as a last resort, since it requires `xp.enonicRepo("dev")`). Add `projectName` if not present (used by `settings.gradle`); leave `appName`/`version`/`group` in place. The metadata fields `appDisplayName`/`vendorName`/`vendorUrl` end up in `application.yaml` in XP 8 — the migrator pipes them across when present in `gradle.properties`, and they can be removed from `gradle.properties` afterwards.
7. `gradle/wrapper/gradle-wrapper.properties` — bump Gradle to 9.4.1 (plugin 4.x requires Gradle 9+). Run via `./gradlew wrapper --gradle-version 9.4.1`.

### Descriptors (run xp8migrator)
8. `src/main/resources/application.xml` → `application.yaml` (`kind: "Application"`, plus `title`/`vendorName`/`vendorUrl` imported from `gradle.properties`)
9. `src/main/resources/site/` → `src/main/resources/cms/` — the migrator splits `site.xml` between `cms/site.yaml` and `cms/cms.yaml`, moves `styles.xml` to `cms/style/style.yaml`, renames `x-data/` to `mixins/`, and converts every part/layout/page/content-type/macro to YAML with the right `kind:`. ~16 files touched.

### Validation
10. Run `./gradlew clean build` — must pass before any file deletion. If it fails, the originals in `site/` and `application.xml` are still in place for diagnosis.

### Cleanup (only after the build is green)
11. `rm -rf src/main/resources/site` and `rm src/main/resources/application.xml` — wipes both the original XML descriptors and the orphan `.js`/`.svg` siblings (the migrator copied those into `cms/`). Don't use `./migrator -x` for this — re-running the migrator on already-migrated output fails the post-migration step.
12. If both `cms/style/style.yaml` and `cms/styles/image.yaml` exist after migration (older migrator versions emitted both), delete `cms/styles/image.yaml` — `cms/style/style.yaml` is the canonical XP 8 form.
13. Then ask before running `./gradlew clean deploy` for a sandbox runtime check.

OK to proceed? Reply "go" to apply all, or list the numbers to skip.
```

### 4. Execute

Apply the approved changes in two passes — descriptors first via the migrator, then build/code edits by hand.

**Descriptor pass (xp8migrator):**

```sh
# from the app project root
curl -fsSL https://raw.githubusercontent.com/enonic/xp8migrator/main/migrator-install.sh | sh
./migrator -e overwrite         # non-interactive — overwrite any pre-existing target files
```

If the migrator errors out, run `./migrator -h` to inspect the available flags. **Don't run the migrator a second time with `-x`** — its
post-migration step tries to move `cms/style/style.yaml` again and fails with `FileAlreadyExistsException`. Cleanup of the original XML
happens via plain `rm` after the build is green (see step 5 below), not via `./migrator -x`.

The migrator leaves the original `site/` tree and `application.xml` in place. Sibling `.js`, `.html`, and `.svg` files are *copied* into the
new `cms/` location during migration. Don't delete anything yet — let `./gradlew clean build` validate the result first. If it fails, the
originals are still there for diagnosis.

On Windows, use `.\migrator.exe` and the PowerShell installer (`migrator-install.ps1`). Useful flags: `-h` (help), `-a <app-name>` (override
`gradle.properties`), `-e ask|overwrite|skip` (conflict policy — use `overwrite` for non-interactive runs).
See <https://github.com/enonic/xp8migrator> for the canonical CLI reference.

If `xp8migrator` is unavailable (offline environment, descriptor not handled by the tool), fall back to hand-edits per
`references/manual-schemes-migration.md`.

**Build / code pass (hand-edits):**

Edit the files in-place (`Edit` tool in Claude Code) for changes to `build.gradle`, `gradle.properties`, and any controller/template
fixes. Keep edits minimal — don't reformat unrelated lines, don't change content the user didn't approve.

### 5. Validate (before any deletion)

Run `./gradlew clean build` first — that catches descriptor-syntax errors and dependency-resolution failures. Report success or paste the
first failure with its file/line. If the build fails, the original `site/` tree and `application.xml` are still in place, so the user can
inspect them while you suggest a fix; ask before retrying.

### 6. Cleanup (only after the build is green AND the user confirms)

Once `./gradlew clean build` passes, **ask the user whether to delete the originals before doing it** — don't remove anything automatically.
A green build with descriptors in both XML and YAML forms is a valid, recoverable state; once the originals are deleted the change is hard
to reverse without git. Phrase the prompt as a yes/no question listing the exact paths, e.g. "Build passed. OK to delete
`src/main/resources/site/`, `src/main/resources/application.xml`, and `./migrator`?"

Only after the user confirms, run:

```sh
rm -rf src/main/resources/site
rm src/main/resources/application.xml
rm ./migrator                   # remove the binary too
```

The migrator already copied sibling `.js`/`.html`/`.svg` files from `site/` into `cms/`, so wiping the entire `site/` directory is safe.
Don't use `./migrator -x` for this — running the migrator on already-migrated output fails its post-migration step.

If both `cms/style/style.yaml` and `cms/styles/image.yaml` exist after migration (older migrator versions emitted both), delete
`cms/styles/image.yaml` — `cms/style/style.yaml` is the canonical XP 8 form.

### 7. Deploy (optional)

After cleanup, **ask the user whether to deploy** — don't run it automatically. `./gradlew clean deploy` writes the JAR into a sandbox's
hot-deploy folder, which mutates state outside the project, so it warrants explicit confirmation. Phrase it as a yes/no question, e.g. "
Build passed. Want me to run `./gradlew clean deploy`?"

If the user says yes, run the gradle command and report only what the gradle command itself returns (success / the first failure with
file/line). **Do not try to verify the deployment by any other means** — don't tail `server.log`, don't poke at the sandbox via `enonic`
CLI, don't grep for the app under XP_HOME. The `deploy` task just drops the JAR in the hot-deploy folder; XP loads it asynchronously and any
runtime load errors live in server logs that aren't this skill's responsibility. If the user wants runtime verification, hand off to the
`xp-app-debugger` skill.

The `xp-app-debugger` skill can help interpret build/runtime errors.

## What changes between XP 7 and XP 8

This is the canonical change set. Apply only the items that exist in the user's app — don't invent files.

> **Verify the public API of any XP `lib-*` before editing JS/Java that calls it.** This skill's lists of removed/renamed functions are a
> starting point, not the source of truth — they may be incomplete or out of date. Library APIs evolve between XP 8 pre-releases. Before
> recommending a replacement function (`getHomeToolUrl`, `extensionUrl`, etc.), confirm it actually exists in the current `lib-*` source by
> reading the relevant `.ts`/`.js` file at <https://github.com/enonic/xp/tree/master/modules/lib/> (raw form:
`https://raw.githubusercontent.com/enonic/xp/master/modules/lib/lib-<name>/src/main/resources/lib/xp/<name>.ts`). Don't assume a named
> function survived just because it was in XP 7. Same rule applies to Java APIs — verify before editing controllers that import XP types.

### Build files

> See `references/examples.md` for the full `xplibs.*` alias tables (every `com.enonic.xp:` library and API), worked `build.gradle`
> examples (site app with version catalog; TS app with custom `dev` task), and TypeScript wiring.

The XP 8 build is reorganized around the **`com.enonic.xp.settings` Gradle settings plugin**. Once it's in place, it supplies the
`com.enonic.xp.app` plugin version and the `xplibs.*` dependency catalog — both of which used to be set in the app's `build.gradle`. The
upgrade therefore *adds* lines to `settings.gradle` and *removes* lines from `build.gradle`.

**`settings.gradle`** — declare the settings plugin:

```diff
+plugins {
+    id( "com.enonic.xp.settings" ) version "<latest>"
+}
+
 rootProject.name = projectName
```

Pick the **latest released** 4.x version of the plugin (alpha/beta releases such as `4.0.0-A3` or `4.0.0-B1` are fine; do **not** use a
`-SNAPSHOT` suffix). The settings plugin and `com.enonic.xp.app` plugin ship together (the settings plugin supplies the app-plugin version),
but they do not track the XP runtime version — see the next paragraph.

**Verify the settings plugin version is actually published before pinning it.** The settings plugin's release cadence does NOT track XP's —
its `4.0.x` numbering is independent of `xpVersion=8.0.x`, and not every XP release ships with a matching settings plugin to the Gradle
plugin portal. Picking `4.0.0-B4` because `xpVersion=8.0.0-B4` will fail with
`Plugin [...] was not found ... could not resolve plugin artifact` if only `4.0.0-A3` / `4.0.0-B1` are live on plugins.gradle.org. Check
before committing to a version:

```sh
curl -s 'https://plugins.gradle.org/m2/com/enonic/xp/settings/com.enonic.xp.settings.gradle.plugin/' | grep -oE 'href="[^"]+/"'
```

Use the highest version that listing actually shows; it may lag the XP runtime version by one or more releases.

**`build.gradle`** — independent edits (the first three are driven by the settings plugin; the last by the Gradle 9 bump):

1. **Drop the version pin** on `com.enonic.xp.app`:

   ```diff
    plugins {
   -    id 'com.enonic.xp.app' version '3.6.2'
   +    id 'com.enonic.xp.app'
    }
   ```

   The settings plugin supplies the version. Pinning it at the app level conflicts with the settings plugin and is wrong for XP 8.

2. **Strip the `app { }` block.** XP 7 wired metadata through it; XP 8 reads metadata from `application.yaml` (`title`, `description`,
   `vendorName`, `vendorUrl`) — see the "Application descriptor" section below:

   ```diff
    app {
   -    name = "${appName}"
   -    displayName = "${appDisplayName}"
   -    vendorName = "${vendorName}"
   -    vendorUrl = "${vendorUrl}"
   -    systemVersion = "${xpVersion}"
    }
   ```

   **Omit `app { }` entirely by default** (the verified XP 8 reference apps don't carry it). Leaving it empty also works. Only keep
   `app { createDefaultDevTask = false }` if the project registers its own custom `dev` task (e.g. an `NpmTask` for TS-based apps);
   otherwise the auto-registered `dev` task is what you want.

3. **Migrate dependencies** to the `xplibs.*` catalog. The catalog has two namespaces:
    - **APIs** (`com.enonic.xp:*-api`) → `xplibs.api.<name>` (e.g. `portal-api` → `xplibs.api.portal`, `core-api` → `xplibs.api.core`,
      `admin-api` → `xplibs.api.admin`)
    - **Libraries** (`com.enonic.xp:lib-*`) → `xplibs.<name>` (e.g. `lib-content` → `xplibs.content`)

   ```diff
    dependencies {
   -    implementation "com.enonic.xp:portal-api:${xpVersion}"
   -    include "com.enonic.xp:lib-content:${xpVersion}"
   -    include "com.enonic.xp:lib-portal:${xpVersion}"
   +    implementation xplibs.api.portal
   +    include xplibs.content
   +    include xplibs.portal
        include "com.enonic.lib:lib-thymeleaf:3.0.0-SNAPSHOT"
    }
   ```

   Only `com.enonic.xp:` dependencies move to the `xplibs` catalog — third-party libraries (`com.enonic.lib:lib-thymeleaf`,
   `com.enonic.lib:lib-xslt`, `com.enonic.lib:lib-asset`, `com.enonic.lib:lib-static`, etc.) keep their full Maven coordinates **or** move
   to a separate Gradle version catalog (see step 5). See `references/examples.md` for the complete `xplibs` alias tables (6 APIs + 24
   libs).

   **`com.enonic.lib:lib-thymeleaf` requires a version bump for XP 8.** XP 7-era apps typically pin `2.1.1`, which is not compatible with XP
    8. The XP 8-targeted build is `3.0.0-SNAPSHOT`, currently published only to the Enonic dev channel — not the public release repo.
       Surface this in the upgrade plan whenever the app declares `lib-thymeleaf:2.1.1` (or any other 2.x), and bump it to `3.0.0-SNAPSHOT`.
       The build needs the dev channel reachable: **replace** any existing `xp.enonicRepo()` line in `repositories { … }` with
       `xp.enonicRepo( "dev" )` (the dev variant is a superset that includes releases — don't add it alongside the plain form, swap it in).
       Do **not** declare a raw `maven { url 'https://repo.enonic.com/snapshot' }` block; use the `xp.enonicRepo( "dev" )` shortcut. Watch
       for similar 2.x → 3.0.0-SNAPSHOT transitions across other `com.enonic.lib:*` libraries during the alpha/beta XP 8 window.

4. **Extract third-party libraries into `gradle/libs.versions.toml`** (recommended for apps with two or more `com.enonic.lib:*` deps). The
   XP 8 reference apps centralize non-`com.enonic.xp` libs in a Gradle version catalog so versions are declared once and referenced as
   `libs.<alias>` in `build.gradle`:

   ```toml
   # gradle/libs.versions.toml
   [versions]
   thymeleaf = "3.0.0-SNAPSHOT"
   xslt      = "2.1.1"
   asset     = "2.0.0-SNAPSHOT"

   [libraries]
   lib-thymeleaf = { module = "com.enonic.lib:lib-thymeleaf", version.ref = "thymeleaf" }
   lib-xslt      = { module = "com.enonic.lib:lib-xslt",      version.ref = "xslt" }
   lib-asset     = { module = "com.enonic.lib:lib-asset",     version.ref = "asset" }
   ```

   Then in `build.gradle`:

   ```diff
    dependencies {
        include xplibs.content
        include xplibs.portal
   -    include "com.enonic.lib:lib-thymeleaf:3.0.0-SNAPSHOT"
   -    include "com.enonic.lib:lib-xslt:2.1.1"
   -    include "com.enonic.lib:lib-asset:2.0.0-SNAPSHOT"
   +    include libs.lib.thymeleaf
   +    include libs.lib.xslt
   +    include libs.lib.asset
    }
   ```

   The catalog file lives at `gradle/libs.versions.toml` (Gradle's default location — no extra wiring in `settings.gradle` needed). Aliases
   in `[libraries]` map dotted accessors in `build.gradle` (`lib-thymeleaf` → `libs.lib.thymeleaf`). This keeps version bumps in one place
   and is the pattern verified against `app-superhero-blog`.

   For apps with only one third-party dep (or zero), keep the inline coordinate — a catalog file is overkill.

5. **Remove top-level `sourceCompatibility` / `targetCompatibility`.** Older XP 7 build files often include:

   ```diff
   -sourceCompatibility = JavaVersion.VERSION_11
   -targetCompatibility = sourceCompatibility
   ```

   Gradle 9 dropped these as `Project` properties — leaving them in fails the build with
   `Could not set unknown property 'sourceCompatibility' for root project ... of type org.gradle.api.Project`. For JS-only XP apps (no
   `src/main/java`) they're vestigial and safe to delete outright. For apps with Java sources, prefer letting the XP plugin's toolchain
   convention handle it (see next bullet) over hand-rolling a `java { sourceCompatibility = ...; targetCompatibility = ... }` block.

6. **Don't add an explicit Java toolchain block — the XP plugin sets it.** The `com.enonic.xp.base` plugin (applied transitively by
   `com.enonic.xp.app`, and applied directly by XP 8 *libraries* like `lib-react4xp`) sets the Java toolchain to **version 25** as a
   convention default whenever the `java` plugin is applied. So a block like:

   ```groovy
   java {
       toolchain {
           languageVersion = JavaLanguageVersion.of(25)
       }
   }
   ```

   is redundant in XP 8 builds and should be removed during the upgrade. Keep an explicit toolchain block only if you need to *override* the
   convention to a different JDK version. Setting `sourceCompatibility`/`targetCompatibility` is also unnecessary — the toolchain handles
   both.

**`gradle.properties`:** bump `xpVersion` to the highest XP 8 version actually available, preferring **stable** over pre-release over
snapshot:

1. **Stable release** (e.g. `8.0.0`, `8.0.1`) — use this if any final XP 8 release exists. Check
   <https://repo.enonic.com/public/com/enonic/xp/core-api/>.
2. **Pre-release** (alpha / beta, e.g. `8.0.0-A3`, `8.0.0-B1`) — use the highest one published to the release repo if no stable exists.
3. **Snapshot** (e.g. `8.0.0-SNAPSHOT`) — only as a last resort, when nothing else is published. Snapshots resolve only through
   `xp.enonicRepo("dev")` — which step 3 of the build edits configures — so they work, but they move under your feet between rebuilds. If
   the
   user is on the very old `7.x` series (< 7.16), warn that XP recommends going through 7.16.x first (see
   <https://raw.githubusercontent.com/enonic/doc-xp/refs/heads/8.0/docs/release/upgrade.adoc>),
   but the source-level edits are the same. Add `projectName = ...` if missing — `settings.gradle` reads it. Keep `appName`, `version`,
   `group` — those are still consumed by Gradle. The metadata fields `appDisplayName`, `vendorName`, `vendorUrl` (and the legacy unprefixed
   `displayName`) **move into `application.yaml`** in XP 8 (per the upstream upgrade guide); they can be removed from `gradle.properties`
   afterwards. The migrator will pick them up from `gradle.properties` if it finds them there, write them into `application.yaml`, and you
   can
   delete the now-redundant entries.

**`gradle/wrapper/gradle-wrapper.properties`:** plugin 4.x requires **Gradle 9+**. Pin to a current 9.x via the wrapper task (which also
refreshes `gradle-wrapper.jar`):

```sh
./gradlew wrapper --gradle-version 9.4.1
```

**JUnit Platform launcher (Gradle 9+).** If the app has JUnit 5 tests, Gradle 9 no longer auto-resolves
`org.junit.platform:junit-platform-launcher` onto the test runtime classpath — running `./gradlew test` fails with
`Failed to load JUnit Platform. Please ensure that all JUnit Platform dependencies are available on the test's runtime classpath, including the JUnit Platform launcher.`
Add the launcher explicitly:

```diff
 dependencies {
     testImplementation 'org.junit.jupiter:junit-jupiter:5.11.4'
+    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
 }
```

The version is supplied transitively by `junit-jupiter`; pinning it is unnecessary.

**`JsonMapGenerator` moved package (test-only).** The concrete `JsonMapGenerator` helper class — commonly used in XP 7 unit tests to drive
`MapSerializable.serialize(MapGenerator)` — moved package and jar in XP 8:

| XP 7                                               | XP 8                                                |
|----------------------------------------------------|-----------------------------------------------------|
| `com.enonic.xp.script.serializer.JsonMapGenerator` | `com.enonic.xp.testing.serializer.JsonMapGenerator` |

Compile error: `cannot find symbol: class JsonMapGenerator ... package com.enonic.xp.script.serializer`. Fix is a one-line import update —
the class is in the `com.enonic.xp:testing` jar (already on the test classpath as `testImplementation`). The interfaces it works with (
`MapGenerator`, `MapSerializable`) stay in `com.enonic.xp.script.serializer` and don't move.

### Application descriptor

`src/main/resources/application.xml` → `application.yaml` (handled by `xp8migrator`; XML is no longer recognized in XP 8).
`kind: "Application"` is mandatory — missing it fails deployment with `Invalid kind "null". Expected "Application"`. See
`references/manual-schemes-migration.md` §6.1 for the exact field map and
<https://raw.githubusercontent.com/enonic/doc-code/refs/heads/master/docs/upgrade.adoc> for an XP 7 → XP 8 example.

**Metadata flow.** The migrator pulls `appDisplayName` / `vendorName` / `vendorUrl` from `gradle.properties` and writes them into
`application.yaml` (as `title` / `vendorName` / `vendorUrl`). Two pre-migrator fixups in real XP 7 apps:

1. **Unprefixed `displayName`** in `gradle.properties` — rename to `appDisplayName` (the migrator only matches the `app`-prefixed form,
   otherwise `title:` ends up empty).
2. **Vendor info hard-coded in the `app{ }` block** of `build.gradle` — lift the literals into `gradle.properties` (or write them straight
   into `application.yaml` post-migration; both end states are valid).

If realized after running the migrator: fix the keys and re-run `./migrator -e overwrite`, or hand-edit `application.yaml`. (Don't use `-x`
on the re-run — see the "Descriptor pass" note.)

### Admin tools

> Also handled by `xp8migrator`. The hand-edit shape below matters for review (and for fixes the migrator may not auto-apply, like the
> system-API list).

`src/main/resources/admin/tools/<name>/<name>.xml` → `<name>.yaml`.

```yaml
kind: "AdminTool"
title:
  text: "My Dashboard"
  i18n: "admin.tool.dashboard.title"
description:
  text: "Application dashboard"
  i18n: "admin.tool.dashboard.description"
allow:
  - "role:system.admin"
apis:
  - "admin:extension"   # always include — replaces admin:widget
  - "admin:event"
  - "admin:status"
  # plus any custom APIs this tool calls
```

Two gotchas:

1. **`title` and `description` must be objects** (`{ text, i18n }`). Plain strings cause an NPE in `AdminToolMapper`. (APIs use plain-string
   `title` — that's correct for APIs, wrong for admin tools.)
2. **`apis:` is strictly enforced.** A tool can only call APIs it declares — calls to undeclared APIs fail at runtime with
   `API [<app>:<name>] is not mounted`. List every API the tool talks to, including the system APIs (`admin:extension`, `admin:event`,
   `admin:status`). For APIs contributed by an `include`d lib (e.g. `lib-asset`'s `asset` API), use the **bare name** (`"asset"`) —
   the lib's `apis/` are merged into your JAR at build time and mount under your app's key, not the lib's. Confirm with
   `unzip -l build/libs/<app-name>.jar | grep apis/`.

In XP 8 every admin tool is shown on the launcher menu. If you have a tool that should *not* appear there, convert it to an API instead (
move `admin/tools/<name>/` to `apis/<name>/`).

In admin tool **controllers**:

- `portalLib.assetUrl`, `widgetUrl`, etc. work the same.
- The admin lib lost `getAssetsUri()`, `getBaseUri()`, `getLauncherPath()`. Replace with `extensionUrl({ application, extension })` and
  `getHomeToolUrl()`.
- **Confirm against current source before editing.** As of XP 8 master the only `/lib/xp/admin` exports are `getToolUrl`, `getHomeToolUrl`,
  `getInstallation`, `getVersion`, `widgetUrl` (deprecated), and `extensionUrl` — `getLauncherUrl()` is also gone, and the launcher script
  is no longer injected at all. Verify the current API
  at <https://raw.githubusercontent.com/enonic/xp/master/modules/lib/lib-admin/src/main/resources/lib/xp/admin.ts>; same rule applies to
  other libs (`lib-portal`, `lib-content`, etc.) — see <https://github.com/enonic/xp/tree/master/modules/lib/>.
- The default Admin Home path is now `/admin` (was `/admin/home`).

In admin tool **HTML templates** (Mustache/Thymeleaf): the launcher script is no longer injected — replace the old launcher include with a
`widgetUrl()` call, and deliver tool config as inline JSON instead of a service fetch.

### Admin widgets → admin extensions

`admin:widget` is renamed to `admin:extension` in XP 8. Update any reference to `admin:widget` (e.g. in the `apis:` list of an admin tool)
to `admin:extension`. The on-disk descriptor file location may also change — consult the `xp-app-creator` skill for current XP 8
widget/extension layout.

### APIs

> Also handled by `xp8migrator`.

`src/main/resources/apis/<name>/<name>.xml` → `<name>.yaml`.

```yaml
kind: "API"
title: "Content API"
allow:
  - "role:system.authenticated"
mount: true
```

`kind: "API"` is mandatory. Note `title:` is a **plain string** for APIs (not the object form admin tools use).

If the app uses `services/` and you want to migrate to the new API model, move each service to `apis/<name>/`, convert the descriptor to
YAML, and update callers from `serviceUrl({ service })` to `apiUrl({ api })`. Services still work in XP 8 — migration is recommended but not
required for the upgrade itself.

### Site descriptors → `cms/` tree (YAML)

The entire `site/` tree relocates to `cms/`, every descriptor becomes YAML with a kind-specific `kind:`, `site.xml` splits into
`cms/site.yaml` + `cms/cms.yaml`, and `x-data/` is renamed to `mixins/`. **Handled by `xp8migrator`.** See
`references/manual-schemes-migration.md` for the systematic transformations: the path/`kind:` map, field renames (including
`<display-name>` → `title:` and its exceptions), structural patterns (`<form/>` → `form: []`, `<occurrences>`, `<config>` flattening,
`<regions>`), and special cases for `application.xml`, `site.xml` split, `styles.xml`, content-type field moves, and `OptionSet` renames.

Two real-world gotchas worth flagging up front (not in the systematic reference):

- **`title:` and `description:` accept either a plain string or a `{ text, i18n }` object.** Use the object form whenever the XP 7 XML had
  an `i18n="…"` attribute on `<display-name>`/`<description>` — it's the form the migrator emits, and it preserves localization keys. Plain
  strings are fine when there's no i18n key. (Admin tools are special — see "Admin tools" above; their `title:` *must* be the object form
  regardless.)
- **Site mappings can use either `match:` or `pattern:`/`invertPattern:`** — both forms are valid in `cms/site.yaml`. The XP 7
  `<match>type:'…'</match>` becomes `match: "type:'…'"`; XP 7 `<pattern>` becomes `pattern: ".*\\/rss"` with an explicit
  `invertPattern: false/true`.

#### `cms/site.yaml` `apis:` list — gotcha

Like admin tools (see above), a site must explicitly mount any **mounted-API** library it uses — most commonly `lib-asset` (which exposes an
`asset` API). Without the declaration, `assetUrl()`/asset-resolution calls fail at runtime. The migrator does NOT infer this — the
declaration must be added by hand:

```yaml
kind: "Site"
mappings:
  - controller: "/lib/rss/rss.js"
    order: 50
    pattern: ".*\\/rss"
    invertPattern: false
apis:
  - "asset"        # required when the site uses lib-asset
```

App-key qualification follows the same rules as admin tool `apis:` lists — a bare name (`"asset"`) refers to the current app's API
namespace; a fully-qualified key (`"<other-app>:<api>"`) targets a *separately-deployed* app's API.

**Important: a lib bundled into your app is not a "separate app".** When the app `include`s a library that publishes a mounted API
(e.g. `com.enonic.lib:lib-asset` providing the `asset` API), the lib's `apis/<name>/` directory is merged into your app's JAR at
build time, and the API is mounted under **your** app's key — not the lib's. So the reference must be the bare name (`"asset"`),
**never** `"com.enonic.lib.asset:asset"` and **never** `"com.enonic.app.<your-app>:asset"` either (the fully-qualified form is
only for cross-app calls). Verify the mount by inspecting the built JAR: `unzip -l build/libs/<app-name>.jar | grep apis/` should
show `apis/asset/...` regardless of which lib contributed the descriptor.

### Server-side configuration files

If the project repo contains server-config files (most commonly a `logback.xml` — remove `<withJansi>true</withJansi>` to avoid a startup
error), apply the changes documented at
<https://raw.githubusercontent.com/enonic/doc-xp/refs/heads/8.0/docs/release/upgrade.adoc>. That guide also covers data migration
(`dump`/`load`), Management API breaking changes, security (`xp.suPassword`, password hashing), and the default
`com.enonic.cms.default` repo behavior change.

## Tips for the conversation

- **Quote what you found.** When presenting the plan, quote the actual current values (`xpVersion = 7.9.0`, plugin `3.6.2`) so the user can
  tell at a glance you read their files instead of guessing.
- **Use diffs, not prose.** When the change is non-trivial (e.g. multi-line `build.gradle` plugin or dependency edits), show a unified-diff
  snippet in the plan so the user can approve precisely what will land.
- **Don't auto-touch generated/wrapper files.** `gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.jar` should not be hand-edited. If
  Gradle needs to be bumped, run the wrapper task: `./gradlew wrapper --gradle-version 9.0.0`.
- **Stop on the first deal-breaker.** If a required file is missing or in an unexpected location, ask before proceeding — don't synthesize a
  guess.
- **Hand off to siblings when relevant.** For new features added during the upgrade, point to `xp-app-creator`. For build failures, point to
  `xp-app-debugger`. For releasing the upgraded app, point to `gradle-release`.

## See also

- <https://raw.githubusercontent.com/enonic/doc-code/refs/heads/master/docs/upgrade.adoc> — upstream Enonic XP 7 → XP 8 app
  upgrade guide (source of truth)
- `references/examples.md` — full `xplibs.*` alias tables (6 APIs + 24 libs), worked `build.gradle` examples (site app with version catalog;
  TS app with custom `dev` task), TypeScript wiring with `@enonic-types/*`
- `references/manual-schemes-migration.md` — comprehensive set of descriptor transformations (paths, `kind:` map, field renames, special
  cases) — read this when the migrator is unavailable or when reviewing what it produced
- <https://raw.githubusercontent.com/enonic/doc-xp/refs/heads/8.0/docs/release/upgrade.adoc> — full instance-level upgrade guide
  (dump/load, security, management API changes)
- `xp-app-creator` skill — current XP 8 app structure, components, build configuration
- `xp-app-debugger` skill — diagnose build/runtime errors after upgrade
- `enonic-cli` skill — running `enonic dump create` / `dump load` for the data side
