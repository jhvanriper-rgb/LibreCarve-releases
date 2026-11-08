# LibreCarve agent guide: running jobs from file to G-code

This guide teaches an AI agent (Claude, or any MCP client) how to set up, run and check a LibreCarve job
without the window: artwork in, G-code out. It is written to be given to the agent as instructions, so it
speaks to "you". Everything here describes commands that exist in the repository. Items that do not
exist yet are listed under "Not available yet" and must not be assumed.

The in-app Help has a shorter version of this guide in its "AI automation" section (pages `ai-automation`,
`ai-connect`, `ai-job-run`, `ai-safety` in `crates/help/pages`). An agent can read it with
`help.page {"id": "ai-job-run"}`. When this guide changes, update those pages too; a test checks that the
commands they name exist.

The Help also has a "Modify" section (step-by-step pages for Erase, Move, Fillet and the other drawing-change commands, starting at `modify-commands`) and a "Toolpaths" section (one page per toolpath command, starting at `toolpaths-start`, each with a `json cam.<id>` example); `help.page` opens them by id, for example `help.page {"id": "modify-fillet"}`.

Keep this file current. Any pull request that adds or changes a `cam.*`, `project.*` or `import`
command parameter updates this guide in the same pull request.

## 1. What you are doing

A LibreCarve job is a session holding a drawing, a material setup, and a list of toolpaths. You build it
with commands. Each command takes JSON and returns JSON, never a dialog. The steps, in order:

0. Optional: set the machine (`cam.machine`, or the guided `cam.machine.wizard`), so that units, post, safe Z, bit change height, time estimates and limit warnings follow it. Ask the person which machine they have; never guess one.
1. Set the material (`cam.material`). Nothing else works before this.
2. Bring in the artwork (`cam.import`), or draw it with the drawing commands.
3. Choose a tool (`cam.tool.pick`, or list them with `cam.tools`) and create toolpaths (`cam.profile`, `cam.pocket`, `cam.hatch`, `cam.engrave`, `cam.chamfer`, `cam.countersink`).
4. Review (`cam.list`), check (`cam.verify`, `cam.simulate`).
5. Write G-code (`cam.post`).
6. Report to the person, who approves before anything goes to the machine.

The same job can run as one command, `job.run` (section 5). Use the steps for a new or unusual job, where you
want to look at each result; use the recipe for a job whose settings you already trust.

## 2. Connecting

LibreCarve speaks the Model Context Protocol on stdio.

```sh
librecarve-cli mcp                              # headless: one in-process session, no window
librecarve-cli mcp --connect 127.0.0.1:7979     # attached to a running app (see below)
librecarve-cli mcp --connect 127.0.0.1:7979 --wait 60   # the app may start later: each call polls for it up to 60 s (default 30)
```

The app side of `--connect` is the control channel. The person turns it on in the app with Window >
Control channel... (an on/off box, a port from 1024 to 65535, default 7979, and an optional "Start
automatically"), or starts the app with `librecarve --control 7979` / `CADCRAFT_CONTROL_PORT`. It is off
at every launch unless they checked auto-start, listens on 127.0.0.1 only, and shows "Control channel on"
in the status bar while open. If you cannot connect, ask the person to turn it on; you cannot do it
yourself, because the window's buttons ignore input that comes from the channel (`ui.click`, `ui.key`),
and `ui.control` only opens the window.

Client configuration:

```json
{ "mcpServers": { "librecarve": { "command": "librecarve-cli", "args": ["mcp"] } } }
```

Generic tool reference: [`docs/mcp.md`](../mcp.md). The tools you use for jobs:

| Tool | Use |
| --- | --- |
| `execute {command, params}` | Run any command with JSON. This is how you run every `cam.*` command. |
| `list_commands {filter?}` | See command ids and parameter docs. Use `filter: "cam."` first in a new session; it is the live source of truth for parameters. |
| `inspect_drawing`, `query_entities` | Look at what is in the drawing (handles, types, extents). |
| `render` | A PNG of the drawing, to look at the artwork. |

Headless mode keeps one session for the life of the process. All job state (material, toolpaths) lives in
that session. If the process ends, the state is gone unless you saved it with `project.save`.

## 3. Rules you must follow

These are not style advice. They exist because a wrong program can break a bit, ruin material or hurt
someone.

1. **Read every result.** A command that fails returns an error with a plain-English message. Never assume
   success. Never report a step as done without its result in front of you.
2. **Order matters.** `cam.material` first. `cam.verify` and `cam.simulate` before `cam.post`.
3. **`cam.verify` failing is a stop sign.** It returns an error when any toolpath point breaks the cut
   rules (gouge, uncut region). Do not post. Report the message, fix the cause (usually tool size, side or
   depth), recalculate, verify again.
4. **Never invent numbers you cannot justify.** Feeds, speeds and depths come from the tool database
   (`cam.tools`) or from the person. If you must guess, say which numbers are guesses.
5. **Units are the job's units.** After `cam.material` every number you pass (depths, tool diameters, feeds)
   is in that job's units (`mm` or `inch`). Library tools are converted for you; tools you give in full are
   used exactly as given.
6. **A person approves before the machine. You cannot send.** You may create files. You may not tell
   anyone the job is "ready to cut" until you have reported the review items in section 9 and they have
   said yes. Never copy a file to a machine, controller or sender yourself. LibreCarve enforces this:
   `send.upload` and `send.stream` are dry runs only, and the Send to Machine window's Upload, Start
   streaming, Resume and Continue buttons refuse clicks and keys from the control channel (`ui.click`,
   `ui.key`, `ui.text`). Do not try to get around it. See section 4, "Sending (dry run only)".
   **Approval gate.** `job.run` with a machine post (such as `onefinity`) sets `require_approval` to true
   by default. When verify passes, the result says `status: "awaiting_approval"` with a `job_id` instead of
   `ok: true`: the G-code has been written, but the job is not finished. Report the result to the person and
   ask for approval, then call `job.approve {job_id: "..."}` after they say yes. If verify failed the job
   never reaches `awaiting_approval`. `job.approve` refuses an unknown id, a second approval, and a job
   with verify violations.
7. **Warnings are information the person needs.** Pass every non-empty `warnings` array on to them.
8. **Small steps.** One command at a time, read the result, then decide. Do not chain guesses.
9. **Do not touch other files.** Write G-code and previews only where the person told you, or in a new
   folder you name in your report. Never overwrite an existing file without being asked.
10. **Ask about open questions before you build.** If the request leaves out something the job depends
    on, ask the person first, in one short message that lists every open question at once. Do not guess.
    Always ask for: material and its **thickness**; the size of the stock and of the design (including any
    dimension left out, such as the height of an oval that was given a width); which bit to use if they
    named none; the units; and, for text, the font and wording. Also ask when the person said "about" or
    gave a range, or when two reasonable readings lead to different cuts. Offer your suggested answer with
    each question so they can just say yes. If nobody is there to answer (an unattended run), say at the top
    of your report which values you assumed, and keep each assumption easy to change.

## 4. Command reference

All are called as `execute {command: "<id>", params: {...}}`. `?` means optional. Parameter docs below
match `list_commands`; if they disagree, `list_commands` wins and this guide has a bug to report.

### Job setup

**`cam.material`** `{thickness, units?: "mm"|"inch", z_zero?: "top"|"bottom", safe_z?, home?: [x,y,z], size?: [w,h], origin?: [x,y], stock_from?: [handle] | true, xy_datum?: "lower_left"|"lower_right"|"upper_left"|"upper_right"|"center"|"spoilboard", hold?: "tabs"|"vacuum"|"clamps", tabs?: {count, width, thickness}}`

**The stock is always an object (2026-10-29).** `stock_from: [handle]` (or `true` for the selection) takes `size` and `origin` from the bounding box of closed shapes in the drawing; it cannot be combined with `size` or `origin`, and an open shape is an error. A material with no `size` has no stock: the design window draws no stock outline, relief pictures (the canvas backdrop and `cam.simulate` of a job with a relief toolpath) are cropped to exactly the area the toolpaths cut with no margin, and zero is the material `origin` (lower left datum; 0, 0 unless set). `cam.post` then returns `stock_warning` ("The stock size is not set, so zero is the material origin (X, Y), not a stock corner. Set the stock in Material Setup.", with the real point; the file is still written, and a spoilboard datum does not warn), and `send.upload` and `send.stream` add the same text to their `warnings`, for the setups they send only. `cam.post` and `send.*` warn only about the setups that are in the program. `job.photo` and `job.model` take `border` (job units, default 0): with no material size they make the stock, picture size plus twice the border, with its lower-left corner at the material's `origin` and the picture inset by the border (with `position`, the picture's corner, the stock corner is that minus the border), which is zero with a lower-left datum; the result's `stock` and a `stock` step say what was made. With a sized stock the picture is centered and `border` is reported as unused. With `keep_toolpaths: true`, a warning says when the kept toolpaths lie outside the stock that was made. Only Photo to Relief in the app has a Border field; 3D Model to Relief has none.
Defaults: units, safe Z and bit change height from the job's machine when it has them, else units `mm`, safe Z 5 mm or 0.2 inch and bit change height 3 inch; `z_zero` top. `hold` says how a part is held when a cut goes
all the way through: `tabs` (every through-cut `cam.profile` gets the material tabs, 4 of 8 x 3 mm or 0.3 x 0.12 inch
unless you give `tabs`; pass `"tabs": false` to `cam.profile` to opt out), `vacuum` or `clamps` (no tabs). With `hold`
unset, a through-cut profile carries a warning that the part can come loose: pass that warning on to the person.

Photo and model reliefs are placed in the stock by three more keys (leaving them out keeps the current value). In the app they are set in the Photo to Relief window, not in Material Setup; `job.photo` and `job.model` also take `image_anchor`, `fit_image` and `image_border` themselves and save them in the job's material (the 3D Model window has no rows for them: it uses whatever Photo to Relief last saved) (all its other settings are kept, and a relief already in the job is moved to match):
`image_anchor` (`top_left`, `top_center`, `top_right`, `middle_left`, `center` (default), `middle_right`,
`bottom_left`, `bottom_center`, `bottom_right`) is the point of the stock the relief is lined up with, separate from
`xy_datum` (where X0 Y0 is). `fit_image` (default true) shrinks a relief larger than the stock, keeping its proportions
and its depth, so it fits; it never enlarges. `image_border` (default 0, job units) is a margin kept clear on every
side; a border that leaves no room is an error. When a relief is already in the job, `cam.material` places it and its
frame and toolpaths and reports `relief_moved` (with a `summary` to pass on), `relief_scaled {from, to, factor}` and
`toolpaths_stale` when it was shrunk (the old toolpaths are not scaled: recalculate them, `cam.post` and `cam.verify`
refuse until you do), `relief_oversize` (with fit off, or when a relief hangs over an edge: also shown by `cam.verify`
`warnings` and `cam.post` `stock_warning`), or `relief_not_moved {reason, stale?}` with a plain reason. A relief tied to
a shape the person drew, one the person moved by hand, or a job with two or more reliefs is left alone. `cam.post`
and `cam.verify` also refuse with "You moved the relief after its toolpaths were made..." or "The relief was resized
after its toolpaths were made..." when a relief toolpath no longer matches its map.
`tabs` work on closed outlines (including `side: "on"`) and on open paths (spread over the part of the path that lies on the stock when `cam.material` has a `size`, else over the whole path). Thickness is measured from the bottom of the cut, or from the stock bottom when the cut goes below it. A warning says when fewer tabs were placed than asked. A path too short for them gets a `too short for tabs` warning; a tab thickness not below the cut depth is an error; `tabs` (other than false/null) on any toolpath command other than `cam.profile` is an error. A requested tab that is not applied is never silent.
**Zero point (`xy_datum`).** The corner choices and `center` make that point of the stock, found from `origin` and `size`, the machine's X0 Y0: posted XY is the drawing XY minus that point, and `size` is required. `spoilboard` keeps X0 Y0 at the drawing origin (the machine's spoilboard corner) whatever the stock does: posted XY equals drawing XY, `size` is optional and `datum_point` is `[0, 0]`. Use it when the person zeroes the machine on the spoilboard and lays the stock anywhere on it. Old jobs and recipes with the five older values load and post exactly as before. In the app the stock is drawn behind the drawing as a pale rectangle with a `0,0` crosshair at the zero point; this is a display only (not an entity, never exported or cut), and the app opens Material Setup first when a job has no material, but only for the Profile, Pocket, Drill, Thread, V-Carve, Chamfer, Countersink and Hatch/Engrave commands started with no parameters from a menu or the command line. JSON, MCP and control calls are never redirected. Returns the material as stored plus
`top_z`, `bottom_z` and `datum_point`. Calling it again replaces the material.

**`cam.machine`** `{preset?: "onefinity-original-woodworker"|"onefinity-elite-woodworker"|"onefinity-pro-woodworker"|"generic-grbl", use?: saved name|"active", units?, name?, manufacturer?, model?, travel?: {x?,y?,z?}, max_feed?, max_plunge?, max_rapid?, accel_mm_s2?, spindle?: {min_rpm?,max_rpm?,manual_start?}, post?: "grbl"|"grbl-manual"|"onefinity"|"linuxcnc"|"mach"|"marlin"|"generic", accessories?, safe_z?, bit_change_z?, notes?, save?, make_active?, clear?}`
Sets the job's machine (saved with the project). A preset or saved machine is converted to `units`; a number set to null is cleared; unset limits are never checked. With no parameters it reports the machine and its `findings`, which only warn (job larger than the travel, feed, plunge or rpm over a limit, hand-started router with the `grbl` post): pass them on to the person. `cam.machine.list` shows models and saved machines; `cam.machine.delete {name}` removes a saved one. `safe_z?` and `bit_change_z?` (in `units`) are the heights a later `cam.material` starts with when the call leaves them out. Homing, soft limits, steps per mm and probe calibration are out of scope (they stay on the controller).

**`cam.machine.wizard`** `{answers?: {machine?, name?, units?, travel_x?, travel_y?, travel_z?, min_rpm?, max_rpm?, manual_start?, post?, touch_plate?, probe?, dust_collection?, safe_z?, bit_change_z?}, step?: "machine"|"work_area"|"spindle"|"post"|"accessories"|"summary", finish?: true}`
The guided setup a first-time user walks through in the window, without the window. It is stateless: pass the answers you have, and it returns `steps` (each with `title`, `help`, `warnings` and `fields`: `key`, `label`, `kind` (`choice`, `text`, `number`, `bool`), `default`, `value`, `help`, `unit?`, `options?`), `answers` with every default filled in, `summary` (`lines`, `findings` from the limit checks on the current job, `ready`, `error`) and `cam_machine` (the parameters Finish sends). `machine` is a model id, `"saved:<name>"` or `"custom"`; an answer left out takes the default of the machine picked, converted to `units`; numbers are in `units`, and null means not set. `step` also returns that step as `current`. `finish: true` saves the machine through `cam.machine` (`save` and `make_active`) and sets it on the job; the result is `cam.machine`'s plus `finished` and `answers`. An answer of the wrong type is an error; one of the right type but out of range shows in `summary.error` with `ready: false`, and `finish` then returns that error. Ask the person for the answers; the defaults are a beginner's starting point, not facts about their machine.

```json
{"command": "cam.machine.wizard", "params": {"answers": {"machine": "onefinity-original-woodworker", "units": "mm"}}}
{"command": "cam.machine.wizard", "params": {"answers": {"machine": "onefinity-original-woodworker", "units": "mm", "safe_z": 6}, "finish": true}}
```

**`cam.tool.pick`** `{op: "profile"|"pocket"|"drill"|"hatch"|"engrave", side?, feeds?, handles?, cut_depth?, kind?, min_diameter?, max_diameter?, max_detail_loss?: 0.01, limit?: 5}`
Ranks the tool database for the operation, the shapes (`handles`, else the selection) and the material
(`feeds`, else the job's `stock`, else hardwood). Pockets and hatches get the largest tool that fits every
region and keeps at least 99% of its area, up to 1/2 inch unless `max_diameter` says otherwise; profiles prefer 1/4 inch; drills the largest end mill no wider
than the smallest hole; engraving a 90 degree V-bit. Returns `pick`, `use: {tool, feeds}` (pass these to
the toolpath command unchanged), `candidates` with numbers in job units and `reasons`, and `warnings`.
When `pick` is null nothing qualified: read `warnings` and `closest`, then raise `max_detail_loss`, change
the shapes, or ask the person for a smaller bit. Do not invent a tool. In `job.run` an operation can say
`"tool": "auto"` or `{"auto": {pick options}}`.

**Overriding a database tool.** `tool: {id, feed?, plunge?, rpm?, pass_depth?, stepover?}` uses the
database tool with those numbers changed (job units). Say which numbers you changed and why.

**Stock.** `cam.material {stock: "mdf"}` sets the feeds material for every toolpath that does not pass
`feeds`.

**`cam.tools`** `{kind?: "end_mill"|"ball_nose"|"v_bit", feeds?: "softwood"|"hardwood"|"mdf"|"plastic"|<your material>, id?, full?: bool}`
Lists the tool database. Use `id` to look at one tool and `full: true` for all fields. With `feeds` the
list shows that material's feed and plunge numbers. Tools are edited with `cam.tool.add`, `.copy`, `.edit`,
`.delete`, `.defaults`, `.export`, `.import`; ask `list_commands {filter: "cam.tool"}`.

### Artwork

**`cam.import`** `{path | data: base64 + name?, at?: [x,y], scale?, units?: "auto"|"mm"|"inch"|"cm"|"m"|"ft", explode?: bool, group?: bool, page?: number (which page of a PDF/AI, default 1)}`
Reads DXF, DWG, SVG, PDF, AI (the PDF stream inside it) or EPS, converts it to the job's units, flattens block
references, selects the cuttable geometry. PDF, AI and EPS are in points (1/72 inch) and come in as millimeters on
a layer named after the file; only their vector paths are read, and raster images, text, clipping and gradients are
reported in `warnings`. Needs `cam.material` first. Returns:

| Field | Meaning |
| --- | --- |
| `handles` | With `group` true (the default): ONE handle, the block reference that holds the whole import. With `group` false: the cuttable objects (at most 10000 listed). Either way they are selected. |
| `block` | The name of the block made for the group (null when `group` is false). |
| `units` | `source`, `job`, `factor`, and `basis`: `given` (you passed `units`), `svg` (SVG is always mm), `pdf` (PDF/AI) or `eps`, `insunits` (read from the DXF header) or `assumed` (the file has no units: it was taken as the job's units). |
| `bounds` | `min`, `max`, `width`, `height` in job units, or null if nothing is cuttable. Check this against the material size. |
| `counts`, `shapes` | Objects by type, and how many closed and open shapes joined up. |
| `ignored` | Objects that cannot be toolpath geometry (text, hatch, dimensions, and so on), by type. |
| `warnings` | Read these. "assumed the job's units" means check the size. |

**Grouped import.** By default (`group: true`) everything that comes in becomes one block named after the file
(`Import - logo.svg`, numbered `(2)`, `(3)` if the name is taken) and ONE block reference is inserted at `at`, so a
file of thousands of small paths is one object that the normal `scale`, `move` and `rotate` commands act on as a
whole. `counts`, `shapes` and `bounds` still describe what is inside. `group: false` keeps the old behavior: loose
objects. `explode` keeps its meaning (flatten block references that are inside the FILE, such as DXF inserts) and
applies before grouping. `ungroup {handles?}` (Tools > Ungroup, or the Drawing tab) breaks the import group (the
block named `Import - <file>`) back into loose paths at the right place and scale. Any other block reference is an
ordinary block: `ungroup` still answers "the objects are not in a group" for it, and `explode` is the command for
those (it also works on an import group). `ungroup` on a group whose layer is locked is an error. You do not need to
ungroup to cut: `cam.profile`, `cam.pocket`, `cam.vcarve`, `cam.hatch`, `cam.engrave`, `cam.drill`,
`cam.countersink`, `cam.thread`, `cam.vector.offset`, `cam.shapes` and `cam.tool.pick` read through a block
reference (insert point, scale and rotation applied) without changing it, and a selection or `handles` that holds the
group works. `cam.vector.boolean` and `cam.nest` replace or move the objects they read, which are inside the block,
so they answer with an error that says the handle is a group and to Ungroup it first. `stretch` treats the group as
one object: a window that holds all of it moves it, a window that cuts through it is an error and changes nothing
(use `scale`, or ungroup first). Locked layers are respected throughout: nothing inside a locked group is replaced
or removed. A recipe's `select` of `"all"`, `{handles}` or `{layer}` finds
a group (by the layers of the objects inside it, and only those objects are cut). Reading through nested groups
stops at 16 levels and 2,000,000 objects (checked before a block is expanded, so a hostile file cannot make one
reference allocate more than that); the result's `warnings` says so. `cam.shapes` with `apply: true` has to
ungroup an import group that holds open pieces, because it replaces them; it reports that in `warnings`. Open pieces
in a locked group, on a locked layer, or in an ordinary block are reported as they are and left in place.

If `bounds` is wildly off (a 2 mm part that should be 50 mm), the file's units were wrong: import again
with `units` set, after undoing or starting a clean session.

Drawing instead of importing: `rectang {p1, p2}`, `circle {center, radius}`, `line {points}`, `pline`, and
the rest of the drawing commands; `list_commands` shows them. Then select with `selectall` or pass
`handles` to the toolpath command.

**Shape check and clean-up:** `cam.shapes {apply?, join_tol?, close_tol?, handles?}`. Without `apply` it only reports: `closed`, `open`, and for each open shape the positions of its two `ends` and the `gap`. With `apply: true` it joins pieces whose ends are within `join_tol` (default 0.1 mm), drops duplicates, and, if `close_tol` is given, closes a small gap with a straight segment. It is one undo step. Run it when `cam.import` reports open shapes and before a pocket or hatch, which need closed shapes. A recipe can do the same with `"clean": {"join_tol": 0.5}`. Look at the picture after closing a gap: the straight line is one you did not draw.

**Text in any font:** `cam.text` turns text into closed outlines on the `TEXT_VECTORS` layer and selects them. Typed text: `{text, at: [x,y], height, font, justify?, rotation?, width_factor?, oblique?, tolerance?, layer?}`. `height` is the capital height in job units and there is no default: ask the person. `font` is an installed TrueType or OpenType font, by file name or family; `cam.fonts {filter?}` lists the names, and an unknown name is an error, never a silent substitute. Existing TEXT and MTEXT objects: leave out `text` and pass `handles` (or select them); the originals stay unless `keep: false`. The result has `count`, `glyphs`, `rings`, `bounds`, `tolerance` and `warnings`. Curves are flattened to `tolerance` (default 0.01 mm, in job units), not to the screen's accuracy.
A letter is a closed outline, and the inside of an O, A, B or R is a closed outline inside it, so the pocket reads it as an island. To **cut the letters**, pocket the outlines on their own. To **clear the background**, draw a boundary (rectangle, oval) round the text and pocket the boundary together with the outlines: the space around the letters and the inside of every counter are cleared and the letters stay standing. TEXT and MTEXT objects can also go straight into `cam.profile`, `cam.pocket`, `cam.hatch` and `cam.engrave`; they are converted the same way. The stroke font (the default text style) has no letter shapes: it gives open lines that only `cam.engrave` can follow, and the result carries a warning. Read the `warnings`: overlapping letters (script fonts) can be misread by a pocket. Look at the picture from `render` before going on.

**Text on a path:** `cam.text_on_path {text, height, path (a handle to a line, arc, circle or polyline), font, side? ("above"|"below"), start?, align? ("left"|"center"|"right"), spacing?, kerning?: [{pair: "AV", adjust: -1.0}, ...], layer?: "TEXT_VECTORS"}` places text along a path. Each letter is rotated to the path's tangent. Text longer than the path stops at the end. Closed paths (a circle) wrap. The output is closed polylines on the given layer.

**Height maps:** `relief.heightmap` reads a grayscale picture (16-bit PNG or TIFF keeps every level; BMP also supported) or a 3D model (`.stl`, `.obj`, `.3mf`) and reports what it holds; it can write a shaded preview picture. It does not change the drawing or the job; `relief.add` keeps a map in the job, and `cam.relief` (rough or finish) cuts it. Embedding a model in the stock takes `floor`, `floor_percent` or `embed: true` after `cam.material`; read the `warnings` (a floor under 0.25 mm is flagged).

**Picture tone (`relief.heightmap` and `relief.add`, pictures only: png, jpg, tif, tiff, bmp):** optional `brightness?` (-1..1, 0 = unchanged), `contrast?` (-1..1, 0), `gamma?` (> 0, 1), `clip_low?` and `clip_high?` (percent of the darkest or lightest pixels ignored, each from 0 up to but not including 50, together under 100; 0 = off) and `smooth?` (Gaussian blur of the cell grid, sigma in cells, >= 0, 0 = off). A missing key or `null` means unchanged, so a call without them gives the same heights as before. A wrong type, a non-finite number or a value out of range is an error that names the key. 3D models (`.stl`, `.obj`, `.3mf`) and textures refuse these keys. Order: clip (each end that is on is stretched to full range), then contrast about the middle, brightness, gamma, then the `skip` merge, `invert`, `smooth`, and `height = base + t * relief_height`. The result lists the values under `tone` only when something was changed. 16-bit TIFF keeps every level; an RGB or RGBA floating-point TIFF is read as 0..1 and clamped; single-channel floating-point TIFF is not supported yet, save it as 16-bit. TIFF and BMP files are kept in the job and the project file like PNG and JPG. The Import 3D Model form starts a freshly chosen picture with clip 0.5 and 0.5 and smooth 1; the commands do not.

**Relief height inside the board (`relief.heightmap` and `relief.add`):** `relief_height_percent` (number above 0, a percent of the thickness) or `relief_height_in_stock` (number above 0, an exact height in the job's units) place the relief inside the stock; both need `cam.material`. The top is flush with the stock top, or `top_below` (0 or more, job units, default 0) under it, and `base` = top - top_below - height. A relief that does not fit (top_below + height deeper than the board) fails with a "cut into the spoil board" message, never clamped; exactly 100 percent is allowed with a warning. Giving both keys, either with `relief_height`, or either with `base`, `embed`, `floor`, `floor_percent` or `depth: "fit"`, is an error; `top_below` alone is an error unless it goes with a picture that has no height. The result carries `relief_height`, `relief_height_percent`, `relief_height_units`, `relief_height_default`, `top_below`, `base` and `relief_depth`. **Unchanged:** `relief_height` alone keeps its old meaning (base 0 unless given, so it can rise above the board); a picture with no material still needs `relief_height`; 3D models keep their natural height; `job.photo` and `job.model` use `depth`; textures keep an absolute `relief_height`. **Default changed:** a picture with a material set and none of `relief_height`, `relief_height_percent`, `relief_height_in_stock`, `base`, `embed`, `floor`, `floor_percent`, `depth` now gets 25 percent (`relief_height_default: true`, with a warning); it used to be an error. A relief that ends up above the stock top gets a warning from `relief.heightmap` and `relief.add` and an entry (`rises_above_material`, `warning`) in `relief.list`, and `cam.relief` refuses it with the fix in plain words. Helpers in `cadcraft_engine::cmd::relief`: `percent_depth`, `total_depth`, `fit_depth`, `convert_length`, `default_floor`. The Import 3D Model form offers % of board (default, 25), Exact height (mm or in), Fit to stock (it sends `embed: true` with `floor` set to `default_floor(units, fit)`, 1/8 inch or 3 mm, so the "Fit: ..." readout equals what is cut; a plain `embed: true` keeps its 1 mm / 0.04 inch floor) and, for models, Natural (the model default). Non-object parameters are an error, and numbers in messages are printed short.

**Automatic tone, border and depth (`relief.heightmap` and `relief.add`, pictures):** `tone: "auto"` chooses `clip_low`, `clip_high`, `gamma` and `smooth` from the picture and returns them under `tone` and, with plain-language `notes`, under `auto`. It is deterministic, and a picture that is already well leveled comes out nearly unchanged. Numbers you pass yourself (`smooth: 3`) win over the automatic ones. `trim_border: true` removes a uniform-color strip along any edge, and `crop: {left?, top?, right?, bottom?}` (whole pixels) cuts first, for example a corner label; `width` and `height` then describe the cropped picture, and `auto.trimmed` and `auto.crop` say what was removed. `depth: "fit"` (needs `cam.material`) sets `relief_height` to the stock minus a floor (3 mm or 1/8 inch; `floor` or `floor_percent` change it) with the top flush, and `bit` (a library id or tool, with `bit_stepover?`) or `max_depth` caps it: a V-bit of angle a and diameter d cuts to d / 2 / tan(a / 2), and `depth_fit: {relief_height, floor, capped, cap, reason}` plus a warning say when the cap applied. `bit` also tells `tone: "auto"` how fine the smoothing must be. `relief.add` stores the numbers chosen, not the request. A picture that is a plain 8-bit file has 256 levels: keep the smoothing, or ask for a 16-bit file. Help page: `help.page {"id": "photo-to-relief"}`.

**`job.photo`** `{path | data + format, width | height, material?: {cam.material params}, tone?: "auto"|"none", clip_low?, clip_high?: 0..<50 (as in relief.add; given numbers win over tone auto), trim_border?, crop?, depth_percent?: 1..100, smoothing?: "auto"|length, depth?: "fit"|number, floor?, floor_percent?, detail?: "tapered"(default)|"fine"|"normal"|"fast", position?: "center"|[x,y], feeds?, rough_max_diameter?, rough_tool?: a tool database id (an end mill, a ball nose or a tapered ball nose; "auto", null or absent keeps the automatic pick of the largest end mill up to `rough_max_diameter`; a V-bit or an unknown id is refused; the `rough` step reports the tool), name?, preview?: png path, simulate?: {path?, view?...}, keep_toolpaths?: false}`
A photograph to a checked relief job in one command. **Steps and progress (2026-11-01):** `job.photo` and `job.model` are step-by-step recipes (`CommandSpec::stepped`, trait `bg::Steps`, `ReliefJob` in `job_photo.rs`). Called with `engine.execute`, the control channel, MCP or a script they run in place and return the same result as before (plus `calculation`: `estimated_s` {rough, finish, total} and `actual_s` {per step name, total}); the window calls `Session::start_steps` and polls `poll_background`, so each long step (`relief.add`, the rough and finish `cam.relief`, `cam.verify`, `cam.simulate`) runs on the worker thread, `background_status().step` names it, and `cancel_background` rolls the whole job back. The rollback is by drawing: it puts the job back into the drawing that started it (not the one now open) and puts the undo and redo lists back whole (not trimmed). While a stepped job runs in the window, `execute` and `start` refuse (Busy, "`Photo to Relief` is still running: cancel it or wait for it to finish") any command that changes the drawing or the job (undoable ones and ids starting `cam.`, `relief.`, `job.`, `send.`), undo and redo (`u`, `undo`, `redo`, `mredo`), `xref`, `attach`, switching, creating and opening drawings (`document.switch`, `new`, `open`), close, save and project open/save, and a grip edit started from the canvas (`Session::step_job_running()` and `job_busy()` tell the window; the window shows the text on the status line); read-only ids (`.list`, `.info`, `.tools`, `.status`, `.query`) and view commands still work. View-only changes (zoom, regen, settings) no longer discard the job. A job that finishes inside `start_steps` is reported by the next `poll_background`. `background_status().step` carries `fraction` and `total` of its own, 0 when it reports nothing (show a spinner). Before the rough pass starts the recipe estimates both toolpaths (`librecarve_cam::relief_estimate`; `job_photo::estimate_calculation` does it from a preview's map for the form) and adds a warning "the rough/finish pass is estimated at about ... " when one is over two minutes (also "the whole job is estimated at ..."); after the rough pass the finish estimate is scaled by the real time. Every step logs `job step start/end` at INFO and `job step slow` at WARN over 30 s. **Depth (changed 2026-10-30, default changed 2026-10-31):** `depth_percent` is the relief depth as a percent of the board thickness, default 5 (changed from 10 on 2026-11-07; it used to be the board minus 1/8 inch or 3 mm); the Photo to Relief window offers 5, 10 and 15 percent presets, and any percent from 1 to 100 can be typed; the top is flush with the top of the material, more than 100 is taken as 100 with a warning, below 1 or not a number is an error, and a V-bit's reach still limits it, as does a tapered ball nose's flute length (a plain ball nose does not). It cannot be combined with `depth`, `floor`, `floor_percent` or `relief_height_*`, which keep their meaning. The picture step reports `depth_percent: {depth_percent, requested_depth, depth_cut, limited_by_bit, toolpath_depth, thickness, clamped}`: `depth_cut` is the relief's depth, `toolpath_depth` how far below the top the finish toolpath goes (it can stop above the relief's deepest point where the bit does not fit; never below), and the finish step has `bottom_z`. `smoothing` (also on `relief.heightmap` and `relief.add`) is `"auto"` (default, the blur chosen for the finish bit) or the blur as a length in job units, 0 for none; it replaces the automatic blur, works on an estimated depth map too, and is refused for 3D models, textures and together with `smooth`. The Quick Start picture track sends the same 5 percent default (`floor_percent` 95). In order, each step reported in `steps`: material (if given, else the job's), `relief.add` with auto tone and depth fit (the finish bit sets the smoothing and caps the depth), the picture's position (centered on the stock, or its lower-left corner at `[x,y]`), a rough `cam.relief` with the largest end mill up to 1/4 inch from `cam.tool.pick` (allowance 0.5 mm or 0.02 inch), a finish `cam.relief` (`fine`: 20 degree V-bit `v-020-0.250`, 0.10 mm passes; `normal`: 60 degree V-bit `v-060-0.250`, 0.3 mm; `fast`: 1/8 inch ball nose `bn-0.125`, 0.8 mm), `cam.verify` and `cam.simulate`. **It never posts.** A job that passes the check returns `status: "awaiting_approval"`, `job_id`, `toolpaths`, `total_time_s` (an estimate), `auto`, `depth_fit`, `verify`, `simulate` and `warnings`: report all of them, then run `cam.post` and `job.approve` after the person says yes. If any step fails, including `cam.verify`, the drawing and the job are put back as they were and the error names the step. It replaces the job's toolpaths unless `keep_toolpaths` is true. When it replaces them, it also removes every height map and frame shape that an earlier `job.photo` run added (maps with the same `recipe` tag; `job.model` likewise removes only `job.model` maps, and Quick Start maps, tagged `job.quickstart`, are left to Quick Start); maps added any other way are never removed. Only the rectangle the recipe drew (`recipe_frame`) is erased, linked or not, and only while it is on the 3D frame layer. `relief.frame` with no shape draws a new rectangle and keeps the map the recipe's (that rectangle becomes `recipe_frame`); a map relinked with `relief.frame` or `relief.fit` to a shape the person gives loses its tag and is the person's, and the recipe's old rectangle is erased then (`old_frame_erased` in the result, or `old_frame_kept` with a warning when the layer is locked; a rectangle the person fitted the map to is never erased). A frame on a locked layer is kept with a warning. What was removed comes back as a `replaced` step: `maps: [{name, frame, frame_kept}]`. Tell the person the estimated time before they choose the detail. `finish_tool` (any ball nose or V-bit: a library id or a tool as `cam.relief` takes it; an end mill or thread mill is refused; overrides `detail`) and `stepover` (job units; default a V-bit 0.005 mm per degree of tip angle, a ball nose the ridge height of `fast`) replace the three presets; the depth fit uses that bit's real geometry, and a bit that reaches almost nothing is refused with the reason. `job.suggest` takes the same two keys. `cam.relief` (and the 3D Relief Toolpath window) cuts any height map in the job whatever its source; `job.photo` is the shortcut that adds the tone, the depth fit, the rough and finish toolpaths, the check and the simulation. By hand: `relief.add` a picture (png, jpg, tif, tiff, bmp) with `width` and `relief_height`, then `cam.relief` with any tool, any stepover and `handles` to cut only inside shapes; you set the tone, the depth and the checks.

**Relief size estimate and the memory warning (item 114).** Before a relief toolpath is built, `cam.relief` estimates how big it will be and compares that with the memory this computer has free. Its result carries `est_moves` (moves in the finished toolpath), `est_memory_mb` (peak MiB: 96 bytes a move, because a move is 48 bytes and the list doubles as it grows, plus about 40 bytes a move for the G-code text) and `available_mb` (free MiB right now; `null` where the system cannot say, as on the web). When `est_memory_mb` is more than `available_mb`, `warnings` gets, for example, "Estimated 14.0 GB; this computer has 8.0 GB free: use a larger stepover, cut a smaller area, or split the job". The warning never stops the job; only the move budget does (60 percent of free memory, never below 20 million moves, read on Linux, Windows and macOS, bounded by the cgroup limit on Linux). `job.photo` and `job.model` put `est_moves` and `est_memory_mb` on their `rough` and `finish` steps, the sums of both and `available_mb` at the top of the result (`available_mb` is left off the steps because it changes from moment to moment), and the warnings under their step names. The estimate is `moves = tips x keep x (1 + (layers - 1) x fill)` for roughing and `tips x keep` for finishing, where `tips = area / (stepover x sample)` and `sample = max(cell / 2, tool diameter / 16)`; with no shape given it uses the whole map, so a cut limited to shapes is estimated high (safe side). The function is `librecarve_cam::relief_size` (`size_report`, `memory_warning`, `available_mb`); calibration and the 8, 16, 32 and 64 GB tiers are tested in `crates/cam/src/relief_size_tests.rs`.

**Quick start (2.5D): `job.suggest`, `job.fit`, `job.quickstart`.** Flat artwork cut at constant depths (profile, pocket, V-carve, engrave, drill); a 3D relief has `job.photo`. Help page: `help.page {"id": "quick-start"}`.

**`job.suggest`** `{material?: wood name, layers?: [names] | handles?: [hex]}` is read-only: it never makes a toolpath or changes the drawing. It needs `cam.material`. Default objects: the selection, else everything visible. Returns `{units, material, thickness, design: {bounds, size}, counts, operations, total_time_s, warnings, next}`. Each operation is `{id: drill|pocket|vcarve|engrave|outside, op, side?, role, handles, shapes, cut_depth, through, tabs?, tool, tool_name, why, feeds, use: {tool, feeds}, candidates: [{id, name, kind, diameter, feed, why}], alternatives, est_time_s (rough), warnings}`, in cutting order (the part last). Rules: a closed shape not inside another is a part (profile outside, cut through, `tabs: true` when the material is held by tabs); a closed shape inside a part is a hole (pocket at half the thickness; circles of 1/4 inch or less, and points, are drilled); text is V-carved; open lines and arcs are engraved. Shapes inside a group (an imported file kept as one block) cannot be given different operations: it warns, so break the group apart. The bit is `cam.tool.pick`'s first candidate; pass an operation's `use` and `handles` to its toolpath command.

**`job.fit`** `{mode?: "fit" (default: only smaller, when larger than the room) | "fill" (largest size that fits) | "size" (width and/or height), width?, height?, margin? (default 0), position?: "center" (default) | "lower_left" | [x, y], handles? | layers?}`: `scalefit` plus a move, in one command, keeping proportions. The room is the stock (`cam.material size`) minus the margin on every side. Returns `{mode, factor, was, now, origin, room, warnings}`; a change of more than 10 times either way warns (usually wrong units). Toolpaths made earlier no longer match: it warns.

**`job.quickstart`** `{material: {cam.material params}, design?: {path | data + name} | {trace: {cam.trace params}} | {model: {job.model params}} | {handles} | {layers} (default: what is drawn), size?: {job.fit params} | false (default: fit, margin 1/2 inch), operations?: "suggest" | [ids] | [{id, tool?, cut_depth?}], tidy?: true, feeds?, simulate?: false | {path?, view?...}, name?}`. In order, each step reported in `steps`: material, import or trace (loose shapes, not a group), tidy and shape check, size, suggest, one toolpath per chosen operation (the suggested bit unless `tool` is given), `cam.verify`, `cam.simulate`. It starts with no toolpaths. It never posts: the result is `awaiting_approval` with `suggestion`, `toolpaths`, `total_time_s`, `verify`, `simulate` and `warnings`; read them, then `cam.post` and `job.approve`. Any failed step, including verify, puts the drawing and the job back as they were. `tidy` is skipped when `design.handles` are given. With `design.model` it is `job.model` (below) and the result has `track: "model"`; `size.width`, `size.height` and `size.position` give the model's size. A job that holds a height map gets `job.suggest` operations `rough` and `finish` (`{op: "relief", mode, tool, allowance?, stepover?, candidates}`; the finish `candidates` are the three detail levels, `detail?: fine|normal|fast`, `rough_max_diameter?`); a 2.5D `job.quickstart` leaves them out unless their ids are listed.

**`job.model`** `{path | data + format, width | height, material?: {cam.material params}, up_axis?: "z"|"y"|"x", scale?, cell?, mirror_x?, mirror_y?, rotate?: 0|90|180|270, depth?: "fit"|number, floor?, floor_percent?, detail?: "fine"|"normal"|"fast", position?: "center"|[x,y], feeds?, rough_max_diameter?, rough_tool? (as in job.photo), name?, preview?: png path, simulate?: {path?, view?...} | false, keep_toolpaths?: false}`: `job.photo` for an STL, OBJ or 3MF model, sharing its steps (`steps` reports material, model, position, rough, finish): `relief.add` reads the model as a height map (the long side is 512 cells unless `cell` is given), fits the depth to the stock and the finish bit, `cam.tool.pick` chooses the largest end mill up to 1/4 inch for the rough toolpath, the finish bit follows `detail`, then `cam.verify` and `cam.simulate`. Never posts; any failed step restores the job. Help page: `help.page {"id": "quick-start"}`.

**Depth from a photo: `relief.depth_from_photo`, and the source keys on `relief.heightmap`, `relief.add` and `job.photo`.** `heights_from: "brightness"` (default, unchanged) reads the picture's brightness as height. `"depth_model"` estimates depth from the photo with the installed depth model, and `"shading"` estimates it from light and shade with no model. `near: "bright" | "dark"` says which end of a picture is near (the carve top); `near: "dark"` flips the picture once, and `invert` on top of it is an error (`effective_invert` is the single flip: `near` flips XOR `invert`). Estimates are always near-high, so `invert` is refused with `depth_model` and `shading`; use `near` for a depth map you read from a file. Estimate keys: `detail` (`"normal"` 518, `"fine"` 770, `"extra_fine"` 1022 or an integer 140..4096; `job.photo` calls it `depth_detail`), `sharp_edges`, `flatten` (0..90 percent, or the exact string `"auto"`; the Photo to Relief window's Background to the floor slider, sent only above 0 or as `"auto"`), `compress`, `photo_detail` (0..100 percent or the exact string `"auto"`, default 0: adds the photo's own fine detail to the estimated depth where there is relief to carve; the part laid flat by `flatten` stays flat and depth just above the floor gets only part of the detail; try 30 to 50 with the depth model, 10 to 20 with shading; both methods), `light` (shading), `threads` (0..1024, default the job's), `rerun_depth` (taken by `relief.add`, `job.photo` and `relief.depth_from_photo`: it makes the depth map again instead of using the one kept in the job or in memory; `relief.heightmap` keeps nothing and refuses it). They are refused with brightness, with 3D models and with textures; a `model: {path}` is not offered. `crop` and `trim_border` apply to the photo before the model; `tone` applies to the depth map. The depth map is a 16-bit PNG kept in the job (`relief.list` shows `heights_from` and `depth_bytes`) and in the project, so a saved project rebuilds the same toolpath on a computer without the model; the same photo, crop and settings reuse it unless `rerun_depth: true`. `"auto"` (the window's Auto boxes, on by default) is decided by the engine from the picture, with fixed rules: `flatten` is the share of the depth map in the far band (within 0.15 of the range above its minimum) when that band is at least 10 percent of the picture and piles up near the minimum (a plain slope or a full-frame texture gives 0), capped at 60; `photo_detail` is 7 to 45 by how much fine detail the photo has over the part of the depth map above the floor (at most 20 for `shading`; 0 when nothing is above the floor). Like `tone` and `rough_tool`, `flatten` and `photo_detail` need the exact lowercase word `"auto"`; any other string, or a number out of range, is an error; the word is refused with brightness, 3D models and textures like the numbers. The picks are recorded as numbers in the depth map's settings (`depth.auto_picked` has the value and the words), so a reused or rebuilt map is identical (`auto_picked.rules` is the version of the rules that picked), a stored Auto map is reused for later Auto calls on the same photo, so existing jobs keep their old pick unless `rerun_depth: true` is given, the same call with those numbers reuses it, and the result says what was chosen in `auto.notes` ("Background to the floor: chose 45% because ...") and `auto.auto_flatten.value` / `auto.auto_photo_detail.value`. The result carries `engine` (name, version, threads), `model` (id, sha256), `detail_px`, `time_s` and `notes`. `relief.depth_from_photo` takes the photo (`path` or `data` + `format`), `method` (`"depth_model"` default, or `"shading"`, the same words as `heights_from`), the estimate keys, `near` (only how the picture is drawn; the job always carves near-high) and `cpu_percent`, and writes the depth map to `output` (a PNG with a `librecarve:near` note) or only reports with `preview`; it never opens a dialog. The depth map fits the board exactly like Photo to Relief: `relief_height_percent` (default 25), `relief_height_in_stock`, `top_below`, `depth: "fit"` (and, on `job.photo`, the same keys); near is the carve top, far is the floor, and a relief that does not fit is refused, never clamped.

**`relief.depth_model`** (Tools > Depth Model...). `{}` reports what is installed (`status`: `installed`, `catalog`, `download_available`, and `installing` while a download runs; the file's checksum is read once and remembered, so polling is cheap), `{use_file: path}` installs a model file only if its size and SHA-256 match the catalog (no consent and no network: it installs a verified local copy; a file of another size is refused from its length alone, never read; the refusals are plain sentences such as "This is not the Depth Anything V2 Small file LibreCarve expects (99 MB). Download it again from <page>.", "LibreCarve could not find the file x.onnx." or "x is a folder, not a file."). The app's window has a **Choose file...** button for the same call, which only a person's own click can press; `status` lists the catalog's pinned `file_url` of each model, `{remove: id}` removes it, `{install: true, consent: true}` (or `download: true`) downloads the model (about 99 MB, once) from the LibreCarve releases page on GitHub and installs it only after the minisign signature on the checksum list, the exact size and the SHA-256 built into the app all agree, and `{cancel: true}` stops a running download and leaves no file. **The release packages carry the model** (Linux tarball, Windows zip, macOS app), so `status` normally says `included: true` and `installed[0].location: "Included with LibreCarve"`: nothing to download, install or remove (`install` answers that it is included, `use_file` is refused, `remove` answers "The model is part of LibreCarve and cannot be removed. Delete LibreCarve to remove it."). `bundled_damaged: true` means the copy that came with LibreCarve fails its size or SHA-256 check and is not used ("The depth model that came with LibreCarve is damaged. Download it again or install a fresh copy of LibreCarve."); then the download below is the repair. Developer builds and `cargo run` have no bundled copy and behave as before. **Never pass `consent: true` on your own: ask the person first and pass it only after they agree to the download.** Without it the call is refused. The call returns `{started: true}` at once and the download runs on a thread of its own (stages: Checking the signature, Downloading the model, Checking the download, Installing), one at a time. Poll `{}`: `installing: {stage, bytes, total, percent, cancelling}` while it runs (poll about once a second, not in a tight loop), and when it is over `installing` is `null` and `last_install: {ok, cancelled, already_installed, message}` says whether the model was installed or why not, in plain words for the person (it carries no command syntax, so work out what to do from it: if it says the file has not been published yet, do not retry, use shading or brightness; if the computer has no `curl`, is offline or is out of disk space, tell the person); it stays until the next install starts and does not depend on the drawing. A failed install changes nothing. A model that is installed and correct is not downloaded again. The command line tool, the MCP server and the browser build have no network access and answer `install` with "not available" (use `use_file` there). Any failure deletes the partial file and says which check failed; nothing is installed unless every check passes. If the model is absent, tell the person it needs downloading once and ask whether to do it; shading and brightness work without it. Using your own model file (`model: {path}`) is not offered. The release programs (the app and the command line tool) are built with the engine (cargo feature `depth-model`, on by default in `apps/cadcraft` and `apps/cadcraft-cli`). A build made with `--no-default-features` has no engine and answers "This copy of LibreCarve was built without the depth model engine, so it cannot estimate depth from a photo. Brightness and Shading still work." (in that case use `heights_from: "shading"` or `"brightness"`, which need no engine, or ask the person for a normal build). **What a failed install says.** The words are for the person and carry no numbers or system text: not published yet (a 404 or 410), no connection, a refused secure connection, a stalled connection, no `curl`, "The server had a problem. Try again later." for any other server answer, "There is not enough free space on this disk..." for a full disk, "LibreCarve could not write to its models folder. Check that the folder is not read-only." for any other folder problem, a file that did not pass the size and checksum checks, and a checksum list whose signature did not verify. **While the release `models-1` of LibreCarve-releases has no model file** (the owner publishes it with the `publish-models` workflow), every download answers "has not been published yet": do not retry, use shading or brightness. `remove`, a new install and `use_file` also drop the depth map kept in memory, so the next estimate runs the model that is there (or says that none is installed) instead of handing back the picture of the model that is gone.

**Automatic tone on an estimate** (`tone: "auto"`, which `job.photo` sends by default) only smooths: it does not clip or change the gamma of a depth map (the result notes say so); give `gamma`, `clip_low` or `clip_high` yourself if you want them. With no `threads`, the engine uses the job's thread count but never more than the physical cores (at most 8). A kept depth map is reused only for the same photo, crop, settings, model file and engine version. A project with a depth map is saved as version 2, so an older LibreCarve says "update" instead of carving the photo as plain brightness; on opening, a damaged or hand-edited depth map (missing, empty, wrong size, not a 16-bit gray PNG, or with `near`/`invert` added to the settings) is refused in plain words. A relative `CADCRAFT_CONFIG_DIR` is refused. `relief.depth_model` is in the Tools menu as Depth Model... (it opens the Depth model window) and Depth Map from Photo... is in the Toolpaths menu, with the 3D relief entries (it opens Photo to Relief with the depth model chosen); with parameters both run as given and never open a window.

**In the desktop app an install needs the person's own click.** `relief.depth_model` with `install: true` (or `download: true`) sent through `app.run`, `engine.execute` or the control channel is not run: LibreCarve shows the question "Download the depth model?" in its window, saying a program asked, and answers `{started: false, awaiting_person: true, message}` (or `{started: false, already_installed: true}` when the model is there). Do not ask again until the person has answered; poll `{}` about once a second for `installing` and `last_install`. A `ui.click` or `ui.key` cannot press Download: the window refuses input that came from the control channel. The control channel shows the person the question; a script gets an error. A script or `cmdline.script` line with `consent: true` is refused too: the app turns on the engine's install permit (`depth_model::require_install_permit`), which only the window's Download button grants, and the error says the person has to confirm in the LibreCarve window. The command line tool and the MCP server have no window and keep the plain `consent: true` rule (they also have no network transport).

**Photo to Relief in the window (for an agent that watches it).** Preview and Create job first stop the depth map shown beside the photo and then run once, however often the button is pressed; Create job with an estimate makes the depth map on the window's own thread, so the window can stop responding for up to a minute (a depth map made a moment before for the same photo and settings is reused). Not offered yet, and so not an error to work around: a model path of the person's own (needs a child process and an operator allowlist), a **Save depth map...** button (use `relief.depth_from_photo` with `output`), a Near choice in Import 3D Model (for a picture, `near` on `relief.heightmap` does it), and `job.photo` as a background command.

**Orientation of a height map (`relief.heightmap` and `relief.add`, picture or model):** a model can arrive mirrored or turned compared with other software. Fix it with `mirror_x?: bool` (left-right), `mirror_y?: bool` (front-back), `rotate?: 0|90|180|270` (degrees counter-clockwise seen from above; `-90` means 270; anything else is an error) and `invert?: bool` (upside down: white is the bottom). They apply to the finished map in this order: mirror_x, then mirror_y, then rotate, so previews, toolpaths and the simulation all agree. `width` and `height` always describe the finished (turned) map, and `origin` stays its lower-left corner. When the cell size does not divide the picture size, the grid reaches less than one cell past the picture; the picture stays where it was and the reported grid corner `origin` moves down or left by that strip. A stored height map without these keys behaves exactly as before. Textures (`relief.texture`, or `relief.add` with `kind`) do not take these options: `mirror_x`, `mirror_y` and `rotate` are refused with an error there. Check with `preview: true` before adding.

**What a relief job shows (canvas and exports).** With a height map in the job, the design window paints a shaded top view of the carve under the drawing: the simulated stock (the `cam.simulate` top view, cropped to the stock) once the job has a relief toolpath, else the shaded height map (the `relief.heightmap` preview shading). The picture therefore changes as the job is built (smooth height-map shading, then the coarse rough-pass simulation, then the finish simulation with its stepover lines and, for 8-bit pictures, gray-level steps); that is expected, not a bug, and the help page `relief-toolpath` says so. It is a display overlay, not an entity, and it is rebuilt in the background only when the height map, the toolpaths or the stock change. The raster scan lines `cam.relief` draws on the `TOOLPATHS` layer are not drawn on the canvas; select the toolpath in the Toolpaths list and check **Show path** to see them (`cam.delete`, undo, `cam.verify`, `cam.simulate` and `cam.post` are unchanged, and the entities stay in the drawing and in DXF). `export`, `save`/`saveas` to `.svg`, `.png` or `.pdf`, and `plot` of model space put the same picture under the vectors (an `<image>` with a base64 PNG in SVG), at its real position and size, at most 2048 pixels on the longer side, and leave the relief scan lines out. A job without a height map exports exactly as before. Exporting a relief job with a relief toolpath runs the simulation, which can take several seconds on a large stock.

**Vector clean-up:** `cam.vector.boolean {op: "union"|"subtract"|"intersect"|"xor", handles?, keep?}` and
`cam.vector.offset {distance, side: "outside"|"inside"|"left"|"right", handles?}`.

**Nesting parts on sheets:** `cam.nest {handles?, sheet?: [w,h], margin?, gap?, tool?, clearance?, rotation?: "none"|degrees, mirror?: false, quantity?: 1, grain?: "free"|"lock"|"lock180", parts?: [{handle, quantity?, grain?}], in_holes?: true, max_sheets?, max_iterations?: 5000000, time_limit?: 30, at?: [x,y], spacing?, layer?: "NEST", create_sheets?: false, keep_originals?: true, dry_run?: false, tolerance?}`
Arranges the selected parts (or `handles`) by their real outlines on sheets. Each outermost closed shape is a part; its holes, the shapes inside its holes and open lines inside it move with it. `sheet` defaults to the active setup's material `size`. Give `gap` (the smallest distance between parts), or `tool` (a library id or `{diameter}`) to make the gap the bit diameter plus `clearance` (1 mm or 0.04 inch); one of them is required, so ask the person for the bit rather than guessing. `margin` defaults to half the gap. `rotation` defaults to 90 (degrees per step; "none" never turns parts). `grain: "lock"` never turns a part and `"lock180"` allows only a half turn; set it per part through `parts`. Copies are drawn on layers `NEST_SHEET_1`, `NEST_SHEET_2`, ... with sheet 1's lower-left corner at `at` (default: right of the parts) and the others in a row `spacing` apart, and they are left selected. `create_sheets: true` adds one setup per sheet (size, and origin at the sheet's corner), not activated. The originals stay unless `keep_originals: false`. The drawing change is one undo step; setups it made are not undone. Result: `placed`, `not_placed`, `sheets` (`index`, `layer`, `origin`, `size`, `parts`, `utilization` percent, `setup`), `copies` (`source` handle, `sheet`, `copy`, `angle`, `mirrored`, `handles`), `handles`, `unplaced` (`source`, `copies`, `reason`), `warnings`, `iterations`, `stopped` (null, `"iterations"` or `"time"`), `gap`, `margin`. Report every `unplaced` row and warning to the person: nothing is dropped silently, but nothing is placed either. The layout is the same for the same input unless the time limit stopped it. `dry_run: true` reports without drawing. Help page: `help.page {"id": "nesting"}`.

```json
{"command": "cam.nest", "params": {"tool": "em-0.250-up", "quantity": 4, "rotation": 90, "grain": "lock180", "create_sheets": true}}
```

### Default tools

The owner's go-to bits are six library tools. Choose from these first and use another only if the person
asks. Pass the id as `tool`, for example `"tool":{"id":"em-0.250-up"}`.

| Bit | End mill id | Ball nose id |
| --- | --- | --- |
| 1/4" | `em-0.250-up` | `bn-0.250` |
| 1/32" | `em-0.031-up` | `bn-0.031` |
| 1/64" | `em-0.016-up` | `bn-0.016` |

Use the 1/4" for clearing and profiles, the 1/32" for fine work and the 1/64" for the finest detail. Use
a ball nose for relief and texture finishing and an end mill for flat work. Do not add tools to the
library unless the person asks.

### Toolpaths

Every toolpath command takes `tool` as either a library id (a string) or a full tool object
`{kind, diameter, pass_depth, feed, plunge, rpm, stepover?, angle_deg?, name?}`. With a library id, pass
`feeds: "<material>"` to pick that material's feed and plunge. They use the current selection unless you
pass `handles: [hex]`. They add a toolpath and draw it on the `TOOLPATHS` layer, which is never treated as
artwork.

The long commands (`cam.profile`, `cam.pocket`, `cam.drill`, `cam.hatch`, `cam.engrave`, `cam.vcarve`,
`cam.relief`, `cam.chamfer`, `cam.countersink`, `cam.thread`, `cam.nest`, `cam.trace`, `cam.import`,
`cam.verify`, `cam.simulate`, `relief.heightmap`, `relief.texture`, `relief.add`) always return the whole
result before your call returns: only the window runs them in the background. Each also takes
`cpu_percent?: 1-100`, the share of the processor cores the calculation may use (default: the
Preferences setting, 50% unless changed). Each logs one INFO line when it starts and one when it ends
(command, shape count, threads, seconds) in the log file, and a WARN if it ran over 10 seconds.

| Command | Parameters beyond `tool` |
| --- | --- |
| `cam.profile` | `side?: "outside"\|"inside"\|"on"` (open shapes only allow `on`), `direction?: "climb"\|"conventional"`, `cut_depth`, `start_depth?`, `pass_depth?`, `allowance?`, `ramp?: {length}`, `tabs?: {count, width, thickness}\|false`, `tolerance?` |
| `cam.pocket` | `direction?`, `cut_depth`, `start_depth?`, `pass_depth?`, `stepover?`, `allowance?`, `ramp?: {length}`, `band?` (cut only rings within this distance of walls and islands), `rest_tool?` (library id of a larger tool that already cleared the area: this tool then cuts only what it left, within reach of the walls and islands), `rest_allowance?` (the allowance that larger pocket used), `tolerance?` |
| `cam.hatch` | `cut_depth`, `start_depth?`, `pass_depth?`, `angle_deg?`, `spacing?`, `cross_hatch?`, `outline?`, `allowance?`, `tolerance?` |
| `cam.engrave` | `cut_depth`, `start_depth?` (one pass along the vectors) |
| `cam.drill` | `cut_depth`, `start_depth?`, `peck?`, `retract?`, `clearance?`. Drills at the centers of circles and points |
| `cam.chamfer` | V-bit only. `width?`, `depth?` (give one or both; the bit's angle sets the other and both must agree with it), `angle_deg?` (the face's angle from the top face, 45 for a 90 degree bit; must match the bit), `side?: "outside"\|"inside"` (closed shapes; on open vectors the tip runs on the line), `tip_offset?` (tip runs this far into the waste, deeper by `tip_offset / tan(half angle)`; closed shapes only), `pass_depth?`, `direction?`, `tolerance?` |
| `cam.countersink` | V-bit or countersink bit. `diameter` (at the top; the bit's angle gives the depth), `angle_deg?` (the screw head's angle, 82 for inch screws; must match the bit), `path?: "spiral"\|"concentric"\|"plunge"` (default spiral), `stepover?` (spiral and concentric: **no default** for a V-bit), `direction?`, `tolerance?`, `pilot?: {tool, feeds?, cut_depth, start_depth?, peck?, retract?, clearance?}`. At the centers of circles and points |
| `cam.thread` | `size?` (`M6`, `M8x1`, `1/4-20`, `#10-32`) or `major?` with `pitch?` or `tpi?`, `length`, `start_depth?`, `side?: "internal"\|"external"`, `hand?: "right"\|"left"`, `direction?: "climb"\|"conventional"`, `passes?` (1 to 100), `spring_passes?` (0 to 10), `hole_diameter?` (points only). Threads at the centers of circles (their diameter is the hole or boss) and points. `tool` must be a thread mill |
| `cam.vcarve` | V-bit only. `cut_depth` (deepest the tip may go), `handles?`, `tolerance?`, `flat_depth?` (flat bottom: the floor depth, within cut_depth and the cone's reach), `flat_tool?` (one end mill clears the floor as a second toolpath, made after the V-bit), `flat_pass_depth?`, `flat_stepover?`, `clear_tools?` (stacked V-carve, see below), `finish_last?: true` |

**Thread milling (`cam.thread`).** The tool must be a thread mill (`kind: "thread_mill"`; library ids
`tm-sp-0.150`, single-point, 0.5 to 1.5 mm pitch, and `tm-mf-1.0-4.6`, multi-form, 1 mm pitch). A full tool
object adds `pitch_min`, `pitch_max`, `cutting_length` (reach from the tip), `rows?` (1), `angle_deg?` (60) and
`tip_offset?` (the lowest tooth's height above the tip, 0). If `cam.tools {"kind": "thread_mill"}` lists none,
the person's database predates them: `cam.tool.defaults` adds the predefined tools that are missing. The
table covers metric coarse and fine M3 to M20 and unified UNC and UNF #2 to 1 inch; anything else needs
`major` and `pitch` (or `tpi`). An internal thread needs the hole drilled first, at about the minor diameter
and deeper than the thread by the tool's `tip_offset` (a warning gives the depth); an external thread needs the
material around the boss cleared. The result adds `thread` with `major`, `minor`, `pitch_diameter`, `pitch`,
`tpi`, `turns`, `helix` (`up` or `down`), `arc` (`G2` or `G3`), `orbit_radii` and `tool_bottom_depth`; quote
the diameters to the person. Errors (not wrong cuts): the tool is not smaller than the minor diameter of an
internal thread, the hole is already at the major diameter, the boss is at the minor diameter, the pitch is
outside the tool's range, the tool cannot reach, or the thread is deeper than the material. The posts write
helical `G2`/`G3` with `Z`; with `arcs: false` (or a post definition with `helical: false`) the helix becomes
lines within 0.01 mm.

Stepover: an end mill defaults to 40% of its diameter. A ball nose or V-bit has **no default** (they are
finishing bits), so `cam.pocket` and `cam.hatch` return an error until you give `stepover` (or `spacing` for
hatch) or the tool has one. Do not guess a number: for finishing, 5% to 20% of the diameter is usual, smaller for
a smoother surface, and ask the person if the finish matters.

Result of each: `{index, name, moves, passes, time_s, warnings, skipped, drawn}`. `index` is the toolpath's
number for later commands. `skipped` counts selected objects that could not be used. A toolpath whose
`warnings` is not empty must be reported.

**Stacked V-carve.** For a big carving, give `cam.vcarve` a `flat_depth` and `clear_tools`: a list (at most 64) of end mills,
each a library id, a full tool or `{id, feed?, plunge?, rpm?, pass_depth?, stepover?}`, **biggest first, each strictly
smaller than the one before** (an equal or bigger one is an error). The first tool clears the whole floor down to
`flat_depth`; each next tool cuts only what the one before left (rest machining); the V-bit toolpath is made last
(`finish_last`, true by default; false makes it first). `flat_pass_depth` and `flat_stepover` are only
defaults, for the clearing tools that give none of their own: a per-tool `{id, pass_depth}` or `{id, stepover}` always
wins (so a 1/32" bit can take 0.01" passes while the 1/4" bit takes the global 0.1"), and a full tool always gives its own
pass depth. A stepover cannot be more than a tool's diameter, so give a stepover per tool when the sizes
differ. Each toolpath's `tool.pass_depth` and `tool.stepover` in the result are the numbers actually used. `finish_last`
only applies with `clear_tools` (without them it is an error). The owner's default stack is the 1/4" end mill (`em-0.250-up`), then the 1/32" end mill (`em-0.031-up`), then the
V-bit; add the 1/64" end mill (`em-0.016-up`) before the V-bit for fine detail. `clear_tools` needs `flat_depth` and
cannot be used with `flat_tool` (the older single-tool form, unchanged: V-bit first, then the floor). A shape too narrow
for a clearing tool is skipped by that tool and still gets the V-bit; `skipped_tools` and `warnings` say which. The
result is the V-bit toolpath's, plus `toolpaths: [{index, name, tool, moves, warnings}, ...]` in the order the toolpaths
were made. A `job.run` operation `{"op": "vcarve", ...}` takes the same parameters, and `tools` aliases work in
`flat_tool` and `clear_tools`.

**Chamfer and countersink sizes come from the bit.** A V-bit cuts only at its own angle: a 90 degree bit
cuts 45 degree chamfers as deep as they are wide, and a countersink as deep as half its diameter. Ask the
person for the chamfer width (or the screw head diameter and angle) and pick a bit of the right angle; do
not change the angle to fit. A request the bit cannot cut is an error that says why (the bit is too small,
the width and depth need another angle, the cut would go below the material), and nothing is added. `cam.chamfer`
returns `chamfer: {width, depth, angle_deg, tip_offset, tip_depth}`; `cam.countersink` returns
`countersink: {diameter, depth, path}`, `holes`, and with `pilot` also `pilot` (the drill toolpath's own result,
which comes first in the list). The library's 82 degree countersink bits are `v-082-0.500` and `v-082-0.750`.
`cam.verify` checks a chamfer against its face and a countersink against its cone. They are not yet `job.run`
operations or `cam.tool.pick` operations.

### Textures and fine detail

**Always do the final clean-up with the 1/64" bit.** A large ball nose cannot follow small letters,
corners or thin shapes. Where a texture or relief meets lettering, a border or any fine edge, the big bit
leaves ragged edges. After the relief pass, trace those edges with the 1/64" end mill (`em-0.016-up`) down
to the floor of the texture, so every edge has a clean vertical wall. The owner tested this: the 1/64" bit
is needed for the final clean-up of letters. Use the 1/32" (`em-0.031-up`) only for large, plain shapes,
or as an earlier rough clean-up before the 1/64" pass. Use a 1/64" ball nose (`bn-0.016`) instead when
the edge should be rounded or the floor is curved.

- Run the clean-up after the relief pass and before any cut that frees the part (a profile through the
  stock goes last).
- For letters, trace outer shapes with `side: "outside"` and the closed loops inside letters (the holes in
  O, a, e, d, p, h) with `side: "inside"`. A loop is a shape that lies fully inside another shape, so check
  the bounds before you choose the side. `outside` on a loop cuts into the letter.
- A 1/32" bit reported a 0.0004 inch gouge at a sharp letter corner and the 1/64" bit cleared it. If
  `cam.verify` reports a gouge, make sure you are using the 1/64" bit. Do not add an `allowance` to fix
  it: that makes it worse.
- Say how long it takes. A 1/64" bit is slow (about 66 minutes for the letters in a 10 inch sign). Tell the
  person the time and let them pick the bit.

**Wave texture (`relief.texture` or `relief.add` with `kind: "waves"`).** Settings that work:

| Setting | Use | Why |
| --- | --- | --- |
| `period` | 0.4 to 0.5 inch (10 to 13 mm) | Readable waves at sign size. Under 0.25 inch they blur into noise. |
| `warp` | 0.1 (in periods) | Gentle sway. With warp 0.5 and warp_size 0.3 the waves broke up into a random bumpy surface. |
| `warp_size` | 2 | A long, smooth sway. A small number makes the waves jitter. |
| `angle` | 0 (horizontal) | Any angle works; keep it the same across a job. |
| `relief_height` | 0.039 inch (1 mm) | Wave height from valley to ridge. |
| `base` | the negative of the depth of the valley bottom | `base` is the height of the deepest point. |

Give a texture next to raised lettering or a border a **pedestal**: put the ridges about 1/16 inch (1.5 mm)
below the top of the stock so the letters and border stand proud of the waves. With ridges level with the
top, the letters have no edge and disappear into the texture. For 1 mm waves, that is `relief_height` 0.039
and `base` -0.102 (valleys 0.102 inch down, ridges 0.0625 inch down).

To cut texture around letters, use `relief.fit` or pass `handles` to `cam.relief` with the inner shape and
every letter ring. Letters then act as islands and are left alone. Cut with a 1/8" ball nose and a stepover
of 0.02 inch; a finer stepover only adds time (0.012 inch took about 57 minutes against 36). Before you
trust the result, run `cam.simulate` with `view: "3d"`, `yaw: 0`, `pitch: 55`, `z_scale: 3`, and look at
the picture.

### Review and check

**`cam.list`** `{accel?}` returns `toolpaths` (each `index, name, kind, tool, time_s, moves`) and the
`material`. `time_s` is an estimate. It assumes 300 mm/s² acceleration unless you pass `accel`; the real
machine's acceleration has not been measured, so quote times as estimates.

**`cam.verify`** `{tol?, setup?, allow_meet?}` checks every toolpath point against the shapes (a thread: against its major and
minor diameters and the thread's bottom; in a two-sided job, the two faces together: see "Two-sided jobs"). Success returns
`{toolpaths, checked, violations: 0}`. Any violation is returned as an **error** whose message names the
count and the first few points. A relief gouge entry carries `shortfall` (that point's) and `worst_shortfall` (the largest of the toolpath); the listed points are the first found, not the worst.

**`cam.simulate`** `{path?, cell?, toolpaths?: [index], setup?, sides?: "one"|"both"}` cuts the stock in software and returns
`{width, height, cell, min_z, bytes}` plus a PNG (`path` writes a file, otherwise `png_base64`). Look at
the picture. `min_z` is the lowest point of the simulated stock; it should match your intended cut depth and must not
go below the bottom of the material unless you meant it to. A job with thread toolpaths also returns `threads`:
for each thread wall (at most 16), `{toolpath, hole, center, diameter, max_depth, rows, dz}` and its unrolled
picture (angle across, height down, the groove dark) in `png_base64`, or in `<path stem>-thread<n>.png` next to
`path`. `max_depth` should be the thread's radial depth.

**Photo to Relief Preview is a turnable 3D picture.** `relief.heightmap` with `preview: true` still returns the flat
shaded `preview_base64`. Add `preview_view` (an object with the view settings of `cam.simulate.view`: `view?: "top"|"3d"`
(default 3d), `yaw?`, `pitch?`, `zoom?`, `pan?`, `z_scale?`, `width?`, `height?`) and the result also has
`preview3d_base64` (a PNG of the same relief as a block with side walls, drawn by the same renderer as `cam.simulate`) and
`preview_view` (`{view, width, height, cell, min_z}`); a `path` inside it is ignored, and a zoom or pan out of range is an
error. The preview's stock is kept in the session (not saved, not undone, replaced by the next `preview: true`), and
**`relief.preview.view`** `{view?, yaw?, pitch?, zoom?, pan?, z_scale?, width?, height?}` redraws it from another point of
view without making the height map again (`png_base64`; error `run Preview first` if there is none). The window uses this
for its Top and 3D buttons, drag, wheel and Reset view while the Preview is on screen; the live preview asks again with the
view the person has now. Drawing samples the stock to at most 400 x 400 vertices at 960 x 640 pixels by default, so it
takes a fraction of a second whatever the picture's size; the height map itself is made in the background as before.

**`cam.simulate.view`** `{path?, view?: "top"|"3d", yaw?, pitch?, z_scale?, width?, height?}` redraws the
last simulation from another angle without cutting it again (error `run Simulate first` if nothing was
simulated or the toolpaths changed since). `cam.simulate` takes the same `view`, `yaw`, `pitch`, `z_scale`,
`width`, `height` (3D default: yaw 30, pitch 45, 960x640); the result gains `view`, and `width`/`height` are the PNG's size.
Both also take `zoom` (0.25 to 50, default 1 = the whole block fits) and `pan` (`[x, y]`, fractions of the picture, x to
the right and y down, each within +-25, default `[0, 0]`), for the 3D view and the flat top view alike. They magnify the
picture about its center and then slide it, and the image size does not change; without them the picture is byte for byte
what it was before they existed. A zoom or pan that is not a number, or is outside the range, is an error, not a clamp. A point at fraction `c` from the center of the fitted picture lands at `c * zoom + pan`.

**`cam.check_gcode`** `{path | text (not both), clearance_z?, spindle?}` reads a G-code file or text (64 MiB at most) and returns
`{ok, moves, units, clearance_z, ends_with_spindle_off, violations: [{line, rule, message}], warnings}` — every safety problem in line order
(`first-move`, `xy-rapid-low`, `cut-without-feed`, `cut-before-spindle`, `end-not-retracted`, `spindle-left-on`, `unproven-code`).
The clearance defaults to the program's own first-move Z; a first Z at or below 0 is an error asking for `clearance_z`. A line it
cannot follow (homing, probing, `G53`, canned cycles, `G92`, work or length offsets after the first move, axis words beside an
unmodeled G code; scaling, rotation and polar codes `G15 G16 G50 G51 G68 G69`; any G, X, Y, Z or M word without a readable
number such as `Z -3`, `Z#1`, `Z[1+2]`, `Znan`, `Z1e1`; an X/Y/Z word twice on a line (`Z-3 Z5`); A/B/C/U/V/W words; `#`, `[`, `$` and `/` (block delete); `M97 M98 M99` and O-words;
units changed after the first move; code after `M2`/`M30`; a `%` with anything else on its line; any G or M code not known to be
harmless, such as `G29`, `G41`, `G52`, `M70`, `M600`, once the program has moved; `M6` after the first move, since a tool
change can move the machine; and `M0`/`M1`/`M117`/`M118` text that contains `#`, `M97`/`M98`/`M99`/`M47`, or a G/X/Y/Z/A/B/C/U/V/W/O
letter followed by a number) is an `unproven-code` violation and makes every position after it unknown. Only the known-harmless codes
(`G4 G40 G61 G61.1 G64 G80 G90.1 G91.1 G93 G94 G98 G99`, `M7 M8 M9`) are warnings, not failures, as is a `G0` that drops Z below zero (`rapid-plunge`). Use it to check a saved file, or a
G-code program from another program, before anything runs on the machine.

**`cam.delete`** `{index}` removes a toolpath; later ones renumber. Use it instead of leaving bad toolpaths
in the list.

### Output

**`cam.post`** `{post?: "grbl"|"onefinity", path?, toolpaths?: [index], setup?: index|name, arcs?: bool, decimals?: 0..9, files?: "one"|"per_toolpath"|"per_setup", group_tools?: bool, dry_run?: bool}`
Writes G-code. A job with several sheets (see "Several sheets in one job" below) is written as one program that
stops the spindle, lifts, parks and pauses (`M0`) between sheets, or as one file for each sheet with
`files: "per_setup"` (named `<name>-setup1`, `<name>-setup2`, ...; each is a complete program). `setup` posts one sheet only.
`files: "per_toolpath"` writes one complete file for each toolpath, in job order (`group_tools` is ignored, nothing is
regrouped), named `<Name>-Step<N>-<Kind>-<size>` (`Tellus-Step1-EndMill-0.25in`; Kind is EndMill, Ball, Vbit or
ThreadMill; size is the bit's diameter in the job's units, then `in` or `mm`; N is zero-padded from 10 toolpaths and
keeps counting across sheets). Each file lifts Z first, asks for its own bit and ends parked. The Onefinity post
cleans the names to letters and digits (`TellusStep1EndMill025in.ngc`); every post replaces characters a file name
cannot hold. The result has `files` (`path`, `file`, `step`, `name`, `tool`, `toolpaths`, `lines`, and `setup` for
several sheets) and `names`, the exact names; use those. Run the files in step order, fitting each bit and setting
Z zero again. `dry_run: true` plans the files and writes nothing (same `names` and `files`, no `text`). Even
with one toolpath the name is the Step name. Each file names its bit (the `T` prompt, or a lift-park-`M0` pause with "Change to the ... set Z zero again" on posts without one), and toolpath comments carry the job-wide numbers. The result has `machine`/`machine_warnings` when a machine is set, and `warnings` for leftover `<name>-Step*` files in the folder (never deleted). A `path` that is an existing folder is refused in every mode. Without `path` it returns the text in `text`; with `path` it writes the file and returns
`{path, names, post, units, lines, bytes}` (`names` is the one file name written, as in the split modes). The post renames the file for the controller (the Onefinity post
keeps letters and digits and uses `.ngc`), so **use the `path` it returns**, not the one you asked for.
Every post lifts Z before any XY move and pauses for the operator to start the router where the machine
needs it; do not edit the G-code to remove that.
The toolpaths are written in job order (`group_tools` defaults to false), so a through-cut you put last
stays last. `group_tools: true` runs toolpaths that use the same bit together within a sheet, bits in
order of first use (used by `files: "one"` and `"per_setup"`, not by `"per_toolpath"`); use it only when no later toolpath depends on an earlier one (a through-cut must not
move ahead of face work). When the same bit comes back after other bits, the result's `warnings` says which
toolpaths would share a bit change, so you can choose. When grouping does move a through-cut ahead of other work,
`warnings` says so ("group_tools moved toolpath N (a through-cut) ahead of toolpath M; the board is cut free before that work").

Which post: the target machine is an Onefinity Woodworker, so use `onefinity` unless the person names
another. See [`onefinity.md`](onefinity.md).

### Several sheets in one job

A job can hold more than one **setup**: a sheet of stock with its own size, origin, thickness and toolpaths.
Setup 0 is the one `cam.material` made. Every setup shares the first one's units (mm or inch).

- **`cam.setup.add`** `{name?, activate?: true, ...cam.material parameters}` adds a sheet and makes it the active one.
  What you leave out is copied from the first sheet, except `size`, `origin` and `xy_datum`. Put each sheet's artwork
  on its own part of the drawing and give the sheet an `origin` (its lower-left corner in the drawing) there: X and Y in
  that sheet's program are measured from its own datum.
- **`cam.setup.use`** `{setup: index|name}` makes a setup the active one. `cam.material` and every toolpath command
  (`cam.profile`, `cam.pocket`, ...) work on the active setup.
- **`cam.setup.rename`** `{name, setup?}`, **`cam.setup.delete`** `{setup, with_toolpaths?: false}` (a job keeps at least
  one setup; a setup that still has toolpaths needs `with_toolpaths: true`), **`cam.setup.list`** `{}` (each setup with its
  sheet, toolpath indices and time).
- `cam.list` adds `setup` to every toolpath row, and `setups` and `active_setup`, but only when the job has more than one
  setup. A single-setup job returns exactly what it always did.
- `cam.simulate` shows the active setup (or `setup`); its toolpaths must all be in one sheet. `cam.verify` checks every
  setup, or just `setup`. `cam.post` writes every setup in order.
- `job.run` takes `setups: [{name?, ...cam.material params}]` and an optional `setup` on each operation (default: the first).
- At each sheet change the program ends the old sheet (spindle off, lift to the bit-change height, park), waits at `M0`
  while the person moves to the next sheet and sets X, Y and Z zero again, lifts Z before any XY move, and restarts the
  spindle (the Onefinity post asks for it with a pause). Tell the person about every such pause.

### Two-sided jobs (flip the stock)

A two-sided job cuts both faces of one stock: the first side, a flip, then the second side. The second side is a setup
marked as the flipped side of the first. Help page: `help.page {"id": "two-sided"}`. Design: [`two-sided.md`](two-sided.md).

- **`cam.flip`** `{axis?: "y"|"x", from?: setup index|name, name?: "Bottom", activate?: true, registration?: {diameter, depth, tool, feeds?, into?: "spoilboard"|"waste", holes?: [[x,y]...], margin?, peck?} | false}`
  adds the flipped side, a copy of the first side's material, and makes it active. `axis: "y"` (default) turns the stock
  over like a page (X mirrored about the stock's center line); `"x"` mirrors Y. The first side's material needs `size`.
  An unnamed first sheet is renamed `Top`. `registration` drills pin holes on the first side (a pocket when the bit is
  smaller than `diameter`): by default two holes on the flip line, `margin` (two diameters) in from the stock's ends;
  given `holes` get their mirror images added. `into: "spoilboard"` (default) needs `depth` more than the thickness;
  `"waste"` keeps the holes in the stock (then say to the person that the spoilboard needs matching pin holes). A hole
  that is not wholly inside the stock, or inside or too near a part, is an error. Returns `index`, `name`, `from`,
  `axis`, `flip_line`, `registration` (`holes`, `toolpath`, `kind`) and `warnings`: pass them on.
- **Drawing the second side:** draw its artwork in place on the first side's sheet, as seen through the stock from
  above. Every toolpath command run while the flipped side is active mirrors the shapes onto the flipped stock (cut
  direction kept); depths are from the top of the flipped stock. `cam.relief` is refused on a flipped side.
- A toolpath calculated on either face carries a warning when the two faces' cuts now meet. Pass it on.
- **`cam.verify`** checks each pair: the same stock on both faces, holes inside the stock and outside every part, no
  second-side cut over a pin, and that the cuts of the two faces do not meet (together they must stay under the
  thickness). Meeting cuts are an error unless `allow_meet: true`; use that only when the person says the part is meant
  to come free from both sides. Success adds `two_sided: [{first, flipped, axis, pins, meet: {cells, area, at, thinnest}}]`.
- **`cam.simulate {sides: "both"}`** draws both faces from above, side by side (the flipped side as the machine sees it),
  red where the cuts meet, and returns `meet` and a `note`. Read the note.
- **G-code:** at the flip the program stops the spindle, lifts to the bit change height, parks and pauses (`M0`) with
  "Flip the stock about the Y axis, seat it on the pins, clamp it, keep X and Y zero, set Z zero again, then press
  Continue" (Onefinity: "Switch the router off, flip the stock ..."), then lifts Z before any XY move. With
  `files: "per_setup"` the flipped side's file starts with that pause. Tell the person about it.
- `cam.setup.list` and `cam.list` show `flip: {axis, of, registration}` on the flipped setup only. `cam.setup.delete`
  refuses the first side while its flipped side exists.
- `job.run` takes a setup entry `{"name": "Back", "flip": {cam.flip params}}` (`from` defaults to 0). The pin hole
  circles are drawn on the `REGISTRATION` layer, which `select: "all"` leaves out; when you select by hand, leave them
  out too, or the second side will cut where the pins are (verify stops on that).

### Sending (dry run only)

**Agents cannot send a program to a machine.** These commands let you check a send and report it; the
person sends from **Send to Machine...** in the Toolpaths palette, after confirming the job there. Help
page: `help.page {"id": "send-to-machine"}`. Design and research: [`send-to-machine.md`](send-to-machine.md).

**`send.targets`** `{save?: {name, kind: "onefinity", host, api?: "file"|"fs"} | {name, kind: "grbl", port, baud?}, delete?: name}`
Lists the person's saved targets (`targets`, each with a `description`) and the serial ports found
(`serial_ports`). Save or delete only when the person asks.

**`send.upload`** `{host | target, api?, name?, file?, group_tools?: bool}`
Checks an upload to an Onefinity controller. It fails unless `cam.verify` passed on the job exactly as
it is now (any toolpath, material, machine or drawing change needs Verify again; a looser `tol` than the
default does not count). `host` must be on the person's own network. With `file`, the file must be
byte-for-byte the G-code `cam.post` writes now for the verified job with the `onefinity` post. Success
returns `{dry_run: true, sent: false, ready: true, plan, reminders, warnings, next}`: `plan` has the job
name, machine, target, `controller_file`, `bits`, `bit_changes`, `router_starts`, `pauses` (line and
message) and `bit_change_height`. Pass `reminders` and `warnings` on to the person. `dry_run: false`,
`confirm` or `send` are refused.

**`send.stream`** `{action?: "prepare"|"status"|"pause"|"stop", port | target, baud?: 115200, post?: "grbl"|"grbl-manual", name?, file?, group_tools?: bool}`
`prepare` (the default) checks a GRBL stream the same way and also reports `lines_to_send`. `status`,
`pause` (feed hold) and `stop` (feed hold, then reset) act on a stream the person started in the
window; use them when the person asks, or to stop the machine. `start`, `resume` and `continue` are
refused: they are buttons for the person at the machine.

`job.run` takes `"send": {"upload": {...}}` or `"send": {"stream": {...}}` and returns the dry-run report
in `send`; a refused check is reported (`ready: false`, a `send check:` warning) without undoing the job.

### Checking for a newer release

**`app.check_updates`** `{}` reads the public list of the ten most recent releases once (one request; pre-releases count, drafts
are skipped) and compares the highest version in it with the running version. Returns `{status: "update_available" | "up_to_date" | "ahead_of_release", current, latest,
prerelease (the latest is marked pre-release; the message then says "(pre-release)"), notes, url, message}`. It is notify only: it downloads and installs nothing, sends no identifying
data, and runs only when you call it (call it only when the person asks; never poll). It waits up to
10 seconds. If it cannot check (offline, rate limited, or a host without the desktop app such as the
command line tool) it returns an error message that includes the releases link. Pass the result on to the
person; do not install anything. Help page: `help.page {"id": "updates"}`.

### Saving the job

`project.save {path}` writes a `.lcarve` project (drawing, material, toolpaths). `project.open {path}`
restores it. Save before any long or risky step and after a job the person wants to keep.

## 5. The whole job in one command: `job.run`

`job.run` takes a recipe (material, drawing, tools, operations, check, G-code) and runs it as one step. If any
step fails, the drawing and job are put back as they were and the error names the step, for example
`operation 1 (pocket): ...`. Help page: `help.page {"id": "job-recipe"}`.

```json
{"command": "job.run", "params": {
  "name": "Tray",
  "material": {"thickness": 19, "z_zero": "top", "safe_z": 5},
  "import": {"path": "/work/tray.dxf"},
  "tools": {"mill": {"kind": "end_mill", "diameter": 6, "pass_depth": 3, "feed": 1500, "plunge": 500, "rpm": 18000}},
  "feeds": "hardwood",
  "operations": [
    {"op": "pocket",  "tool": "mill", "cut_depth": 6},
    {"op": "profile", "tool": "mill", "side": "outside", "cut_depth": 19}
  ],
  "simulate": {"path": "/work/out/tray.png"},
  "post": {"post": "onefinity", "path": "/work/out/tray.ngc"}
}}
```

Keys: `material` and `operations` are required. `operations[].op` is `profile`, `pocket`, `drill`, `hatch` or
`engrave`; the other fields are those of the matching command. `select` says what an operation cuts: `"all"`
(default: everything visible and unlocked), `"selection"`, `{"layer": "CUT"}`, `{"layers": [...]}` or
`{"handles": [...]}`. `verify` is on by default; a recipe that fails the check returns an error and **no
G-code**. A recipe replaces the job's toolpaths unless `keep_toolpaths` is true.

The result has `ok`, the toolpaths (name, kind, tool, moves, time), `total_time_s`, `tool_changes`, the verify
counts, the post summary, `gcode` (when no `path` was given) and `warnings`. Everything in section 9 still
applies: read the result, pass on the warnings, and get approval. A recipe result is not approval.

The recipe's `import` step uses `cam.import`: the material is set first, the file's units are converted to the
job's units, block references are flattened, and the import's warnings come back in the job's `warnings` with an
`import: ` prefix. The `import` step in `steps` reports `imported`, `units` and `bounds`; check that the size
is what you expect. The `import` params are those of `cam.import`.

### Saved recipes

A recipe can be a JSON file, so a proven recipe runs on a new drawing. `recipe_path` loads it; every other key
in the call replaces the same key from the file (whole value, no deep merge), and `null` removes a key.
`save_recipe` writes the effective recipe (file plus overrides, without `recipe_path`, `save_recipe`,
`overwrite_recipe`) after a successful run; `.json` is appended unless present. An existing file is refused
before anything runs unless it is the file loaded by `recipe_path` or `"overwrite_recipe": true` is passed. A
recipe file cannot contain `recipe_path`, `save_recipe` or `overwrite_recipe`, and files over 8 MiB are refused.
The report has `recipe: {from?, saved?}`. Relative paths inside a recipe are used as they are.

```json
{"command": "job.run", "params": {"recipe_path": "/work/recipes/tray.json",
  "import": {"path": "/work/new-tray.dxf"},
  "post": {"post": "onefinity", "path": "/work/out/new-tray.ngc"}}}
```

To make the file in the first place, add `"save_recipe": "/work/recipes/tray"` to a job that worked.

## 6. Worked example: SVG to Onefinity G-code

A 100 x 50 mm part, 18 mm MDF, cut out with a 6 mm end mill. Every call is `execute`; read each result.

```json
{"command": "cam.material", "params": {"thickness": 18, "units": "mm", "z_zero": "top", "xy_datum": "lower_left"}}
{"command": "cam.import",   "params": {"path": "/work/part.svg"}}
{"command": "cam.tools",    "params": {"kind": "end_mill", "feeds": "mdf"}}
{"command": "cam.profile",  "params": {"tool": "<id from cam.tools>", "feeds": "mdf", "side": "outside", "cut_depth": 18,
                                         "tabs": {"count": 4, "width": 8, "thickness": 3}, "ramp": {"length": 20}}}
{"command": "cam.list",     "params": {}}
{"command": "cam.verify",   "params": {}}
{"command": "cam.simulate", "params": {"path": "/work/out/part.png"}}
{"command": "cam.post",     "params": {"post": "onefinity", "path": "/work/out/part.ngc"}}
{"command": "project.save", "params": {"path": "/work/out/part"}}
```

What to check between steps:

- After `cam.import`: `bounds.width` and `bounds.height` match the part (100 x 50). `warnings` empty or
  understood. `shapes.open` is 0 for a cut-out.
- After `cam.profile`: `warnings` empty, `skipped` 0, `passes` is `cut_depth / pass_depth` rounded up.
- After `cam.verify`: no error.
- After `cam.simulate`: open the PNG. `min_z` is the lowest point of the simulated stock; compare it with the
  intended cut depth (below the top surface when Z zero is the top) and with the material thickness.
- After `cam.post`: `lines` is plausible, and the returned `path` ends in `.ngc`.

## 7. Recovering from errors

| You see | Do |
| --- | --- |
| `no material set: run cam.material first` | Call `cam.material`, then repeat. |
| `nothing to cut: select lines, arcs, circles...` | Nothing is selected, or the selection is dimensions, hatches or other non-geometry. Re-import, or select with `selectall` / pass `handles`. Text objects count: they are cut as their letter outlines. |
| `shape N is open: only side on can follow an open shape` | An outline has a gap. Use `side: "on"`, close the shape in the artwork, or check `cam.import` `shapes.open`. |
| `cam.verify` error with violations | Do not post. Read the first points; common causes are a tool too large for an inside corner or the wrong `side`. Delete the toolpath (`cam.delete`), change one thing, recalculate, verify. |
| `BadParams ... must be ...` | A number or name was wrong or out of range. The message says which. Fix that parameter only. |
| `job.run` error naming a step | Nothing was changed; the job is as it was. Fix that step's parameter and run the recipe again. |
| A path or file error | Check the path exists and is readable. Do not search the disk; ask the person for the file. |

Commands in the `cam.*` family are not undoable in the session. To discard a toolpath use `cam.delete`.
Drawing changes (including `cam.import`) can be undone with the `undo` command.

## 8. Managing several jobs

- Use **one session per job**. Headless mode is one session per `librecarve-cli mcp` process, so start a
  fresh process for a new job.
- Name outputs after the job, in a folder you create for it: `part.ngc`, `part.png`, `part.lcarve`.
  Onefinity file names are letters and digits only.
- Keep a short run log in the job folder (`part-run.md`): the commands you ran, the key results
  (`bounds`, `time_s`, `min_z`, `violations`), every warning, and the final file names.
- For a batch of parts, finish and report one job completely before starting the next.

## 9. What to report to the person

Before saying a job is ready, give them:

1. **Part and material:** size from `bounds`, thickness, units.
2. **Toolpaths:** one line each: name, tool and diameter, depth, passes, estimated time.
3. **Checks:** `cam.verify` result (checked count, 0 violations) and the simulation picture.
4. **Warnings:** every one, in the person's words.
5. **Output:** the exact file path returned by `cam.post`, the post used, and that it pauses for the router.
   If the person wants to send it, the result of a `send.upload` or `send.stream` check, and that they
   send it themselves with **Send to Machine...**.
6. **Assumptions:** any guessed number, any `assumed` units, the 300 mm/s² time assumption.
7. **The question:** "Approve for cutting?" Nothing proceeds without a yes.

## 10. Not available yet

Do not call these; they do not exist.

- Text from an SVG file (`cam.import` does not read it). Text in the drawing is cut with `cam.text`.
- Merging overlapping letters of a script font into one outline (`cam.text` warns about them).
- Inlay toolpaths. (V-carve exists as `cam.vcarve`, 3D relief toolpaths as `cam.relief`, drilling with pecking as `cam.drill`, thread milling as `cam.thread`.)
- Tapered (pipe) threads, thread forms other than 60 degrees, and `thread` operations in `job.run` recipes.
- Sending, for agents. Only the person sends, from the Send to Machine window; you can check a send
  with `send.upload` or `send.stream` (dry runs) and pause or stop a stream the person started.
- Jogging, homing, probing or unlocking a machine. LibreCarve never does these.
- Posts other than `grbl` and `onefinity`.
- Two-sided nesting, filler parts by priority, and nesting around parts already on a sheet (`cam.nest` always starts from empty sheets).

## 11. Open questions this guide cannot answer yet

- Starting a new drawing (`new`, or the `new_drawing` tool) does **not** clear the toolpath list: seen
  in a live app on 2026-10-14, where the earlier job's toolpath was still listed and would have gone
  into the next job's G-code. After a new drawing, run `cam.list` and `cam.delete` any toolpath that
  is not part of the new job, or use a fresh session per job.
- Feeds in the tool database are starting points, not tested on the target machine. The person decides.
