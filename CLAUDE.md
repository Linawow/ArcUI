# ArcUI — Claude Code Instructions

ArcUI is a comprehensive World of Warcraft addon suite for WoW 12.0 (Midnight), currently v3.7.x.
It covers cooldown management, buff/debuff tracking bars, resource bars, CDM (CooldownViewer)
integration, custom icons/timers, and proc tracking. Related addon: ArcUI_ProcTracker (v1.0.5+).

> **Shared playbook:** the global **`wow-addon-dev`** skill is the cross-addon layer for all Arc
> addons — the CurseForge release pipeline, WoW 12.0 secret/taint rules, dev workflow, reference
> sources, and the changelog template (`changelog-template.md`). Reference it for release/packaging
> and general addon-dev tasks; this file is ArcUI's project-specific layer on top of it.

Target client: WoW 12.0.5 (Midnight), Interface 120005 — current live build 12.0.5.67823.
(The .toc supports 120000/120001/120005. Verify the live build against the API mirror per the
build-check protocol in Reference Sources before doing API work.)

---

## ABSOLUTE RULES — never violate these

- **NEVER use pcall anywhere in ArcUI.** No exceptions. It has all been removed; do not reintroduce it.
- **ALL new features and options MUST default to false/disabled.** Users opt-in, never opt-out.
- **Surgical fixes only.** Fix exactly what was asked. No "while I'm here" restructuring, no broad
  migrations, no refactors without explicit approval.
- **Run `luac -p` on every modified Lua file before declaring a task done.** Run it yourself; do not
  ask the user to.
- **Zero-idle-CPU philosophy:** event-driven > polling, throttled > per-frame, single watcher >
  per-icon hooks. Avoid OnUpdate when events exist. ALWAYS ask before adding any constant polling.
- **Never re-copy files from old uploads or backups over the working tree** unless explicitly told
  "use these new files." Always work from the live project files.
- **Option labels must never truncate.** Every AceConfig control (toggle, select, input, color,
  range, execute) MUST have a `width` large enough that its full `name` is visible in the panel —
  truncated labels like "Glow Col..." are BANNED. When in doubt, widen (a color/short toggle
  usually needs ≥0.8; longer names ≥1.0–1.5). Check new options render cleanly, not just compile.

---

## WoW 12.0 SECRET VALUES (critical — most bugs trace back to violating these)

Secret values are tainted runtime values addons cannot inspect.

**Cannot do with secrets:**
- Compare them (`<`, `>`, `==` against numbers), do arithmetic, `tonumber()`, or store in
  SavedVariables (they get replaced with nil on save).
- Passing a secret into an API call marks the receiving object secret.

**Can do:**
- `if x then` works as a nil check on a secret VALUE (number/string/table/userdata) — BUT a
  secret BOOLEAN throws on a boolean test (`if secretBool then` errors "attempt to perform boolean
  test on a secret boolean value"). And don't rely on `if` for truthiness of secret numbers either
  (`a or b` / `and` truth-test a secret the same way). Detect-don't-test: derive state from a
  non-secret signal. Concrete: `GetTotemInfo`'s `haveTotem` is a secret boolean — NEVER `if
  haveTotem`. Instead use `GetTotemDuration(slot)` (returns nothing when empty, a durObj when
  active — a normal nil-testable userdata) fed into a Cooldown, then read `Cooldown:IsShown()`.
- `issecretvalue(x)` to test. `HasSecretAspect()` on objects; `SetToDefaults()` clears all secret
  state on a frame. `IsAnchoringSecret()` to test anchor taint (secret anchors propagate DOWN the
  anchor chain, not up).
- Comparing different types yields non-secret false; comparing nil secrets yields non-secret true.
- `/dump` prints secret contents (debugging only).

**Duration objects ARE secret:**
- `IsZero()`, `GetSpan()`, `GetCooldownDuration()` ALL return secrets. Never extract numbers.
- Pass the durObj directly to `Cooldown:SetCooldownFromDurationObject(durObj)`.
- APIs always return a durObj (zero-span when inactive), never nil.
- Use `C_UnitAuras.GetAuraDurationRemaining(unit, auraInstanceID)` and `SetTimerDuration`.

**BANNED with secrets (12.0.1):** `SetCooldown`, `SetCooldownFromExpirationTime`,
`SetCooldownDuration`, `SetCooldownUNIX`, `ActionButton_ApplyCooldown`.

**Secret-accepting APIs (safe sinks):** `SetText`, `SetValue`, `SetAlpha`, `SetVertexColor`,
`SetTexture` (secret numbers only, not strings), `C_StringUtil.*`, `AbbreviateNumbers`,
`Cooldown:SetCooldownFromDurationObject`.

**Known NON-secret values:**
- `GetUnitAuras`/`UNIT_AURA` vectors themselves (contents are secret); `auraInstanceID` is non-secret.
- Player spellcasts (including pets), even in combat — `UNIT_SPELLCAST_SUCCEEDED` on "player" is safe.
- `UnitHealthMax`/`UnitPowerMax` for player units; `UnitStagger` for player. `UnitPowerMax` returns
  non-secret 0 if the unit doesn't use that power type.
- `UnitIsUnit` for target/focus/mouseover/softenemy/softinteract/softfriend tokens (compound tokens
  like boss1target remain secret).
- `maxCharges` (non-secret as of 12.0.1); `isEnabled` and `isActive` from spell/action cooldown APIs.
- `isOnGCD` (true/false/nil) from `GetSpellState()` is NON-secret. NEVER compare
  `GetSpellCooldown.duration` — it is SECRET.
- NEVER compare `currentCharges` — SECRET.
- Unit names/GUIDs/IDs are secret in instances (instance-based, not combat-based).

**Display helpers:** Curve objects for secret-compatible color transitions;
`C_StringUtil.TruncateWhenZero` suppresses secret zeros in display text.

**Charge bars:** hook `SetCachedChargeValues(count, shown)` on `CooldownViewerCooldownItemMixin` —
count is non-secret for CDM spells. `SPELL_UPDATE_COOLDOWN` + `isOnGCD` misses casts within the
charge GCD; catch final-charge spends with `UNIT_SPELLCAST_SUCCEEDED` on "player".

**Scheduling that needs real numbers:** only use NON-secret sources — `GetTime()` plus
locally-tracked cast times, isActive/isEnabled flags.

---

## TAINT RULES (12.0) — how addon code silently breaks Blizzard's CDM

Taint is an execution credential. Addon code always runs tainted; Blizzard code runs secure
until it READS any value an addon wrote — from that read onward the rest of that execution is
"tainted by 'ArcUI'", protected APIs hand BLIZZARD'S OWN CODE secret values, and its unguarded
comparisons throw. A tainted Blizzard run can also STORE a secret into Blizzard state (e.g.
`previousCooldownChargesCount`), after which every later refresh of that frame errors until
reload. Errors abort CDM's refresh loop mid-iteration → icons vanish.

**NEVER:**
- Write any non-`_arc` field on a Blizzard frame (`frame.spellOutOfRange = x` caused the 3.6.9
  vanishing-icons regression — CDM read the tainted flag in RefreshData, then died comparing
  secret charges in CacheChargeValues/SetCachedChargeValues).
- Insert/replace entries in Blizzard-owned tables, or replace Blizzard methods (`frame.X = fn`).
- Call Blizzard refresh entry points directly from addon code (`viewer:RefreshData()`,
  `item:RefreshIconColor()`) — the run executes tainted and can poison stored state. (Known
  pre-existing instance: CDMEnhance.lua ~3271 calls `viewerFrame:RefreshData()` on a timer —
  predates 3.6.8 and hasn't manifested, but it's an audit candidate, not a pattern to copy.)

**SAFE:**
- `hooksecurefunc` (global or object method) — does not taint the original.
- READING Blizzard fields (reading never taints Blizzard; only their reads of OUR writes do).
- Safe-sink widget methods (SetText/SetAlpha/SetVertexColor/SetTexture/Show/Hide).
- Writing anything on ArcUI-created frames, and `_arc*`-prefixed fields on Blizzard frames —
  Blizzard never reads `_arc*` keys, so taint can't propagate from them.

**Why taint bugs look intermittent / class-specific:** the tainted write is usually conditional
(stale-state mismatch), secrets only materialize under combat/instance restrictions, the error
depends on frame iteration order (taint starts mid-loop at the read), and the trigger population
is config-dependent (e.g. `rangeCheckSpellID` only exists on ranged-spell cooldown frames — why
hunters hit the 3.6.9 bug and self-buff-heavy melee never did). "Works on my character" proves
nothing for taint bugs.

---

## Architecture

- **Namespace:** `local ADDON, ns = ...`. `ns.API` (Core.lua), `ns.Display` / `ns.Resources` /
  `ns.Catalog` / `ns.CDMEnhance` (modules), `ns.db` (AceDB).
- **CDM scanning:** `ScanAllCDMIcons()` scans the 4 viewers → `cdmIconCache` →
  `OnCDMScanComplete()`.
- **Settings flow:** DEFAULT → global → perIcon. `GetIconSettings()` returns the merged result;
  `GetOrCreateIconSettings()` returns the sparse entry. `InvalidateCache()` is REQUIRED after
  changes.
- **CooldownState module is ONLY for cooldowns.** Auras are handled by
  `OptimizedApplyIconVisuals` (event-driven). `ApplyCooldownStateVisuals` skips auras.
- **Event-driven hooks** (SetIsActive, aura gained/lost): frame state is already correct at hook
  time — check frame properties directly, no redundant API queries.
- **Mouse/tooltips:** `SetScript("OnEnter", fn)` and `RegisterForDrag("LeftButton")` re-enable the
  mouse; set tooltips BEFORE click-through; force tooltips off when click-through is on.
- **Libraries:** AceDB, AceConfig, AceConfigDialog, LibSharedMedia, LibCustomGlow (Arc's fork:
  ArcGlow-1.0), LibDeflate, AceSerializer. CDM = Blizzard's native CooldownViewer; read CDM source
  in docs/ directly when diagnosing CDM interactions.

---

## Bug analysis methodology (mandatory)

- ALWAYS read every relevant file line by line, from line 1 to the end, before touching code —
  regardless of file length. Bugs hide in unrelated-seeming lines.
- Grep-only diagnosis consistently misses root causes. Grep may supplement full reads, never
  replace them.
- When a fix is proposed, trace its interaction with every code path that touches the same frames,
  settings keys, or events before shipping.
- When a regression is reported, diff against the last known-good git commit first.

---

## Changelogs & output conventions

- **Assembling the content (recollection THEN diff) — do this first:** the changelog content is
  DERIVED, never pre-canned. Two passes: (1) **Recollection** — draft from what you/the
  `MEMORY.md` feature notes know changed this version (gives the framing + why each change helps
  the player). (2) **Diff for the full picture** — run `git diff HEAD` (ArcUI commits ONLY on
  release, so the working tree vs the last "Release X.Y.Z" commit IS the complete change set since
  that release) and reconcile: add anything recollection missed, DROP pure-internal/refactor/
  restructure changes, translate felt-but-internal changes into a player benefit (e.g. the
  aura-engine rework → "Bar Performance"), keep only user-facing items. Recollection = framing;
  diff = completeness. Never write a release changelog from recollection alone. Only AFTER this do
  you format/destinations/approval below.
- **Changelog format (CurseForge):** `## Section` header, then `- **Title** — Description` all on
  one line. NO blank lines between entries within a section; contiguous bullets under the header.
  No newline between the bold title and the description. Matches the 3.6.4 format exactly.
- **Changelog content:** plain English, user-facing only. No technical jargon, no implementation
  details.
- **Changelog file:** ArcUI now uses a single canonical **`CHANGELOG.md`** that the auto-packager
  reads (`.pkgmeta` `manual-changelog`) — add each release as a `## X.Y.Z` section on top, with
  `###` New Features / Improvements / Bug Fixes under it. (The old per-version
  `CHANGELOG_vX_X_X_V1.md` files are legacy.) See the **Releasing** section below.
- **Approval gate:** before any CurseForge release, show the user the finished changelog and get
  their explicit OK before uploading. Never release a changelog unseen. (Shared rule — see the
  `wow-addon-dev` skill's `changelog-template.md`.)
- **In-game changelog window (ArcUI-specific) — update it EVERY release:** ArcUI ships a "What's
  New" popup, `ns.Changelog` in `ArcUI_Changelog.lua`. The changelog has THREE destinations that
  all carry the SAME approved notes — CurseForge, Wago, and this in-game window. So every release,
  after the changelog is approved, add a new entry at the TOP of `CL.versions` (newest first)
  mirroring it. Convert the CurseForge/Wago markdown into the Lua structure: section → color
  (`New Features`→`C_NEW`, `Improvements`→`C_IMP`, `Bug Fixes`→`C_FIX`), and each
  `- **Title** — Description` line → `{ title = "Title", desc = "Description" }` (split the bold
  title and the description into the two fields; drop the `**` and ` — `). Same plain-English,
  user-facing tone; same approval gate. Don't write it from scratch — translate the one approved
  changelog into all three. See memory `changelog-module` and `changelog-template.md` §C.
- **Deliverables:** only Lua/TOC code changes plus a brief summary. No extra documentation files,
  no change-summary documents, unless explicitly requested.

---

## Releasing (CurseForge auto-pipeline)

ArcUI auto-releases via **GitHub Actions + the BigWigs packager** (`.github/workflows/release.yml`) —
same pattern as ArcUI_ProcTracker; the global `wow-addon-dev` skill has the full reference. Set up
2026-06-18, currently **uncommitted in the working tree** with the 3.7.2 restructure; it **goes live
with the first tag (3.7.2).**

**Releasing X.Y.Z:**
1. Bump `## Version: X.Y.Z` in `ArcUI.toc`.
2. Add a `## X.Y.Z` section (newest on top) to **`CHANGELOG.md`** — the single canonical file the
   packager reads (`##` = version, `###` = New Features / Improvements / Bug Fixes). Assemble it
   recollection-THEN-diff per the changelog conventions above.
3. `luac -p` every touched Lua file.
4. **Show the user the changelog and get explicit approval** (the gate above).
5. Mirror the SAME approved notes into the in-game "What's New" (`ArcUI_Changelog.lua` `CL.versions`,
   newest first). Wago is now automated — the packager uploads the file + this changelog there too.
6. Commit, then tag with **NO "v" prefix** and push:
   `git tag -a X.Y.Z -m "Release X.Y.Z"` → `git push origin HEAD` → `git push origin X.Y.Z`.
   The tag triggers the Action: builds per `.pkgmeta`, uploads to CurseForge as **`ArcUI-X.Y.Z`**
   (project `1391614`, read from the toc's `X-Curse-Project-ID`), and creates the GitHub release.
   Do NOT `gh release create` or upload to CurseForge by hand.

**Don't break:** toc `## X-Curse-Project-ID: 1391614`; repo secret `CF_API_KEY`; `.pkgmeta`
`manual-changelog: CHANGELOG.md`; the `-n ":{package-name}-{project-version}"` label in the workflow.
On CurseForge the project's **Automatic Packaging must be OFF** (Actions is the only packager).
**Wago auto-upload IS wired** — repo secret `WAGO_API_TOKEN` + the toc's `## X-Wago-ID:mNw7Q2No`.

---

## Workflow

- One bug or feature per session. Commit after each fix is confirmed working in-game.
- Before any risky change, ensure the current state is committed so it can be rolled back.
- After edits: `luac -p` every touched file, then summarize briefly what changed and why.
- Communication style: be direct and concise. No lengthy preambles or restated requirements.
- When a new WoW patch drops: `git -C E:\WoWDev\wow-ui-source pull` (BigWigs `live` branch), verify
  its build now matches `.build.info` per the Reference Sources build-check protocol, and re-mirror
  the relevant API_changes wiki page into `E:\WoWDev\reference\wiki\`. All reference material goes
  in `E:\WoWDev\`, never in the addon folder.

---

## Reference Sources

Reference material lives OUTSIDE this addon folder — never save reference files, mirrors, or
clones inside the ArcUI directory.

- **E:\WoWDev\wow-ui-source** — local clone of **BigWigsMods/WoWUI, `live` branch** (NOT
  Gethe/wow-ui-source — Gethe's `beta` froze at 12.0.1.66220 when 12.0.5 went live, which is what
  caused stale-API mistakes). BigWigs commits every retail build within hours, so it tracks the
  live client. Ground truth for actual API/CDM behavior; grep and read it directly. NOTE the
  layout: files live under `AddOns\` with NO `Interface\` prefix. CDM (CooldownViewer) source:
  `AddOns\Blizzard_CooldownViewer`. API signatures and secret-value annotations:
  `AddOns\Blizzard_APIDocumentationGenerated`.
- **Build-check protocol (run before trusting the clone for API work):** the clone's commit
  message IS its build number. Compare it to the live client:
  - clone build: `git -C E:\WoWDev\wow-ui-source log -1 --format=%s` → e.g. `12.0.5.67823`
  - client build: the `12.0.x.xxxxx` token on line 2 of `E:\World of Warcraft\.build.info`
  - if the clone is behind, `git -C E:\WoWDev\wow-ui-source pull`. Branch must match the client's
    track: **`live`** for current retail (default); `ptr`/`ptr2` for an upcoming PTR (e.g. a
    12.0.7 line); **`beta` is frozen/dead — never use it.**
- **townlong-yak** (https://www.townlong-yak.com/framexml/) — fastest mirror, updates within
  minutes of a build dropping. Browse `/framexml/builds` for the newest build list and per-build
  files. Use it to grab a single file at the bleeding edge, diff two builds, or read a PTR line
  the local clone isn't tracking — i.e. whenever even BigWigs lags or you need a build the clone
  doesn't have. When the clone and the live client disagree, the client wins; townlong-yak is how
  you see what the client actually has.
- **warcraft.wiki.gg** — community WoW API wiki. Fetch for event payloads, API signatures, and
  per-patch change lists (e.g. https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes). If the
  wiki and the Blizzard source disagree, trust the source and say so.
- **E:\WoWDev\reference\wiki\** — locally mirrored wiki pages. Check there before fetching the
  live wiki.

---

## Architecture / File Map

Load order is defined by `ArcUI.toc` (libs → core → options → CDM module → Arc Auras → CDM
options). Everything shares one private namespace: `local ADDON, ns = ...`.

**Folder layout (3.7.2 restructure):** only `ArcUI_DB.lua`, `ArcUI_Core.lua`, `ArcUI_Minimap.lua`
and `ArcUI_Options.lua` remain at the addon root. The rest of the non-CDM code moved into
subfolders: `Bars\` (display/resources/catalog/cooldown bars/custom tracking + their options
panels), `Presets\` (presets + talent picker), `Utilities\` (data repair + the not-loaded
legacy/orphan files), `Cooldown_Reminder\`, and `Import_Export\`. The CDM side is unchanged:
`CDM_Module\` with `CDM_Enhance\`, `CDM_Groups\`, `Arc_Auras\`. Paths below reflect this.

### How the modules connect (big picture)

- **Bars side (root + `Bars\`):** `ArcUI_DB` defines schema/defaults and config accessors on
  `ns.API`. `ArcUI_Options` creates the AceDB (`ns.db`), registers the master options tree, and
  calls each module's `Init()` at PLAYER_LOGIN. `ArcUI_Core` hooks CDM frames for aura events and
  drives bar updates through `ns.Display` (aura/stack bars), `ns.Resources` (resource bars), and
  `ns.CooldownBars` (cooldown/charge/timer bars + the player spell catalog). `ns.CustomTracking`
  is the deterministic cast-event tracking engine feeding the same bars. `ns.Catalog` discovers
  CDM auras and creates/removes bars. `ns.BarGroupAlign` (BGA) does pixel-snap math for bars
  anchored to CDM group containers.
- **CDM side (CDM_Module):** `ns.CDMShared` is the hub (constants, DB helpers, throttles, panel
  callbacks). `ns.CDMGroups` owns groups/free icons/per-spec profiles; `ns.FrameController` is the
  single authority for frame lifecycle/assignment, using `ns.FrameRegistry` for lookup and
  `ns.FrameActive` for active-state detection. `ns.CDMEnhance` styles every CDM icon and exposes
  the merged-settings flow (`GetIconSettings`/`InvalidateCache`); feature files (CooldownState,
  AuraFrames, Glows, Keybinds, CustomLabel, SpellUsability, TextColor, Masque, ACH, BPH) all hang
  off it. `ns.ArcAuras` (+Cooldown/+Timer) are custom item/spell/timer icons that register into
  CDMGroups via `ns.CDMGroups.Integration`.
- **Visuals dispatch split (critical):** `ns.CooldownState` owns COOLDOWN visuals via hidden
  shadow Cooldown frames (single-pass `Apply()` merges alpha/swipe/glow/text color);
  `ns.AuraFrames.UpdateAuraFrame` (a.k.a. `OptimizedApplyIconVisuals`) owns AURA visuals,
  event-driven off `ns.FrameActive`. All actual glow rendering goes through `ns.Glows`.
- **Positioning:** `ns.CDMGroupsAnchors` (group↔group/external-frame anchoring),
  `ns.CDMContainerSync` (ArcUI containers ↔ Blizzard CDM viewers / Edit Mode),
  `ns.EditModeContainers` (LibEQOL drag wrappers).
- **Import/export:** `ns.UnifiedIE` auto-detects string type and dispatches to
  `ns.BarsImportExport` / `ns.CDMImportExport` / `ns.CDMMasterExport` / `ns.CRImportExport`.
  `ns.CDMSharedProfiles` syncs CDM profiles across same-class characters by reference.

### Root (addon root — load order)

- **ArcUI_DB.lua** — DB schema (`ns.DB_DEFAULTS`), threshold presets, and `ns.API` config
  accessors: `GetBarConfig/GetResourceBarConfig/GetCooldownBarConfig`, `GetActive*Bars` (cached;
  `InvalidateActiveBarCache`), `InitializeNew*Bar`.
- **ArcUI_Core.lua** — event-driven aura engine: hooks CDM frame methods
  (`OnAuraInstanceInfoSet/Cleared`, `OnUnitAuraUpdatedEvent`, `RefreshData`, totems) to drive bar
  updates; owns `ScanAllCDMIcons()`/`GetAllCDMIcons()` (`ns.API`), alternate-cooldown-ID mapping,
  hide-CDM-icon-when-tracked-by-bar logic, and `ns.Sounds` LSM registration. **Primary target of
  the bar aura-engine rework** (see the `cdm-aura-bars` skill).
- **ArcUI_Minimap.lua** — LibDBIcon minimap button; opens options via `ns.API.OpenOptions()`.
- **ArcUI_Options.lua** — creates the AceDB, registers the top-level AceConfig tree ("ArcUI"),
  aggregates every panel's `GetOptionsTable()`, calls module `Init()`s at login, fires panel
  open/close callbacks into CDM modules. Exports `ns.API.OpenOptions`. Slash: `/arcui`,
  `/arcbars`, `/ab`, `/aui`.

### Bars\ — bars engine & options

- **Bars\ArcUI_BarGroupAlign.lua** — `ns.BarGroupAlign` pixel-snap/measure utilities
  (`GetMatchedDimension`, `GetIconInset*`, `ApplySizeAndAnchor`) for bars anchored to
  `ns.CDMGroups.groups[*].container`. Used by Display/Resources/CooldownBars.
- **Bars\ArcUI_Display.lua** — `ns.Display`: renders all buff/debuff/stack/duration bars (textures,
  text, ticks, curves, animations). Key: `UpdateBar`, `UpdateDurationBar`, `ApplyAppearance`,
  `ApplyAllBars`, `RefreshAllBars`, `HideBar`, preview mode. Hooks
  `ns.CDMGroups.UpdateGroupVisibility` for visibility sync.
- **Bars\ArcUI_Resources.lua** — `ns.Resources`: primary/secondary resource bars (thresholds,
  smoothing, segmented/fragmented modes, spell-cost forecasting, per-spec auto-power profiles).
  Heavy event surface (UNIT_POWER_FREQUENT, RUNE_POWER_UPDATE, shapeshift, etc.). Key:
  `UpdateBar`, `ApplyAppearance`, `UpdateAllBars`, `RefreshVisibility`, `PowerTypes`/
  `SecondaryTypes` metadata.
- **Bars\ArcUI_Catalog.lua** — `ns.Catalog`: discovers CDM auras (frame scan + DataProvider out of
  combat), bar create/remove from catalog entries (`CreateArcUIDisplay`/`RemoveArcUIDisplay`),
  reload-required tracking. Rescans on spec/talent change.
- **Bars\ArcUI_CooldownBars.lua** — `ns.CooldownBars`: player spellbook catalog (`ScanPlayerSpells`,
  `spellCatalog` — also consumed by Cooldown Reminder and options panels) plus CRUD/runtime for
  cooldown, charge, resource and timer bars. Slash: `/cdbar`, `/stackbar`.
- **Bars\ArcUI_CustomTracking.lua** — `ns.CustomTracking`: deterministic custom aura/cooldown
  engine driven by `UNIT_SPELLCAST_SUCCEEDED` (stacks, decay, modifiers, talent/spec conditions);
  definitions edited via Bars\ArcUI_CustomOptions.lua.
- **Bars\ArcUI_TrackingOptions.lua** — `ns.TrackingOptions`: Aura Bars setup tab
  (`GetBuffDebuffSetupTable`) and Resources setup tab (`GetResourceSetupTable`);
  `AreTalentConditionsMet()`.
- **Bars\ArcUI_AppearanceOptions.lua** — `ns.AppearanceOptions`: unified appearance panel for all
  bar types (color/fill/size/text/border/ticks); integrates `ns.Presets`; applies via
  `ns.Display.ApplyAppearance` / `ns.Resources.ApplyAppearance`.
- **Bars\ArcUI_CooldownBarOptions.lua** — `ns.CooldownBarOptions`: options tab for cooldown/charge
  bars (reads `ns.CooldownBars` state and presets).
- **Bars\ArcUI_TimerBarOptions.lua** — `ns.TimerBarOptions`: "Custom Bars" tab for timer bars
  (triggers, generators/spenders, duration) — backed by `ns.CooldownBars.activeTimers`.
- **Bars\ArcUI_CustomOptions.lua** — `ns.CustomOptions`: UI for `ns.CustomTracking` custom aura/
  cooldown definitions.

### Presets\ — presets & talent picker

- **Presets\ArcUI_TalentPicker.lua** — `ns.TalentPicker`: talent-tree picker UI and runtime
  `CheckTalentConditions()` used by bars, resources, Arc Auras, and options for conditional
  visibility.
- **Presets\ArcUI_Presets.lua** — `ns.Presets`: appearance skin/profile system (snapshot/apply/
  copy/paste/save/load, category filtering, bar-type compatibility); library in
  `ns.db.global.skinLibrary`.

### Utilities\ — helpers

- **Utilities\ArcUI_DataRepair.lua** — `ns.DataRepair`: SavedVariables compaction (strip defaults
  on logout, restore on login), ghost-bar/corruption cleanup, character purge. Slash: `/arcrepair`.
  (Other files in `Utilities\` are present but NOT loaded — see that section below.)

### Cooldown_Reminder\

- **Cooldown_Reminder\ArcUI_CooldownReminder.lua** — `ns.CooldownReminder` + `.Engine`:
  shadow-Cooldown-widget detection of spell/item cooldown-ready transitions; queued/stacked pulse
  animations, sounds, TTS, per-trigger glows. Slash: `/arcuicr`.
- **Cooldown_Reminder\ArcUI_CooldownReminder_Options.lua** — `ns.GetCooldownReminderOptionsTable()`:
  catalog browser, per-spell trigger editor (type/sound/TTS/animation/glow/priority).
- **Cooldown_Reminder\ArcUI_CR_ImportExport.lua** — `ns.CRImportExport`: ARCUI_CR string export/
  import for CR settings + whitelist; also bundled into ARCMASTER exports.

### Import_Export\

- **Import_Export\ArcUI_Bars_ImportExport.lua** — `ns.BarsImportExport`: export/import of all bar
  configs (aura/cooldown/resource/timer) with add/replace modes.
- **Import_Export\ArcUI_UnifiedImportExport.lua** — `ns.UnifiedIE`: single import window;
  auto-detects string type (bars/CDM/Master/CR) and routes to the right importer;
  character-migration tools.

### CDM_Module — shared infrastructure

- **ArcUI_CDM_Shared.lua** — `ns.CDMShared` hub: viewer/category constants, DB helpers
  (`GetCDMGroupsDB`, `GetSpecIconSettings`, profile access), styling master toggle, event
  throttling, options-panel open/close callback registry.
- **ArcUI_FrameActive.lua** — `ns.FrameActive`: O(1) active-state tracking for CDM icon frames
  (shown/bound/aura-owned) with `OnChanged`/`OnAuraInstanceChanged` callbacks; adaptive
  UNIT_AURA registration. Standalone (no module deps).
- **ArcUI_CDMSetup.lua** — `ns.CDMSetup`: requirements checker/alerts (WoW version, CDM enabled,
  viewer settings, Edit Mode layout type, ElvUI/MasqueBlizzBars conflicts) + one-click fixes.
- **ArcUI_CDMContainerSync.lua** — `ns.CDMContainerSync`: bidirectional position/size sync
  between ArcUI group containers and Blizzard CDM viewers (push via SetPoint hooks, pull from
  Edit Mode), persisted via LibEditModeOverride.
- **ArcUI_EditModeContainers.lua** — `ns.EditModeContainers`: LibEQOL drag-overlay wrappers for
  all groups; "Drag Groups" toggle; Edit Mode integration.
- **ArcUI_CDM_ImportExport.lua** — `ns.CDMImportExport`: per-spec CDM export/import (group
  layouts, icon positions, iconSettings, global defaults, layout profiles, Arc Auras).
- **ArcUI_CDM_MasterExport.lua** — `ns.CDMMasterExport`: ARCMASTER global export/import across
  all characters/specs (scans raw `ArcUIDB.char`); pending-import queue for alts; cherry-pick
  selector used by UnifiedIE.
- **ArcUI_CDM_SharedProfiles.lua** — `ns.CDMSharedProfiles`: same-class profile sync via
  lightweight references into source characters' SavedVariables (Push/Pull/CheckAndSync,
  detach/purge).

### CDM_Module/CDM_Enhance — icon styling & state visuals

- **ArcUI_CDMEnhance.lua** — `ns.CDMEnhance`, the styling core: `EnhanceFrame`/`ApplyIconStyle`,
  merged settings (`GetIconSettings`, `GetOrCreateIconSettings`, `InvalidateCache`,
  `GetEffectiveIconSettingsForFrame`), ready/proc glow orchestration, cooldown curves,
  `GetEnhancedFrames`, mouse/tooltip state. Hooks CDM mixin methods; delegates to the files below.
- **ArcUI_CooldownFormatter.lua** — `ns.CooldownFormatter` (CF): 12.0.5 Cooldown text threshold
  APIs (milliseconds/abbrev) with feature probing.
- **ArcUI_GCDFilter.lua** — `GCDFilter.Install`: hooks visual Cooldown SetCooldown to strip GCD
  swipe, re-pushing real durObjs; defers to CooldownState when IAO is active.
- **ArcUI_Glows.lua** — `ns.Glows`: THE glow renderer for everything (LCG pixel/autocast/button/
  proc, Blizzard ants/ach_proc, CDM flash). Keyed Start/Stop per frame, Masque shape matching,
  forced alpha, resize handling, CDM VisualAlerts patch.
- **ArcUI_AuraFrames.lua** — `ns.AuraFrames` (AF): aura-frame visuals — aura-active glow,
  alpha/desaturate state visuals, threshold glow tickers, `UpdateAuraFrame` (the
  event-driven aura path), post-RefreshLayout sweep. Driven by `ns.FrameActive` callbacks.
- **ArcUI_CooldownState.lua** — `ns.CooldownState`: cooldown-only state detection via dual
  hidden shadow Cooldown frames; `FeedShadow` → relay → single-pass `Apply()` merging alpha,
  swipe, glow, text color, usability. (See "CooldownState is ONLY for cooldowns" rule above.)
- **ArcUI_Keybinds.lua** — `ns.Keybinds`: keybind text overlays on CDM + Arc Auras icons; caches
  bindings from standard/addon action bars; debounced refresh on binding events.
- **ArcUI_CustomLabel.lua** — `ns.CustomLabel`: up to 3 custom text overlays per icon with
  state-based visibility and curve alpha; `UpdateVisibility` called from CooldownState relay.
- **ArcUI_Masque.lua** — `ns.Masque`: Masque group registration for CDM/free/custom-group
  icons; conflict handling. Slash: `/arcmasque`.
- **ArcUI_CDMSpellUsability.lua + ArcUI_SpellUsabilityOptions.lua** — `ns.CDMSpellUsability`:
  usability vertex-color tinting + usable glow; caches CDM's RefreshIconColor/SetVertexColor
  state instead of polling; merges usability alpha into CooldownState's readyAlpha (single-writer
  alpha). NOTE: despite the name, SpellUsabilityOptions.lua contains the RUNTIME module; the
  per-icon options UI lives in CDMEnhanceOptions.
- **ArcUI_AssistedCombatHighlight.lua** — `ns.AssistedCombatHighlight`: Blizzard Assisted Combat
  "next cast" highlight on CDM/Arc Auras frames (ants or proc style); settings in
  `ArcUIDB.char[*].achSettings`.
- **ArcUI_ButtonPressHighlight.lua** — `ns.ButtonPressHighlight`: press-feedback overlay (hold/
  flash modes) via UseAction/CastSpellByID/CastSpellByName hooks; settings in `bphSettings`.
- **ArcUI_CDMTextColor.lua** — `ns.CDMTextColor`: cooldown countdown text coloring via
  ColorCurves; captures durObjs from SetCooldownFromDurationObject hook; 0.5s ticker only while
  frames are active.
- **ArcUI_CDMEnhanceOptions.lua** — the master per-icon/global-defaults options builder
  (`ns.GetCDMAuraIconsOptionsTable`, `ns.GetCDMCooldownIconsOptionsTable`,
  `ns.GetCDMGlobalAura/CooldownDefaultsOptionsTable`, `ns.GetCDMUtilitiesOptionsTable`);
  owns selection/edit-all/multi-select state (`ns.CDMEnhanceOptions`) and `ns.OptionsHelpers`
  (settings setters with edit-all expansion) used by other options modules.
- **ArcUI_CustomLabelOptions.lua** — `ns.CustomLabelOptions.GetAura/CooldownArgs()` injected
  into CDMEnhanceOptions; also exports `ns.AssistedCombatHighlightOptions`.

### CDM_Module/CDM_Groups — group/frame management

- **ArcUI_CDMGroups_Registry.lua** — `ns.FrameRegistry`: unified frame registry/lookup by
  cooldownID across viewers, groups, free icons, Arc Auras (`FindByCooldownID`, `Register`,
  `GetValidFrameForCooldownID`).
- **ArcUI_FrameController.lua** — `ns.FrameController`: SINGLE AUTHORITY for frame lifecycle —
  detects CDM rebuilds, scans viewers, assigns frames to groups/free (`Reconcile`,
  `AssignFrameToOwner`), installs anti-reclamation hooks, 2Hz visual maintainer, `OnFrameRebind`
  callback registry.
- **ArcUI_CDMGroups_Maintain.lua** — hook callbacks/installers (`HookFrame*`) that re-assert
  frame properties when CDM fights back; `SetupFreeIconFrame`, `FindFrameInViewers`.
- **ArcUI_CDMGroups_Layout.lua** — icon sizing/slot math, grid alignment, tooltip/click-through
  application (`SetupFrameInContainer`, `RefreshAllGroupLayouts`, `CalculateSlotPosition`).
- **ArcUI_CDMGroups_Placeholders.lua** — `ns.CDMGroups.Placeholders`: draggable placeholder
  frames for inactive cooldownIDs while options panel is open; slot badges and cooldown picker.
- **ArcUI_CDMGroups_StateManager.lua** — `ns.CDMGroups.StateManager`: protection windows during
  spec/talent change and profile load (`IsRestoring`, `IsInAnyProtection`) that block
  save/reflow.
- **ArcUI_CDMGroups_ImportRestore.lua** — `ns.CDMGroups.ImportRestore`: post-import placement —
  known icons restored, unknown placed as free grid; persists across reload, auto-expires.
- **ArcUI_CDMGroupsAnchors.lua** — `ns.CDMGroupsAnchors`: group→group/external-frame anchoring,
  mouse-follow, taint-safe re-anchoring on combat end/vehicle/Edit Mode; frame picker.
- **ArcUI_CDMGroups.lua** — `ns.CDMGroups`, the group core: `groups`/`freeIcons`/
  `savedPositions`, container pool, create/delete groups, drag mode, per-spec profiles
  (`LoadProfile`, `SaveCurrentToProfile`), visibility conditions (combat/mounted/group/etc. via
  `UpdateGroupVisibility` — hooked by bars for visibility sync), spec-change handling.
- **ArcUI_CDMGroups_DynamicLayout.lua** — `ns.CDMGroups.DynamicLayout`: reflow mode (fill gaps
  when auras hide) and dynamic positioning (auras flow around cooldown "walls"); dirty-group
  flagging.
- **ArcUI_CDMGroups_Integration.lua** — `ns.CDMGroups.Integration`: external-frame registration
  API used by Arc Auras (`RegisterExternalFrame`, `AssignToGroup`, `SavePosition`).
- **ArcUI_CDMGroupsOptions.lua** — `ns.GetCDMGroupsOptionsTable()`: group management panel
  (create/rename/delete, layout, appearance, strata, visibility conditions, anchoring UI).

### CDM_Module/Arc_Auras — custom item/spell/timer icons

- **ArcUI_ArcAuras.lua** — `ns.ArcAuras` core: frame creation/lifecycle/registry for tracked
  items/trinkets/spells/timers, CDMGroups positioning integration, settings cache, arcID scheme
  (`MakeTrinketID/MakeItemID/MakeSpellID`). Direct `ArcUIDB` access (bypasses AceDB). Slash:
  `/arc`.
- **ArcUI_ArcAurasCooldown.lua** — `ns.ArcAurasCooldown`: event-driven spell cooldown engine
  (shadow Cooldown frames + SPELL_UPDATE_* events) driving swipes, desaturation, usability tint,
  ready/proc glows; `spellData`/`spellsByID` consumed by ACH/BPH/Keybinds.
- **ArcUI_ArcAurasTimer.lua** — `ns.ArcAurasTimer`: custom timer engine (cast/cooldown/proc
  triggers, stack modes refresh/independent/consume), persists active state across reloads.
- **ArcUI_ArcAurasTotems.lua** — `ns.ArcAurasTotems`: per-totem-slot icons (frame type `"totem"`,
  arcID `arc_totem_<slot>`). COPY approach: `GetTotemDuration(slot)` durObj → cooldown →
  `IsShown()` = active (secret-safe, combat+instances, zero taint). Active maps to `readyState`
  (reuses its glow suite), empty to `cooldownState`. NOT in `ArcAurasCooldown.spellData`
  (nil-spellID would crash spell loops); self-drives off `PLAYER_TOTEM_UPDATE`. Enable UI in the
  Arc Auras Main panel; auto-creates a centered "Totems" group. See memory `totem-slots-feature`.
- **ArcUI_CustomIcons_Presets.lua** — `ns.CustomIconsPresets`: pure-data preset timer library +
  install helpers (`ns.AddTimerFromPreset`).
- **ArcUI_CustomIcons_Options.lua** — `ns.GetCustomIconsOptionsTable()`: Custom Icons tab
  (timer creation form, preset picker, per-timer editor).
- **ArcUI_ArcAuras_Options.lua** — `ns.ArcAurasOptions`: main Arc Auras panel — unified catalog
  (trinkets/items/spells/timers), selection state shared with Custom Icons tab.

### Files present but NOT loaded (not in ArcUI.toc)

- **Utilities\ArcUI_TimerBars.lua** — old standalone timer-bar system, superseded by timer bars
  inside Bars\ArcUI_CooldownBars.lua. NOTE: `ns.TimerBars` still EXISTS at runtime —
  CooldownBars.lua (~line 9389) installs it as a compat shim aliasing `ns.CooldownBars` timer
  functions, and Bars_ImportExport + AppearanceOptions call through it.
- **Utilities\ArcUI_ResourceOptions.lua** (`ns.ResourceOptions`) — orphaned resource options
  panel; the live UI is Bars\ArcUI_TrackingOptions (setup) + Bars\ArcUI_AppearanceOptions
  (appearance).
- **Utilities\ArcUI_CustomTextures.lua** — LSM registration for ~140 community bar textures.
- **Cooldown_Reminder\ArcUI_CRDebugger.lua** — dev tool: `/crdebug` cooldown state-transition
  tracer for the Cooldown Reminder engine.
- **ArcUI_Profiler.lua** — deleted in working tree (was a profiling tool; LibPleebug fills this
  role via `ns.lpmsg`).
