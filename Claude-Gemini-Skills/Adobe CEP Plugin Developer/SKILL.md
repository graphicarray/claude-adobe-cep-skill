---
name: adobe-cep-plugin-dev
description: Build, debug, update and ship Adobe CEP panels (HTML/JS extensions with ExtendScript) for Illustrator, Photoshop, After Effects and Premiere Pro. Use whenever the user wants to create a new Adobe plugin, extension or panel, fix a bug in or add features to an existing CEP plugin or .zxp, or mentions CSInterface, evalScript, manifest.xml, CSXS, ExtendScript or .jsx, PlayerDebugMode, ZXPSignCmd, Adobe Exchange, or a panel that is missing, blank or shows "EvalScript error". Also use for CEP vs UXP decisions and CEP-to-UXP migration planning, even if the user only says "Adobe plugin", "Illustrator panel", "Premiere extension" or "After Effects tool with a UI".
---

# Adobe CEP plugin development (Illustrator, Photoshop, After Effects, Premiere)

CEP panels have two halves: an HTML/JS UI running in embedded Chromium, and ExtendScript (ES3-era
JavaScript) running inside the Adobe app. Almost every CEP bug lives at one of five layers:
loading, panel JS, the bridge between them, host script logic, or the user's environment.
This skill is built around finding the right layer fast and fixing it with the smallest change.

## Step 0: know the platform situation (it changes advice)

Adobe published a formal CEP-to-UXP schedule in September 2026. CEP still works today, but it is
being phased out per app and **removed from every flagship app from December 2029**. The
practical consequences differ a lot by host, so read `references/platform-status-2026.md` before
recommending an approach, and re-check Adobe's pages with web search if a decision depends on dates.

Short version (as of 20 Oct 2026):
- **Photoshop**: UXP is mature. Marketplace stops accepting new CEP plugins in **March 2027**, CEP is
  disabled by default from **December 2027**. Recommend UXP for new work.
- **Premiere**: UXP is GA. Recommend UXP for brand-new products; CEP is fine for existing panels.
- **After Effects / Illustrator**: UXP only reaches public beta in Nov 2026 / Spring 2027.
  **CEP + ExtendScript is the practical choice today.**
- **ExtendScript is not being retired**, so `.jsx` work is safe.

Give the user this in two or three sentences, then do what they asked. Respect their decision: if they
want CEP, build CEP well and use the migration-friendly structure (thin bridge, no Node unless needed).
Never claim CEP "no longer works" in general; it does.

## Step 1: triage the request

Decide which workflow applies, and collect missing facts in **one** short message (don't interrogate):

| Need | Ask for (only what's missing) |
|---|---|
| New plugin | Which host(s)? What should it do (user action → result)? Will it be sold on Adobe Marketplace? |
| Fix / update existing | Host + exact version, OS/chip, symptom + exact error text, how it's installed, the source or `.zxp`, log or DevTools output |

If the user uploaded a plugin, look in `/mnt/user-data/uploads` first; it may already answer most questions.
Work on a **copy** in `/home/claude/work/` (uploads are read-only).

## Workflow A: new plugin

1. Confirm hosts and the feature in one sentence (*user action → host operation → visible result*).
2. Scaffold a working, installable panel (generates manifest, `.debug`, UI, bridge, ES3 host scripts):
   ```bash
   python <skill-dir>/scripts/scaffold_cep_panel.py --name "Layer Tools" \
       --bundle-id com.acme.layertools --hosts ILST,PHXS,AEFT,PPRO --out /home/claude/work/com.acme.layertools
   ```
   Host aliases: `ai`, `ps`, `ae`, `pr`. Add `--node` only if a Node dependency is genuinely required.
   If the user's machine has Node and wants React/Vue/Svelte, HMR and typed `evalTS` calls, suggest
   **Bolt CEP** (`npx create-bolt-cep`) instead; see `references/extendscript-bridge.md`.
3. Implement the feature as `NS.api.<verbNoun>` functions in `host/hosts/<HOST>.jsx` (ES3, JSON in/out) and
   call them from the panel with `Bridge.call('verbNoun', [args])`. Read the host reference file for the
   target app(s) first (see Host quick sheet below); they hold the traps that waste the most time.
4. Verify statically (below), then deliver with install + test instructions.

## Workflow B: fix, update or extend an existing plugin

1. Get the code (folder, ZIP, or `.zxp`, which is just a ZIP). Only modify plugins the user wrote or has
   rights to change.
2. Run the audit immediately and use it to focus the investigation:
   ```bash
   python <skill-dir>/scripts/audit_cep_plugin.py /home/claude/work/<extension-or-zxp>
   ```
   It flags manifest mistakes (id mismatches, missing paths, bad host ranges, runtime too new), ES3 violations
   in `.jsx`, Node used but not enabled, unsafe `evalScript` string building, missing error handling, and
   host-specific pitfalls. It is heuristic: read each finding before acting on it.
3. Classify the symptom into a layer using `references/maintaining-existing-plugins.md` and the decision
   trees in `references/debugging-and-troubleshooting.md`. State the **root cause in one sentence** before editing.
4. Make the **smallest correct change**. Keep style, ES3 level and architecture. Never change
   `ExtensionBundleId` or `Extension Id`. Bump the version with `scripts/bump_version.py`.
5. Re-run the audit and `node --check` on changed panel JS. Summarise: files changed, why, how to verify.
6. For new features, follow "Adding a feature" in `references/maintaining-existing-plugins.md`.
7. Add a one- or two-sentence migration-readiness note only if relevant (Node use, `cep.process`,
   Photoshop timeline); don't push a rewrite unprompted.

## Golden rules (and why)

- **ExtendScript is ES3.** `let`, `const`, arrows, backticks, `forEach`, `JSON` (without a polyfill) and trailing
  commas make the *entire file* fail to parse, which surfaces as the useless reply `EvalScript error.` and every
  function being undefined. This is the most common cause of "buttons do nothing".
- **Everything crosses the bridge as a JSON string through one envelope** (`{ok, data|error}`), built with
  `JSON.stringify(JSON.stringify(args))`. Hand-concatenated `evalScript` strings break on quotes, backslashes and
  names like `O'Brien`, and let untrusted text become code.
- **One namespace object per product in ExtendScript.** All panels in an app share one global scope; bare
  globals collide with other vendors' panels.
- **Never use `alert()` in host scripts.** It blocks the host and can steal focus. Return a message instead.
- **Restore what you change** (Photoshop `rulerUnits`/dialogs, Illustrator coordinate system and
  interaction level). Leaking preference changes is a user-visible bug.
- **Group undo correctly per host**: After Effects `beginUndoGroup`/`endUndoGroup` inside a *single* host call;
  Photoshop `suspendHistory`; Illustrator and Premiere have no undo-group API.
- **Batch host work** into few calls; the host's scripting thread is single and long loops freeze its UI.
- **Sign releases with a timestamp.** An expired certificate with no timestamp stops the extension; with one it
  keeps working. Never ship `.debug`.
- **`PlayerDebugMode` must be set for the right `CSXS.N`**, and the host restarted. Set CSXS.9-14.
- **Never claim you tested inside an Adobe app.** You can't launch them here. Say what you verified
  (XML validity, JS syntax, audit, simulated host logic) and give the user a short manual test list.

## Host quick sheet

Read the file for each target host before writing host code.

| Host | ID | Read | Headline traps |
|---|---|---|---|
| Photoshop | `PHXS` + `PHSP` | `references/host-photoshop.md` | ActionManager + `DialogModes.NO`, restore `rulerUnits`, Rosetta on Apple Silicon, UXP clock |
| Illustrator | `ILST` | `references/host-illustrator.md` | y-axis up, `artboardRect` order, no events, focus freeze on doc creation |
| After Effects | `AEFT` | `references/host-aftereffects.md` | undo groups per call, 1-based collections, match names, scripting-permission pref |
| Premiere | `PPRO` | `references/host-premiere.md` | PProPanel is the reference, QE DOM unsupported, array of paths for import |

Other references: `architecture-and-manifest.md` (manifest, versions, install paths, logs),
`extendscript-bridge.md` (bridge patterns, ES3 table, events, files), `debugging-and-troubleshooting.md`
(symptom decision trees), `packaging-signing-distribution.md` (ZXPSignCmd, Exchange, release checklist),
`platform-status-2026.md` (CEP/UXP timeline and decision rules).

## Scripts (all in `scripts/`, Python 3, no dependencies)

| Script | Purpose |
|---|---|
| `scaffold_cep_panel.py` | Generate a complete panel for any mix of ILST/PHXS/AEFT/PPRO |
| `audit_cep_plugin.py` | Static audit of a folder or `.zxp` (`--json`, `--fail-on warn`) |
| `enable_debug_mode.py` | Set `PlayerDebugMode` for CSXS.9-14 (macOS/Windows; `--status`, `--disable`, `--dry-run`) |
| `dev_install.py` | Link/copy a dev build into the CEP extensions folder; `--clear-cache`, `--uninstall` |
| `bump_version.py` | Bump bundle + extension versions, optional changelog entry |
| `package_zxp.py` | Stage (excludes `.debug`, secrets, `scripts/`), sign with timestamp, verify |

`enable_debug_mode.py`, `dev_install.py` and `package_zxp.py` act on the user's machine (and need their
certificate), so give them the exact commands to run locally rather than running them in the sandbox.
`package_zxp.py` reads the certificate password from an environment variable; never ask the user to paste a
password or certificate into chat.

## Verifying your work (what you CAN check here)

```bash
xmllint --noout CSXS/manifest.xml .debug         # XML well-formed
for f in js/*.js; do node --check "$f"; done      # panel JS syntax
python <skill-dir>/scripts/audit_cep_plugin.py .  # manifest + ES3 + pitfalls (0 errors expected)
```
For non-trivial host logic, run it in a Node `vm` context with the native `JSON` deleted and mock `app`
objects. That catches logic and envelope bugs without Adobe apps. Ask the user to run the manual test plan
in `references/maintaining-existing-plugins.md` on the real host.

## Delivering

- Copy the finished extension folder to `/mnt/user-data/outputs/`, zipped (`zip -r name.zip <folder>`), and
  present it. Exclude `.debug` from anything meant for release.
- Reply briefly: what was built or changed, how to install (`enable_debug_mode.py`, `dev_install.py`, restart
  the app, Window > Extensions (Legacy)), what was verified vs not, and next steps (signing, test plan).
- For a bug fix, lead with the root cause and the diff summary, not the whole file listing.


---

<!-- references/platform-status-2026.md -->

# CEP vs UXP: platform status and decision guide

Snapshot date: **20 Oct 2026**. Dates come from Adobe's developer blog post
"Investing in the Future of Creative Cloud Extensibility" (24 Sep 2026) and the
Adobe Help Center page "Creative Cloud CEP to UXP plugin transition". Adobe
revises these schedules, so when the answer matters (a go/no-go decision, a
client commitment), re-check the two sources above with web search.

## Contents
- Official timeline
- What it means per host app
- Apple Silicon / Rosetta
- Conflicting claims you may run into
- How to tell the user (decision rules)
- Building CEP panels so migration stays cheap
- Features with no UXP equivalent
- Sources

## Official timeline

| Host | UXP status | New CEP submissions to Adobe Marketplace stop | CEP disabled by default |
|---|---|---|---|
| Photoshop | UXP plugins available since 2020/2021 | **March 2027** | **December 2027** |
| Premiere | UXP GA + Hybrid plugins available now | December 2027 | December 2028 |
| InDesign | UXP since 2023; GA Hybrid summer 2027 | January 2028 | December 2028 |
| After Effects | UXP **public beta by November 2026** | December 2028 | December 2028 |
| Illustrator | UXP **public beta by Spring 2027** | December 2028 | December 2028 |
| Media Encoder | UXP public beta live now | December 2028 | December 2028 |

Rules that apply to every flagship app:
- Each app gets a guaranteed minimum of **two years from its own UXP public beta**
  before CEP is removed from new versions of that app.
- **From December 2029, CEP is no longer included** in these apps.
- Plugins already in the Marketplace keep working and can still receive updates until then.
- Only *new* submissions are blocked at each cutoff. Plugins distributed outside the
  Marketplace (own website, other stores) are not affected by the submission cutoffs.
- "Disabled by default" means users can switch CEP back on, but it is a visible
  friction for every customer.
- **ExtendScript is not part of this transition.** `.jsx` scripts keep working.
- Acrobat, Adobe Express and Lightroom are not affected.

Premiere note: Adobe rebranded "Premiere Pro" to "Premiere" with version 26.0
(January 2026). The CEP host ID is still `PPRO`.

## What it means per host app

**Photoshop.** UXP is the mature, default choice. New CEP work is hard to justify
unless the plugin is private/internal, is distributed outside the Marketplace, or
depends on CEP-only capabilities (Node.js modules, heavy ExtendScript). Existing
CEP panels still need maintenance, and the clock is real: Marketplace cutoff
March 2027, disabled by default December 2027.

**Premiere.** UXP is GA and Hybrid plugins exist, so recommend UXP for brand-new
products. CEP remains reasonable for updating existing panels and for features that
lean on the (large) ExtendScript API, which PProPanel documents by example.

**After Effects.** UXP is only entering public beta (Nov 2026). Before that, AE's UXP
hosted Adobe's own plugins only. CEP + ExtendScript is the practical route for
third-party panels today. Plan for migration after the beta matures.

**Illustrator.** Public UXP beta is expected Spring 2027; until then UXP is not
available to third parties. CEP + ExtendScript is the practical route today. Third-party
engineering shops report that historically CEP kept loading for years after UXP
shipped in other apps.

## Apple Silicon / Rosetta

Adobe states that on Apple Silicon, **CEP in Photoshop runs through Rosetta 2**, and
Apple plans to retire Rosetta in 2027. Adobe's Help Center says: if your macOS no
longer supports Rosetta, Photoshop still runs natively but **its CEP plugins will not
work**. Only Photoshop CEP plugins need Rosetta. Support threads also show users on
Apple Silicon seeing greyed-out legacy panels until Photoshop was opened in Rosetta
mode (Get Info on the app, tick "Open using Rosetta").

When a user says "my Photoshop panel disappeared", ask for OS + Photoshop version and
check Rosetta before anything else.

## Conflicting claims you may run into

One panel vendor's support pages claim Adobe "removed support for CEP" in Photoshop
2025 and later. Adobe's own September 2026 schedule says CEP is still shipped and only
becomes disabled-by-default in December 2027. Treat the vendor claim as unverified:
the two are most easily reconciled by the Rosetta/Legacy Extensions settings above
(Photoshop > Preferences > Plugins > Load Extension Panels, a.k.a. "Legacy Extensions").
Do not assert either version as fact. Have the user check on their exact Photoshop
version and OS, and cite Adobe's page for the official position.

## How to tell the user (decision rules)

1. **Ask which host(s) and whether the plugin will be sold on Adobe Marketplace**, because
   that is what the cutoffs affect.
2. **New plugin, Photoshop:** recommend UXP. Offer CEP only if the user has a reason
   (private use, outside-Marketplace distribution, CEP-only dependency). If they still want CEP,
   build it, say clearly what the dates imply, and use the migration-friendly structure below.
3. **New plugin, Premiere:** recommend considering UXP; CEP is acceptable if the user
   wants ExtendScript depth or already has a CEP codebase.
4. **New plugin, After Effects or Illustrator:** CEP is the practical choice today.
   Mention the UXP beta dates in one sentence and use the migration-friendly structure.
5. **Existing plugin:** keep it on CEP, fix and extend it, and give a short
   migration-readiness note (see `maintaining-existing-plugins.md`). Do not push a rewrite
   unprompted. Adobe itself describes migration as a "reconstruction", not a port.
6. Keep the tone factual. Never be alarmist and never claim CEP "no longer works" in
   general; it does work today in current apps.

## Building CEP panels so migration stays cheap

- Keep **all host calls behind one thin API layer** (the scaffold's `cep-bridge.js` +
  `NS.api` functions). The UI never touches `evalScript` strings directly.
- Keep the **UI framework-agnostic and free of CEP globals** except in the bridge. Plain
  HTML/CSS/JS, or React/Vue/Svelte via a bundler, ports to UXP/webview far more easily.
- Write host logic as **pure ExtendScript functions that take and return JSON**. That logic
  is what you rewrite against the UXP DOM later; the contract stays the same.
- **Avoid Node.js** in the panel unless truly required. Node modules and native `.node`
  add-ons are the biggest migration blockers. Prefer `cep.fs` or host-side file APIs.
- Do not store critical state only in `localStorage`; use a settings JSON file with a
  schema version so it can be read by a future UXP version.
- Document which host functions use undocumented APIs (e.g. Premiere QE DOM).

## Features with no UXP equivalent

From Adobe's CEP-to-UXP technical migration guide:
- `registerExtensionUnloadCallback` and `storeProxyCredentials`: the UXP `shell` module
  does not and will not have equivalents.
- `batchPlay` is Photoshop-specific; it is not available in Premiere, InDesign or Media Encoder.
- Architectures differ (UXP has a stricter security model), so expect to adopt
  UXP-supported patterns rather than a 1:1 port. Webview and Hybrid plugins bridge the gaps.

## Sources
- Adobe Developer Blog, 24 Sep 2026: blog.developer.adobe.com/en/publish/2026/09/investing-in-the-future-of-creative-cloud-extensibility-uxp-comes-to-our-flagship-applications
- Adobe Help Center: helpx.adobe.com/creative-cloud/apps/integration-with-other-apps/manage-plugins/cep-uxp-plugin-transition.html
- CEP to UXP Technical Migration Guide: developer.adobe.com/uxp/migration-center/uxp-for-cep-devs/technical-migration-guide/
- UXP developer forum: forums.creativeclouddeveloper.com


---

<!-- references/architecture-and-manifest.md -->

# CEP architecture, manifest, versions, install paths

## Contents
- The two halves of a CEP extension
- Folder layout
- Annotated manifest
- Host IDs and version numbers
- CEP runtime versions (Chromium / Node)
- Extension types and lifecycle
- CEFCommandLine flags
- Install locations, cache, logs
- Versioning rules

## The two halves of a CEP extension

1. **Panel side (HTML/CSS/JS)** runs in an embedded Chromium (CEF). It draws the UI,
   may use Node.js if enabled, and talks to the host through `CSInterface`
   (a wrapper over `window.__adobe_cep__`).
2. **Host side (ExtendScript, `.jsx`)** runs inside the Adobe app, with access to that
   app's DOM. It is an ES3-era JavaScript engine.

The two halves are separate JavaScript engines. The panel calls `evalScript(string)`; the
host evaluates the string and returns **a string**. Everything crossing the boundary is
serialised text, so treat the boundary as an API with JSON in and JSON out.

## Folder layout

```
com.acme.mypanel/            <- folder name usually = bundle id
├── CSXS/
│   └── manifest.xml         <- required, exact folder name "CSXS"
├── .debug                   <- dev only: remote-debug ports (hidden file; do NOT ship)
├── index.html               <- MainPath
├── css/  js/  lib/
├── host/                    <- ExtendScript (.jsx)
└── icons/                   <- optional panel icons
```

## Annotated manifest

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ExtensionManifest Version="7.0"
    ExtensionBundleId="com.acme.mypanel"        <!-- unique, stable forever -->
    ExtensionBundleVersion="1.2.0"              <!-- bump on every release -->
    ExtensionBundleName="My Panel"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <ExtensionList>
    <Extension Id="com.acme.mypanel.panel" Version="1.2.0"/>   <!-- Id #1 -->
  </ExtensionList>
  <ExecutionEnvironment>
    <HostList>
      <Host Name="ILST" Version="[23.0,99.9]"/>
      <Host Name="PHXS" Version="[20.0,99.9]"/>
      <Host Name="PHSP" Version="[20.0,99.9]"/>
      <Host Name="AEFT" Version="[16.0,99.9]"/>
      <Host Name="PPRO" Version="[13.0,99.9]"/>
    </HostList>
    <LocaleList><Locale Code="All"/></LocaleList>
    <RequiredRuntimeList>
      <RequiredRuntime Name="CSXS" Version="9.0"/>
    </RequiredRuntimeList>
  </ExecutionEnvironment>
  <DispatchInfoList>
    <Extension Id="com.acme.mypanel.panel">                    <!-- Id #2: must match #1 -->
      <DispatchInfo>
        <Resources>
          <MainPath>./index.html</MainPath>
          <!-- <ScriptPath>./host/index.jsx</ScriptPath> -->
          <CEFCommandLine>
            <!-- <Parameter>--enable-nodejs</Parameter> -->
            <!-- <Parameter>--mixed-context</Parameter> -->
          </CEFCommandLine>
        </Resources>
        <Lifecycle><AutoVisible>true</AutoVisible></Lifecycle>
        <UI>
          <Type>Panel</Type>
          <Menu>My Panel</Menu>
          <Geometry>
            <Size><Height>400</Height><Width>320</Width></Size>
            <MinSize><Height>200</Height><Width>240</Width></MinSize>
            <MaxSize><Height>1200</Height><Width>800</Width></MaxSize>
          </Geometry>
        </UI>
      </DispatchInfo>
    </Extension>
  </DispatchInfoList>
</ExtensionManifest>
```

Points that cause real bugs:
- The **extension Id appears twice** (ExtensionList and DispatchInfoList) and must match,
  and it must also match the Id inside `.debug`. A mismatch = panel missing or no debugger.
- `ExtensionManifest Version="7.0"` is the schema version, not your plugin version.
- **Host filtering:** extensions whose host ID/version doesn't match the running app are
  silently ignored. There's no error dialog.
- **`RequiredRuntime`** is the minimum CEP version. Setting it higher than the CEP version
  inside the host means the panel won't load. `9.0` is a safe floor for CC 2019+ and
  works in current apps; raise it only when you need newer CEP APIs.
- **Version ranges** use inclusive brackets: `[23.0,99.9]`. Wide ranges are fine for
  local/private builds. For **Adobe Developer Distribution (Exchange)** uploads, the max
  must not exceed what Adobe considers supported for that product; community reports show
  uploads rejected with "maximum version ... is greater than the maximum supported
  version" for `PPRO`/`AUDT`/`ILST`. If rejected, lower the max to what the error accepts.
- Malformed XML or a wrong `MainPath` leaves the panel absent or blank. Validate with
  `xmllint --noout CSXS/manifest.xml`.

## Host IDs and version numbers

| App | Host ID(s) | Base version (Help Center table, spring 2026) |
|---|---|---|
| Photoshop | `PHXS` and `PHSP` (list both) | 27.5 (27.11 shipped Sep 2026) |
| Illustrator | `ILST` | 30.3 |
| After Effects | `AEFT` | 26.0 |
| Premiere (Pro) | `PPRO` | 26.0.2 (26.5.x by Sep 2026) |
| Audition | `AUDT` | – |
| InDesign / InCopy | `IDSN` / `AICY` | – |
| Bridge | `KBRG` | – |

Historical mapping (CC 2019 → CEP 9): Photoshop 20, Illustrator 23, Premiere 13, After Effects 16.

Which `CSXS.N` a given host release reads differs by release (Illustrator 2024 users
had to set `CSXS.11`; Premiere 26.3 reads `CSXS.12`; AE 2025 reports mention both 11 and
12). Rule: **set `PlayerDebugMode` for CSXS.9 through CSXS.14** (the `enable_debug_mode.py`
script does this) instead of guessing.

## CEP runtime versions

| CEP | Chromium | Node | Notes |
|---|---|---|---|
| 9 | older Chromium (see CEP 9 cookbook) | n/a | CC 2019/2020 |
| 10 | see CEP 10 cookbook | 12.3.1 (NW.js 0.38) | Chromium CSS deprecations apply |
| 11 | see CEP 11 cookbook | 15.9.0 (NW.js 0.50.1) | |
| 12 | **CEF 99 (Chromium 99.0.4844)** | **17.7.1** | Adobe announced Aug 2024 (Photoshop 25.12); macOS OpenGL→Metal |

Cells marked "see cookbook" were not verified for this skill; check the HTML Extension Cookbook for that
CEP version in github.com/Adobe-CEP/CEP-Resources before relying on a specific Chromium version.

Write panel JavaScript in **ES2017 or earlier** (no optional chaining `?.`, no `??`, no object spread) if you
must support old hosts, or transpile with Vite/esbuild. CEP 12 had a reported WebSocket
regression in Photoshop 25.12; if a user's network code broke after a host update, test
the transport first.

## Extension types and lifecycle

`<Type>` values: `Panel` (dockable), `ModalDialog` (blocks host until closed),
`Modeless` (floating dialog), `Custom` (no UI; used for background/"invisible" extensions
started by `<StartOn>` events such as `com.adobe.csxs.events.ApplicationActivate`),
and `Embedded` (a panel that cannot be undocked).

`AutoVisible=true` shows the panel when the app starts if it was open last session.
Panel JS state lives only while the panel is open; persist settings to disk.

## CEFCommandLine flags

| Flag | Effect | Warning |
|---|---|---|
| `--enable-nodejs` | `require()`, `process`, and `cep_node` available in the panel | Largest migration blocker; security surface |
| `--mixed-context` | Node and browser share one JS context | Needed by many npm libraries; changes iframe behaviour |
| `--allow-file-access` / `--allow-file-access-from-files` | Local `file://` access from page scripts | Only if you really load local files this way |

With Node enabled and separate contexts, some libraries (e.g. jQuery) load in module
context and fail to attach to `window`; `--mixed-context` fixes that.

## Install locations, cache, logs

| Scope | Windows | macOS |
|---|---|---|
| User (use for dev) | `%APPDATA%\Adobe\CEP\extensions\` | `~/Library/Application Support/Adobe/CEP/extensions/` |
| All users | `C:\Program Files\Common Files\Adobe\CEP\extensions\` (and `(x86)` variant) | `/Library/Application Support/Adobe/CEP/extensions/` |

The same extension in both scopes causes confusing "which copy loaded?" bugs. Keep one.

- **Cache (macOS):** `~/Library/Caches/CSXS/cep_cache/`. Clear it if edits to the panel don't appear.
- **Logs:** macOS `~/Library/Logs/CSXS/CEP12-<HOST>.log` (e.g. `CEP12-PPRO.log`);
  Windows `%TEMP%\CEP12-<HOST>.log`. The number matches the CEP version (CEP11, CEP12, …).
- Where the panel appears: **Window > Extensions (Legacy)** in current apps (older apps: Window > Extensions).
  Photoshop's switch for these panels: Preferences > Plugins > "Load Extension Panels".

## Versioning rules

- If two installed copies share the same bundle ID and same version, the one whose
  manifest has the latest modified date is loaded. During development, bump the version or
  make sure only one copy exists.
- **Never change `ExtensionBundleId` or `Extension Id`** on an existing product. It breaks
  upgrades, stored settings, and the Marketplace listing.
- Use semantic versions; bump `ExtensionBundleVersion` **and** each `<Extension Version>` together.
  (`scripts/bump_version.py` does both.)


---

<!-- references/extendscript-bridge.md -->

# The panel ⇄ ExtendScript bridge

## Contents
- How `evalScript` really behaves
- The envelope pattern (use it everywhere)
- Passing arguments safely
- ES3 limits and polyfills
- Shared global namespace
- Host → panel events
- Paths, files, network, settings
- Theme sync
- TypeScript / Bolt CEP
- Performance and responsiveness

## How `evalScript` really behaves

- It takes a **string of ExtendScript**, runs it in the host, and returns **a string** to a
  callback. Objects, numbers, `undefined` all arrive as strings (`"undefined"` is common).
- On any uncaught error, or a syntax error in a file, the callback receives the literal
  string **`EvalScript error.`** with no message or line number. That's why every host function must
  catch its own errors and return them.
- The call is **asynchronous for the panel** but **blocks the host's scripting thread**. A
  long loop freezes the host UI. Calls queue; ordering across separate `evalScript`
  calls is preserved but state is not transactional.
- In After Effects, undo groups reportedly do not survive across separate `evalScript` calls, so grouped work
  should be in **one** call (see `host-aftereffects.md`).
- Do not call `alert()`/`confirm()` in host scripts: it opens a modal in the host,
  can steal focus (a known Illustrator problem), and stalls the queue. Return a message to the panel instead.

## The envelope pattern (use it everywhere)

Host side (ExtendScript, ES3):
```js
var NS = (typeof NS === 'undefined') ? {} : NS;     // unique per product, see below
NS.api = NS.api || {};
NS.call = function (fnName, argsJson) {
  try {
    var args = argsJson ? JSON.parse(argsJson) : [];
    var fn = NS.api[fnName];
    if (typeof fn !== 'function') return JSON.stringify({ ok: false, error: 'Unknown host function: ' + fnName });
    return JSON.stringify({ ok: true, data: fn.apply(NS.api, args) });
  } catch (e) {
    return JSON.stringify({ ok: false, error: String(e && e.message ? e.message : e),
                            name: e && e.name, line: e && e.line, file: e && e.fileName });
  }
};
```

Panel side:
```js
function callHost(fn, args) {
  const script = 'NS.call(' + JSON.stringify(fn) + ',' + JSON.stringify(JSON.stringify(args || [])) + ')';
  return new Promise((resolve, reject) => {
    csInterface.evalScript(script, (raw) => {
      if (raw === 'EvalScript error.') return reject(new Error('Host script missing or has a syntax error'));
      let res; try { res = JSON.parse(raw); } catch (e) { return reject(new Error('Bad host reply: ' + raw)); }
      res.ok ? resolve(res.data) : reject(Object.assign(new Error(res.error), res));
    });
  });
}
```
`assets/template/js/cep-bridge.js` implements exactly this, plus host loading, theme sync and a
browser preview mode. It was exercised against a simulated `__adobe_cep__`, but not inside a real Adobe app. Start from it.

## Passing arguments safely

- Always `JSON.stringify(JSON.stringify(args))` (double stringify): the inner call builds
  JSON, the outer produces a valid JavaScript string literal with quotes and backslashes escaped.
- Never splice user text into a script with string concatenation. Names such as `O'Brien` or
  `C:\folder` break the script, and untrusted text becomes code injection.
- In the host, return only JSON-friendly values. Convert `UnitValue` → `.as('px')`,
  `Time` → `.seconds`, `File` → `.fsName`, collections → arrays you build.

## ES3 limits and polyfills

ExtendScript is ES3 plus a few extras. Whole-file syntax errors make *every* function in
that file undefined, which surfaces as `EvalScript error.`.

| Not available / unsafe | Use instead |
|---|---|
| `let`, `const`, arrow functions, template literals, classes, `async/await`, default params, spread | `var`, `function`, string concatenation |
| `JSON` (often missing) | include `host/json-lite.jsx` (or Crockford json2) before use |
| `Array.prototype.forEach/map/filter/indexOf/some/reduce` | `for` loops, or small polyfills |
| `Object.keys`, `Array.isArray` | `for (k in o) if (o.hasOwnProperty(k))`, `o instanceof Array` |
| Trailing commas in literals | remove them |
| Reserved words as property names (`o.class`, `o.default`) | `o['class']` |

Teams that want modern syntax write TypeScript/ES6 and **compile to ES3**: Bolt CEP does this
with Rollup/Babel and `types-for-adobe` typings. If the project already uses a build step,
keep it; if not, plain ES3 is fine and has fewer moving parts.

## Shared global namespace

All CEP panels in one host app share the **same ExtendScript global scope**. Two panels
that both define `function init()` overwrite each other. Always hang everything off a
unique namespace object (`AcmePanel.api.*`) and never define bare globals. Bolt CEP does the
same: it appends functions to a per-panel namespace for this reason.

## Host → panel events

From ExtendScript:
```js
NS.emit = function (type, data) {
  try {
    if (!NS._xlib) NS._xlib = new ExternalObject('lib:PlugPlugExternalObject');
    var ev = new CSXSEvent();
    ev.type = type;
    ev.data = (typeof data === 'string') ? data : JSON.stringify(data);
    ev.dispatch();
    return true;
  } catch (e) { return false; }
};
```
In the panel: `csInterface.addEventListener(type, function (evt) { /* evt.data is a string */ });`
Event names should be namespaced (`com.acme.mypanel.progress`).

ExtendScript cannot listen for panel events directly. The panel pushes data with
`evalScript`, the host pushes back with `CSXSEvent`. Application-level notifications
(document opened, selection changed) vary by host; check the host reference file. Where no
event exists, **poll sparingly** (≥ 500 ms) and only while the panel is visible.

## Paths, files, network, settings

- Get the extension root: `csInterface.getSystemPath(SystemPath.EXTENSION)`
  (value `"extension"`). Use **forward slashes** everywhere, including Windows; strip any
  `file://` prefix. Percent-decode (CSInterface does).
- User data dir for settings: `SystemPath.USER_DATA`. Store a **settings JSON file with a
  `schemaVersion`** for anything important; `localStorage` lives in the CEP cache and
  disappears if the cache is cleared.
- File I/O without Node: `window.cep.fs.readFile/writeFile/showOpenDialogEx/...` (returns
  `{err, data}`; `err === 0` is success). With Node enabled you can use `fs`, but see the
  migration warning in `platform-status-2026.md`.
- Host-side file access (`File`, `Folder`) works too. After Effects needs
  Preferences > Scripting & Expressions > **Allow Scripts to Write Files and Access Network**
  enabled (preference key `Pref_SCRIPTING_FILE_NETWORK_SECURITY=1`, edited only while AE is closed).
- Network: panel `fetch`/XHR works subject to CORS and CEF's Chromium version; Node's `http`
  is the fallback. Photoshop has a preference "Allow Extensions to Connect to the Internet".
- Open links with `cep.util.openURLInDefaultBrowser(url)`, not `window.open`.

## Theme sync

`csInterface.getHostEnvironment().appSkinInfo.panelBackgroundColor.color` returns
`{red, green, blue}` (0–255). Listen to `com.adobe.csxs.events.ThemeColorChanged`
(`CSInterface.THEME_COLOR_CHANGED_EVENT`) and re-apply. The scaffold maps these to CSS variables
and picks light/dark text by luminance, so panels match each app's theme without hard-coded colours.

## TypeScript / Bolt CEP

When the user wants a modern toolchain for a **new** panel (React/Vue/Svelte, HMR, typed bridge),
recommend **Bolt CEP** (`npx create-bolt-cep`, also `yarn`/`pnpm`). Highlights:
Vite + TypeScript + Sass, hot-module reload in the host, `evalTS("fn", args)` for end-to-end typed
calls to ExtendScript, `listenTS`/`dispatchTS` typed events, multi-panel and multi-host config in
`cep.config.ts`, and built-in ZXP packaging and GitHub Actions releases. ExtendScript is written in
`src/jsx/` (ES6 compiled to ES3). Use it if the user's machine has Node; this skill's scaffold is the
zero-dependency alternative and the only one that can be generated in an offline sandbox.

## Performance and responsiveness

- Batch work into **one** host call (loop in ExtendScript), not hundreds of `evalScript` calls.
- For long jobs, process in chunks of ~50–200 items per call and emit progress events so the
  panel can show a bar and the host can repaint between chunks.
- Cache expensive reads in the panel; avoid polling the DOM of large documents.
- Avoid `$.sleep` in panels; it freezes the host.


---

<!-- references/debugging-and-troubleshooting.md -->

# Debugging and troubleshooting CEP

## Contents
- One-time dev setup
- The three debuggers
- "Panel doesn't appear" decision tree
- "Panel is blank" decision tree
- "Buttons do nothing / EvalScript error."
- "Works for me, not for customers"
- "Broke after a host update"
- Reading the CEP log

## One-time dev setup

1. **Enable unsigned extensions** (`PlayerDebugMode`) for every CSXS version, then restart the host:
   `python scripts/enable_debug_mode.py` (macOS `defaults write com.adobe.CSXS.N PlayerDebugMode 1`;
   Windows `reg add HKCU\Software\Adobe\CSXS.N /v PlayerDebugMode /t REG_SZ /d 1 /f`).
   The value is a **string `"1"`** on Windows. On macOS, if the setting doesn't take, run
   `killall cfprefsd` (the preferences daemon caches values).
2. **Install a dev copy** by symlink/junction so edits are live: `python scripts/dev_install.py`.
3. Add a **`.debug`** file in the extension root (the scaffold writes one), with one `Host`
   entry per app and a **unique port per host**.
4. Keep **one** copy of the extension installed (user scope *or* system scope).

## The three debuggers

| Layer | Tool | How |
|---|---|---|
| Panel JS (HTML/JS) | Chromium DevTools | Open `http://localhost:<port>` (port from `.debug`) in Chrome, click the panel's link. Console shows `console.log` and errors. Use `location.reload()` in the console to reload the panel. If modern Chrome can't attach to the old CEF, use **CEFClient** from CEP-Resources. |
| Host script (ExtendScript) | VS Code + **Adobe ExtendScript Debugger** (`Adobe.extendscript-debug`) | Breakpoints in `.jsx`, attach to the running app. ExtendScript Toolkit is no longer updated. |
| Host/CEP loader | CEP log files | See the end of this file. |

A blank DevTools page usually means the `.debug` Extension Id differs from the manifest, the port
doesn't match the URL, or the file isn't in the extension root (it must be exactly `.debug`, hidden,
no extension).

## "Panel doesn't appear" decision tree

Work top to bottom; stop at the first fix that works.
1. Is it under **Window > Extensions (Legacy)**? (Photoshop: also Preferences > Plugins > Load Extension Panels.)
2. Restarted the host after changing debug mode or installing? `PlayerDebugMode` is read at launch.
3. Debug mode set under the **right `CSXS.N`**? Set 9–14 (script does this).
4. Installed in a CEP `extensions` folder, with `CSXS/manifest.xml` directly inside the extension folder?
5. `xmllint --noout CSXS/manifest.xml` passes? Extension Id identical in both places?
6. Does `HostList` contain the app's ID and a version range that includes the installed version?
   (Find the version in the app's About screen.) Out-of-range = silently hidden.
7. `RequiredRuntime` not higher than the host's CEP? Lower to `9.0`.
8. Same bundle id installed twice (user + system scope)? Remove one.
9. Stale cache? Clear `~/Library/Caches/CSXS/cep_cache/` (macOS) and restart.
10. macOS Apple Silicon + **Photoshop**: needs Rosetta (see `platform-status-2026.md`).
11. Check the CEP log (below) for manifest or signature errors.

## "Panel is blank" decision tree

1. Log says `invalidSignature` → unsigned extension without debug mode, or modified after signing. Enable
   debug mode for dev, or re-sign for release.
2. `MainPath` wrong (case-sensitive on macOS) or `index.html` missing.
3. JavaScript error on load: open DevTools and read the console. Common: using syntax newer than the
   host's Chromium (optional chaining on old CEP), `require` without `--enable-nodejs`, CDN scripts
   blocked offline (bundle assets locally).
4. Node-dependent libraries failing: add `--mixed-context` alongside `--enable-nodejs`.
5. Absolute `file://` paths or paths with spaces/Unicode: build URLs from `getSystemPath('extension')`.

## "Buttons do nothing / EvalScript error."

`EvalScript error.` means a host exception or syntax error with no detail. Narrow it down:
1. **Syntax error in the `.jsx` file**: the entire file fails to parse, so every function is
   undefined. Search for ES6+ (`let`, `const`, `=>`, backticks, trailing commas, `.forEach`).
   `python scripts/audit_cep_plugin.py <folder>` flags these.
2. **Host script never loaded**: the panel must load `json-lite.jsx`, `index.jsx` and the host file
   (the scaffold's `Bridge.loadHost()` does). Call `ping` first; if it fails, loading is the problem.
3. **Wrong namespace**: the JS calls `NS.call` but the host file defined `OtherNS`.
4. **Host state**: no document/sequence/composition open; host in a modal tool; another panel overwrote a global.
5. **Undefined API on this version**: guard with `typeof x !== 'undefined'` and return a readable error.
6. Run the same function in the VS Code ExtendScript debugger to get a real error message and line.

## "Works for me, not for customers"

| Check | Why |
|---|---|
| Is the ZXP **signed with a timestamp**? | Unsigned or invalid signature = "Extension does not run". An expired cert **without** a timestamp also stops it; expired **with** a valid timestamp still runs. |
| Did you ship `.debug`, `node_modules`, `.git`, source maps? | Bloat and an open debug port. Use `package_zxp.py` which excludes them. |
| Hard-coded paths, your username, drive letters | Use `getSystemPath` and `Folder.userData`/`Folder.temp`. |
| macOS Photoshop on Apple Silicon | Rosetta. |
| AE: scripting file/network permission | Pref off by default. |
| Non-English host | Menu/property display names localized; use match names / runtime lookup. |
| Different install scope or older host | Version range, `RequiredRuntime`. |
| Fonts / locale / decimal separators | Never parse numbers from localized UI strings. |

## "Broke after a host update"

1. Compare the new host version with `HostList`; widen the max (`[min,99.9]` for private builds).
2. Check CEP version changes: CEP 12 moved to CEF 99/Node 17.7.1 and caused a WebSocket
   regression in Photoshop 25.12; Photoshop-on-Apple-Silicon needs Rosetta.
3. Re-run the ExtendScript through the debugger; DOM behaviour (especially Premiere QE) changes.
4. Read Adobe's release notes / the CEP-Resources releases page for known issues.
5. Plan the UXP migration if the host is Photoshop (see `platform-status-2026.md`).

## Reading the CEP log

- macOS: `~/Library/Logs/CSXS/CEP12-<HOST>.log` (`PHXS`, `ILST`, `AEFT`, `PPRO`). Windows: `%TEMP%\CEP12-<HOST>.log`.
  The `12` follows the CEP version in the host (`CEP11-AEFT.log` on older AE builds).
- Search for: `invalidSignature`, `No debug port file found`, `Error occurs while getting debug port`
  (= `.debug` missing or in the wrong place), and manifest parse errors.
- Delete the log, reproduce the problem once, then read it. A fresh log is far easier to read.


---

<!-- references/maintaining-existing-plugins.md -->

# Maintaining and extending an existing CEP plugin

## Contents
- Ground rules
- Intake: what to collect
- Getting the code (folder, ZIP, ZXP)
- Triage workflow
- Making the change safely
- Adding a feature
- Manual test plan
- Migration-readiness note

## Ground rules

- Only modify plugins the user **wrote or has the right to change**. If the code belongs to a
  third party's commercial product, do not help bypass licensing, signatures or protections.
- Work on a **copy**. Never edit files under `/mnt/user-data/uploads` (read-only); copy to
  `/home/claude/work/<name>/`, edit there, deliver to `/mnt/user-data/outputs/`.
- **Smallest correct change.** Don't reformat the whole codebase, rename files, or swap frameworks
  while fixing a bug. Reviewers and users need to see exactly what changed.
- **Never change** `ExtensionBundleId` or `Extension Id` (upgrades, settings and listings depend on them).
- Keep the existing style, language level (ES3 in `.jsx`), and architecture unless the user asks otherwise.
- State what was **not** verified. Claude cannot launch Adobe apps in this environment.

## Intake: what to collect

Ask only for what's missing, in a single message:
1. Host app(s) and **exact versions** (Help > About), OS and chip (Apple Silicon / Intel / Windows).
2. **What happens vs what should happen**; exact error text; screenshots help.
3. How it is installed (ZXP via Exchange, manual folder, installer) and from where.
4. Log file contents if available (`CEP12-<HOST>.log`) and DevTools console output.
5. Whether source code is available (folder/ZIP/repo), or only the `.zxp`.
6. Scope: bug fix only, new feature, compatibility update, or all three.

## Getting the code

- **Folder / ZIP / repo**: unzip to a work dir; find `CSXS/manifest.xml` (it's the root marker).
- **ZXP**: a ZXP is a ZIP archive. `unzip plugin.zxp -d work/` yields the extension folder plus
  signature files (`META-INF/`); the original signature becomes invalid once files change, so the
  edited version must be re-signed by the owner. If JS is minified or obfuscated, **ask for the source**;
  patching minified bundles is fragile.
- Run the audit immediately: `python scripts/audit_cep_plugin.py work/<extension>`. It reports manifest
  problems, ES3 violations in `.jsx`, missing `PlayerDebugMode`-independent issues, Node usage, and
  other risks. Use its output to focus the investigation.

## Triage workflow

Classify the symptom into a layer before changing code (this prevents fixing the wrong thing):

| Layer | Typical symptom | First look |
|---|---|---|
| 1. Loading | Not in menu, blank | manifest, host range, signature, debug mode, Rosetta |
| 2. Panel JS/UI | Error in DevTools, layout broken | console, CEF version features, CSS |
| 3. Bridge | `EvalScript error.`, empty replies | host script load order, namespace, JSON handling |
| 4. Host script | Wrong result, host exception | ExtendScript logic, DOM differences by version |
| 5. Environment | Only some machines | signing/timestamp, OS, locale, permissions |

Then: reproduce (or reason from logs) → form one hypothesis → make one change → re-test → repeat.
Use `references/debugging-and-troubleshooting.md` for the layer-specific procedure.

## Making the change safely

1. Write the **root cause in one sentence** before editing.
2. Edit the minimum lines. Preserve public host function names and event names (other code, saved
   presets, or scripts may call them).
3. Add defensive checks at boundaries: document/sequence/comp exists; collections non-empty;
   JSON parse guarded; host returns the `{ok, data|error}` envelope.
4. If the fix changes **stored settings**, add a `schemaVersion` and a small migration so old
   settings files still load.
5. Bump the version (`scripts/bump_version.py --part patch`) and add a changelog line.
6. Re-run `audit_cep_plugin.py` and `node --check` on changed panel JS files.
7. Produce a **diff summary** for the user: files changed, why, and how to verify.

## Adding a feature

1. Restate the feature as: *user action → host operation → visible result*.
2. Put the host operation in `host/<HOST>.jsx` as `NS.api.<verb><Noun>` taking/returning JSON (ES3).
   If it modifies the document, wrap it for undo where the host allows (AE `beginUndoGroup` in the same
   call; Photoshop `suspendHistory`; Premiere/Illustrator have no undo-group API).
3. Expose it in the panel via `Bridge.call('<verbNoun>', [args])`, never raw `evalScript` strings.
4. Add UI (buttons/inputs) that disables itself while the call runs and shows errors from the envelope.
5. Handle empty states ("Open a document first") with a message, not an exception.
6. If the feature spans several hosts, share the panel code and branch inside the host files.
7. Update `README`/changelog, bump the **minor** version.

## Manual test plan (hand to the user; Claude can't run it)

For each target host × OS (at least *oldest supported* and *newest* host version):
1. Fresh install with debug mode **off** (signed ZXP) → panel appears in Window > Extensions (Legacy).
2. Panel opens with no console errors; theme matches (try light and dark UI themes).
3. Run the fixed/new feature with: no document open, one document, large document, and a locked/hidden
   item if relevant.
4. Run it twice in a row; undo once and confirm the expected state (where undo is supported).
5. Close and reopen the panel; settings persist.
6. Restart the host; panel still loads.
7. Update over a previous version (installer handles upgrade; settings preserved).

## Migration-readiness note (give a short version; don't push a rewrite)

Score the plugin and mention only the items that apply:
- Uses Node (`require`, `cep_node`, native `.node` modules)? → highest effort in UXP.
- Uses `cep.process.createProcess` or external binaries? → UXP has stricter process rules.
- Uses `registerExtensionUnloadCallback` / `storeProxyCredentials`? → **no UXP equivalent**.
- Heavy ExtendScript surface, esp. Photoshop ActionManager? → rewrite against `batchPlay` (Photoshop only)
  or the host's UXP DOM.
- Large, framework-based UI (React/Vue)? → ports well; the UI layer carries over.
- Premiere QE DOM usage? → verify equivalents in UXP before committing.
Cite the dates in `platform-status-2026.md` for the specific host, in one or two sentences.


---

<!-- references/packaging-signing-distribution.md -->

# Packaging, signing, installing, distributing

## Contents
- Signature rules (why this matters)
- Build a signed ZXP
- Install channels
- Adobe Developer Distribution / Exchange
- Release checklist
- Certificate renewal

## Signature rules

CEP checks the signature **every time the host loads the extension**.

| Situation | Result |
|---|---|
| No signature | Extension does not run (unless `PlayerDebugMode`) |
| Invalid signature (file changed after signing) | Does not run |
| Valid signature | Runs |
| Certificate expired, **no timestamp** | Does not run |
| Certificate expired, **valid timestamp** | Runs |

So: **always sign with a timestamp (`-tsa`).** A timestamped signature stays valid after the
certificate expires, because it proves the cert was valid when you signed.

## Build a signed ZXP

Tool: **ZXPSignCmd** (Adobe-CEP/CEP-Resources → `ZXPSignCMD/`). Use **4.1.3 or newer**: older
builds segfaulted on macOS and Windows when `-tsa` was supplied. On macOS, make the binary
executable (`chmod +x`); one user couldn't run it from the mounted DMG. Copy it out first.

```bash
# 1. Create a certificate (once; keep the .p12 and password safe and backed up)
ZXPSignCmd -selfSignedCert US NY "My Company" "My Company" "PASSWORD" MyCert.p12

# 2. Sign a clean copy of the extension folder, with a timestamp
ZXPSignCmd -sign ./staging MyPanel.zxp MyCert.p12 "PASSWORD" -tsa http://timestamp.digicert.com/

# 3. Verify and inspect (shows certificate + timestamp info)
ZXPSignCmd -verify MyPanel.zxp -certInfo
```

`scripts/package_zxp.py` does the staging (copies the folder, **excludes** `.debug`, `.git`,
`node_modules`, source maps, tests, OS junk), signs with a timestamp, and verifies, in one command.
Self-signed certificates are accepted by CEP for direct distribution. A certificate from a
commercial CA is optional and mainly adds user trust on installer screens.

If signing crashes, remove `-tsa` to confirm the TSA is the cause, then upgrade ZXPSignCmd or try
another timestamp server (`https://timestamp.geotrust.com/tsa`, `http://time.certum.pl/`).

Claude cannot sign for the user: the certificate and password are theirs. Give them the
command (or the script) and a verification step. Never ask the user to paste a certificate password into chat.

## Install channels

| Channel | When | Notes |
|---|---|---|
| **Adobe Exchange / Creative Cloud Desktop app** | Marketplace distribution | Handles most installs; Exchange builds the ZXP from your folder; CC Desktop manages updates. |
| **Third-party ZXP installers** (e.g. aescripts + aeplugins ZXP Installer) | Direct sales, free tools | Users drag the ZXP into the app. |
| **UPIA** (Unified Plugin Installer Agent, command line) | Enterprise/offline deployments | Lives under Adobe Desktop Common's RemoteComponents/UPI folder; verify the path on the target machine. Used as `UnifiedPluginInstallerAgent --install file.zxp`. |
| **Manual copy** | Dev and quick support | Copy the folder into the CEP `extensions` directory (see `architecture-and-manifest.md`); needs debug mode if unsigned. |

If the package carries files beyond the standard HTML extension (extra assets, installation
instructions), Adobe's packaging guide says to provide a `.mxi` configuration file.

## Adobe Developer Distribution / Exchange

- **New CEP submissions**: Photoshop stops March 2027; Premiere December 2027; InDesign January
  2028; After Effects, Illustrator and Media Encoder December 2028. **Updates to already-listed CEP
  plugins stay possible.** Outside-Marketplace distribution is unaffected. (See `platform-status-2026.md`.)
- **Host version max**: uploads can be rejected with "The maximum version you mentioned is greater
  than the maximum supported version for product X". Lower the max in `HostList` until accepted
  (community reports say upload failures also happen with ranges ending 26.9 for PPRO/AUDT).
- Reviews check: per-app install instructions, no altered Adobe product icons or trademarks, a
  unique descriptive name, and screenshots. Adobe assets used without authorization cause rejection.
- Read Adobe's current "Prepare a plugin for Adobe Exchange review" checklist before submitting; requirements change.

## Release checklist

1. Bump version (`scripts/bump_version.py --part patch|minor|major`) → updates `ExtensionBundleVersion`
   and every `<Extension Version>`.
2. Run `scripts/audit_cep_plugin.py` → fix all ERRORs.
3. Smoke-test on each target host (see the manual test plan in `maintaining-existing-plugins.md`).
4. `scripts/package_zxp.py` → signed, timestamped ZXP; verify output shows a timestamp.
5. Install the ZXP on a **clean machine/user account with debug mode OFF** and confirm the panel runs.
6. Update changelog and release notes; keep the previous ZXP for rollback.

## Certificate renewal

Repackage and re-release **well before** the certificate expires (timestamped signatures survive, but
you still want fresh certs for new builds). Keep the `.p12` and password in a password manager;
losing them means future updates carry a different signer.


---

<!-- references/host-premiere.md -->

# Premiere (PPRO)

Adobe renamed Premiere Pro to **Premiere** with 26.0 (January 2026). The CEP host ID is
still `PPRO`. UXP is **GA** with Hybrid plugins, so recommend UXP for new products. CEP remains
fine for maintaining existing panels. Adobe's schedule: new CEP Marketplace submissions stop
December 2027; CEP disabled-by-default December 2028; removed from December 2029.

## Contents
- Manifest and loading
- The reference sample: PProPanel
- Supported API vs QE DOM
- Time, sequences, clips, markers
- Import, export, encode
- Events
- Known problems
- Docs and tools

## Manifest and loading

- Host ID `PPRO`; current base 26.0.2 (26.5.x by September 2026); CC 2019 was 13.
- Premiere 26.3 reads `CSXS.12` (reported). Set `PlayerDebugMode` for CSXS.9–14.
- If the panel is listed but opens blank, check `~/Library/Logs/CSXS/CEP12-PPRO.log`
  (macOS) for `invalidSignature` and confirm `defaults read com.adobe.CSXS.12 PlayerDebugMode`
  returns 1; restart Premiere after changing it.
- Shared panels with Audition (`AUDT`) hit the Developer Distribution version-range
  error; see `architecture-and-manifest.md`.

## The reference sample: PProPanel

Adobe's **PProPanel** (Adobe-CEP/Samples/PProPanel, file `jsx/PPRO/Premiere.jsx`) is the
canonical, deliberately exhaustive example. Adobe staff and long-time developers send everyone
there first. It shows bins, import, sequences, insert/overwrite edits, clip and sequence
markers, proxies/ingest, metadata, exporting, and event binding. When asked "how do I do X in
Premiere with CEP", **search PProPanel for X before writing code**, and mirror its calling
pattern. Its UI is plain by design; reuse the host code, not the look.

## Supported API vs QE DOM

- The documented ExtendScript API is large and keeps growing (project, sequences, tracks,
  clips, markers, metadata, encoder, application settings).
- The **QE DOM** (`app.enableQE()` then `qe.*`) is **undocumented and unsupported** (Adobe
  community manager's words). Calls are guessed, parameters undocumented, and
  behaviour changes between releases. Use it only when no supported route exists, isolate it in one
  file, wrap each call in try/catch, and tell the user the feature may break after updates.
  Example: for proxies, PProPanel's `ingestFiles()` is the supported route, not `createProxy`.
- No API exists to add custom fields to markers.

## Time, sequences, clips, markers

```js
var seq = app.project ? app.project.activeSequence : null;
if (!seq) throw new Error('Open a sequence first');

// Time objects: use .seconds (float) or .ticks (string). 254016000000 ticks per second.
var playhead = seq.getPlayerPosition();          // Time
var sec = playhead.seconds;

// Sequence markers
var m = seq.markers.createMarker(sec);           // seconds
m.name = 'My Panel marker';
m.comments = 'Created by panel';

// Tracks and clips
var v1 = seq.videoTracks[0];                     // 0-based here
for (var i = 0; i < v1.clips.numItems; i++) { var clip = v1.clips[i]; /* clip.name, clip.start.seconds */ }

// Insert/overwrite a project item at a time
v1.insertClip(projectItem, sec);                 // ripples
v1.overwriteClip(projectItem, sec);              // replaces
```
- Collections are exposed with `.numItems`/`.numTracks`, not always `.length`; check the
  reference before looping.
- Clip markers belong to the *project item* (`projectItem.getMarkers()`); sequence markers
  belong to the sequence.
- Sequence presets cannot be built from script; PProPanel's workaround is to ship a project
  containing template sequences and clone them.

## Import, export, encode

```js
// Import (takes an ARRAY of path strings)
app.project.importFiles([ '/path/clip.mov' ], true /*suppressUI*/, app.project.getInsertionBin(), false);

// Export current sequence directly or via Media Encoder
seq.exportAsMediaDirect(outPath, presetPath, app.encoder.ENCODE_WORKAREA);
// app.encoder.encodeSequence(seq, outPath, presetPath, app.encoder.ENCODE_ENTIRE, 1);
```
Verify argument order against PProPanel for your Premiere version before shipping.
Presets (`.epr`) are located relative to the app install; resolve at runtime.

## Events

`app.bind('onProjectChanged', fn)` and related `onSourceClipSelectedInProjectPanel`
style events exist for project and selection changes. The handler is a host function: have it
call `NS.emit('com.acme.mypanel.projectChanged', {...})` so the panel is notified.
`app.setSDKEventMessage('text', 'info' | 'warning' | 'error')` writes to the Events panel and is the
supported way to surface host-side messages without modal dialogs.

## Known problems

| Symptom | Cause / fix |
|---|---|
| Panel lists but is blank | Signature invalid / debug mode under the wrong CSXS number; read `CEP12-PPRO.log`. |
| `qe` is undefined | Forgot `app.enableQE()`, or QE changed in this release. |
| Cursor disappears on Windows after closing panel | Historic CEP bug (CEP-1769); update the host. |
| `importFiles` ignores paths | Passed a `File` object or string instead of an **array of path strings**. |
| Sequence creation fails | Presets can't be created from script; use template sequences. |
| Upload rejected on Developer Distribution | Host max version above the supported max; lower it. |

## Docs and tools

- Premiere ExtendScript API docs: ppro-scripting.docsforadobe.dev (community-hosted) and the
  Premiere section of Adobe's developer site.
- VS Code + **Adobe ExtendScript Debugger** (ExtendScript Toolkit is no longer updated).
- Adobe-CEP/Samples/PProPanel; CEP-Resources `CEP_HTML_Test_Extension` for CEP API coverage.
- UXP for Premiere: developer.adobe.com/premiere-pro/uxp/


---

<!-- references/host-photoshop.md -->

# Photoshop (PHXS / PHSP)

Read `platform-status-2026.md` first: Photoshop is the host where CEP is closest to its end
(Marketplace cutoff March 2027, disabled by default December 2027). For new work recommend UXP;
this file is for building and maintaining the CEP + ExtendScript version.

## Contents
- Manifest and loading
- DOM vs ActionManager
- Patterns that avoid classic bugs
- Events
- Known environment problems
- Docs and tools

## Manifest and loading

- List **both** `<Host Name="PHXS" .../>` and `<Host Name="PHSP" .../>`.
- Panel menu: **Window > Extensions (Legacy)**. User switch: Preferences > Plugins >
  **Load Extension Panels** (and **Allow Extensions to Connect to the Internet** for network).
- Photoshop 25.12 upgraded to CEP 12 (CEF 99, Node 17.7.1), per Adobe's August 2024 announcement.

## DOM vs ActionManager

Photoshop's scripting DOM is incomplete. Real plugins combine two layers:

1. **DOM** (`app.activeDocument`, `layers`, `artLayers`, `selection`, etc.): readable and stable. Use it first.
2. **ActionManager** (`executeAction`, `ActionDescriptor`, `ActionReference`,
   `charIDToTypeID`, `stringIDToTypeID`): reaches nearly everything the UI can do. Capture
   code with the **ScriptListener** plugin, then clean it up (use `stringIDToTypeID` names
   or a tidy `cTID/sTID` helper, drop the dead `idnull` variables, parameterise values).
   Raw ScriptListener output works as-is, and a recorded descriptor maps closely to what UXP's
   `batchPlay` takes later, so keep it isolated in small functions.

Always pass **`DialogModes.NO`** to `executeAction`/`executeActionGet` in panels; a dialog
opened from a panel call blocks the host.

## Patterns that avoid classic bugs

```js
// 1. Never assume a document exists.
if (app.documents.length === 0) throw new Error('Open a document first');

// 2. Units: set what you need, ALWAYS restore (try/finally is valid ES3).
var oldRuler = app.preferences.rulerUnits, oldType = app.preferences.typeUnits;
app.preferences.rulerUnits = Units.PIXELS;
try { /* work */ } finally {
  app.preferences.rulerUnits = oldRuler; app.preferences.typeUnits = oldType;
}

// 3. Read sizes unit-independently.
var wPx = doc.width.as('px');               // UnitValue → number, ignores ruler units
var b = layer.bounds;                        // [L,T,R,B] as UnitValues
var layerW = b[2].as('px') - b[0].as('px');

// 4. One history state for a multi-step operation.
//    suspendHistory takes the function as a STRING evaluated in global scope,
//    so it must be reachable through the namespace.
doc.suspendHistory('My Panel: Apply', 'NS.internal.applyAll()');

// 5. Suppress dialogs for the duration of a batch, then restore.
var oldDialogs = app.displayDialogs; app.displayDialogs = DialogModes.NO;
try { /* work */ } finally { app.displayDialogs = oldDialogs; }
```

Other points:
- `app.activeDocument.activeLayer` can be a `LayerSet` (group); check `typeof layer.kind`
  or `layer.typename` before calling `ArtLayer`-only members.
- Photoshop layer indexes (`doc.layers[i]`) are top-to-bottom for the visible list, but
  ActionManager `itemIndex` references are bottom-to-top and 1-based with a background layer
  offset. When mixing the two, resolve by **layer ID** (`layerID` property) instead.
- Changing the selection or active layer from a panel while the user has a modal tool active
  (type edit mode, transform) can fail. Return a clear error ("finish the current edit")
  rather than retrying in a loop.
- Heavy per-pixel work in ExtendScript is slow; prefer built-in filters/actions, or export a
  smaller proxy.

## Events

Photoshop exposes document/layer/tool events to CEP through a registration handshake
demonstrated in Adobe's **"Photoshop Events"** CEP sample (Adobe-CEP/Samples). Copy its
registration code rather than writing it from memory, and unregister on panel close to avoid
duplicate callbacks after the panel is reopened.

## Known environment problems

| Symptom | Cause / fix |
|---|---|
| Panels greyed out / missing on Apple Silicon | Photoshop CEP runs through Rosetta 2. Enable Rosetta for the app (Get Info → Open using Rosetta), then re-enable Load Extension Panels. If macOS no longer supports Rosetta, CEP panels will not work. |
| WebSocket failures starting at PS 25.12 | CEP 12 / CEF 99 transport change. Test with another WebSocket library or HTTP fallback. |
| Panel loads in dev, not on customers' machines | Unsigned ZXP or expired certificate with no timestamp. See `packaging-signing-distribution.md`. |
| Works on PS 2024, absent on newest | Host version range in manifest; widen `[min,99.9]`. |

## Docs and tools

- Photoshop JavaScript Scripting Reference and Guide (Adobe); community mirror: theiviaxx.github.io/photoshop-docs.
- ScriptListener plugin (Adobe Photoshop scripting downloads) for ActionManager code capture.
- VS Code + **Adobe ExtendScript Debugger** extension (`Adobe.extendscript-debug`) for stepping through `.jsx`.
- CEP samples: github.com/Adobe-CEP/Samples (includes Photoshop Events and a hello-world panel).


---

<!-- references/host-illustrator.md -->

# Illustrator (ILST)

Status: Illustrator has **no public UXP yet** (public beta expected Spring 2027). CEP +
ExtendScript is the practical route for third-party panels today. Adobe's schedule keeps CEP
loading until at least December 2028 (disabled-by-default) and removes it from December 2029.

## Contents
- Manifest and loading
- Coordinates (the #1 source of "shape is in the wrong place" bugs)
- Safe patterns
- Events and polling
- Known problems
- Docs and tools

## Manifest and loading

- Host ID `ILST`. Current base version 30.3 (Help Center table); CC 2019 was 23.
- Panel menu: Window > Extensions (Legacy).
- Illustrator 2024 users needed `PlayerDebugMode` under `CSXS.11`; set CSXS.9–14 to be safe.
- Historically a modal CEP panel on Windows sent keyboard events to the host; avoid
  `ModalDialog` unless required and test shortcuts.

## Coordinates

- **Default document coordinates:** origin at the artboard-0 *bottom-left*, **y increases
  upward**. `artboard.artboardRect` is `[left, top, right, bottom]` with `top > bottom`.
  `pathItems.rectangle(top, left, width, height)` takes the **top-left** corner in that same space.
- **Artboard coordinates:** set `app.coordinateSystem = CoordinateSystem.ARTBOARDCOORDINATESYSTEM`
  to get origin top-left of the *active artboard*, y downward. This is easier for UI-driven
  tools, but it is a global app setting: **restore it afterwards** or you will break the
  user's other scripts.
- Units: ExtendScript values are in **points**. Convert (`mm * 2.834645669`) explicitly; do
  not depend on the document's ruler units.
- Always verify placement with a real document in the test plan. Coordinate bugs are silent.

## Safe patterns

```js
// Guard: scripts error when no document is open.
if (app.documents.length === 0) throw new Error('Open a document first');
var doc = app.activeDocument;

// Selection may be null, an array of items, or (during text edit) a text range.
var sel = doc.selection;
var items = (sel && typeof sel === 'object' && sel.length !== undefined) ? sel : [];

// Silence alerts for a batch, then restore.
var oldUI = app.userInteractionLevel;
app.userInteractionLevel = UserInteractionLevel.DONTDISPLAYALERTS;
try { /* work */ } finally { app.userInteractionLevel = oldUI; }

// Repaint after big edits so the user sees progress.
app.redraw();

// Menu commands exist for things the DOM lacks (fit to window, outline, etc.).
app.executeMenuCommand('fitin');
```

- Prefer `doc.pageItems`/`doc.selection` + `typename` checks over assuming `PathItem`;
  groups, compound paths, symbols and text frames are different classes.
- `layers.getByName(name)` **throws** if missing; wrap in try/catch or scan `layers`.
- Illustrator's scripting has **no undo-group API**; a long script becomes many undo steps. Keep
  panel actions small and tell users, or perform edits on a duplicated layer.
- Export: `doc.exportFile(File, ExportType.PNG24, new ExportOptionsPNG24())` and similar;
  save: `doc.saveAs(File, new IllustratorSaveOptions())`.

## Events and polling

ExtendScript in Illustrator has no document/selection events. Choices, in order of preference:
1. **User-triggered refresh** (a button, or refresh on panel focus).
2. **Throttled polling** every ≥ 500–1000 ms, only while the panel is visible. Stop on
   `com.adobe.csxs.events.WindowVisibilityChanged` (verify the event on your CEP version).
3. Ask the user to consider the Illustrator C++ SDK or future UXP if truly real-time sync is required.

## Known problems

| Symptom | Cause / fix |
|---|---|
| Creating a document from the panel hangs until the user clicks Illustrator | Focus issue when a script creates a document. Split into two `evalScript` calls: create, then (in the callback) operate; return a flag from the first. |
| Script errors on startup scripts / "no active document" | No document exists at launch; guard with `app.documents.length`. |
| Shapes land far from expected position | Coordinate system mix-up (see above). |
| Panel missing after Illustrator update | `HostList` max version, or `PlayerDebugMode` under a new `CSXS.N`. |

## Docs and tools

- Illustrator Scripting Guide + JavaScript Reference (Adobe does not bundle them with the app;
  download the PDFs from Adobe, or use the community docs at ai-scripting.docsforadobe.dev; verify the URL).
- VS Code + Adobe ExtendScript Debugger for `.jsx`.
- Adobe-CEP/Samples for Illustrator panel samples.


---

<!-- references/host-aftereffects.md -->

# After Effects (AEFT)

Status: UXP reaches **public beta by November 2026**; before that, AE's UXP hosted Adobe's
own plugins only. CEP + ExtendScript is the practical route today. Adobe's schedule: new CEP
submissions stop and CEP becomes disabled-by-default in December 2028; removed from December 2029.

## Contents
- Manifest and loading
- Undo groups (the classic CEP trap)
- DOM conventions that cause bugs
- Permissions
- Events and background work
- Rendering and automation
- Known problems
- Docs and tools

## Manifest and loading

- Host ID `AEFT`. Current base version 26.0; CC 2019 was 16.
- Panel menu: Window > Extensions (Legacy).
- AE 2025 reports exist for both `CSXS.11` and `CSXS.12`; set `PlayerDebugMode` for CSXS.9–14.
- `PlayerDebugMode` is read **at launch**; restart AE after changing it.
- ScriptUI panels (`.jsx` in `Scripts/ScriptUI Panels`) are a separate, simpler mechanism.
  Recommend them when the user only needs a small native-looking panel with no web UI.

## Undo groups

`app.beginUndoGroup()`/`app.endUndoGroup()` only group work done inside **one**
`evalScript` call. Community reports show that begin/work/end issued as three separate
calls produces no undo entry; one experienced developer's explanation is that a group left open when a
script call finishes is closed automatically (treat that mechanism as reported, not documented). Put the whole operation in one host function:

```js
NS.api.addNullDemo = function () {
  var comp = app.project.activeItem;
  if (!(comp instanceof CompItem)) throw new Error('Select a composition first');
  app.beginUndoGroup('My Panel: Add null');
  try {
    var n = comp.layers.addNull();
    n.name = 'Demo null';
  } finally {
    app.endUndoGroup();                 // always close, even on error
  }
  return n.name;
};
```
If the panel drives a loop (one layer per call), instead send the whole list to the host
and loop there, or queue the operations in the panel and run them in one call.

## DOM conventions that cause bugs

- **Collections are 1-based**: `comp.layers[1]`, `app.project.item(1)`, `comp.layer(1)`.
  Off-by-one errors are the most common AE scripting bug.
- `app.project.activeItem` can be `null`, a `FootageItem`, or a `FolderItem`; test with
  `instanceof CompItem`.
- Time is in **seconds**; use `comp.frameRate` and `comp.frameDuration` to convert to frames.
- Reference properties with **match names** (`'ADBE Transform Group'`, `'ADBE Position'`) not
  display names, because display names are localized and break on non-English AE.
- Use `app.findMenuCommandId('Exact Menu Text')` then `app.executeCommand(id)` for
  features with no DOM access; menu text is also localized, so resolve at runtime and
  fail gracefully.
- Layer/comp names are not unique. Track items by `.id` (project items) or index+comp.
- Text: replace via `textProp.setValue(new TextDocument('...'))` (fetch the existing
  `TextDocument`, modify it, set it back to keep styling).

## Permissions

Writing files or using the network from ExtendScript needs
**Preferences > Scripting & Expressions > Allow Scripts to Write Files and Access Network**.
If it is off, `File.open('w')`, `Socket`, and `system.callSystem` fail. Check at runtime
(`app.preferences.getPrefAsLong('Main Pref Section', 'Pref_SCRIPTING_FILE_NETWORK_SECURITY') === 1`)
and tell the user which switch to flip. Editing the prefs file by hand is only safe with AE closed,
as AE overwrites it on exit.

## Events and background work

- ExtendScript has no document events in AE. Poll lightly (e.g. active item id) or add a refresh button.
- `app.scheduleTask('NS.api.tick()', delayMs, repeat)` runs host code later, but it
  executes on the host thread: keep it short.
- Avoid `$.sleep`; it freezes the UI.

## Rendering and automation

- Render queue: `app.project.renderQueue.items.add(comp)`; templates by name
  (`outputModule(1).applyTemplate('...')`); set `file` on the output module.
- Headless/batch rendering: the **`aerender`** command-line tool in the AE install
  (`Support Files/aerender`). Spawn it from the panel with Node (`child_process`) or host
  `system.callSystem`. Note that Node means migration friction.
- MOGRT/Essential Graphics: expose properties via the Essential Graphics panel and drive
  them through `comp.motionGraphicsTemplateName` and exported `.mogrt`.

## Known problems

| Symptom | Cause / fix |
|---|---|
| Undo menu shows nothing / many steps | Undo group split across `evalScript` calls. |
| `Can't write file` / `Socket` errors | Scripting permission pref is off. |
| Works in English AE only | Display names or menu strings used instead of match names. |
| Panel blank after AE update | Check `CEP12-AEFT.log`; unsigned/expired signature, or `PlayerDebugMode` under a new CSXS number. |
| Slow panel | Many small `evalScript` calls; batch them. |

## Docs and tools

- After Effects Scripting Guide: ae-scripting.docsforadobe.dev (community-maintained; verify URL).
- VS Code + Adobe ExtendScript Debugger.
- Adobe-CEP/Samples → `AfterEffectsPanel` (shows invoking AE ExtendScript from a CEP panel).


---

# Appendix: bundled scripts and template files

These files are in the original .skill archive and are not inlined here:

- `assets/template/css/styles.css`
- `assets/template/host/hosts/AEFT.jsx`
- `assets/template/host/hosts/ILST.jsx`
- `assets/template/host/hosts/PHXS.jsx`
- `assets/template/host/hosts/PPRO.jsx`
- `assets/template/host/index.jsx`
- `assets/template/host/json-lite.jsx`
- `assets/template/index.html`
- `assets/template/js/cep-bridge.js`
- `assets/template/js/main.js`
- `scripts/audit_cep_plugin.py`
- `scripts/bump_version.py`
- `scripts/dev_install.py`
- `scripts/enable_debug_mode.py`
- `scripts/package_zxp.py`
- `scripts/scaffold_cep_panel.py`
