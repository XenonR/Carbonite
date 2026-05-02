# AGENTS.md

Guidance for AI agents working in this Carbonite addon folder.

## Project Overview

This is a World of Warcraft addon named `Carbonite`. It is not a typical build-first software project: the runtime is the WoW client, file load order is controlled by `.toc` and XML include files, and most verification happens in-game after `/reload`.

The local working folder is a deployed addon directory:

`Interface\AddOns\Carbonite`

Treat it as live addon code. Avoid broad rewrites, generated-file churn, or asset deletion unless the user explicitly asks for it.

## High-Level Structure

- `Carbonite.lua` - main Ace3 addon object (`Nx`), version flags, saved-variable defaults, slash commands, startup/lifecycle, global data migration, TomTom compatibility, gathering support, and assorted core systems.
- `NxMap.lua` - largest module; Carbonite map UI, minimap integration, world map hooks, coordinate conversion, icons, POIs, routing, targets, overlays, and edit mode.
- `NxUI.lua` - Carbonite UI framework helpers: windows, menus, lists, controls, skinning, toolbars, tooltips, and utility functions.
- `NxOptions.lua` - AceConfig options UI and command handlers for settings changes.
- `NxCom.lua` - addon communication, player sharing, channels, friend/guild/party tracking, and communication debug UI.
- `NxHUD.lua` - waypoint arrow/HUD window.
- `NxTravel.lua` - taxi/flight/transport route logic and travel-time data.
- `Carbonite.xml` - frame templates and root frames such as `NxFrame`, minimap button, slider template, quest detail scroll frame, and watch-list item template.
- `Taintless.xml` - bundled taint mitigation script; keep it isolated unless explicitly updating that upstream file.
- `Bindings.xml` - keybinding declarations.
- `Data\` - expansion-specific map, zone, transport, hotspot, and guide data.
- `Locales\` - AceLocale strings for all supported locales.
- `Libs\` - embedded Ace3 and support libraries loaded through `Libs\libs.xml`.
- `Gfx\` and `Snd\` - addon assets. Do not recompress, rename, or regenerate assets casually.
- `Docs\` - legacy user-facing documentation and license text.
- `Integrations\Sliverdragon.lua` - integration-looking code, but it is not currently included by any root TOC/XML load path. Add it to load files before depending on it.

## Load Order

The `.toc` files are the source of truth for runtime loading. Keep them synchronized when adding required code files.

Current root TOCs:

- `Carbonite.toc` - Retail, loads `Data\retail\...`
- `Carbonite-Classic.toc` - Classic Era, loads `Data\classic\...`
- `Carbonite-Tbc.toc` - TBC Classic, loads `Data\tbc\...`
- `Carbonite-Mists.toc` - MoP Classic, loads `Data\mop\...`

Each TOC follows this shape:

1. `Libs\libs.xml`
2. `Locales\Locales.xml`
3. `Taintless.xml`
4. `Carbonite.lua`
5. `NxUI.lua`
6. `NxOptions.lua`
7. `NxCom.lua`
8. `NxHUD.lua`
9. expansion `Data\<flavor>\data.xml`
10. expansion `Data\<flavor>\NxMapData.lua`
11. `NxMap.lua`
12. `Data\Shared\NxMapGuide.lua`
13. `NxTravel.lua`
14. `Carbonite.xml`

Important load-order implications:

- `Carbonite.lua` creates the global `Nx` table and submodule tables such as `Nx.Map`, `Nx.Com`, `Nx.HUD`, `Nx.Travel`, and `Nx.Opts`.
- Later modules extend those existing tables with methods. Do not make a module local-only if other files expect `Nx.<Module>`.
- Locale files load before `Carbonite.lua`, so `LibStub("AceLocale-3.0"):GetLocale("Carbonite")` is expected to work in core files.
- Expansion data loads before `NxMap.lua`; map code assumes data tables already exist.
- `Carbonite.xml` is last and creates the frame that calls `Nx:NXOnLoad(self)`.

## Runtime Architecture

Carbonite uses Ace3, but much of the code is legacy global-table style.

- Main addon object:
  - `Nx = LibStub("AceAddon-3.0"):NewAddon("Carbonite", "AceConsole-3.0", "AceTimer-3.0", "AceEvent-3.0", "AceComm-3.0")`
- Main database:
  - `Nx.db = LibStub("AceDB-3.0"):New("CarbData", defaults, true)`
  - Saved variable: `CarbData`
  - Some TOCs also declare per-character `CarbMigr`.
- Main lifecycle:
  - `Nx:OnInitialize()` sets versions, `Nx.db`, Ace callbacks, config, and module comms.
  - `Nx:NXOnLoad(frm)` is called by `Carbonite.xml`, registers `/Carb`, initial events, and stores `Nx.Frm`.
  - `Nx:SetupEverything()` initializes data, events, options, UI, item/proc/title/minimap systems, comms, HUD, map, gathering, travel, and user events.
  - `Nx:NXOnEvent(event, ...)` and `Nx:NXOnUpdate(elapsed)` are wired from the XML frame.
- Embedded AceEvent is added to several tables in `Nx:InitEvents()`, including `Nx.Com`, `Nx.Map.Guide`, `Nx.AuctionAssist`, and `Nx.Travel`.

When changing startup or module initialization, check both the Ace lifecycle and XML frame lifecycle. A change that works after login can still fail during early frame creation.

## Expansion And API Compatibility

The code supports multiple WoW API eras with flags in `Carbonite.lua`, including:

- `Nx.isClassic`, `Nx.isClassicEra`, `Nx.isTBCClassic`, `Nx.isWotlkClassic`, `Nx.isCataClassic`, `Nx.isMoPClassic`, `Nx.isRetail`
- build-threshold flags such as `Nx.OldMapIDs`, `Nx.TBCMaps`, `Nx.MOPMaps`, `Nx.TWWMaps`, and `Nx.MidMaps`

Do not assume a single API surface. Prefer existing compatibility patterns already in the file, such as:

- `C_AddOns.IsAddOnLoaded or IsAddOnLoaded`
- `C_Item.GetItemInfo or GetItemInfo`
- `C_Spell.DoesSpellExist or DoesSpellExist`
- map-ID switches using `Nx.OldMapIDs`

This working copy lives under a `ChromieCraft_3.3.5a` install path, but the included TOCs target Retail, Classic Era, TBC Classic, and MoP Classic. If the user asks for Wrath/3.3.5a behavior, verify the intended TOC/client target before changing interface numbers or load paths.

## Data Files

Expansion data is split by flavor:

- `Data\classic`
- `Data\tbc`
- `Data\mop`
- `Data\retail`
- `Data\Shared`

Each expansion `data.xml` includes:

- `MapWorldHotspots.lua`
- `Zones.lua`
- `Guides\Guide.xml`
- `ZoneConnections.lua`

Each expansion TOC also separately loads that expansion's `NxMapData.lua`.

Common data patterns:

- `Data\<flavor>\NxMapData.lua` populates `Nx.Map` data such as continent blocks, map zones, and `Map.MapInfo`.
- `Data\<flavor>\Zones.lua` populates `Nx.Zones` and related zone metadata using localized names.
- `Data\<flavor>\ZoneConnections.lua` is used by travel/path logic.
- `Data\<flavor>\MapWorldHotspots.lua` feeds map hotspot rendering.
- `Data\<flavor>\Guides\GuideDefault.lua` initializes `Nx.GuideData = {}`.
- `Data\<flavor>\Guides\*.lua` appends guide categories, usually keyed by localized names.
- `Data\Shared\NxMapGuide.lua` defines shared guide UI/category behavior and contains version-specific map-ID logic.

When adding a guide file:

1. Add the Lua file under the relevant `Data\<flavor>\Guides\` folder.
2. Add it to that folder's `guide.xml`.
3. Add required locale keys in all locale folders or intentionally fall back only where AceLocale supports it.
4. Test the guide category in-game.

When changing zone IDs or map coordinates, check all expansion folders that may need the same concept represented with different IDs.

## Localization

Locales load through `Locales\Locales.xml`, which includes one XML per locale:

- `enUS`, `deDE`, `esES`, `esMX`, `frFR`, `itIT`, `koKR`, `ptBR`, `ruRU`, `zhTW`, `zhCN`

Each locale folder usually includes:

- `Main.lua`
- `HarvestNodes.lua`
- `Com.lua`
- `Map.lua`
- `Zones.lua`
- `Guide.lua`
- `FlightMasters.lua`
- `Options.lua`
- `Bindings.lua`

The usual pattern is:

```lua
local L = LibStub("AceLocale-3.0"):NewLocale("Carbonite", "enUS", true, true)
if not L then return end

L["Carbonite"] = true
```

Guidance:

- Add new user-facing strings to `enUS` first.
- For non-English locales, preserve existing style. If no translation is available, ask the user or use the existing fallback convention rather than inventing translations.
- Do not concatenate localized grammar-heavy phrases if a full sentence key would be safer.
- Be careful with guide keys like `L["Alchemy"] .. " " .. L["Trainer"]`; changing one locale term can change data table keys.

## Saved Variables And Migrations

Saved data is versioned manually in `Carbonite.lua` with constants such as:

- `Nx.VERSIONDATA`
- `Nx.VERSIONCHAR`
- `Nx.VERSIONGATHER`
- `Nx.VERSIONGOPTS`
- `Nx.VERSIONHUDOPTS`
- `Nx.VERSIONTRAVEL`
- `Nx.VERSIONCAP`

`Nx:InitGlobal()` resets or migrates structures when versions are old.

Guidance:

- Prefer additive defaults in `defaults` or module defaults when possible.
- If changing persisted structure, update the relevant version constant and add explicit migration/reset logic.
- Avoid resetting `CarbData` wholesale unless the user explicitly wants that behavior.
- Profile settings live mostly under `Nx.db.profile`; per-character travel data lives under `Nx.db.char`.

## Development Conventions

- Client API Interface is version 30300
- Keep using the existing `Nx.*` module-table style.
- Use `local L = LibStub("AceLocale-3.0"):GetLocale("Carbonite")` in files that need localized strings.
- Prefer existing helper functions and UI primitives from `NxUI.lua` before creating new UI patterns.
- Preserve the legacy WoW Lua style where it helps consistency: table methods with `:`, global frame names where XML expects them, and `Nx.prt` for user/debug chat output.
- Use `hooksecurefunc` for Blizzard hooks where possible; avoid replacing secure/global functions directly.
- Avoid heavy work in `OnUpdate`; many map and comm paths already run frequently.
- Keep asset paths in WoW format, for example `Interface\\AddOns\\Carbonite\\Gfx\\...`.
- Do not edit embedded libraries in `Libs\` unless the task is specifically to update the vendored library.
- `Libs\libs.xml` is the load list, not the directory listing. A library folder existing under `Libs\` does not mean it is loaded.

## Practical Verification

There is no package manifest or automated test suite in this folder.

Useful local checks:

- `rg --files` - inventory files quickly.
- `rg "pattern"` - search code/data.
- `Get-Content <file> -TotalCount <n>` - inspect files in PowerShell.

In this environment, use non-login PowerShell commands when possible; the user's profile may emit an execution-policy error.

In-game checks:

1. Enable script errors: `/console scriptErrors 1`
2. Reload UI: `/reload`
3. Confirm the Carbonite minimap button and map open.
4. Open options with `/carb options`.
5. Test slash commands affected by the change, especially `/carb`, `/way`, and `/cbway`.
6. For map/data changes, test on the relevant expansion client/TOC and inspect the zone, guide, hotspot, travel route, or coordinate conversion directly.

## Common Pitfalls

- Updating only one TOC when a new core file must load for every supported client.
- Adding a data file but forgetting the relevant XML include.
- Using Retail-only APIs without a Classic fallback.
- Assuming modern map IDs where `Nx.OldMapIDs` expects old IDs, or the reverse.
- Changing localized strings without updating all locale files that rely on the key.
- Modifying `Carbonite.xml` frame names without updating Lua references.
- Editing secure frames or world-map hooks without testing combat lockdown and taint behavior.
- Treating `Integrations\Sliverdragon.lua` as active code before adding it to a load path.
- Running broad formatters over large legacy Lua files; the diff becomes hard to review and risky.

## Before Finishing A Change

- Re-check the affected TOC/XML load paths.
- Search for the touched option, data key, map ID, frame name, or locale key across the whole addon.
- If saved variables changed, document migration/reset behavior.
- If testing in-game was not possible, say so clearly and list the exact manual checks the user should run.
