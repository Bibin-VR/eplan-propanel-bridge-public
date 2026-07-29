?# eplanbridge

Drive EPLAN Pro Panel 2022.0.3 from outside the application, and build a
6-axis robot-arm control schematic with it.

## What this is

A programmable control surface for EPLAN Pro Panel, in the shape of the
existing `gxbridge` MCP server, plus the project it was built to produce:
`RoboArm_6DOF_FX5U`, a 20-page schematic for an FX5U-controlled 6-axis arm —
273 symbols and 129 connections, all placed by script.

Two documents carry the knowledge:
[`FINDINGS.md`](FINDINGS.md) is the construction spec (pin geometry, the
place → generate → close sequence, what is unresolved and why), and
[`ATTEMPTS.md`](ATTEMPTS.md) records the dead ends. Surface-level findings —
how the remoting bridge was established at all — live in
[`docs/eplan-automation-surface.md`](docs/eplan-automation-surface.md). Read the
relevant one before changing anything; each records what was measured and what
is still a guess.

## Why

EPLAN has no supported way to build a schematic in bulk from outside the GUI.
Drawing 273 symbols and wiring them by hand is slow and unrepeatable; doing it
from a script makes the drawing a build artefact that can be rebuilt, diffed
and corrected. This project works out what that takes and records the parts
that are not obvious — chiefly that placement alone connects nothing.

## Why it is built this way

EPLAN's remoting channel exposes exactly one useful verb, `ExecuteAction`. The
interesting API — `DataModel`, `DataModel.E3D`, `HEServices` — is **in-process
only**. The bridge between them is EPLAN's `ExecuteScript` action.

But EPLAN's script compiler carries **no reference to `Eplan.EplApi.DataModelu.dll`**.
Any compile-time mention of that namespace fails the whole script, and a failed
compile is reported as `Succeed=False` with an **empty message** — no error text
anywhere. That single fact dictates the design.

```
  PowerShell / MCP server           (out of process, net48)
        |  EplanRemoteClient.ExecuteAction
        v
  ExecuteScript /ScriptFile:EplanBridgeStub.cs
        |  (the ONLY reflection in the system)
        v
  EplanBridge.Core.dll              (in process, typed C#, full API access)
```

`EplanBridgeStub.cs` names no EPLAN API type, so it always compiles. It loads
`EplanBridge.Core.dll` and calls `Bridge.Invoke(command, argument)`. Everything
below that boundary is ordinary typed C# — which means **the compiler checks the
API usage**, instead of discovering mistakes at runtime.

The stub loads the DLL from a **byte array**, not `Assembly.LoadFrom`. That
leaves the file unlocked, so `EplanBridge.Core` can be rebuilt without
restarting EPLAN. Verified working.

## Layout

| Path | Purpose |
|---|---|
| `src/EplanBridge.Core/` | net48 class library, typed EPLAN API access |
| `bin/EplanBridge.Core.dll` | build output, loaded into EPLAN |
| `scripts/EplanBridgeStub.cs` | reflection stub run by `ExecuteScript` |
| `tools/Invoke-EplanBridge.ps1` | call one bridge command from PowerShell |
| `tools/Invoke-EplanAction.ps1` | dispatch EPLAN's own actions (`generate`, `renumber`, `reports`) |
| `tools/Build-RoboArmSchematic.ps1` | build every schematic page — idempotent, clears each page first |
| `tools/Get-ProjectAudit.ps1` | **the finish check**: functions, connections and untagged count per page |
| `tools/Set-RoboArmParts.ps1` | allocate a part to every function that represents a real device — idempotent |
| `docs/parts-catalog.json` | all 654 parts in the local database, dumped via `parts.search` |
| `tools/Get-SymbolGeometry.ps1` | measure a symbol's connection-point offsets before designing with it |
| `tools/Build-PanelSpace.ps1` | create the Pro Panel 3D space and mounting plate — **not idempotent** |
| `tools/Build-PanelLayout.ps1` | draw the cabinet arrangement on the panel-layout page — idempotent |
| `tools/New-RoboArmPages.ps1` | create the page shells (already run) |
| `tools/Restore-PageNumbering.ps1` | shift a page block's counters — **does not work**, kept as evidence |
| `ipc/` | `request.txt` in, `response.json` out |
| `projects/` | working projects |
| `docs/eplan-automation-surface.md` | how the remoting/script/DLL bridge was established |
| `docs/substitutions.md` | what stands in for the absent Mitsubishi hardware |
| `docs/symbol-catalog.json` | all 1,035 `IEC_symbol` symbols with descriptions |
| `FINDINGS.md` | the schematic construction spec — read before changing the build |
| `ATTEMPTS.md` | dead ends, so they are not repeated |

## Verify

With a dedicated EPLAN instance running on port 57315:

```powershell
.\tools\Get-ProjectAudit.ps1
```

Success is the last three lines reading exactly:

```
total functions   = 273
total connections = 129
untagged (=+)     = 273
```

and every page `=RA1+CAB/2` through `/17` reporting a non-zero function count.
The audit reopens the project from disk, so a passing run is also the
persistence proof — there is no `Save` in this API, only `Project.Close()`.

For parts allocation:

```powershell
.\tools\Invoke-EplanBridge.ps1 -Command project.parts `
    -Argument 'C:\Users\Kishor\Tovex-Work\eplan-propanel-bridge\projects\RoboArm_6DOF_FX5U.elk'
```

Success is `withoutPart` reading `0`:

```
{"totalFunctions":169,"withPart":144,"withoutPart":0,
 "cannotCarryArticle":25,"distinctParts":9}
```

`totalFunctions` is 169 here and 273 in the audit above, and both are correct —
the audit sums 16 page-filtered queries, this is one project-wide query, and
`GetFunctions(null)` does not return the 104 PLC connection points. Do not
"fix" either number to match the other.

To rebuild the schematic from scratch and re-verify:

```powershell
.\tools\Build-RoboArmSchematic.ps1
.\tools\Invoke-EplanAction.ps1 -Actions 'generate /TYPE:CONNECTIONS /PROJECTNAME:"C:\Users\Kishor\Tovex-Work\eplan-propanel-bridge\projects\RoboArm_6DOF_FX5U.elk" /REBUILDALLCONNECTIONS:1'
.\tools\Invoke-EplanBridge.ps1 -Command project.close -Argument 'C:\Users\Kishor\Tovex-Work\eplan-propanel-bridge\projects\RoboArm_6DOF_FX5U.elk'
.\tools\Get-ProjectAudit.ps1
```

`untagged = 273` is the expected value, not a failure to investigate: device
numbering does not work yet (FINDINGS §3). It will drop to 0 when it does.

## Build

```powershell
cd src\EplanBridge.Core
dotnet build -c Release -o ..\..\bin
```

Targets `net48` because the EPLAN API is .NET Framework. The only SDK installed
is .NET 10, which cannot load Framework assemblies in-process — but it builds
`net48` fine (targeting pack present). EPLAN references use `<Private>false</Private>`
so its DLLs are never copied; the process already has them loaded.

## Use

```powershell
.\tools\Invoke-EplanBridge.ps1 -Command ping
.\tools\Invoke-EplanBridge.ps1 -Command project.info `
    -Argument 'C:\Users\Kishor\Tovex-Work\eplan-propanel-bridge\projects\RoboArm_6DOF_FX5U.elk'
```

`-Port` defaults to `57315` (the dedicated instance). Start a dedicated instance
with an explicit port rather than an ephemeral one:

```
Eplan.exe /Variant:"Pro Panel" EplanServer /EPLANSERVERPORT:<port>
```

## Commands

| Command | Argument | Returns |
|---|---|---|
| `ping` | — | bridge version, CLR, bitness |
| `project.info` | `.elk` path | name, paths, page count, page list |
| `page.create` | `project=…;plant=…;location=…;counter=N;type=…;description=…` | created page name and type |
| `project.close` | `.elk` path | confirmation — **this is what flushes writes to disk** |
| `symbol.list` | `library=&lt;.slk&gt;;filter=…;limit=N` | symbol name, id and FUNC_DESC |
| `function.place` | `project=…;page=…;library=…;symbol=…;variant=N;x=…;y=…` | placed symbol and its created type |
| `page.contents` | `project=…;page=…` | functions, with function definition and identifier letter |
| `page.clear` | `project=…;page=…` | count removed — destructive, no undo |
| `page.placements` | `project=…;page=…` | **every** placement by type — use this, not `page.contents`, to tell whether a report page is populated |
| `page.pins` | `project=…;page=…` | each symbol's connection points in **absolute** coordinates |
| `page.connections` | `project=…;page=…` | connections EPLAN derived on the page |
| `project.connections` | `.elk` path | project-wide connection count by representation type |
| `page.setcounter` | `project=…;page=…;counter=N` | renames a page — **fails, `PAGE_COUNTER` is read-only after creation** |
| `report.create` | `project=…;types=TableOfContents,…` | creates reports from the project's templates |
| `device.renumber` | `project=…;scheme=…;start=N;step=N` | assigns device tags — **currently fails, see FINDINGS.md §3** |
| `page.clearall` | `project=…;page=…` | removes **every** placement including graphics — destructive |
| `space.create` | `project=…;name=…;height=…;width=…;depth=…` | Pro Panel 3D installation space + article-free mounting panel |
| `space.list` | `.elk` path | installation spaces and their contents |
| `graphics.rect` | `project=…;page=…;x1=…;y1=…;x2=…;y2=…;filled=0\|1` | one rectangle — graphics only, carries no function |
| `graphics.text` | `project=…;page=…;x=…;y=…;height=…;text=…` | one text item; `text=` may not contain `;` |
| `parts.search` | `filter=…;group=…;limit=N` | parts matching a substring over part number, type, manufacturer and description — read-only |
| `function.setpart` | `project=…;part=…;page=…;symbol=…` | assigns one part to matching functions; `page=` optional (whole project when omitted). Adds only — **there is no counterpart that removes** |
| `project.parts` | `.elk` path | **the parts finish check**: functions with/without a part, grouped by part number |

Page types accepted by `page.create`: `Circuit`, `TitlePage`, `TableOfContents`,
`PLCDiagram`, `PLCCardOverview`, `TerminalDiagram`, `PanelLayout`, `Overview`,
and the rest of `DocumentTypeManager.DocumentType`.

## Rules for adding a command

1. One command, one verb, one effect. Never "create or update".
2. Return JSON always. Never throw across the boundary; never show a dialog —
   a modal dialog blocks every later remote call, since EPLAN only services
   remote calls when idle.
3. Data-model **writes must be wrapped in a `LockingStep`**, or EPLAN throws
   `NoLockingStepException` (S063110).
4. A project already open cannot be re-opened. Use `GetProject` first
   (returns null when not open), then `OpenProject`.
5. State in the command's docs what it cannot do.

## API gotchas found the hard way

- **There is no `Save`.** The only persistence trigger is `Project.Close()`.
  Verified: after creating 19 pages, `Page.eod` was still 1,134 bytes at its
  original timestamp; after `project.close` it became 22,680 bytes, and
  reopening from disk (`wasAlreadyOpen:false`) returned all 20 pages. **Never
  report a write as done before closing** — an in-memory read-back proves
  nothing.
- **The page description property is `PAGE_NOMINATIOMN`** — EPLAN's own
  misspelling, documented as "Page description # 11011". There is no
  `PAGE_DESCRIPTION`. `PAGE_SUPPLEMENTARYFIELD` is a *different*, indexed
  property (11901) and is the wrong choice.
- **`MultiLangString` has no string constructor.** Its only ctor takes a native
  `EMultiLangString*`. Build it empty then
  `AddString(ISOCode.Language.L_en_US, text)`, as EPLAN's own sample scripts do.
- Page name parts go in the `PagePropertyList` passed to `Page.Create`
  (`DESIGNATION_PLANT`, `DESIGNATION_LOCATION`, `PAGE_COUNTER`); other
  properties are set on `page.Properties` afterwards.
- **Place symbols with `SymbolVariant.Create(page)`, never `Function.Create`.**
  `Function.Create` only builds generic `Function`s. For a symbol backed by a
  specialised class (terminals, cables, PLC boxes) it creates the object and
  *then* throws `NotImplementedException` — leaving a half-initialised object on
  the page while reporting failure. Observed exactly: four terminals reported
  FAIL yet all appeared in `page.contents`. `SymbolVariant.Create` returns the
  correct subclass; set `SymbolReference.Location` afterwards.
- **Device tags cannot be set as a property.** `FUNC_DEVICETAG_FULL` throws
  `NotImplementedException`; `NameParts` has no `FUNC_DEVICETAG_MAINNAME`; there
  are no `IDENTLETTER`/`COUNTER` name-part properties on
  `FunctionBasePropertyList`. EPLAN assigns device tags with its own `renumber`
  action — place first, number after.
- Symbol names are opaque codes (`SL`, `XBS`, `QL3`). Choose them by
  `FUNC_DESC` via `symbol.list`, not by guessing at the abbreviation.
  `IEC_symbol` holds 1,035 symbols, 1,030 with descriptions. Note `symbol.list`
  filters on the *name*, which is near useless — dump the whole library once
  (`limit=2000`, no filter) and grep the descriptions offline.
- **Placing symbols does not connect them.** EPLAN auto-connects connection
  points that share an X coordinate and face each other, but only when
  `generate /TYPE:CONNECTIONS /REBUILDALLCONNECTIONS:1` is run. Place →
  generate → close. Skip the middle step and you get a drawing that looks
  right and is electrically empty.
- **`PinBase.Location` is relative to the symbol's insertion point**, not the
  page. `page.pins` does the addition for you. Pole pitch is a property of the
  symbol — 8 mm is common, 16 mm exists, and the two do not interoperate.
- **`renumber /TYPE:PAGES` is not a safe probe.** It renumbers every page in
  the project from one start value and cannot be scoped to a structure. It is
  not reversible: `PAGE_COUNTER` is read-only after creation (S063113).
- **An invalid scheme name makes EPLAN list the valid ones.** Numbering scheme
  names are not readable from disk (settings are opaque `.eod` databases), but
  passing a nonsense name returns `S029123` enumerating every valid scheme.
  The same trick is worth trying wherever a name must be guessed.
- **`Create` is an instance method on the EPLAN data-model types**, not a
  static factory: `new Page(); page.Create(...)`, and likewise
  `InstallationSpace`, `MountingPanel`, `Rectangle`, `Text`. Calling
  `Type.Create(...)` gives `CS0120`.
- **`MountingPanel.Create(project, height, width, depth)` is article free** —
  dimensions only, no part number. It is the one Pro Panel object that can be
  built in a project with no parts. Note height comes first.
- **Read bridge responses with `-Encoding UTF8`.** PowerShell 5.1's
  `Get-Content -Raw` defaults to the ANSI codepage, so any non-ASCII the bridge
  returns is silently corrupted — `Zähler` arrives as `ZÃ¤hler`, and 19 symbol
  descriptions in the catalogue were mangled before this was caught.

## Status

Working and verified:

- remoting connect / ping to a dedicated instance
- action dispatch, with three distinguishable failure classes
- in-process execution of a compiled assembly
- typed `DataModel` read access
- data-model **write** — project creation via `ProjectManager.CreateProject`
- data-model **write** — page creation, 19 pages in one run, 0 failures
- persistence verified by close → reopen → recount
- rebuild without restarting EPLAN

`RoboArm_6DOF_FX5U` holds 20 pages: the template title page plus a `=RA1+CAB`
structure covering supply, 24 V DC, safety and STO, FX5U CPU and I/O, two
FX5-40SSC-S modules, SSCNET III/H topology, six MR-J4-B axis pages, terminals
and cabinet layout.

**Schematic content is built and wired.** Verified by reading back from disk
after a close→reopen:

```
total functions   = 273     (16 content pages, 0 placement failures)
total connections = 129     (all Circuit)
untagged (=+)     = 273
```

**Parts are allocated.** 144 functions carry a real part from the local
database, 0 are left without one, verified from disk after a close→reopen:

```
withPart = 144   withoutPart = 0   cannotCarryArticle = 25   distinctParts = 9
```

Every one is a **substitute** — this machine holds no Mitsubishi data, so an
`SEW.MC07B0015-5A3-4-00` vector drive stands in for each MR-J4-B amplifier and a
`SEW.DRN90L4/FE/TH` for each servo motor. The register is
[`docs/substitutions.md`](docs/substitutions.md); do not read the BOM as a
specification.

Reproduce with `tools\Get-ProjectAudit.ps1`. Full spec in
[`FINDINGS.md`](FINDINGS.md); every dead end in [`ATTEMPTS.md`](ATTEMPTS.md).

Known incomplete, with the cause identified for each:

- **No device tags.** All 273 functions read `=+`. Numbering fails with
  `S029079 Unable to generate numbering groups` through both the `renumber`
  action and the typed API. Function definitions, identifier letters, the scheme
  name and — as of 2026-07-29 — missing parts have all been ruled out. Six
  attempts, FINDINGS §3.
- **Table of contents and terminal diagrams are empty.** The project defines no
  report templates, so `reports /TYPE:PROJECT` succeeds and generates nothing —
  FINDINGS §4.
- **PLC pages carry no connections** (122 symbols on `/6`–`/11`). Those are
  EPLAN evaluation page types; demonstrated by controlled test — FINDINGS §1.2.
- **No horizontal busbars.** Auto-connect works on shared X only, so 24 V
  distribution is drawn as one column per feeder — FINDINGS §1.4.
- **Cabinet layout is a drawing, not device placements.** Page `/19` carries a
  1:7 mounting-plate arrangement (37 rectangles, 25 texts) and the project has
  a real article-free Pro Panel `InstallationSpace` + `MountingPanel`. No device
  is *mounted* on that panel. **The reason it could not be has gone** — mounting
  needs parts with physical dimensions, and 144 functions now have them. What is
  missing is the command: a `panel.mount` wrapping
  `MountingPanelService.CreateArticlePlacement`. Not written, not run —
  FINDINGS §6.
- Page numbers on disk sit one below the design numbers used throughout these
  documents, after an unscoped `renumber /TYPE:PAGES` — mapping table in
  FINDINGS §5.

Not built: the MCP server wrapper itself, and 3D device mounting.
`DataModel.E3D` is opened — one installation space with an article-free
mounting panel — but nothing is mounted in it.

**Constraint on content:** this installation has **no Mitsubishi FX5U / MELSEC /
MR-series parts or macros**. The parts database holds 654 parts —
`RIT 233, SIE 156, FES 100, ESS 51, A-B 37, LAPP 31, PXC 30` and a handful more,
catalogued in [`docs/parts-catalog.json`](docs/parts-catalog.json) — and none of
them is Mitsubishi. Available macro libraries are Rittal (419), Siemens (152),
Phoenix (55), Allen-Bradley (35). Any FX5U panel built here uses substitute
hardware until Mitsubishi data is imported.
