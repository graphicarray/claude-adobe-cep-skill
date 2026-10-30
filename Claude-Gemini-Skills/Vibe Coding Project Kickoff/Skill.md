# Vibe Project Kickoff (single-file skill)

**How to use:** attach this file to a new Claude chat (or put it in your project as `PROJECT-KICKOFF.md`) and say what you want to build, or paste the error you want debugged. Claude follows the instructions below. Platform notes, document templates and the debugging protocol are in the sections at the end of this file.

AI coding goes wrong when the AI invents structure while it codes: files land in random places, features contradict each other, and every new session forgets the rules. This skill moves those decisions into a few short documents that Claude re-reads at the start of every session, then builds in small, verified slices.

The user builds mostly: Adobe CEP panels, Chrome extensions, desktop apps, websites, and also asks for debugging. They are not necessarily a professional developer, so explain decisions in plain words and never ask them to hand-write code or configs.

## Step 0: Pick the mode

| Situation | What to do |
|---|---|
| New project or new idea | Full kickoff: Steps 1-5 |
| Existing code, no planning docs | **Adopt mode**: scan the repo, then write `AGENTS.md`, `docs/ARCHITECTURE.md` and `PLAN.md` from what actually exists (do not invent a different structure), then continue at Step 5 |
| Something is broken / an error / "not working" | Read the **Debugging protocol** section at the end of this file and follow it. Skip kickoff unless the project has no docs at all |
| Project already has `AGENTS.md` and `PLAN.md` | Read both, find the first unchecked item in `PLAN.md`, continue the Step 5 loop |

Then identify the project type and read the matching platform section (at the end of this file) before writing any doc, because the platform decides what is even possible (permissions, language limits, packaging, store rules):

- Adobe CEP panel -> the **Platform: Adobe CEP panel** section at the end of this file
- Chrome extension -> the **Platform: Chrome extension** section at the end of this file
- Desktop app -> the **Platform: Desktop app** section at the end of this file
- Website / web app -> the **Platform: Website / web app** section at the end of this file
- Anything else -> ask what runtime and distribution channel it targets, then proceed with the generic flow

## Step 1: Short interview (one round, max two)

Ask only what you cannot reasonably decide yourself. If the user already wrote a detailed brief, skip the interview and record your assumptions in the PRD instead. Use AskUserQuestion when available (max 4 questions), otherwise plain text. Cover:

1. What problem does it solve, and for whom? (one sentence is enough)
2. Target platform and version (e.g. Premiere Pro 2025, Chrome only, Windows + macOS, which browsers)
3. Must-have features for version 1, and anything explicitly out of scope
4. Distribution: private use, Chrome Web Store, Adobe Exchange, own website, paid or free
5. Any stack preference or existing code/branding/design to follow

Offer sensible defaults as options so the user can just pick. Unanswered questions become entries under "Assumptions" in the PRD, clearly marked, rather than blocking progress.

## Step 2: Write the docs, in this order

Create the files in the project folder (use the file-writing tool; if no file tools exist, output them in chat one at a time). Use the templates in the **Document templates** section at the end of this file. Keep each doc short and specific (aim under 150 lines) because the AI re-reads them every session and long docs get skimmed or drift out of sync.

| # | File | Contains | Skip when |
|---|---|---|---|
| 1 | `PRD.md` | Problem, users, goals, v1 scope, out of scope, feature list with IDs (F01, F02...), success checks, assumptions | never |
| 2 | `docs/PLATFORM.md` | Platform limits, permissions/security rules, packaging, store or distribution requirements (from the platform reference) | never for CEP, extension, desktop |
| 3 | `docs/DESIGN.md` | UI rules book: layout, colors, type, spacing, components, states (empty/loading/error), dark/light, accessibility | project has no UI |
| 4 | `docs/ARCHITECTURE.md` | Tech stack with reasons, folder tree, module responsibilities, data flow, **Database** section (tables + relations), **API** section (every endpoint or message type) | drop Database/API sections if the project has none (say "none" explicitly) |
| 5 | `AGENTS.md` | AI rules and guardrails (see below). Also create a one-line `CLAUDE.md` containing `@AGENTS.md` so Claude Code picks it up | never |
| 6 | `PLAN.md` | Milestones with checkboxes: M0 walking skeleton, then one milestone per feature group, then release | never |
| 7 | `docs/TESTING.md` | How to run tests, what to test per layer, manual checklist for the host app (Premiere, Chrome, etc.) | never |
| 8 | `docs/DEBUGGING.md` | Where logs live, how to attach a debugger for this platform, known issues log (starts nearly empty) | never |
| 9 | `README.md` | What it is, install/run/build, folder overview, links to the docs | never |

Feature specs (the "step by step guide" for each feature) are **not** written up front. Write `docs/features/F01-name.md` just before building that feature (Step 5), because specs written weeks ahead go stale and the AI follows stale specs faithfully.

Why this order: the PRD fixes the goal, the platform limits prune what is possible, design and architecture then work inside those limits, and the guardrails file can only be written once the stack and folders are known.

### What goes in AGENTS.md

This is the most important file. Include:

- One paragraph: what the project is and the stack
- "Read first every session": `PLAN.md`, `docs/ARCHITECTURE.md`, and the feature spec being worked on
- Folder rules: where new code goes, naming conventions, no new top-level folders without asking
- Coding rules specific to this platform (e.g. ExtendScript is ES3, MV3 service workers hold no state, `contextIsolation` stays on)
- Security rules: no secrets in code, minimum permissions, sanitize input, no remote code
- Working rules: one feature at a time, no unrequested refactors, no new dependencies without asking and saying why, update `PLAN.md` checkboxes after each task, commit after each working feature
- "Ask before": changing the stack, the database schema, the API contract, permissions, or deleting files
- "Done means": the feature's acceptance checks pass and the manual checklist item was verified

## Step 3: Review gate (do not skip)

Before any code, reread all the docs once and fix problems yourself:

- Do the PRD features all appear in the architecture and the plan?
- Does the design or architecture require something the platform doc forbids?
- Do the API and database sections agree with each other and with the folder tree?
- Is every v1 feature testable in a concrete way?

Then give the user a short summary: the stack chosen and why, the milestones, the top 3 risks, and your assumptions. Ask for approval or changes. Wait. The point of the gate is that fixing a wrong decision now costs minutes, while fixing it after 30 files exist costs hours.

## Step 4: Walking skeleton (milestone M0)

Build the smallest version that loads in the real host and does one trivial thing end to end: the panel opens in the Adobe app and shows a button that talks to the host script; the extension loads and its popup reads and writes storage; the desktop window opens and one IPC call works; the site renders one page and deploys locally. Verify it actually runs, commit, check the box in `PLAN.md`. This proves the toolchain, folder layout and packaging early, when problems are cheap.

## Step 5: Feature loop (repeat per feature)

1. Write `docs/features/Fxx-name.md`: user story, behavior, edge cases, files to touch, acceptance checks (observable, e.g. "clicking Export creates `out.png` and shows a success message")
2. Build only that feature
3. Verify against the acceptance checks: run tests, and walk the manual checklist when the feature touches the host app. Report honestly what was verified and what could not be (for example, Claude cannot open Premiere; say so and give the user exact steps to check)
4. Fix failures using the **Debugging protocol** section at the end of this file
5. Tick `PLAN.md`, add anything learned to `docs/DEBUGGING.md` or `AGENTS.md`, commit with a clear message
6. Move on only when the feature works

Small slices keep each AI change reviewable and make bugs easy to localize to the last commit.

## Release (final milestone)

Version number, changelog, production build/package, signing if the platform needs it, the store or distribution checklist from `docs/PLATFORM.md`, README final pass.

## Replying to the user

Keep updates short: what was created or built, what needs their decision, and the next step. Do not paste whole documents into chat when they are saved as files; point to the file names. If something was assumed, say so in one line.

---

## Platform: Adobe CEP panel

Use for panels in Photoshop, Illustrator, Premiere Pro, After Effects, InDesign, Audition and similar.

### Check before planning
Adobe is moving several apps to UXP, and CEP support differs per app and version. Before committing to CEP, verify with a quick web search that the target app and version still load CEP panels and which CEP/CSXS version it uses. If the target app supports UXP and the user is free to choose, mention it as an option and let them decide; keep CEP if they need older versions or an existing codebase.

### How a CEP panel is built
- **Panel UI**: HTML/CSS/JS running in an embedded Chromium. Its Chromium version is old and depends on the CEP version, so check feature support before using modern JS/CSS.
- **Host scripting**: ExtendScript (`.jsx`), running inside the Adobe app. It is **ES3-style JavaScript**: no `let`/`const`, arrow functions, template strings, `Array.map` etc. unless polyfilled, and no built-in `JSON` (bundle `json2.js`).
- **Bridge**: panel JS calls `CSInterface.evalScript("fn(args)", callback)`. Calls are asynchronous and return strings only, so serialize data (JSON) both ways and always handle errors in the callback.
- **Manifest**: `CSXS/manifest.xml` declares extension ID, version, host apps with version ranges, panel size, entry HTML, and the ExtendScript file. Wrong host ranges are a top reason a panel does not appear in the menu.
- **Node.js (optional)**: enabled via `CEFCommandLine` parameters in the manifest (e.g. `--enable-nodejs`, `--mixed-context`). Needed for file system or network access beyond browser APIs. Keep it minimal, and treat it as a security surface.

### Typical folder layout
```
extension-root/
  CSXS/manifest.xml
  .debug                  (dev only, not shipped)
  client/                 index.html, css/, js/, lib/CSInterface.js
  host/                   index.jsx (+ json2.js)
  icons/
  build/ or scripts/      packaging scripts
```

### Development setup
- Unsigned extensions only load when **PlayerDebugMode** is on: set `PlayerDebugMode` = `1` for the matching `CSXS.<version>` key (Windows registry `HKEY_CURRENT_USER/Software/Adobe/CSXS.<n>`; macOS `defaults write com.adobe.CSXS.<n> PlayerDebugMode 1`). Write a small script for this in the project rather than doing it by hand.
- Dev install path: Windows `C:\Users\<user>\AppData\Roaming\Adobe\CEP\extensions\` (or the Program Files common folder for all users); macOS `~/Library/Application Support/Adobe/CEP/extensions/`. Symlink the project folder there so edits show up without copying.
- Remote debugging: add a `.debug` file naming the extension ID and a port per host app; open `http://localhost:<port>` in Chrome to get DevTools for the panel. Restart or reload the panel after ExtendScript changes (ExtendScript is cached per session; use the panel reload or relaunch the app).
- Claude cannot run the Adobe app. Every feature needs exact manual verification steps for the user, and the panel should expose a visible log area or console so the user can paste errors back.

### Pitfalls to write into AGENTS.md
- ExtendScript stays ES3; never paste modern JS into `.jsx`.
- Every `evalScript` call needs error handling and string-only returns.
- Escape quotes and backslashes in strings sent through `evalScript` (file paths on Windows are the usual culprit).
- Wrap host actions that change the document in an undo group where the host API supports it.
- Do not assume a document or selection is open; check and show a friendly message.
- Never hardcode absolute user paths; use the extension path from `CSInterface` and OS-aware paths.

### Packaging and distribution
- Sign into a `.zxp` with `ZXPSignCmd` using a certificate (self-signed for private use, a proper cert for distribution), with a timestamp server.
- Install `.zxp` with an installer such as ZXPInstaller or Adobe's UPIA-based tools; for Adobe Exchange, follow the current Exchange submission requirements (verify these online, they change).
- Bump the version in `manifest.xml` on every release.

---

## Platform: Chrome extension

Default to **Manifest V3** (MV2 is no longer accepted for Chrome). If the user also wants Firefox or Safari, note the differences in `docs/PLATFORM.md` (Firefox supports MV3 with some API differences and a `browser_specific_settings` block; use `webextension-polyfill` if cross-browser is a goal).

### Parts of an extension
- **manifest.json**: name, version, permissions, host permissions, entry points. Single source of truth for what the extension may do.
- **Service worker (background)**: event-driven, **not persistent**. It is shut down when idle, so global variables are lost. Persist state in `chrome.storage` (`local`, `sync`, or `session`) and register event listeners at the top level of the file, not inside async callbacks. No DOM access, and no `window`; use an offscreen document if DOM APIs are needed.
- **Content scripts**: run inside web pages in an isolated world. They can read and change the page DOM but cannot use most `chrome.*` APIs; they talk to the service worker with `chrome.runtime.sendMessage` / ports.
- **Popup / side panel / options page**: normal HTML pages, extension origin.
- **Declarative APIs**: `declarativeNetRequest` replaces blocking `webRequest` for modifying or blocking requests.

### Hard rules in MV3
- No remotely hosted code and no `eval`/`new Function`; all JS ships in the package. Default CSP forbids inline scripts, so use separate `.js` files.
- Request the minimum permissions. Prefer `activeTab` over broad host permissions. Use `optional_permissions` / `optional_host_permissions` for features not everyone needs.
- Message passing needs a clear message schema (put it in the API section of `ARCHITECTURE.md`) and validation of every incoming message and every page-sourced value.
- Never inject user or page data with `innerHTML`; use `textContent` or a sanitizer.
- API keys must not be embedded in the extension (it is readable by anyone). Proxy through a server the user controls, or have the user paste their own key into the options page (stored in `chrome.storage.local`).

### Typical folder layout
```
src/
  manifest.json
  background/service-worker.js
  content/content.js (+ content.css)
  popup/ popup.html, popup.js, popup.css
  options/
  shared/ (types, messaging, storage helpers)
  icons/ 16, 32, 48, 128
```
If using a bundler (Vite + CRXJS, WXT, or plain esbuild), put the choice and reasons in the architecture doc. Prefer plain files for small extensions; add a bundler only when TypeScript, a UI framework, or npm packages are needed.

### Development and debugging
- Load unpacked via `chrome://extensions` with Developer mode on; click reload after every change (a bundler with HMR/watch mode removes this step).
- Service worker logs: "Inspect views: service worker" on the extension card. Popup: right-click the popup -> Inspect. Content script logs appear in the page's own DevTools console.
- "Errors" button on the extension card shows uncaught errors. A service worker that "goes inactive" is normal; test flows after letting it idle.

### Chrome Web Store checklist (put in docs/PLATFORM.md)
- Single, clear purpose, and a description that matches what the code does.
- Justification text for every permission, especially host permissions and `tabs`/`history`/`cookies`.
- Privacy policy URL if any user data is collected or handled; data-use disclosures completed in the developer dashboard.
- Assets: 128x128 icon, at least one 1280x800 (or 640x400) screenshot, small promo tile; listing text.
- Developer account (one-time registration fee). Reviews can take days; broad permissions slow them down. Verify current policy online before submission.
- Bump `version` in `manifest.json` for every upload.

---

## Platform: Desktop app

### Choosing the framework (decide in the interview, record in ARCHITECTURE.md)
| Option | Good for | Trade-off |
|---|---|---|
| **Electron** | Web-skilled builders, huge ecosystem, same Chromium everywhere | Large install size, higher memory |
| **Tauri** | Small, fast apps; Rust backend with web UI | Needs Rust for native logic; webview differs per OS |
| Native / Python (PyQt, etc.) | Heavy OS integration or existing code | Different UI skills, packaging effort |

If the user has no preference, recommend Electron for fast vibe-coding with a web UI, or Tauri when small size matters. Check current versions online before pinning dependencies.

### Electron rules (write into AGENTS.md)
- Process split: **main** (Node, owns windows, files, OS), **preload** (the only bridge), **renderer** (UI, treat like a web page).
- Keep `contextIsolation: true` and `nodeIntegration: false`. Expose a small, explicit API from preload with `contextBridge.exposeInMainWorld`; never expose raw `ipcRenderer` or `require`.
- Every IPC channel is listed in the API section of `ARCHITECTURE.md` with input/output shape; validate inputs in the main process. Use `ipcMain.handle` / `ipcRenderer.invoke` for request-response.
- Load only local content or trusted URLs, set a Content-Security-Policy, and open external links with `shell.openExternal` after checking the URL scheme.
- Store settings in the OS user-data directory (`app.getPath('userData')`), not next to the executable.
- Secrets: use the OS keychain (e.g. `keytar` or Electron `safeStorage`), not plain JSON.

### Tauri rules
- Native logic lives in Rust commands; the front end calls them through `invoke`. List each command in the API section.
- Grant the smallest set of capabilities/permissions per window in the Tauri config; add file-system and shell scopes narrowly.

### Typical layout (Electron)
```
src/
  main/        main process, windows, IPC handlers, services
  preload/     contextBridge API
  renderer/    UI code
  shared/      types and constants shared across processes
resources/     icons, installers assets
```

### Development and debugging
- Dev with hot reload for the renderer (Vite or similar); main-process changes need an app restart (use a watcher like `electronmon`).
- Renderer: DevTools (Ctrl/Cmd+Shift+I). Main process: console output in the terminal, or `--inspect` with Chrome DevTools. Log to a file in `userData/logs` so the user can send it back; show its location in `docs/DEBUGGING.md`.

### Packaging and release
- Packagers: electron-builder or Electron Forge (Tauri has its own bundler). Produce installers per OS (NSIS/MSI for Windows, dmg for macOS, AppImage/deb for Linux).
- **Code signing**: Windows needs a code-signing certificate to avoid SmartScreen warnings; macOS needs an Apple Developer account for signing and **notarization**, otherwise Gatekeeper blocks the app. Note these costs in the PRD if distributing publicly.
- Auto-update (electron-updater / Tauri updater) needs signed builds and a release host; decide whether v1 needs it.
- Test the packaged build on a clean machine or VM, not only in dev mode; paths, permissions and missing dependencies show up only there.

---

## Platform: Website / web app

### Decide in the interview
- Is it a **content site** (marketing, blog, docs), a **web app** (logins, data, dashboards), or both?
- Who updates the content: the user in code, or a non-technical person through a CMS?
- Hosting and budget, domain, expected traffic, languages, and whether SEO matters.

### Stack guidance (record the choice and reasons in ARCHITECTURE.md)
- Static / content-heavy: Astro or Next.js static export; plain HTML/CSS/JS is fine for a one-page site.
- Web app with accounts and data: a full-stack framework (Next.js, SvelteKit, Nuxt) plus a managed database/auth service (Postgres via Supabase or Neon, or SQLite for small tools) to avoid hand-rolling auth.
- Prefer boring, well-documented tools: the AI writes better code for popular stacks. Check current major versions online before pinning.

### Rules to write into AGENTS.md
- Responsive from the start (mobile first); test at 360, 768, 1280 px.
- Secrets only in environment variables (`.env` is git-ignored; provide `.env.example`). Server-only keys never reach client bundles.
- Validate input on the server even if the client validates too; use parameterized queries or an ORM; hash passwords with a standard library (never roll your own auth or crypto).
- Accessibility basics: semantic HTML, labels on inputs, keyboard focus visible, alt text, sufficient contrast.
- SEO basics for public pages: unique title and meta description, one `h1`, Open Graph tags, sitemap and robots, fast loading (optimize images, lazy-load).
- Handle empty, loading and error states on every data-driven page.

### Typical layout (framework-dependent)
```
src/ or app/   pages/routes, components, lib (data access, helpers)
public/        static assets
db/ or prisma/ schema and migrations
tests/
.env.example
```

### Development and debugging
- Local dev server with hot reload; browser DevTools (console, network, Elements); server logs in the terminal.
- Reproduce data issues with seed data (`db/seed`) rather than by hand.
- Deploy a preview early (Vercel, Netlify, Cloudflare Pages, or the user's host) as part of the walking skeleton so deployment problems surface in week one, not at launch.

### Release checklist
Environment variables set on the host, database migrations applied, custom domain + HTTPS, error monitoring or at least log access, backups for any database, 404 and error pages, analytics/privacy notice if tracking users (cookie banner where the law requires it), final Lighthouse pass for performance, accessibility and SEO.

---

## Debugging protocol

Used when something is broken, whether it is a bug found during the feature loop or a project the user brings in already failing. AI debugging fails mostly through guessing: changing several things at once, "fixing" symptoms, and piling up edits that hide the real cause. This protocol forces evidence before edits.

### 1. Get the facts
Ask for (or collect yourself, if you can run the code) only what is missing:
- The exact error text or screenshot, not a paraphrase
- What the user did, what they expected, what happened instead
- Environment: OS, host app/browser and version, project version, whether it ever worked and what changed since (last commit, new dependency, app update)
- Console output from the right place (see the platform list below)

If you cannot run the code (Adobe apps, Chrome, installed desktop builds), say so and tell the user precisely where to look and what to paste back. Keep that request short and numbered.

### 2. Reproduce
Find the smallest steps that trigger it every time. If it cannot be reproduced, add temporary logging around the suspected area and ask the user to try again, rather than guessing.

### 3. Form one hypothesis at a time
State it in one sentence: "The service worker loses `token` after going idle, so the second call fails." Read the relevant code and the project docs (`docs/PLATFORM.md` often explains the cause) before changing anything.

### 4. Smallest possible fix
Change the minimum that tests the hypothesis. Do not refactor, rename, or "clean up" in the same edit. If the hypothesis was wrong, **revert that change** before trying the next one, so failed attempts do not accumulate.

### 5. Verify
Re-run the reproduction steps and the nearby happy path. Check that previous features still work. State plainly what was verified and what was not.

### 6. Record
Add a row to the known-issues table in `docs/DEBUGGING.md` (symptom, cause, fix). If the same class of bug could recur, add a rule to `AGENTS.md` or a check to `docs/TESTING.md` regression list. Commit with a message that names the bug.

### Escalation rule
If two attempted fixes fail, stop editing. Add logging, re-read the architecture and platform docs, question the assumption behind the hypotheses, and consider whether the cause is in the environment (cache not reloaded, old build installed, wrong host version, stale service worker, PlayerDebugMode off) rather than the code. Many "bugs" are a stale build.

### Where to look, by platform
| Platform | Evidence sources |
|---|---|
| CEP panel | Remote DevTools at `http://localhost:<port>` from the `.debug` file; ExtendScript errors returned through `evalScript` callbacks (`EvalScript error.`); host app's ExtendScript Toolkit/VS Code debugger; confirm PlayerDebugMode, manifest host version range, and that the app was restarted |
| Chrome extension | `chrome://extensions` -> Errors button; service worker "Inspect"; popup inspect; page console for content scripts; check the extension was reloaded and the page refreshed |
| Desktop app (Electron) | Renderer DevTools; main-process terminal output or `--inspect`; log file in `userData/logs`; test the packaged build, not only dev mode |
| Website | Browser console and Network tab; server/terminal logs; hosting provider logs; compare local vs deployed environment variables |

### Debugging a project with no docs
Run Adopt mode first (see Step 0 above), at least `AGENTS.md` and a short `docs/ARCHITECTURE.md`, so fixes follow the project's real structure instead of the AI's guess. Do this quickly: read the entry points, the manifest or package file, and the folder tree, then write the docs from what exists.

---

## Document templates

Fill these in with project-specific content. Delete sections that do not apply instead of leaving empty headings. Keep sentences concrete: names, versions, file paths, numbers.

### PRD.md

```markdown
# <Project name> - Product Requirements

## Problem
<2-3 sentences: who has what pain today>

## Users
<who, skill level, where they use it>

## Goals (v1)
- <measurable outcome>

## Non-goals (v1)
- <things deliberately not built>

## Features
| ID | Feature | Priority | Notes |
|----|---------|----------|-------|
| F01 | ... | Must | |
| F02 | ... | Should | |

## Success checks
- <how we know v1 is done, observable>

## Assumptions
- <anything decided without the user's confirmation; mark clearly>

## Open questions
- <only real blockers>
```

### docs/PLATFORM.md

```markdown
# Platform constraints

## Target
<host app/browser/OS + minimum versions>

## Hard limits
- <language/runtime limits, forbidden APIs, lifecycle gotchas>

## Permissions and security
- <minimum permissions requested and why each is needed>
- <where secrets/API keys live>
- <input sanitization rules>

## Packaging and distribution
- <build output, signing, install location>
- <store/marketplace requirements: review rules, privacy policy, screenshots, listing text>
```

### docs/DESIGN.md

```markdown
# Design rules

## Principles
<3-5 rules, e.g. "dense but readable", "keyboard first">

## Tokens
- Colors: <name = hex> for background, surface, text, muted, primary, danger, success (light and dark if both)
- Typography: <font, sizes for title/body/caption>
- Spacing scale: <e.g. 4/8/12/16/24>
- Radius, borders, shadows: <values>

## Layout
<panel/popup/window dimensions, grid, responsive behavior>

## Components
<buttons, inputs, lists, modals: variants and states>

## States every screen must handle
Empty, loading, error, success, disabled.

## Accessibility
<contrast, focus order, labels, min target size>
```

### docs/ARCHITECTURE.md

```markdown
# Architecture

## Stack
| Layer | Choice | Why |
|-------|--------|-----|

## Folder structure
<tree with one-line purpose per folder>

## Modules and responsibilities
<module -> what it owns, what it must not do>

## Data flow
<how a user action travels through the layers; a short ASCII diagram is fine>

## Database
<"None" or: tables, columns with types, primary/foreign keys, indexes, migrations approach>

## API / messages
| Name | Direction/Method + path | Input | Output | Errors |
|------|-------------------------|-------|--------|--------|

## Config and environment
<env vars, settings storage, defaults>

## Key decisions
<decision, alternatives rejected, reason>
```

### AGENTS.md

```markdown
# <Project name> - AI working rules

## Project
<one paragraph + stack>

## Read first, every session
1. PLAN.md (find the next unchecked item)
2. docs/ARCHITECTURE.md
3. The feature spec in docs/features/ for the current task

## Structure rules
- New code goes in: <folders>
- Naming: <conventions>
- Do not create new top-level folders without asking.

## Platform rules
- <the 5-8 most important constraints from docs/PLATFORM.md>

## Security rules
- No secrets in code or commits.
- Request the minimum permissions; ask before adding any.
- Validate/sanitize all external input.

## Working rules
- One feature at a time. Do not refactor or "improve" unrelated code.
- No new dependency without asking, and say why it is needed.
- After each task: update PLAN.md checkboxes. After each working feature: commit.
- If a fix fails twice, stop and gather evidence (see docs/DEBUGGING.md) instead of trying a third guess.
- Never claim something works unless you ran or checked it; say what you could not verify.

## Ask before
Changing the stack, database schema, API contract, permissions, build setup, or deleting files.

## Definition of done
Acceptance checks in the feature spec pass, tests pass, manual checklist item verified, PLAN.md updated, committed.
```

### PLAN.md

```markdown
# Plan

## M0 - Walking skeleton
- [ ] Project scaffolded per ARCHITECTURE.md
- [ ] Loads in the real host (<Premiere / Chrome / window / browser>)
- [ ] One trivial end-to-end action works
- [ ] Build/package command works
- [ ] First commit

## M1 - <feature group>
- [ ] F01 <name> (spec: docs/features/F01-name.md)
- [ ] F02 <name>

## M2 - ...

## Release
- [ ] Version + changelog
- [ ] Production build / package / signing
- [ ] Distribution checklist (docs/PLATFORM.md)
- [ ] README final pass

## Notes / decisions log
- <date>: <decision>
```

### docs/features/Fxx-name.md

```markdown
# Fxx - <name>

## User story
As a <user>, I want <action> so that <benefit>.

## Behavior
<numbered steps of what happens, including UI text>

## Edge cases
- <empty input, permission denied, host app has no document open, offline, huge file...>

## Files to touch
- <path>: <change>

## Acceptance checks
- [ ] <observable result>
- [ ] <observable result>
- [ ] Existing features still work: <which>

## How to verify
<commands to run + manual steps in the host app>
```

### docs/TESTING.md

```markdown
# Testing

## Commands
- Unit: `<command>`
- Lint/typecheck: `<command>`
- Build: `<command>`

## What to test per layer
<logic = unit tests; message/IPC/API = contract tests; UI = manual checklist or e2e tool>

## Manual checklist (host app)
- [ ] Install/load fresh build
- [ ] <core flow 1>
- [ ] <core flow 2>
- [ ] Error path: <e.g. no document open>
- [ ] Uninstall/reload leaves no leftovers

## Regression list
<bugs that came back once; add a check for each>
```

### docs/DEBUGGING.md

```markdown
# Debugging

## Where to look
- Logs: <paths / console location>
- Debugger: <how to attach for this platform>
- Reload/restart steps after a change: <exact steps>

## Known issues log
| Date | Symptom | Cause | Fix |
|------|---------|-------|-----|
```

### README.md

```markdown
# <Project name>
<one-line description>

## Features
## Requirements
## Install / run (development)
## Build / package
## Folder overview
## Docs
Links to PRD.md, docs/ARCHITECTURE.md, docs/DESIGN.md, PLAN.md
## License
```
